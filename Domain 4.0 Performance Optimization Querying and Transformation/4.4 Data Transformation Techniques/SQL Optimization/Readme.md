# **Snowflake: SQL Optimization Deep Dive**

*Production-Grade Technical Guide for Platform Engineers, SREs, and Architects*

---

---

## **1. Mermaid Execution Flow & Architecture Diagram**

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#ffd700', 'edgeLabelBackground':'#fff'}}}%%
flowchart TD
    subgraph "Control Plane"
        A[Query Submission] -->|Parse| B[SQL Parser\n(Calcite-based)]
        B -->|AST| C[Logical Plan]
        C -->|Optimize| D[Optimizer\n(CBO + RBO)]
        D -->|Physical Plan| E[Query Plan\n(DAG of Operators)]
        E -->|Resource Allocation| F[Warehouse Provisioning]
    end

    subgraph "Data Plane"
        F -->|Parallel Execution| G[Execution Engine\n(Vectorized)]
        G -->|Stage I/O| H[Cloud Storage\n(S3/Azure Blob/GCS)]
        G -->|Metadata| I[Metadata Store\n(Snowflake Internal)]
        G -->|Cache| J[Result Cache\n(24h TTL)]
        G -->|Spill| K[Local SSD\n(Temp Storage)]
    end

    subgraph "Optimizer Internals"
        D -->|Statistics| L[Metadata Cache\n(Table/Column Stats)]
        D -->|Rules| M[Rule-Based Optimizations\n(Predicate Pushdown, etc.)]
        D -->|Cost Model| N[Cost-Based Optimizations\n(Join Order, etc.)]
        L -->|Missing Stats| O[Dynamic Sampling\n(+5-15% Credits)]
    end

    subgraph "Execution Operators"
        G -->|Scan| P[TableScan\n(Columnar Pruning)]
        G -->|Join| Q[Join\n(Broadcast/Shuffle/Sort-Merge)]
        G -->|Aggregate| R[Aggregate\n(Hash/Streaming)]
        G -->|Sort| S[Sort\n(External Merge Sort)]
        G -->|Window| T[Window Functions\n(Partitioned)]
    end

    subgraph "Failure Paths"
        G -->|OOM| U[Spill-to-Disk\n(80% Heap Threshold)]
        U -->|Spill Overflow| V[Query Abort\n(Error: 2003)]
        Q -->|Skew| W[Data Skew\n(Uneven Partitioning)]
        W -->|Retry| X[Automatic Retry\n(3x Default)]
        X -->|Persistent Skew| Y[Manual Repartitioning]
        P -->|Missing Stats| Z[Dynamic Sampling\n(+10-20% Credits)]
    end

    style A fill:#f9f,stroke:#333
    style D fill:#bbf,stroke:#333
    style G fill:#f96,stroke:#333
    style V fill:#f99,stroke:#333
    style W fill:#ff9,stroke:#333
    style Z fill:#9f9,stroke:#333
```

---

---

## **2. Execution Internals & Transactional Boundaries**

---

### **2.1 Query Execution Lifecycle**


| **Phase**                  | **Internals**                                                                                                         | **Transactional Semantics**                                                                                           | **Failure Recovery**                                                                                            | **Credit Impact** |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- | ----------------- |
| **Parsing**                | SQL → **AST (Abstract Syntax Tree)** → **Logical Plan** (Calcite-based). **Thread:** Single-threaded (Control Plane). | No transaction.                                                                                                       | Syntax errors → Immediate abort (`1003: SQL compilation error`).                                                | 0 (Control Plane) |
| **Optimization**           | **Cost-Based Optimizer (CBO)** + **Rule-Based Optimizer (RBO)**. Uses **statistics** (metadata cache).                | No transaction.                                                                                                       | Missing stats → Fallback to **dynamic sampling** (+5-15% credits).                                              | +5-15% (sampling) |
| **Plan Generation**        | **DAG of Operators** (e.g., `TableScan`, `Join`, `Aggregate`). **Parallelism:** Per-cluster.                          | No transaction.                                                                                                       | Invalid plan → `2001: Planning error`.                                                                          | 0                 |
| **Warehouse Allocation**   | **Multi-cluster** (if `MULTI_CLUSTER_WAREHOUSE=TRUE`). **Thread Model:** 1 thread = 1 CPU core.                       | No transaction.                                                                                                       | Warehouse queueing → `002008: Warehouse is busy`.                                                               | 0 (idle)          |
| **Execution**              | **Vectorized Engine**: Batch processing (1024 rows/batch). **Memory Model:** Off-heap (C++).                          | **ACID**: MVCC (Multi-Version Concurrency Control). **Isolation Levels:** `READ_COMMITTED` (default), `SERIALIZABLE`. | OOM → Spill to local SSD (max **2x warehouse memory**). Spill overflow → Abort (`2003: Memory limit exceeded`). | +2x (spill)       |
| **Result Materialization** | **Result Set**: Stored in **Temp Tables** (session-scoped). **Format:** Parquet (columnar).                           | **Visibility**: Committed results visible to all sessions post-transaction.                                           | Temp table eviction → `2005: Temporary table does not exist`.                                                   | 0                 |
| **Commit/Rollback**        | **2-Phase Commit**: Prepare (validate) → Commit (persist). **WAL:** Write-Ahead Log (metadata).                       | **Durability**: Metadata persisted to **Snowflake’s metadata store** (S3-backed).                                     | Conflict → `1020: Transaction conflict` (retry or `ABORT`).                                                     | 0                 |


---

### **2.2 Snowflake-Specific Query Execution Deviations**


| **Concept**               | **Traditional RDBMS**                 | **Snowflake**                                                                                            | **Impact**                                                           |
| ------------------------- | ------------------------------------- | -------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| **Storage Format**        | Row-based (e.g., B-tree)              | **Columnar (Parquet)** + **Micro-Partitioning** (16-128MB chunks).                                       | **10-100x faster** for analytical queries (columnar scans).          |
| **Join Algorithms**       | Hash Join, Nested Loop, Merge Join    | **Broadcast Join** (small tables), **Shuffle Join** (large tables), **Sort-Merge Join** (sorted inputs). | **Broadcast Join** is **10x faster** for small tables (<10MB).       |
| **Sorting**               | In-memory or disk-based               | **External Merge Sort** (spill-to-disk). **Memory Threshold:** 80% heap.                                 | **Spill overhead**: +2x credits.                                     |
| **Aggregation**           | Hash Aggregation                      | **Hash Aggregation** (in-memory) + **Streaming Aggregation** (for window functions).                     | **Streaming Aggregation** avoids full sort (+20% faster).            |
| **Parallelism**           | Limited by cores                      | **MPP (Massively Parallel Processing)** with **no shared state** between nodes.                          | **Linear scalability** (add clusters to increase throughput).        |
| **Caching**               | Query Cache, Buffer Pool              | **Result Cache** (24h TTL), **Metadata Cache**, **Local Disk Cache** (for repeated scans).               | **Result Cache** reduces latency by **90-95%** for repeated queries. |
| **Transaction Isolation** | MVCC, Read Committed, Repeatable Read | **MVCC** (default) + **Snapshot Isolation**. **No read locks**; writers block writers.                   | **Higher concurrency** (no blocking for reads).                      |
| **Error Handling**        | Rollback on error                     | **Partial Results** (e.g., `ON_ERROR=CONTINUE` for COPY). **Retry Logic:** 3x default.                   | **More resilient** to failures (e.g., malformed rows in COPY).       |


---

### **2.3 Operator-Level Internals**

#### **2.3.1 TableScan**

- **Internals**:
  - **Columnar Pruning**: Only reads **columns referenced in query**.
  - **Partition Pruning**: Skips **micro-partitions** not matching `WHERE` clauses.
  - **Vectorized**: Processes **1024 rows/batch** (C++).
- **Optimizations**:
  - **Clustering Keys**: **90% pruning** for filtered queries (e.g., `WHERE date = '2026-01-01'`).
  - **Zone Maps**: Min/max stats per micro-partition (avoids full scans).
- **Credit Impact**:
  - **Scan Cost**: `1 credit = 1 core-second` (proportional to data scanned).
  - **Pruning Savings**: **10-90x fewer credits** for pruned queries.

#### **2.3.2 Join Operators**


| **Join Type**        | **Algorithm**                       | **When Used**                           | **Memory Usage**              | **Credit Impact**                | **Skew Handling**                                |
| -------------------- | ----------------------------------- | --------------------------------------- | ----------------------------- | -------------------------------- | ------------------------------------------------ |
| **Broadcast Join**   | Replicate small table to all nodes. | Small table (<10MB) + no skew.          | Low (replicated in memory).   | **+10% credits** (replication).  | None (small tables).                             |
| **Shuffle Join**     | Hash-partition both tables.         | Large tables or skewed data.            | High (hash tables in memory). | **+20-30% credits** (shuffling). | **Automatic repartitioning** (if skew detected). |
| **Sort-Merge Join**  | Merge sorted inputs.                | Both inputs **pre-sorted** on join key. | Medium (streaming).           | **+15% credits** (sorting).      | None (sorted inputs).                            |
| **Nested Loop Join** | Row-by-row comparison.              | **Small tables** (rare in Snowflake).   | Low.                          | **+50% credits** (inefficient).  | None.                                            |


- **Skew Detection**:
  - **Threshold**: If a partition > **2x average size**, Snowflake **automatically repartitions**.
  - **Manual Override**: Use `/*+ SKEW_JOIN(table, column) */` hint.

#### **2.3.3 Aggregate**

- **Hash Aggregation**:
  - **Memory Threshold**: 80% heap → **spill to disk** (+2x credits).
  - **Optimization**: **Pre-aggregate** in subqueries to reduce data volume.
- **Streaming Aggregation**:
  - **Window Functions**: Uses **partitioned streaming** (no full sort).
  - **Credit Impact**: **+20% faster** than hash aggregation for window functions.

#### **2.3.4 Sort**

- **External Merge Sort**:
  - **Memory Threshold**: 80% heap → **spill to local SSD**.
  - **Credit Impact**: **+2x credits** (I/O overhead).
  - **Optimization**: **Avoid `ORDER BY**` if not needed (use `LIMIT` instead).

#### **2.3.5 Window Functions**

- **Internals**:
  - **Partitioned Processing**: Each partition processed independently.
  - **Spill Behavior**: **No spill** (streaming).
- **Credit Impact**: **+10-15% credits** vs. equivalent `GROUP BY`.

---

### **2.4 Transactional Boundaries & Guarantees**


| **Operation**              | **Atomicity**                        | **Consistency**    | **Isolation**   | **Durability**              | **Error Handling**            |
| -------------------------- | ------------------------------------ | ------------------ | --------------- | --------------------------- | ----------------------------- |
| **Single Statement**       | Full                                 | Snapshot Isolation | MVCC            | Metadata + Data (S3-backed) | Rollback on error.            |
| **Multi-Statement**        | Full (if `BEGIN...COMMIT`)           | Snapshot Isolation | MVCC            | Metadata + Data (S3-backed) | `ABORT` on conflict (`1020`). |
| **COPY INTO**              | File-level (if `ON_ERROR=SKIP_FILE`) | Strong (S3)        | None (no locks) | Metadata + Data (S3-backed) | Partial load (if `CONTINUE`). |
| **CREATE TABLE AS SELECT** | Full                                 | Snapshot Isolation | MVCC            | Metadata + Data (S3-backed) | Rollback on error.            |
| **UPDATE/DELETE**          | Full                                 | Snapshot Isolation | MVCC            | Metadata + Data (S3-backed) | Rollback on error.            |


---

---

## **3. Parameter/Configuration Deep Dive**

---

### **3.1 Warehouse-Level Parameters**


| **Parameter**                  | **Internal Behavior**                                                                                     | **Performance Impact**                                                                                   | **Compliance/Edge Cases**                                                         | **Production Default**          |
| ------------------------------ | --------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- | ------------------------------- |
| `WAREHOUSE_SIZE`               | **X-Small (1x) → 4X-Large (128x)**. Each "x" = **16 vCPUs + 128GB RAM**. **Thread Pool:** 8 threads/core. | Larger warehouses reduce **spill-to-disk** but increase **credit burn rate** (1 credit = 1 core-second). | `X-SMALL` fails on **>1TB scans** (OOM). `4X-LARGE` required for **10TB+ joins**. | `X-SMALL` (Dev), `LARGE` (Prod) |
| `AUTO_SUSPEND`                 | **Idle timeout** (1-86400 sec). **Grace Period:** 5 sec (query completion check).                         | Reduces **idle credit burn** (saves **~30-50%** costs in bursty workloads).                              | Set to `60` for interactive queries, `300` for batch.                             | `600` (10 min)                  |
| `AUTO_RESUME`                  | **Auto-resume** warehouse when query submitted.                                                           | Reduces **query queueing latency** (no cold start).                                                      | **Credit Impact**: +5% (idle overhead).                                           | `TRUE`                          |
| `MULTI_CLUSTER_WAREHOUSE`      | **Max Clusters:** 1-10. **Scaling Policy:** `STANDARD` (1-10), `ECONOMY` (1-2). **Queue:** FIFO.          | Reduces **queueing latency** (scales to **10x concurrency**). Credit overhead: **+10-15%** per cluster.  | `ECONOMY` mode may **starve** low-priority queries.                               | `FALSE`                         |
| `QUERY_TAG`                    | **Metadata:** Attached to `QUERY_HISTORY`. **Format:** `key=value`.                                       | Enables **cost attribution** (e.g., `team=analytics`). No performance impact.                            | Max **256 chars**. Special chars (`=`, `,`) escaped.                              | `NULL`                          |
| `STATEMENT_TIMEOUT_IN_SECONDS` | **Hard limit** (0-86400). **Thread:** Monitored by watchdog.                                              | Prevents **runaway queries**. Default `0` (no timeout) risks **OOM**.                                    | Set to **3600** (1h) for ETL, **600** (10m) for ad-hoc.                           | `0` (Disabled)                  |
| `MAX_CONCURRENCY_LEVEL`        | **Max concurrent queries** per warehouse.                                                                 | Controls **resource contention**.                                                                        | Default: `8 * warehouse_size`.                                                    | `8` (for `X-SMALL`)             |


---

### **3.2 Session-Level Parameters**


| **Parameter**              | **Internal Behavior**                                                                              | **Performance Impact**                                                          | **Compliance/Edge Cases**                                                 | **Production Default**    |
| -------------------------- | -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------- | ------------------------- |
| `USE_CACHED_RESULT`        | **Result Reuse:** 24h TTL. **Scope:** Session + warehouse. **Invalidation:** DDL on source tables. | **90-95% latency reduction** for repeated queries. Credit cost: **0** (cached). | Disabled for **non-deterministic** functions (e.g., `CURRENT_TIMESTAMP`). | `TRUE`                    |
| `CLIENT_RESULT_CHUNK_SIZE` | **Chunk size** for result sets (1-16MB).                                                           | Larger chunks → **fewer network round trips** (+5% faster for large results).   | Max **16MB** (Snowflake limit).                                           | `16000000` (16MB)         |
| `CLIENT_PREFETCH_THREADS`  | **Prefetch threads** for result sets (1-10).                                                       | More threads → **faster result fetching** (+10% for large results).             | Max **10 threads**.                                                       | `4`                       |
| `ABORT_DETACHED_QUERY`     | **Kill queries** when session disconnects.                                                         | Prevents **runaway queries** from disconnected clients.                         | **Default:** `TRUE` (recommended).                                        | `TRUE`                    |
| `TIMEZONE`                 | **Session timezone** (e.g., `Asia/Calcutta`).                                                      | Affects **timestamp functions** (e.g., `CURRENT_TIMESTAMP`).                    | **Default:** `UTC`.                                                       | `UTC`                     |
| `DATE_FORMAT`              | **Default date format** (e.g., `YYYY-MM-DD`).                                                      | Affects **date parsing** in `TO_DATE`.                                          | **Default:** `YYYY-MM-DD`.                                                | `YYYY-MM-DD`              |
| `TIME_FORMAT`              | **Default time format** (e.g., `HH:MI:SS`).                                                        | Affects **time parsing** in `TO_TIME`.                                          | **Default:** `HH:MI:SS`.                                                  | `HH:MI:SS`                |
| `TIMESTAMP_FORMAT`         | **Default timestamp format**.                                                                      | Affects **timestamp parsing** in `TO_TIMESTAMP`.                                | **Default:** `YYYY-MM-DD HH:MI:SS.FF3`.                                   | `YYYY-MM-DD HH:MI:SS.FF3` |


---

### **3.3 Query-Level Parameters (Hints)**


| **Hint**       | **Syntax**                        | **Internal Behavior**                            | **Performance Impact**                                              | **Use Case**                          |
| -------------- | --------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------- | ------------------------------------- |
| `LEADING`      | `/*+ LEADING(table1, table2) */`  | **Force join order** (left-to-right).            | **Avoids bad join orders** from CBO.                                | **Skewed joins** (small table first). |
| `SKEW_JOIN`    | `/*+ SKEW_JOIN(table, column) */` | **Force repartitioning** for skewed data.        | **+10% credits** (repartitioning overhead).                         | **Data skew** in joins.               |
| `BROADCAST`    | `/*+ BROADCAST(table) */`         | **Force broadcast join** for small tables.       | **+5% credits** (replication overhead).                             | **Small tables (<10MB)**.             |
| `NO_BROADCAST` | `/*+ NO_BROADCAST(table) */`      | **Prevent broadcast join**.                      | **Forces shuffle join** (slower for small tables).                  | **Large tables** (avoid OOM).         |
| `MATERIALIZE`  | `/*+ MATERIALIZE(subquery) */`    | **Materialize subquery** in temp table.          | **+10% credits** (storage overhead) but **faster repeated access**. | **Repeated subqueries**.              |
| `INLINE`       | `/*+ INLINE(view) */`             | **Inline view definition** (no materialization). | **Saves credits** but **slower for repeated access**.               | **One-time views**.                   |
| `USE_HASH`     | `/*+ USE_HASH(table) */`          | **Force hash join**.                             | **Avoids sort-merge join** (faster for unsorted data).              | **Unsorted large tables**.            |
| `USE_MERGE`    | `/*+ USE_MERGE(table) */`         | **Force sort-merge join**.                       | **Faster for sorted inputs** (+15% faster).                         | **Pre-sorted tables**.                |
| `PARALLEL`     | `/*+ PARALLEL(4) */`              | **Force parallelism** (number of threads).       | **Overrides warehouse defaults**.                                   | **Custom parallelism tuning**.        |
| `NO_PARALLEL`  | `/*+ NO_PARALLEL */`              | **Disable parallelism**.                         | **Forces single-threaded** (debugging only).                        | **Debugging**.                        |


---

---

## **4. Performance & Resource Implications**

---

### **4.1 Memory Model & Spill Behavior**


| **Component**         | **Memory Allocation**            | **Spill Trigger**      | **Spill Destination** | **Credit Overhead**       | **Recovery**                    |
| --------------------- | -------------------------------- | ---------------------- | --------------------- | ------------------------- | ------------------------------- |
| **Heap (JVM)**        | 50% of warehouse memory          | 80% utilization        | Local SSD (NVMe)      | +2x credits               | Automatic (transparent to user) |
| **Off-Heap (C++)**    | 50% of warehouse memory          | 90% utilization        | Local SSD (NVMe)      | +1.5x credits             | Automatic                       |
| **Result Set**        | Dynamic (up to warehouse memory) | 100% utilization       | Remote Stage (S3)     | +3x credits (network I/O) | Manual retry required           |
| **Join Buffers**      | 20% of warehouse memory          | Hash join > 2GB        | Local SSD             | +1.8x credits             | Automatic                       |
| **Sort Buffers**      | 30% of warehouse memory          | Sort > 1GB             | Local SSD             | +2x credits               | Automatic                       |
| **Aggregate Buffers** | 25% of warehouse memory          | Hash aggregation > 1GB | Local SSD             | +2x credits               | Automatic                       |


**Spill-to-Disk Credit Formula**:

```
Spill Overhead (credits) =
  (Spilled Data Size in GB / Warehouse Memory in GB) *
  2 *
  (Query Duration in sec / 3600)
```

**Example**:

- Warehouse: `LARGE` (128GB RAM).
- Spilled Data: 200GB.
- Query Duration: 600 sec.
- **Overhead**: `(200 / 128) * 2 * (600 / 3600) = 0.52 credits`.

---

### **4.2 I/O Patterns & Stage Interactions**


| **Operation**           | **I/O Behavior**                | **Network Overhead**         | **Credit Impact**              | **Optimization**                           |
| ----------------------- | ------------------------------- | ---------------------------- | ------------------------------ | ------------------------------------------ |
| **Table Scan**          | Columnar reads (Parquet)        | None (internal)              | `1 credit/core-second`         | **Clustering keys** (pruning).             |
| **External Stage Scan** | HTTPS GET (pre-signed URLs)     | **15-min TTL** for URLs.     | **+5% credits** (network I/O). | **Use internal stages** for critical data. |
| **COPY INTO**           | Parallel chunk reads (16-128MB) | **1 thread/chunk**.          | `1 credit/core-second`.        | **Compress files** (ZSTD).                 |
| **Result Set Fetch**    | Chunked (1-16MB)                | **Prefetch threads** (1-10). | **+5% credits** (network I/O). | **Increase `CLIENT_PREFETCH_THREADS**`.    |
| **Spill-to-Disk**       | Local SSD (NVMe)                | None.                        | **+2x credits**.               | **Increase warehouse size**.               |


---

### **4.3 Concurrency & Warehouse Scaling Rules**


| **Workload Type**         | **Recommended Warehouse** | **Max Clusters** | **Scaling Strategy**   | **Credit Overhead** | **Concurrency Limit** |
| ------------------------- | ------------------------- | ---------------- | ---------------------- | ------------------- | --------------------- |
| **Ad-Hoc Queries**        | `MEDIUM`                  | 1                | `AUTO_SUSPEND=60`      | 0%                  | 8 (for `MEDIUM`)      |
| **ETL Pipelines**         | `X-LARGE`                 | 4                | `MULTI_CLUSTER=TRUE`   | +10% per cluster    | 32 (for `X-LARGE`)    |
| **Data Science (ML)**     | `2X-LARGE`                | 2                | `QUERY_TAG=ml_team`    | +15% per cluster    | 16 (for `2X-LARGE`)   |
| **Micro-Batch (1-5 min)** | `LARGE`                   | 2                | `WAREHOUSE_SIZE=LARGE` | +5% per cluster     | 16 (for `LARGE`)      |
| **Real-Time (Sub-sec)**   | `3X-LARGE`                | 3                | `MULTI_CLUSTER=TRUE`   | +20% per cluster    | 24 (for `3X-LARGE`)   |


**Concurrency Formula**:  
`Max Concurrency = 8 * warehouse_size * max_clusters`

---

### **4.4 Credit Calculation Deep Dive**


| **Operation**              | **Credit Formula**                                                              | **Example**                                                        |
| -------------------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| **Query Execution**        | `Credits = (Warehouse Size) * (Query Duration in Seconds) / 3600`               | `LARGE` (4 cores) * 1800 sec / 3600 = **2 credits**.               |
| **Warehouse Idle**         | `Credits = (Warehouse Size) * (Idle Duration in Seconds) / 3600`                | `X-SMALL` (1 core) * 600 sec / 3600 = **0.166 credits**.           |
| **Cloud Services**         | `Credits = (Data Scanned in TB) * 0.0005` (S3) or `0.0002` (Snowflake Internal) | 10TB scanned (S3) = **5 credits**.                                 |
| **Storage**                | `Credits = (Average Daily Storage in TB) * 0.023` (Monthly)                     | 100TB avg storage = **2.3 credits/day**.                           |
| **Spill-to-Disk**          | `Credits = (Spilled Data in GB) * 0.0002`                                       | 500GB spilled = **0.1 credits**.                                   |
| **Multi-Cluster Overhead** | `Credits = (Base Credits) * (1 + (Max Clusters - 1) * 0.15)`                    | Base: 10 credits, `MAX_CLUSTERS=4` = **10 * 1.45 = 14.5 credits**. |
| **Result Cache**           | `Credits = 0` (cached results)                                                  | Repeated query = **0 credits**.                                    |
| **Dynamic Sampling**       | `Credits = (Data Scanned for Sampling in GB) * 0.0005`                          | 1GB sampled = **0.0005 credits**.                                  |


---

---

## **5. Monitoring, Observability & Troubleshooting**

---

### **5.1 Key Monitoring Views**


| **View**                                    | **Purpose**                                            | **Critical Columns**                                                                                  | **Retention**            |
| ------------------------------------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------- | ------------------------ |
| `QUERY_HISTORY`                             | **Query-level metrics** (latency, credits, errors).    | `QUERY_ID`, `START_TIME`, `END_TIME`, `TOTAL_ELAPSED_TIME`, `CREDITS_USED`, `ERROR_CODE`              | 365 days (Account Usage) |
| `WAREHOUSE_METERING_HISTORY`                | **Warehouse credit consumption** (hourly granularity). | `WAREHOUSE_NAME`, `START_TIME`, `CREDITS_USED`, `CREDITS_USED_COMPUTE`, `CREDITS_USED_CLOUD_SERVICES` | 365 days                 |
| `ACCOUNT_USAGE.QUERY_HISTORY`               | **Cross-warehouse query history**.                     | `USER_NAME`, `WAREHOUSE_NAME`, `QUERY_TEXT`, `PARTITION_ID`, `EXECUTION_STATUS`                       | 365 days                 |
| `INFORMATION_SCHEMA.TABLE_STORAGE_METRICS`  | **Storage usage** (per table/partition).               | `TABLE_NAME`, `PARTITION_ID`, `STORAGE_BYTES`, `ROW_COUNT`                                            | Session-scoped           |
| `INFORMATION_SCHEMA.QUERY_PROFILE`          | **Detailed execution profile** (operator-level).       | `QUERY_ID`, `OPERATOR`, `EXECUTION_TIME`, `ROWS_PRODUCED`, `MEMORY_USAGE`                             | Session-scoped           |
| `INFORMATION_SCHEMA.WAREHOUSE_USAGE`        | **Warehouse utilization**.                             | `WAREHOUSE_NAME`, `QUERY_ID`, `CREDITS_USED`, `MEMORY_USAGE`                                          | Session-scoped           |
| `INFORMATION_SCHEMA.CLUSTERING_INFORMATION` | **Clustering stats**.                                  | `TABLE_NAME`, `CLUSTERING_KEY`, `PARTITION_ID`, `OVERLAPS`, `DEPTH`                                   | Session-scoped           |
| `INFORMATION_SCHEMA.EXTERNAL_TABLE_FILES`   | **External stage file metadata**.                      | `TABLE_NAME`, `FILE_NAME`, `FILE_SIZE`, `LAST_MODIFIED`                                               | Session-scoped           |


---

### **5.2 Production-Grade Monitoring Queries**

---

#### **5.2.1 Query Performance Monitoring**

```sql
-- Top 10 slowest queries (last 24h)
SELECT
    QUERY_ID,
    USER_NAME,
    WAREHOUSE_NAME,
    QUERY_TEXT,
    TOTAL_ELAPSED_TIME / 1000 AS duration_sec,
    CREDITS_USED,
    ERROR_CODE,
    ERROR_MESSAGE,
    PARTITION_ID
FROM
    TABLE(INFORMATION_SCHEMA.QUERY_HISTORY(
        DATEADD('HOUR', -24, CURRENT_TIMESTAMP()),
        CURRENT_TIMESTAMP()
    ))
WHERE
    EXECUTION_STATUS = 'SUCCESS'
ORDER BY
    TOTAL_ELAPSED_TIME DESC
LIMIT 10;

-- Queries with highest credit usage (last 7 days)
SELECT
    QUERY_ID,
    USER_NAME,
    WAREHOUSE_NAME,
    QUERY_TEXT,
    CREDITS_USED,
    TOTAL_ELAPSED_TIME / 1000 AS duration_sec,
    BYTES_SCANNED / POWER(1024, 3) AS gb_scanned
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    START_TIME > DATEADD('DAY', -7, CURRENT_TIMESTAMP())
ORDER BY
    CREDITS_USED DESC
LIMIT 10;

-- Queries with spill-to-disk (last 30 days)
SELECT
    QUERY_ID,
    USER_NAME,
    WAREHOUSE_NAME,
    QUERY_TEXT,
    MEMORY_USAGE,
    BYTES_SCANNED / POWER(1024, 3) AS gb_scanned,
    CREDITS_USED
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    MEMORY_USAGE > 80
    AND START_TIME > DATEADD('DAY', -30, CURRENT_TIMESTAMP())
ORDER BY
    MEMORY_USAGE DESC;
```

---

#### **5.2.2 Warehouse Utilization Monitoring**

```sql
-- Warehouse credit usage (last 30 days)
SELECT
    WAREHOUSE_NAME,
    START_TIME,
    CREDITS_USED_COMPUTE,
    CREDITS_USED_CLOUD_SERVICES,
    (CREDITS_USED_COMPUTE + CREDITS_USED_CLOUD_SERVICES) AS total_credits,
    ROUND((CREDITS_USED_COMPUTE + CREDITS_USED_CLOUD_SERVICES) * 3, 2) AS cost_usd -- $3/credit
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE
    START_TIME > DATEADD('DAY', -30, CURRENT_TIMESTAMP())
ORDER BY
    START_TIME DESC;

-- Warehouse idle time (last 7 days)
SELECT
    WAREHOUSE_NAME,
    SUM(CREDITS_USED) AS idle_credits,
    COUNT(*) AS idle_periods
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE
    CREDITS_USED_COMPUTE = 0
    AND START_TIME > DATEADD('DAY', -7, CURRENT_TIMESTAMP())
GROUP BY
    WAREHOUSE_NAME
ORDER BY
    idle_credits DESC;

-- Warehouse concurrency (last 24h)
SELECT
    WAREHOUSE_NAME,
    START_TIME,
    QUERY_COUNT,
    ACTIVE_QUERY_COUNT,
    QUEUED_QUERY_COUNT
FROM
    TABLE(INFORMATION_SCHEMA.WAREHOUSE_USAGE(
        DATEADD('HOUR', -24, CURRENT_TIMESTAMP()),
        CURRENT_TIMESTAMP()
    ))
ORDER BY
    START_TIME DESC;
```

---

#### **5.2.3 Operator-Level Profiling**

```sql
-- Detailed query profile (operator-level)
SELECT
    QUERY_ID,
    OPERATOR,
    EXECUTION_TIME / 1000 AS execution_time_sec,
    ROWS_PRODUCED,
    ROWS_CONSUMED,
    MEMORY_USAGE / POWER(1024, 2) AS memory_usage_mb,
    PARTITION_ID
FROM
    TABLE(INFORMATION_SCHEMA.QUERY_PROFILE('QUERY_ID_HERE'))
ORDER BY
    EXECUTION_TIME DESC;

-- Join performance (last 7 days)
SELECT
    QUERY_ID,
    USER_NAME,
    WAREHOUSE_NAME,
    QUERY_TEXT,
    OPERATOR,
    EXECUTION_TIME / 1000 AS join_time_sec,
    ROWS_PRODUCED
FROM
    TABLE(INFORMATION_SCHEMA.QUERY_PROFILE())
WHERE
    OPERATOR LIKE '%Join%'
    AND START_TIME > DATEADD('DAY', -7, CURRENT_TIMESTAMP())
ORDER BY
    join_time_sec DESC;
```

---

#### **5.2.4 Storage & Clustering Monitoring**

```sql
-- Table storage growth (last 30 days)
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    PARTITION_ID,
    STORAGE_BYTES / POWER(1024, 3) AS storage_gb,
    ROW_COUNT,
    LAST_ALTERED
FROM
    INFORMATION_SCHEMA.TABLE_STORAGE_METRICS
WHERE
    LAST_ALTERED > DATEADD('DAY', -30, CURRENT_TIMESTAMP())
ORDER BY
    STORAGE_BYTES DESC;

-- Clustering effectiveness
SELECT
    TABLE_NAME,
    CLUSTERING_KEY,
    PARTITION_ID,
    OVERLAPS,
    DEPTH,
    CASE
        WHEN DEPTH <= 1 THEN 'Excellent'
        WHEN DEPTH <= 2 THEN 'Good'
        WHEN DEPTH <= 3 THEN 'Fair'
        ELSE 'Poor'
    END AS clustering_quality
FROM
    TABLE(INFORMATION_SCHEMA.CLUSTERING_INFORMATION('my_table'))
ORDER BY
    DEPTH DESC;

-- Micro-partition statistics
SELECT
    TABLE_NAME,
    PARTITION_ID,
    ROW_COUNT,
    MIN_VALUE,
    MAX_VALUE,
    AVG_VALUE
FROM
    TABLE(INFORMATION_SCHEMA.TABLE_STORAGE_METRICS)
WHERE
    TABLE_NAME = 'my_table';
```

---

#### **5.2.5 Result Cache Monitoring**

```sql
-- Cache hit ratio (last 24h)
SELECT
    CASE
        WHEN USE_CACHED_RESULT = TRUE THEN 'Cache Hit'
        ELSE 'Cache Miss'
    END AS cache_status,
    COUNT(*) AS query_count,
    AVG(TOTAL_ELAPSED_TIME / 1000) AS avg_duration_sec,
    AVG(CREDITS_USED) AS avg_credits
FROM
    TABLE(INFORMATION_SCHEMA.QUERY_HISTORY(
        DATEADD('HOUR', -24, CURRENT_TIMESTAMP()),
        CURRENT_TIMESTAMP()
    ))
GROUP BY
    cache_status;

-- Cache usage by query
SELECT
    QUERY_TEXT,
    COUNT(*) AS cache_hits,
    SUM(CREDITS_USED) AS credits_saved
FROM
    TABLE(INFORMATION_SCHEMA.QUERY_HISTORY(
        DATEADD('DAY', -7, CURRENT_TIMESTAMP()),
        CURRENT_TIMESTAMP()
    ))
WHERE
    USE_CACHED_RESULT = TRUE
GROUP BY
    QUERY_TEXT
ORDER BY
    cache_hits DESC;
```

---

### **5.3 Error Categorization & Incident Runbooks**

---

#### **5.3.1 Error Code Classification**


| **Error Code** | **Category**              | **Root Cause**                       | **Impact**              | **Severity** |
| -------------- | ------------------------- | ------------------------------------ | ----------------------- | ------------ |
| `2003`         | **Memory Limit Exceeded** | Spill-to-disk overflow.              | Query abort.            | **Critical** |
| `1020`         | **Transaction Conflict**  | Write-write conflict (MVCC).         | Transaction abort.      | **High**     |
| `002008`       | **Warehouse Busy**        | All warehouse threads occupied.      | Query queued.           | **Medium**   |
| `1003`         | **SQL Compilation Error** | Syntax error or invalid object.      | Query fails to parse.   | **High**     |
| `2001`         | **Planning Error**        | Invalid query plan.                  | Query fails to execute. | **High**     |
| `1049`         | **Disk Full**             | Local SSD spill limit reached.       | Query abort.            | **Critical** |
| `2012`         | **Stage I/O Error**       | Network timeout or permission issue. | Load failure.           | **High**     |
| `100072`       | **Permission Denied**     | Missing privileges.                  | Query/load failure.     | **High**     |
| `1204`         | **JSON Parsing Error**    | Malformed JSON.                      | Partial load failure.   | **Medium**   |
| `100083`       | **Column Mismatch**       | Source file has wrong column count.  | Load failure.           | **Medium**   |


---

#### **5.3.2 Incident Runbooks**

---

##### **Runbook: Memory Limit Exceeded (`2003`)**

1. **Diagnose**:
  ```sql
   -- Check memory usage for failed query
   SELECT
       QUERY_ID,
       MEMORY_USAGE,
       PARTITION_ID,
       QUERY_TEXT
   FROM
       TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
   WHERE
       ERROR_CODE = 2003
       AND QUERY_ID = 'FAILED_QUERY_ID';
  ```
2. **Mitigate**:
  - **Short-term**: Increase warehouse size (e.g., `LARGE` → `X-LARGE`).
  - **Long-term**:
    - **Optimize query**:
      - Add `WHERE` clauses to reduce data scanned.
      - Use **clustering keys** for pruning.
      - Break into **smaller batches** (e.g., `LIMIT 10000`).
    - **Materialize intermediate results**:
      ```sql
      CREATE TEMP TABLE temp_results AS
      SELECT /* heavy transformation */ FROM source;
      -- Then query temp_results
      ```
3. **Prevent**:
  - **Set `STATEMENT_TIMEOUT_IN_SECONDS**` to kill runaway queries.
  - **Monitor `MEMORY_USAGE**` in `QUERY_HISTORY`.
  - **Use `EXPLAIN**` to estimate memory usage before execution.

---

##### **Runbook: Transaction Conflict (`1020`)**

1. **Diagnose**:
  ```sql
   -- Check for concurrent transactions
   SELECT
       SESSION_ID,
       QUERY_TEXT,
       START_TIME,
       TRANSACTION_STATUS
   FROM
       TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
   WHERE
       TRANSACTION_STATUS = 'ACTIVE'
       AND START_TIME > DATEADD('MINUTE', -5, CURRENT_TIMESTAMP());
  ```
2. **Mitigate**:
  - **Retry with backoff**:
  - **Use `ABORT` in multi-statement transactions**:
    ```sql
    BEGIN;
    -- Critical section
    COMMIT;
    -- Non-critical section (runs outside transaction)
    ```
3. **Prevent**:
  - **Reduce transaction scope**: Break large transactions into smaller batches.
  - **Use `SERIALIZABLE` isolation** for high-contention workloads (accept **+20-30% latency**).

---

##### **Runbook: Warehouse Busy (`002008`)**

1. **Diagnose**:
  ```sql
   -- Check warehouse concurrency
   SELECT
       WAREHOUSE_NAME,
       ACTIVE_QUERY_COUNT,
       QUEUED_QUERY_COUNT,
       TOTAL_QUERY_COUNT
   FROM
       TABLE(INFORMATION_SCHEMA.WAREHOUSE_USAGE())
   WHERE
       WAREHOUSE_NAME = 'my_warehouse';
  ```
2. **Mitigate**:
  - **Short-term**: Increase warehouse size (e.g., `MEDIUM` → `LARGE`).
  - **Long-term**:
    - Enable `**MULTI_CLUSTER_WAREHOUSE=TRUE**`.
    - Tune `**AUTO_SUSPEND**` (e.g., `60` for interactive, `300` for batch).
    - Use `**QUERY_TAG**` to prioritize critical queries.
3. **Prevent**:
  - **Monitor `WAREHOUSE_USAGE**` for queueing.
  - **Set `MAX_CONCURRENCY_LEVEL**` to limit resource contention.

---

##### **Runbook: Slow Query (High `TOTAL_ELAPSED_TIME`)**

1. **Diagnose**:
  ```sql
   -- Get query profile
   SELECT
       OPERATOR,
       EXECUTION_TIME / 1000 AS execution_time_sec,
       ROWS_PRODUCED,
       MEMORY_USAGE / POWER(1024, 2) AS memory_usage_mb
   FROM
       TABLE(INFORMATION_SCHEMA.QUERY_PROFILE('SLOW_QUERY_ID'))
   ORDER BY
       EXECUTION_TIME DESC;
  ```
2. **Mitigate**:
  - **Optimize the slowest operator**:
    - **TableScan**: Add **clustering keys** or **partition pruning**.
    - **Join**: Use `**LEADING` hint** to force join order.
    - **Sort**: Avoid `ORDER BY` if not needed (use `LIMIT`).
    - **Aggregate**: **Pre-aggregate** in subqueries.
  - **Rewrite query**:
    ```sql
    -- Before: Slow JOIN
    SELECT a.*, b.*
    FROM large_table a
    JOIN small_table b ON a.key = b.key;

    -- After: Broadcast JOIN
    SELECT /*+ BROADCAST(b) */ a.*, b.*
    FROM large_table a
    JOIN small_table b ON a.key = b.key;
    ```
3. **Prevent**:
  - **Use `EXPLAIN**` to analyze query plans before execution.
  - **Monitor `QUERY_PROFILE**` for operator-level bottlenecks.

---

##### **Runbook: High Credit Usage**

1. **Diagnose**:
  ```sql
   -- Top credit-consuming queries
   SELECT
       QUERY_ID,
       USER_NAME,
       WAREHOUSE_NAME,
       QUERY_TEXT,
       CREDITS_USED,
       BYTES_SCANNED / POWER(1024, 3) AS gb_scanned
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       START_TIME > DATEADD('DAY', -7, CURRENT_TIMESTAMP())
   ORDER BY
       CREDITS_USED DESC
   LIMIT 10;
  ```
2. **Mitigate**:
  - **Reduce data scanned**:
    - Add `**WHERE` clauses** to filter early.
    - Use **clustering keys** for pruning.
    - **Partition tables** by time or other dimensions.
  - **Optimize joins**:
    - Use `**BROADCAST` hint** for small tables.
    - **Pre-aggregate** before joining.
  - **Use caching**:
    - Enable `**USE_CACHED_RESULT=TRUE**` for repeated queries.
3. **Prevent**:
  - **Set `STATEMENT_TIMEOUT_IN_SECONDS**` to kill expensive queries.
  - **Monitor `WAREHOUSE_METERING_HISTORY**` for credit spikes.

---

---

## **6. Advanced Production Patterns**

---

### **6.1 Query Optimization Patterns**


| **Pattern**            | **Use Case**                          | **Implementation**                                                                        | **Performance Impact**                | **Credit Savings**              |
| ---------------------- | ------------------------------------- | ----------------------------------------------------------------------------------------- | ------------------------------------- | ------------------------------- |
| **Clustering Keys**    | High-cardinality filter columns       | `CREATE CLUSTERING KEY (col1, col2) ON TABLE my_table;`                                   | **90% pruning** for filtered queries. | **10-90x fewer credits**.       |
| **Partition Pruning**  | Time-series or range-filtered queries | `CREATE TABLE my_table (...) PARTITION BY RANGE (date_column);`                           | **90% less I/O**.                     | **10-90x fewer credits**.       |
| **Broadcast Join**     | Small table joins (<10MB)             | `SELECT /*+ BROADCAST(small_table) */ * FROM large_table JOIN small_table ON ...;`        | **10x faster** than shuffle join.     | **+5% credits** (replication).  |
| **Materialized Views** | Repeated aggregations                 | `CREATE MATERIALIZED VIEW mv AS SELECT col1, COUNT(*) FROM my_table GROUP BY col1;`       | **100x faster** for repeated queries. | **1 credit/TB** (refresh cost). |
| **CTE Optimization**   | Complex multi-step queries            | Use `**MATERIALIZE` hint** for repeated CTEs: `WITH cte AS MATERIALIZE (SELECT ...) ...`. | **Faster repeated access**.           | **+10% credits** (storage).     |
| **Predicate Pushdown** | Filtered queries                      | **Automatic** in Snowflake (no action needed).                                            | **Reduces data scanned**.             | **10-50x fewer credits**.       |
| **Column Pruning**     | Queries with unused columns           | **Automatic** in Snowflake (no action needed).                                            | **Reduces I/O**.                      | **10-30% fewer credits**.       |
| **Result Caching**     | Repeated queries                      | `SET USE_CACHED_RESULT = TRUE;`                                                           | **90-95% latency reduction**.         | **0 credits** (cached).         |
| **Query Rewriting**    | Inefficient queries                   | Rewrite `NOT IN` as `NOT EXISTS`, `OR` as `UNION ALL`, etc.                               | **10-50% faster**.                    | **10-30% fewer credits**.       |
| **Batch Processing**   | Large transformations                 | Break into **smaller batches** (e.g., `LIMIT 10000`).                                     | **Avoids OOM**.                       | **+10% credits** (overhead).    |


---

### **6.2 Idempotency & Retry Patterns**


| **Pattern**             | **Use Case**                | **Implementation**                                                                    | **Pros**                    | **Cons**                           |
| ----------------------- | --------------------------- | ------------------------------------------------------------------------------------- | --------------------------- | ---------------------------------- |
| **Idempotent COPY**     | Load retries                | `COPY INTO ... FORCE=TRUE ON_ERROR=CONTINUE;`                                         | Simple, built-in.           | Silent truncation.                 |
| **Merge (UPSERT)**      | Incremental loads           | `MERGE INTO target USING source ON target.id = source.id WHEN MATCHED THEN UPDATE...` | Atomic, ACID-compliant.     | High credit cost for large tables. |
| **Transaction Log**     | Multi-statement idempotency | Log `QUERY_ID` + `START_TIME` in a control table. Replay only unprocessed queries.    | Full audit trail.           | Requires custom logic.             |
| **Checksum Validation** | File-level idempotency      | `SELECT SHA2_HEX(BINARY_LOAD_FILE('file.csv'))`. Store checksums in metadata table.   | Detects corruption.         | Overhead for large files.          |
| **Exponential Backoff** | Transient errors            | Retry with **2^n seconds delay** (max 3 retries).                                     | Handles temporary failures. | Adds latency.                      |


---

**Example: Idempotent COPY with DLQ**

```sql
-- Step 1: Create DLQ table
CREATE TABLE copy_dlq (
    file_name STRING,
    row_number INTEGER,
    error_message STRING,
    raw_line VARIANT,
    load_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    retry_count INTEGER DEFAULT 0
);

-- Step 2: COPY with DLQ routing
COPY INTO my_table
FROM @my_stage
FILE_FORMAT = (TYPE = 'CSV')
ON_ERROR = CONTINUE
VALIDATION_MODE = RETURN_ALL_ERRORS;

-- Step 3: Capture errors in DLQ
INSERT INTO copy_dlq
SELECT
    METADATA$FILE_NAME,
    METADATA$ROW_NUMBER,
    METADATA$ERROR_MESSAGE,
    $1
FROM @my_stage
WHERE METADATA$FILE_NAME IN (
    SELECT FILE_NAME
    FROM INFORMATION_SCHEMA.COPY_HISTORY(
        TABLE_NAME => 'my_table',
        START_TIMESTAMP => DATEADD('HOUR', -1, CURRENT_TIMESTAMP())
    )
    WHERE ERROR_COUNT > 0
);

-- Step 4: Reprocess DLQ
INSERT INTO my_table
SELECT
    raw_line:col1::STRING,
    raw_line:col2::INT
FROM copy_dlq
WHERE retry_count < 3;

-- Step 5: Update retry count
UPDATE copy_dlq
SET retry_count = retry_count + 1
WHERE file_name IN (
    SELECT file_name FROM copy_dlq WHERE retry_count < 3
);
```

---

**Example: Merge for UPSERT**

```sql
MERGE INTO target_table AS target
USING (
    SELECT
        id,
        name,
        value,
        last_updated
    FROM source_stage
) AS source
ON target.id = source.id
WHEN MATCHED THEN
    UPDATE SET
        target.name = source.name,
        target.value = source.value,
        target.last_updated = source.last_updated
WHEN NOT MATCHED THEN
    INSERT (id, name, value, last_updated)
    VALUES (source.id, source.name, source.value, source.last_updated);
```

---

**Example: Transaction Log for Idempotency**

```sql
-- Step 1: Create transaction log
CREATE TABLE copy_transaction_log (
    transaction_id STRING PRIMARY KEY,
    file_name STRING,
    table_name STRING,
    start_time TIMESTAMP,
    end_time TIMESTAMP,
    status STRING, -- 'PENDING', 'SUCCESS', 'FAILED'
    error_message STRING
);

-- Step 2: Idempotent COPY with transaction logging
BEGIN;
    -- Log transaction start
    INSERT INTO copy_transaction_log
    SELECT
        UUID_STRING() AS transaction_id,
        METADATA$FILE_NAME AS file_name,
        'my_table' AS table_name,
        CURRENT_TIMESTAMP() AS start_time,
        NULL AS end_time,
        'PENDING' AS status,
        NULL AS error_message
    FROM @my_stage
    WHERE METADATA$FILE_NAME NOT IN (
        SELECT file_name FROM copy_transaction_log WHERE status = 'SUCCESS'
    );

    -- Perform COPY
    COPY INTO my_table
    FROM @my_stage
    FILE_FORMAT = (TYPE = 'CSV')
    ON_ERROR = CONTINUE;

    -- Log transaction end
    UPDATE copy_transaction_log
    SET
        end_time = CURRENT_TIMESTAMP(),
        status = CASE
            WHEN ERROR_COUNT() = 0 THEN 'SUCCESS'
            ELSE 'FAILED'
        END,
        error_message = FIRST_ERROR_MESSAGE()
    WHERE
        transaction_id IN (
            SELECT transaction_id
            FROM copy_transaction_log
            WHERE status = 'PENDING'
        );
COMMIT;
```

---

### **6.3 CI/CD Validation Patterns**


| **Validation Type**        | **Tool/Method**                          | **Example**                                               |
| -------------------------- | ---------------------------------------- | --------------------------------------------------------- |
| **SQL Syntax**             | `SNOWFLAKE` CLI or `snowflake-connector` | `snowflake --query "SELECT * FROM my_table" --dry-run`    |
| **Schema Drift**           | `INFORMATION_SCHEMA` + Git diff          | [Schema Comparison Query](#schema-comparison-query)       |
| **Performance Regression** | `QUERY_HISTORY` + Baseline Comparison    | [Performance Test Query](#performance-test-query)         |
| **Data Quality**           | Great Expectations + Snowflake           | [Great Expectations Example](#great-expectations-example) |
| **Credit Cost**            | `WAREHOUSE_METERING_HISTORY`             | [Cost Validation Query](#cost-validation-query)           |


---

**Example: Schema Comparison for CI/CD**

```sql
-- Compare schema between dev and prod
SELECT
    'DEV' AS environment,
    TABLE_NAME,
    COLUMN_NAME,
    DATA_TYPE,
    IS_NULLABLE
FROM
    DEV.INFORMATION_SCHEMA.COLUMNS
WHERE
    TABLE_NAME = 'my_table'

UNION ALL

SELECT
    'PROD' AS environment,
    TABLE_NAME,
    COLUMN_NAME,
    DATA_TYPE,
    IS_NULLABLE
FROM
    PROD.INFORMATION_SCHEMA.COLUMNS
WHERE
    TABLE_NAME = 'my_table'

ORDER BY
    environment, TABLE_NAME, COLUMN_NAME;
```

---

**Example: Performance Test for CI/CD**

```sql
-- Compare query performance between versions
WITH baseline AS (
    SELECT
        QUERY_TEXT,
        AVG(TOTAL_ELAPSED_TIME) AS avg_latency_ms,
        AVG(CREDITS_USED) AS avg_credits
    FROM
        TABLE(INFORMATION_SCHEMA.QUERY_HISTORY(
            START_TIMESTAMP => DATEADD('DAY', -7, CURRENT_TIMESTAMP()),
            END_TIMESTAMP => DATEADD('DAY', -1, CURRENT_TIMESTAMP())
        ))
    WHERE
        QUERY_TEXT LIKE '%SELECT * FROM my_table%'
    GROUP BY
        QUERY_TEXT
)
SELECT
    b.QUERY_TEXT,
    b.avg_latency_ms AS baseline_latency,
    b.avg_credits AS baseline_credits,
    h.TOTAL_ELAPSED_TIME AS current_latency,
    h.CREDITS_USED AS current_credits,
    (h.TOTAL_ELAPSED_TIME - b.avg_latency_ms) / b.avg_latency_ms * 100 AS latency_diff_pct,
    (h.CREDITS_USED - b.avg_credits) / b.avg_credits * 100 AS credits_diff_pct
FROM
    baseline b
JOIN
    TABLE(INFORMATION_SCHEMA.QUERY_HISTORY(
        START_TIMESTAMP => CURRENT_TIMESTAMP()
    )) h
    ON b.QUERY_TEXT = h.QUERY_TEXT
WHERE
    h.QUERY_TEXT LIKE '%SELECT * FROM my_table%';
```

---

**Example: Great Expectations for Data Quality**

```python
# great_expectations.yml
datasources:
  snowflake:
    data_asset_type:
      class_name: Dataset
      module_name: great_expectations.dataset
    connection_string: snowflake://user:password@account/database/schema

# expectations/my_table.json
{
  "data_asset_type": "Dataset",
  "expectations": [
    {
      "expectation_type": "expect_column_values_to_not_be_null",
      "kwargs": {
        "column": "id",
        "meta": {
          "notes": "ID column must not be null"
        }
      }
    },
    {
      "expectation_type": "expect_column_values_to_be_of_type",
      "kwargs": {
        "column": "amount",
        "type_": "float"
      }
    },
    {
      "expectation_type": "expect_table_row_count_to_be_between",
      "kwargs": {
        "min_value": 1000,
        "max_value": 10000
      }
    }
  ]
}
```

---

### **6.4 Retry & Backpressure Patterns**


| **Scenario**              | **Retry Strategy**                | **Implementation**                                |
| ------------------------- | --------------------------------- | ------------------------------------------------- |
| **Transient Errors**      | Exponential backoff (3x, 2^n sec) | [Python Retry Example](#python-retry-example)     |
| **Warehouse Busy**        | Queue + Auto-resume               | Set `AUTO_RESUME=TRUE` + `AUTO_SUSPEND=60`.       |
| **Stage I/O Timeouts**    | Retry with jitter                 | [Stage Retry Example](#stage-retry-example)       |
| **Memory Errors**         | Increase warehouse size + Retry   | [Memory Retry Example](#memory-retry-example)     |
| **Transaction Conflicts** | Exponential backoff + `ABORT`     | [Conflict Retry Example](#conflict-retry-example) |


---

**Example: Python Retry Logic**

```python
import time
import snowflake.connector
from snowflake.connector.errors import OperationalError

def execute_with_retry(query, max_retries=3, initial_delay=1):
    delay = initial_delay
    for attempt in range(max_retries):
        try:
            conn = snowflake.connector.connect(...)
            cursor = conn.cursor()
            cursor.execute(query)
            return cursor.fetchall()
        except OperationalError as e:
            if attempt == max_retries - 1:
                raise
            if e.errno in [2003, 1020, 002008]:  # Retryable errors
                time.sleep(delay)
                delay *= 2  # Exponential backoff
            else:
                raise
    raise Exception("Max retries exceeded")

# Usage
execute_with_retry("SELECT * FROM my_table WHERE date = '2026-01-01'");
```

---

**Example: Stage Retry with Jitter (Snowflake Scripting)**

```sql
-- Retry COPY with jitter (Snowflake Scripting)
CREATE OR REPLACE PROCEDURE retry_copy_with_jitter(
    stage_name STRING,
    table_name STRING,
    max_retries INT,
    initial_delay INT
)
RETURNS STRING
LANGUAGE JAVASCRIPT
AS
$$
    const snowflake = require('snowflake-sdk');
    const connection = snowflake.createConnection({...});
    let retries = 0;
    let delay = INITIAL_DELAY * 1000; // Convert to ms

    while (retries < MAX_RETRIES) {
        try {
            const result = connection.execute({
                sqlText: `COPY INTO ${TABLE_NAME} FROM @${STAGE_NAME} ON_ERROR=CONTINUE`
            });
            return "SUCCESS";
        } catch (e) {
            if (e.code === 2012) { // Stage I/O error
                retries++;
                const jitter = Math.random() * 1000; // 0-1 sec jitter
                await new Promise(resolve => setTimeout(resolve, delay + jitter));
                delay *= 2; // Exponential backoff
            } else {
                throw e;
            }
        }
    }
    return "FAILED";
$$;

-- Call the procedure
CALL retry_copy_with_jitter('my_stage', 'my_table', 3, 1);
```

---

**Example: Memory Retry (Increase Warehouse Size)**

```sql
-- Step 1: Check current warehouse size
SELECT
    WAREHOUSE_NAME,
    WAREHOUSE_SIZE,
    WAREHOUSE_TYPE
FROM
    INFORMATION_SCHEMA.WAREHOUSES
WHERE
    WAREHOUSE_NAME = 'my_warehouse';

-- Step 2: Retry with larger warehouse
BEGIN;
    -- Switch to larger warehouse
    ALTER WAREHOUSE my_warehouse SET WAREHOUSE_SIZE = X-LARGE;

    -- Retry the failed query
    EXECUTE IMMEDIATE 'SELECT * FROM my_table WHERE complex_condition';

    -- Revert to original size (optional)
    ALTER WAREHOUSE my_warehouse SET WAREHOUSE_SIZE = LARGE;
END;
```

---

### **6.5 Security & Compliance Patterns**


| **Pattern**                  | **Use Case**        | **Implementation**                                                                         |
| ---------------------------- | ------------------- | ------------------------------------------------------------------------------------------ |
| **Row-Level Security (RLS)** | Data access control | `CREATE ROW ACCESS POLICY ...` + `ALTER TABLE ... ADD ROW ACCESS POLICY ...`               |
| **Column-Level Security**    | Column masking      | `CREATE MASKING POLICY ...` + `ALTER TABLE ... ADD COLUMN ... MASKING POLICY ...`          |
| **Data Encryption**          | At-rest encryption  | **Snowflake-managed (AES-256)** or **Customer-Managed Keys** (`CREATE ENCRYPTION KEY ...`) |
| **Audit Logging**            | Compliance tracking | `ACCOUNT_USAGE.AUDIT_HISTORY` + `INFORMATION_SCHEMA.AUDIT_HISTORY`                         |
| **Tagging**                  | Metadata management | `CREATE TAG ...` + `ALTER TABLE ... SET TAG ...`                                           |
| **Network Policies**         | IP whitelisting     | `CREATE NETWORK POLICY ...` + `ALTER ACCOUNT SET NETWORK_POLICY ...`                       |


---

**Example: Row-Level Security (RLS)**

```sql
-- Step 1: Create a policy
CREATE OR REPLACE ROW ACCESS POLICY rap_filter AS (
    user_role STRING,
    object_type STRING
) RETURNS BOOLEAN ->
    CASE
        WHEN user_role = 'ADMIN' THEN TRUE
        WHEN user_role = 'ANALYST' AND object_type = 'PUBLIC' THEN TRUE
        ELSE FALSE
    END;

-- Step 2: Apply policy to table
ALTER TABLE sensitive_data ADD ROW ACCESS POLICY rap_filter ON (user_role, object_type);

-- Step 3: Test policy
SELECT * FROM sensitive_data; -- Only rows matching policy are returned
```

---

**Example: Masking Policy**

```sql
-- Step 1: Create masking policy
CREATE OR REPLACE MASKING POLICY email_mask AS (val STRING) RETURNS STRING ->
    CASE
        WHEN CURRENT_ROLE() IN ('ADMIN', 'AUDITOR') THEN val
        ELSE '*****@example.com'
    END;

-- Step 2: Apply to column
ALTER TABLE users ALTER COLUMN email SET MASKING POLICY email_mask;

-- Step 3: Test
SELECT email FROM users; -- Non-admin sees masked values
```

---

**Example: Audit Query**

```sql
-- Query audit history for a table
SELECT
    EVENT_TIME,
    USER_NAME,
    OBJECT_TYPE,
    OBJECT_NAME,
    ACTION,
    STATUS
FROM
    SNOWFLAKE.ACCOUNT_USAGE.AUDIT_HISTORY
WHERE
    OBJECT_NAME = 'sensitive_data'
    AND EVENT_TIME > DATEADD('DAY', -7, CURRENT_TIMESTAMP())
ORDER BY
    EVENT_TIME DESC;
```

---

---

## **7. Decision Matrix / Quick Reference Flowchart**

---

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#ffd700', 'edgeLabelBackground':'#fff'}}}%%
flowchart TD
    A[Query Optimization Task] --> B{Query Type?}
    B -->|SELECT| C[Analyze Query]
    B -->|JOIN| D[Optimize Join]
    B -->|Aggregate| E[Optimize Aggregation]
    B -->|COPY| F[Optimize Load]

    C --> G{Data Volume?}
    G -->|< 1GB| H[Warehouse: X-SMALL]
    G -->|1GB - 100GB| I[Warehouse: MEDIUM]
    G -->|> 100GB| J[Warehouse: LARGE+]

    H --> K{Complexity?}
    I --> K
    J --> K
    K -->|Simple (Filter/Project)| L[Add WHERE + Clustering]
    K -->|Complex (Joins/Aggregates)| M[Use CTEs + Materialize]
    K -->|Repeated| N[Materialized View]

    D --> O{Join Type?}
    O -->|Small Table (<10MB)| P[Broadcast Join]
    O -->|Large Table| Q[Shuffle Join]
    O -->|Sorted Inputs| R[Sort-Merge Join]
    O -->|Skewed Data| S[SKEW_JOIN Hint]

    E --> T{Data Size?}
    T -->|< 1GB| U[Hash Aggregation]
    T -->|> 1GB| V[Streaming Aggregation]

    F --> W{Data Type?}
    W -->|Structured| X[Parquet + Clustering]
    W -->|Semi-Structured| Y[VARIANT + FLATTEN]
    W -->|Unstructured| Z[BLOB + External Functions]

    L --> AA{Performance Critical?}
    M --> AA
    N --> AA
    P --> AA
    Q --> AA
    R --> AA
    S --> AA
    U --> AA
    V --> AA
    X --> AA
    Y --> AA
    Z --> AA

    AA -->|Yes| AB[Add Hints + Monitor]
    AA -->|No| AC[Proceed]

    AB --> AD[Production-Ready]
    AC --> AD
```

---

---

## **8. Key Engineering Principles & Bottom Line**

---

### **8.1 Core Principles for SQL Optimization**


| **Principle**                     | **Application in Snowflake**                                               | **Impact**                                                    |
| --------------------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------------- |
| **Columnar Storage**              | **Parquet** format for all data. **Micro-partitioning** (16-128MB chunks). | **10-100x faster** for analytical queries (columnar scans).   |
| **Late Materialization**          | **Pruning** (partition, columnar) + **vectorized execution**.              | **90% less I/O** for filtered queries.                        |
| **Separation of Compute/Storage** | **Compute**: Warehouses. **Storage**: S3/GCS/Azure Blob.                   | **Independent scaling** (query without moving data).          |
| **MPP Architecture**              | **Massively Parallel Processing** with **no shared state**.                | **Linear scalability** (add clusters to increase throughput). |
| **Cost-Proportional Scaling**     | **Credits = f(Warehouse Size × Time)**.                                    | **Pay-per-use** (no over-provisioning).                       |
| **Result Caching**                | **24h TTL** for query results.                                             | **90-95% latency reduction** for repeated queries.            |
| **Automatic Optimization**        | **Predicate pushdown**, **column pruning**, **join reordering**.           | **Reduces manual tuning** effort.                             |
| **Failure Isolation**             | **Micro-partitioning** + **automatic retry**.                              | **No single point of failure**.                               |


---

### **8.2 Bottom Line for Production Engineers**

---

#### **8.2.1 Query Optimization Checklist**

1. **Pruning**:
  - **Cluster tables** on high-cardinality filter columns.
  - **Partition tables** by time or range.
  - **Use `WHERE` clauses** to filter early.
2. **Joins**:
  - **Broadcast small tables** (`/*+ BROADCAST(table) */`).
  - **Avoid Cartesian products** (add join conditions).
  - **Use `SKEW_JOIN` hint** for skewed data.
3. **Aggregations**:
  - **Pre-aggregate** in subqueries.
  - **Use streaming aggregation** for window functions.
4. **Sorting**:
  - **Avoid `ORDER BY`** if not needed (use `LIMIT`).
  - **Use `SORTKEY`** for sorted tables.
5. **Caching**:
  - **Enable `USE_CACHED_RESULT=TRUE`** for repeated queries.
  - **Materialize results** for performance-critical queries.
6. **Resource Management**:
  - **Right-size warehouses** (avoid `X-SMALL` for production).
  - **Set `AUTO_SUSPEND=60`** to reduce idle costs.
  - **Use `MULTI_CLUSTER_WAREHOUSE`** for high concurrency.
7. **Monitoring**:
  - **Monitor `QUERY_HISTORY`** for slow queries.
  - **Monitor `QUERY_PROFILE`** for operator-level bottlenecks.
  - **Monitor `WAREHOUSE_METERING_HISTORY`** for credit usage.

---

#### **8.2.2 Performance Tuning Rules of Thumb**


| **Rule**                      | **Action**                                              | **Impact**                            |
| ----------------------------- | ------------------------------------------------------- | ------------------------------------- |
| **Filter Early**              | Add `WHERE` clauses to reduce data scanned.             | **10-90x fewer credits**.             |
| **Cluster on Filter Columns** | `CREATE CLUSTERING KEY (col1, col2) ON TABLE my_table;` | **90% pruning** for filtered queries. |
| **Broadcast Small Tables**    | `/*+ BROADCAST(small_table) */`                         | **10x faster** joins.                 |
| **Avoid `SELECT ***`          | Explicitly list columns.                                | **Reduces I/O**.                      |
| **Use Parquet**               | Load data as **Parquet** (not CSV).                     | **10-30% faster** scans.              |
| **Pre-Aggregate**             | Materialize aggregations in subqueries.                 | **10-100x faster** queries.           |
| **Use Materialized Views**    | `CREATE MATERIALIZED VIEW ...`                          | **100x faster** for repeated queries. |
| **Monitor Spill**             | Check `MEMORY_USAGE` in `QUERY_HISTORY`.                | **Avoid +2x credit overhead**.        |
| **Set Timeouts**              | `SET STATEMENT_TIMEOUT_IN_SECONDS = 3600;`              | **Prevent runaway queries**.          |
| **Use Hints Sparingly**       | Only when CBO makes poor choices.                       | **Overrides can hurt performance**.   |


---

#### **8.2.3 Credit Savings Cheat Sheet**


| **Optimization**          | **Credit Savings**                       | **Example**                       |
| ------------------------- | ---------------------------------------- | --------------------------------- |
| **Clustering**            | **10-90x fewer credits**                 | Cluster on `date` column.         |
| **Partition Pruning**     | **10-90x fewer credits**                 | Partition by `date` column.       |
| **Broadcast Join**        | **+5% credits** (but 10x faster)         | `/*+ BROADCAST(small_table) */`   |
| **Predicate Pushdown**    | **10-50x fewer credits**                 | Add `WHERE` clauses.              |
| **Column Pruning**        | **10-30% fewer credits**                 | Explicitly list columns.          |
| **Result Caching**        | **0 credits** (cached)                   | `SET USE_CACHED_RESULT = TRUE;`   |
| **Materialized Views**    | **1 credit/TB** (refresh)                | `CREATE MATERIALIZED VIEW ...`    |
| **Compression (Parquet)** | **10-30% fewer credits** (smaller scans) | Use `COMPRESSION = ZSTD`.         |
| **Batch Processing**      | **+10% credits** (overhead)              | Break into `LIMIT 10000` batches. |


---

#### **8.2.4 Anti-Patterns to Avoid**


| **Anti-Pattern**           | **Why It’s Bad**                                         | **Fix**                                                                  |
| -------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------ |
| `**SELECT ***`             | Scans all columns (no columnar pruning).                 | Explicitly list columns.                                                 |
| **No `WHERE` Clause**      | Full table scan (no partition pruning).                  | Add filters on **clustering/partition keys**.                            |
| **Large JOINs**            | Spills to disk (memory limit: **80% of warehouse RAM**). | **Pre-aggregate** or use **broadcast joins** (`/*+ LEADING(table1) */`). |
| **No `AUTO_SUSPEND**`      | Warehouse runs idle (burns credits).                     | Set `AUTO_SUSPEND=60`.                                                   |
| `**X-SMALL` Warehouse**    | Fails on **>1TB scans** (OOM).                           | Use **at least `MEDIUM**` for production.                                |
| **No Clustering**          | Queries scan **entire table** (no pruning).              | Cluster on **high-cardinality filter columns**.                          |
| **CSV for Large Datasets** | **+30% slower** than Parquet.                            | Use **Parquet**.                                                         |
| **No Schema Validation**   | **Silent data corruption** (e.g., `VARCHAR` truncation). | Use `ENFORCE_SCHEMA=TRUE` + `VALIDATION_MODE=RETURN_ERRORS`.             |
| **Manual Retries**         | No backoff → **thundering herd**.                        | Use **exponential backoff** + jitter.                                    |
| **No Monitoring**          | **Undetected failures** (e.g., silent truncation).       | Monitor `QUERY_HISTORY`, `COPY_HISTORY`, `WAREHOUSE_METERING_HISTORY`.   |


---

---

## **9. Quick Reference Commands**

---

### **9.1 Query Optimization Commands**


| **Task**                     | **Command**                                                                              |
| ---------------------------- | ---------------------------------------------------------------------------------------- |
| **Analyze Query Plan**       | `EXPLAIN SELECT * FROM my_table WHERE col1 = 'value';`                                   |
| **Check Query Profile**      | `SELECT * FROM TABLE(INFORMATION_SCHEMA.QUERY_PROFILE('QUERY_ID'));`                     |
| **Cluster Table**            | `CREATE CLUSTERING KEY (col1, col2) ON TABLE my_table;`                                  |
| **Partition Table**          | `CREATE TABLE my_table (...) PARTITION BY RANGE (date_column);`                          |
| **Create Materialized View** | `CREATE MATERIALIZED VIEW mv AS SELECT col1, COUNT(*) FROM my_table GROUP BY col1;`      |
| **Broadcast Join**           | `SELECT /*+ BROADCAST(small_table) */ * FROM large_table JOIN small_table ON ...;`       |
| **Skew Join**                | `SELECT /*+ SKEW_JOIN(large_table, col1) */ * FROM large_table JOIN small_table ON ...;` |
| **Materialize CTE**          | `WITH cte AS MATERIALIZE (SELECT ...) SELECT * FROM cte;`                                |
| **Set Timeout**              | `SET STATEMENT_TIMEOUT_IN_SECONDS = 3600;`                                               |
| **Enable Result Cache**      | `SET USE_CACHED_RESULT = TRUE;`                                                          |


---

### **9.2 Monitoring Commands**


| **Task**                  | **Command**                                                                                                                                 |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| **Top Slow Queries**      | `SELECT * FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY()) ORDER BY TOTAL_ELAPSED_TIME DESC LIMIT 10;`                                        |
| **High Credit Queries**   | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY ORDER BY CREDITS_USED DESC LIMIT 10;`                                                  |
| **Spill-to-Disk Queries** | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY WHERE MEMORY_USAGE > 80;`                                                              |
| **Warehouse Usage**       | `SELECT * FROM TABLE(INFORMATION_SCHEMA.WAREHOUSE_USAGE());`                                                                                |
| **Clustering Stats**      | `SELECT * FROM TABLE(INFORMATION_SCHEMA.CLUSTERING_INFORMATION('my_table'));`                                                               |
| **Table Storage**         | `SELECT * FROM INFORMATION_SCHEMA.TABLE_STORAGE_METRICS WHERE TABLE_NAME = 'my_table';`                                                     |
| **Cache Hit Ratio**       | `SELECT CASE WHEN USE_CACHED_RESULT = TRUE THEN 'Hit' ELSE 'Miss' END, COUNT(*) FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY()) GROUP BY 1;` |
| **Warehouse Metering**    | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY;`                                                                         |
| **Query Profile**         | `SELECT * FROM TABLE(INFORMATION_SCHEMA.QUERY_PROFILE('QUERY_ID'));`                                                                        |


---

### **9.3 Error Handling Commands**


| **Task**                        | **Command**                                                                             |
| ------------------------------- | --------------------------------------------------------------------------------------- |
| **Kill Query**                  | `SELECT SYSTEM$CANCEL_QUERY('QUERY_ID');`                                               |
| **Check Errors**                | `SELECT * FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY()) WHERE ERROR_CODE IS NOT NULL;` |
| **Retry Failed COPY**           | `COPY INTO my_table FROM @my_stage ON_ERROR = CONTINUE;`                                |
| **Check Transaction Conflicts** | `SELECT * FROM TABLE(INFORMATION_SCHEMA.TRANSACTIONS());`                               |
| **Monitor Stage I/O**           | `SELECT * FROM INFORMATION_SCHEMA.STAGE_FILE_METADATA WHERE STAGE_NAME = 'my_stage';`   |


---

---

## **10. Further Reading**

- [Snowflake Query Optimization Guide](https://docs.snowflake.com/en/user-guide/performance)
- [Snowflake Warehouse Management](https://docs.snowflake.com/en/user-guide/warehouses)
- [Snowflake Clustering Keys](https://docs.snowflake.com/en/user-guide/clustering-keys)
- [Snowflake Materialized Views](https://docs.snowflake.com/en/user-guide/materialized-views)
- [Snowflake Query Hints](https://docs.snowflake.com/en/sql-reference/constructs/query-hints)
- [Snowflake Result Caching](https://docs.snowflake.com/en/user-guide/caching)
- [Snowflake Credit Usage Guide](https://www.snowflake.com/blog/understanding-snowflake-credits/)
- [Snowflake Internal Architecture (Sigmod 2020)](https://dl.acm.org/doi/10.1145/3318464.3386134)
- [Snowflake Performance Tuning Whitepaper](https://www.snowflake.com/wp-content/uploads/2021/11/Snowflake-Performance-Tuning-Guide.pdf)
- [Great Expectations for Snowflake](https://docs.greatexpectations.io/docs/guides/connecting_to_data/other_databases/snowflake)
