# **Snowflake Query Performance Evaluation: Production-Grade Technical Deep Dive**


## **1. Query Performance Fundamentals**

### **Mermaid: Snowflake Query Execution Architecture**
```mermaid
%% Snowflake Query Execution Architecture
flowchart TD
    subgraph Client["Client Layer"]
        A[("Client App\n(SnowSQL, JDBC, etc.)")] -->|SQL Query| B[("Query Router")]
    end

    subgraph Snowflake["Snowflake Layer"]
        B --> C[("Query Parser")]
        C --> D[("Query Optimizer")]
        D --> E[("Query Compiler")]
        E --> F[("Query Plan Cache")]
        F --> G[("Query Execution Engine")]
        G --> H[("Virtual Warehouse")]
        H --> I[("Metadata Service")]
        H --> J[("Storage Service\n(Cloud Storage)")]
        H --> K[("Result Cache")]
    end

    subgraph Execution["Query Execution"]
        G --> L[("Query Steps")]
        L --> M[("Scan")]
        L --> N[("Join")]
        L --> O[("Aggregate")]
        L --> P[("Sort")]
        L --> Q[("Filter")]
        M --> J
        N --> J
        O --> J
        P --> J
        Q --> J
    end

    subgraph Results["Results"]
        G --> R[("Result Set")]
        R --> S[("Client")]
        K --> R
    end

    subgraph Monitoring["Monitoring & Observability"]
        T[("QUERY_HISTORY")]
        U[("QUERY_PROFILE")]
        V[("ACCOUNT_USAGE")]
        W[("INFORMATION_SCHEMA")]
    end
    G --> T
    G --> U
    G --> V
    G --> W

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef client fill:#4285f4,stroke:#1976d2;
    classDef snowflake fill:#29abe2,stroke:#1a8fb8;
    classDef execution fill:#009688,stroke:#00796b;
    classDef results fill:#ff9800,stroke:#f57c00;
    classDef monitoring fill:#9c27b0,stroke:#7b1fa2;
    class A client;
    class B,C,D,E,F,G,H,I,J,K snowflake;
    class L,M,N,O,P,Q execution;
    class R,S results;
    class T,U,V,W monitoring;
```

### **Key Query Performance Metrics**

| **Metric** | **Definition** | **Target Value** | **Measurement Method** | **Impact of Poor Performance** |
|------------|---------------|------------------|--------------------------|---------------------------------|
| **Execution Time** | Time taken to execute the query (end-to-end) | <1 sec (OLTP), <10 sec (OLAP) | `QUERY_HISTORY.EXECUTION_TIME` | High latency, poor user experience |
| **Compilation Time** | Time taken to parse, optimize, and compile the query | <100ms | `QUERY_HISTORY.COMPILATION_TIME` | Slow query startup, high CPU usage |
| **Queue Time** | Time query spends waiting in queue | <100ms | `QUERY_HISTORY.QUEUE_TIME` | Resource contention, warehouse overloading |
| **Scan Time** | Time spent scanning data (from storage) | Minimize | `QUERY_PROFILE.SCAN_TIME` | High I/O costs, slow queries |
| **Join Time** | Time spent joining tables | Minimize | `QUERY_PROFILE.JOIN_TIME` | Inefficient joins, high CPU usage |
| **Aggregate Time** | Time spent aggregating data | Minimize | `QUERY_PROFILE.AGGREGATE_TIME` | Poor aggregation strategies |
| **Sort Time** | Time spent sorting data | Minimize | `QUERY_PROFILE.SORT_TIME` | Missing `ORDER BY` optimization |
| **Bytes Scanned** | Amount of data read from storage | Minimize | `QUERY_HISTORY.BYTES_SCANNED` | High storage costs, inefficient filtering |
| **Partitions Scanned** | Number of micro-partitions scanned | Minimize | `QUERY_HISTORY.PARTITIONS_SCANNED` | Poor clustering, missing partition pruning |
| **Rows Produced** | Number of rows returned by the query | As needed | `QUERY_HISTORY.ROWS_PRODUCED` | High memory usage, network overhead |
| **Credit Usage** | Snowflake credits consumed | Minimize | `QUERY_HISTORY.CREDITS_USED` | High costs |
| **Warehouse Utilization** | Percentage of warehouse capacity used | <80% | `QUERY_HISTORY.WAREHOUSE_SIZE` | Resource contention, throttling |
| **Spill to Disk** | Amount of data spilled to disk (due to memory limits) | 0 | `QUERY_PROFILE.SPILL_TO_DISK` | High memory pressure, slow performance |
| **Spill to Remote** | Amount of data spilled to remote storage | 0 | `QUERY_PROFILE.SPILL_TO_REMOTE` | Severe memory pressure, very slow performance |

### **Snowflake's Unique Architecture for Performance**

Snowflake's **multi-cluster, shared-data architecture** enables **high performance** and **scalability** by separating **compute** (virtual warehouses) from **storage** (cloud storage). Key features that impact performance:

1. **Micro-Partitioning**:
   - Data is **automatically divided** into **micro-partitions** (50MB-500MB each).
   - Enables **partition pruning** (only scan relevant partitions).
   - Enables **parallel processing** (each partition processed independently).

2. **Columnar Storage**:
   - Data is stored in **columnar format** (Parquet).
   - Enables **column pruning** (only read required columns).
   - Enables **vectorized execution** (process columns as vectors).

3. **Caching Layers**:
   - **Result Cache**: Caches query results for **24 hours** (if query and data are unchanged).
   - **Metadata Cache**: Caches table metadata (e.g., statistics, schema).
   - **Local Disk Cache**: Caches frequently accessed data in **SSD** (per warehouse).

4. **Query Optimization**:
   - **Automatic Clustering**: Improves **partition pruning** for large tables.
   - **Query Rewriting**: Optimizes queries (e.g., predicate pushdown, join reordering).
   - **Vectorized Execution**: Processes data in **batches** (SIMD instructions).

5. **Multi-Cluster Warehouses**:
   - **Multi-cluster warehouses** can **scale out** to handle **concurrent queries**.
   - **Auto-scaling**: Automatically adds/removes clusters based on load.

6. **Serverless Services**:
   - **Metadata service**, **query optimization**, and **cloud services** run on **serverless compute**.
   - No warehouse required for these operations.

## **2. Query Execution Internals**

### **A. Query Lifecycle**

```mermaid
%% Query Lifecycle
flowchart TD
    A[("Client Submits Query")] --> B[("Query Router")]
    B --> C[("Query Parser")]
    C --> D[("Query Optimizer")]
    D --> E[("Query Compiler")]
    E --> F{Query Plan Cache Hit?}
    F -->|Yes| G[("Retrieve Cached Plan")]
    F -->|No| H[("Generate New Plan")]
    H --> E
    G --> I[("Query Execution Engine")]
    I --> J[("Virtual Warehouse Allocation")]
    J --> K[("Query Execution")]
    K --> L[("Result Materialization")]
    L --> M[("Result Cache")]
    M --> N[("Return Results to Client")]
    K --> O[("Update Statistics")]
    O --> N

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef client fill:#4285f4,stroke:#1976d2;
    classDef router fill:#29abe2,stroke:#1a8fb8;
    classDef parser fill:#009688,stroke:#00796b;
    classDef optimizer fill:#ff9800,stroke:#f57c00;
    classDef compiler fill:#e91e63,stroke:#c2185b;
    classDef cache fill:#9c27b0,stroke:#7b1fa2;
    classDef engine fill:#3f51b5,stroke:#303f9f;
    classDef warehouse fill:#795548,stroke:#5d4037;
    classDef execution fill:#00bcd4,stroke:#0097a7;
    classDef result fill:#8bc34a,stroke:#689f38;
    class A client;
    class B router;
    class C parser;
    class D optimizer;
    class E compiler;
    class F,G cache;
    class H compiler;
    class I engine;
    class J warehouse;
    class K execution;
    class L,M result;
    class O execution;
    class N result;
```

#### **1. Query Parsing**
- **Lexical Analysis**: Tokenizes the SQL query into **keywords, identifiers, literals**.
- **Syntax Analysis**: Validates the query against Snowflake's **SQL grammar**.
- **Semantic Analysis**: Resolves **object references** (tables, columns, functions) and **data types**.
- **Error Handling**: Returns **syntax errors** or **semantic errors** (e.g., undefined table).

#### **2. Query Optimization**
- **Predicate Pushdown**: Moves **filters** closer to the data source (e.g., `WHERE` clauses pushed to scans).
- **Column Pruning**: Removes **unreferenced columns** from scans.
- **Partition Pruning**: Skips **irrelevant micro-partitions** based on filters.
- **Join Reordering**: Reorders **joins** to minimize intermediate results.
- **Common Subexpression Elimination (CSE)**: Reuses **subqueries** or **expressions**.
- **Constant Folding**: Evaluates **constant expressions** at compile time.
- **Query Rewriting**: Transforms queries into **equivalent, more efficient forms** (e.g., `NOT IN` → `NOT EXISTS`).

#### **3. Query Compilation**
- **Logical Plan**: Generates a **logical execution plan** (operators like `Scan`, `Join`, `Aggregate`).
- **Physical Plan**: Converts the logical plan into a **physical execution plan** (e.g., `HashJoin`, `SortMergeJoin`).
- **Plan Caching**: Caches the **physical plan** for **repeated queries** (if schema and data are unchanged).
- **Cost Estimation**: Estimates **execution cost** (credits, time) for the plan.

#### **4. Query Execution**
- **Warehouse Allocation**: Allocates **compute resources** from the virtual warehouse.
- **Parallel Execution**: Executes **query steps in parallel** (one thread per micro-partition).
- **Data Scanning**: Reads data from **cloud storage** (S3, Azure Blob, GCS) or **local cache**.
- **Operator Execution**: Executes **scan, join, aggregate, sort, filter** operators.
- **Spill Handling**: Spills data to **disk** or **remote storage** if memory limits are exceeded.
- **Result Materialization**: Collects and **materializes** the final result set.

#### **5. Result Caching**
- **Result Cache**: Stores query results in **SSD cache** for **24 hours** (if query and underlying data are unchanged).
- **Metadata Cache**: Updates **statistics** and **metadata** (e.g., table size, row counts).
- **Client Results**: Returns results to the **client** (JDBC, ODBC, REST API, etc.).

### **B. Query Plan Anatomy**

#### **1. Query Plan Structure**
Snowflake's query plan is a **tree of operators** that describes how the query will be executed. Each operator has:
- **Type**: `Scan`, `Join`, `Aggregate`, `Sort`, `Filter`, etc.
- **Input/Output**: Data flowing into/out of the operator.
- **Properties**: Operator-specific settings (e.g., join type, aggregate function).
- **Statistics**: Estimated **rows, bytes, cost** for the operator.

#### **2. Common Query Plan Operators**

| **Operator** | **Type** | **Description** | **Performance Impact** | **Optimization Tips** |
|--------------|----------|-----------------|------------------------|-----------------------|
| **TableScan** | Scan | Reads data from a table | High I/O, high memory | Use **clustering**, **partition pruning**, **column pruning** |
| **ExternalScan** | Scan | Reads data from external tables | High I/O, depends on cloud storage | Use **predicate pushdown**, **partition pruning** |
| **Filter** | Filter | Filters rows based on a condition | Low CPU, reduces data volume | Push **filters early** in the plan |
| **Project** | Project | Selects specific columns | Low CPU, reduces data volume | Use **column pruning** |
| **Join** | Join | Joins two datasets | High CPU, high memory | Use **proper join type** (HashJoin, MergeJoin, NestedLoop) |
| **Aggregate** | Aggregate | Groups and aggregates data | High CPU, high memory | Use **approximate functions** (e.g., `APPROX_COUNT_DISTINCT`) |
| **Sort** | Sort | Sorts data | High CPU, high memory | Use **`ORDER BY` sparingly**, use **`LIMIT`** to reduce sorting |
| **Union** | SetOp | Combines results from multiple queries | High CPU, high memory | Use **`UNION ALL`** instead of `UNION` if duplicates are acceptable |
| **WindowFunction** | Window | Applies window functions (e.g., `ROW_NUMBER()`, `SUM() OVER()`) | High CPU, high memory | Use **partitioning** to reduce window size |
| **Limit** | Limit | Limits the number of rows returned | Low CPU | Push **`LIMIT` early** in the plan |
| **Offset** | Offset | Skips a number of rows | High CPU (if large offset) | Avoid **large offsets** (use keyset pagination instead) |

#### **3. Query Plan Example**
```sql
-- Example query
SELECT
    c.customer_id,
    c.name,
    COUNT(o.order_id) AS order_count,
    SUM(o.amount) AS total_amount
FROM
    customers c
JOIN
    orders o ON c.customer_id = o.customer_id
WHERE
    o.order_date > '2023-01-01'
    AND c.region = 'US'
GROUP BY
    c.customer_id, c.name
ORDER BY
    total_amount DESC
LIMIT 100;
```

```text
-- Query Plan (simplified)
GlobalLimit [limit=100]
  LocalLimit [limit=100]
    Sort [order_by=[total_amount DESC]]
      Aggregate [group_by=[customer_id, name]]
        HashJoin [type=INNER]
          TableScan [table=customers, filters=[region='US']]
          Filter [filter=order_date > '2023-01-01']
            TableScan [table=orders]
```

#### **4. Query Plan Visualization**
Use `EXPLAIN` to view the query plan:
```sql
-- Basic EXPLAIN
EXPLAIN SELECT * FROM my_table WHERE id = 1;

-- EXPLAIN with cost estimation
EXPLAIN SELECT * FROM my_table WHERE id = 1
  WITH COST ESTIMATION = TRUE;

-- EXPLAIN with statistics
EXPLAIN SELECT * FROM my_table WHERE id = 1
  WITH STATISTICS = TRUE;

-- EXPLAIN for a specific query ID
EXPLAIN LAST QUERY;
```

### **C. Execution Phases**

| **Phase** | **Description** | **Duration** | **Resource Usage** | **Optimization Opportunities** |
|-----------|-----------------|--------------|--------------------|---------------------------------|
| **Parsing** | Parse SQL query into logical plan | 10-100ms | CPU (serverless) | Use **valid SQL**, avoid complex expressions |
| **Optimization** | Optimize logical plan into physical plan | 10-500ms | CPU (serverless) | Use **simple queries**, avoid nested subqueries |
| **Compilation** | Compile physical plan into executable code | 10-200ms | CPU (serverless) | Use **cached plans** where possible |
| **Queueing** | Wait for warehouse resources | 0-1000ms | None | Use **larger warehouses**, **multi-cluster warehouses** |
| **Execution** | Execute query on warehouse | 10ms-10min | CPU, Memory, I/O | Optimize **query design**, **clustering**, **warehouse size** |
| **Result Materialization** | Collect and return results | 10-1000ms | CPU, Memory | Use **`LIMIT`**, **pagination** to reduce result size |

## **3. Monitoring and Profiling Tools**

### **A. Key Monitoring Views**

| **View** | **Purpose** | **Retention** | **Key Columns** | **Example Query** |
|----------|-------------|---------------|-----------------|-------------------|
| `QUERY_HISTORY` | Detailed history of all executed queries | 365 days | `query_id`, `query_text`, `execution_time`, `bytes_scanned`, `credits_used`, `warehouse_size`, `start_time`, `end_time` | `SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_HISTORY()) WHERE start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP()) ORDER BY start_time DESC;` |
| `ACCOUNT_USAGE.QUERY_HISTORY` | Account-level query history | 365 days | Same as `QUERY_HISTORY` + `user_name`, `role_name` | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY WHERE start_time > DATEADD('day', -7, CURRENT_TIMESTAMP()) ORDER BY start_time DESC;` |
| `QUERY_PROFILE` | Detailed execution profile for a query | Session | `query_id`, `step_id`, `operation`, `rows_produced`, `bytes_scanned`, `execution_time`, `spill_to_disk`, `spill_to_remote` | `SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('01a2b3c4-d5e6-78f9')) ORDER BY step_id;` |
| `ACCOUNT_USAGE.QUERY_PROFILE` | Account-level query profile | 365 days | Same as `QUERY_PROFILE` | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_PROFILE WHERE query_id = '01a2b3c4-d5e6-78f9';` |
| `WAREHOUSE_LOAD_HISTORY` | Warehouse utilization history | 365 days | `warehouse_name`, `start_time`, `end_time`, `running_queries`, `queued_queries`, `credit_usage` | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY WHERE warehouse_name = 'MY_WH' AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP());` |
| `WAREHOUSE_METERING_HISTORY` | Warehouse credit usage history | 365 days | `warehouse_name`, `start_time`, `end_time`, `credits_used`, `query_type` | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY WHERE warehouse_name = 'MY_WH' AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP());` |
| `TABLE_STORAGE_METRICS` | Table storage metrics | 365 days | `table_name`, `storage_bytes`, `row_count`, `partition_count` | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS WHERE table_name = 'MY_TABLE';` |
| `MATERIALIZED_VIEW_REFRESH_HISTORY` | Materialized view refresh history | 365 days | `view_name`, `refresh_time`, `status`, `rows_refreshed`, `bytes_refreshed` | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_REFRESH_HISTORY WHERE view_name = 'MY_MV';` |

### **B. Query Profile Deep Dive**

The **Query Profile** provides **detailed execution metrics** for a specific query, including:
- **Step-by-step execution** (operators, rows, bytes, time).
- **Spill metrics** (spill to disk, spill to remote).
- **Parallelism** (number of threads, partitions processed).
- **Resource usage** (CPU, memory).

#### **1. Query Profile Example**
```sql
-- Get query ID from QUERY_HISTORY
SELECT query_id FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_text LIKE '%SELECT * FROM my_table%'
ORDER BY start_time DESC
LIMIT 1;

-- Get query profile for the query ID
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('01a2b3c4-d5e6-78f9'));
```

#### **2. Query Profile Output**
| **Column** | **Description** | **Example Value** | **Performance Insight** |
|------------|-----------------|-------------------|-------------------------|
| `query_id` | Unique identifier for the query | `01a2b3c4-d5e6-78f9` | Use to correlate with `QUERY_HISTORY` |
| `step_id` | Unique identifier for each step in the query plan | `1`, `2`, `3` | Identify bottlenecks in the plan |
| `parent_step_id` | Parent step ID (for nested steps) | `1`, `NULL` | Understand step hierarchy |
| `operation` | Type of operation (e.g., `TableScan`, `HashJoin`) | `TableScan` | Identify expensive operations |
| `options` | Operation-specific options | `table_name=my_table` | Understand operation details |
| `rows_produced` | Number of rows produced by the step | `1000000` | High rows = potential bottleneck |
| `bytes_scanned` | Bytes scanned by the step | `1073741824` (1GB) | High bytes = inefficient scanning |
| `execution_time` | Time spent in the step (ms) | `5000` (5 seconds) | High time = bottleneck |
| `spill_to_disk` | Bytes spilled to disk | `1048576` (1MB) | Spill = memory pressure |
| `spill_to_remote` | Bytes spilled to remote storage | `0` | Remote spill = severe memory pressure |
| `threads` | Number of threads used | `8` | Parallelism level |
| `partitions_processed` | Number of partitions processed | `100` | Partition pruning effectiveness |
| `bytes_processed` | Bytes processed by the step | `2147483648` (2GB) | Data volume processed |

#### **3. Query Profile Analysis Workflow**
1. **Identify Slow Queries**:
   ```sql
   SELECT
       query_id,
       query_text,
       execution_time,
       bytes_scanned,
       credits_used,
       warehouse_size
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       execution_time > 10000  -- >10 seconds
       AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY
       execution_time DESC;
   ```

2. **Get Query Profile**:
   ```sql
   SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('01a2b3c4-d5e6-78f9'));
   ```

3. **Analyze Bottlenecks**:
   - Look for steps with **high `execution_time`** (bottlenecks).
   - Look for steps with **high `bytes_scanned`** (inefficient scanning).
   - Look for **spill to disk/remote** (memory pressure).
   - Look for **low parallelism** (`threads`).

4. **Optimize Query**:
   - **High `bytes_scanned`**: Add **filters**, use **clustering**, use **partition pruning**.
   - **High `execution_time` for joins**: Use **proper join type**, **reduce join size**.
   - **Spill to disk/remote**: Increase **warehouse size**, **reduce data volume**.
   - **Low parallelism**: Check **warehouse concurrency**, **query complexity**.

### **C. Performance Metrics Dashboard**

#### **1. Query Performance Dashboard**
```sql
-- Create a query performance dashboard view
CREATE VIEW QUERY_PERFORMANCE_DASHBOARD AS
SELECT
    query_id,
    query_text,
    user_name,
    role_name,
    warehouse_name,
    warehouse_size,
    start_time,
    end_time,
    execution_time,
    compilation_time,
    queue_time,
    bytes_scanned,
    partitions_scanned,
    rows_produced,
    credits_used,
    DATEDIFF('second', start_time, end_time) AS total_duration_seconds,
    bytes_scanned / NULLIF(execution_time, 0) AS scan_rate_mb_per_sec,
    credits_used / NULLIF(execution_time, 0) AS credits_per_sec,
    CASE
        WHEN execution_time > 10000 THEN 'Slow (>10s)'
        WHEN execution_time > 1000 THEN 'Medium (1-10s)'
        ELSE 'Fast (<1s)'
    END AS performance_category,
    CASE
        WHEN spill_to_disk > 0 THEN 'Spill to Disk'
        WHEN spill_to_remote > 0 THEN 'Spill to Remote'
        ELSE 'No Spill'
    END AS spill_category
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;
```

#### **2. Warehouse Performance Dashboard**
```sql
-- Create a warehouse performance dashboard view
CREATE VIEW WAREHOUSE_PERFORMANCE_DASHBOARD AS
SELECT
    warehouse_name,
    warehouse_size,
    start_time,
    end_time,
    running_queries,
    queued_queries,
    credit_usage,
    DATEDIFF('second', start_time, end_time) AS duration_seconds,
    running_queries * 100.0 / NULLIF(warehouse_size_multiplier, 0) AS utilization_percent,
    credit_usage / NULLIF(DATEDIFF('second', start_time, end_time), 0) AS credits_per_sec
FROM
    (
        SELECT
            warehouse_name,
            warehouse_size,
            start_time,
            end_time,
            running_queries,
            queued_queries,
            credit_usage,
            CASE
                WHEN warehouse_size = 'XSMALL' THEN 1
                WHEN warehouse_size = 'SMALL' THEN 2
                WHEN warehouse_size = 'MEDIUM' THEN 4
                WHEN warehouse_size = 'LARGE' THEN 8
                WHEN warehouse_size = 'XLARGE' THEN 16
                WHEN warehouse_size = 'XXLARGE' THEN 32
                WHEN warehouse_size = 'XXXLARGE' THEN 64
                WHEN warehouse_size = 'XXXXLARGE' THEN 128
                ELSE 1
            END AS warehouse_size_multiplier
        FROM
            SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
        WHERE
            start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
    )
ORDER BY
    start_time DESC;
```

#### **3. Table Storage Dashboard**
```sql
-- Create a table storage dashboard view
CREATE VIEW TABLE_STORAGE_DASHBOARD AS
SELECT
    table_name,
    schema_name,
    database_name,
    storage_bytes,
    row_count,
    partition_count,
    storage_bytes / NULLIF(row_count, 0) AS avg_row_size_bytes,
    storage_bytes / 1024 / 1024 / 1024 AS storage_gb,
    last_altered
FROM
    SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE
    last_altered > DATEADD('day', -30, CURRENT_TIMESTAMP())
ORDER BY
    storage_gb DESC;
```

## **4. Identifying Performance Bottlenecks**

### **A. Bottleneck Categories**

| **Bottleneck Type** | **Symptoms** | **Root Causes** | **Diagnosis** | **Solutions** |
|--------------------|--------------|-----------------|---------------|---------------|
| **I/O Bottleneck** | High `bytes_scanned`, high `execution_time`, low CPU usage | Full table scans, missing filters, poor clustering | `QUERY_PROFILE.bytes_scanned`, `QUERY_HISTORY.bytes_scanned` | Add filters, use clustering, use partition pruning, use column pruning |
| **CPU Bottleneck** | High `execution_time`, high CPU usage, low `bytes_scanned` | Complex joins, aggregations, window functions | `QUERY_PROFILE.execution_time`, `WAREHOUSE_LOAD_HISTORY.running_queries` | Optimize joins, use approximate functions, reduce data volume |
| **Memory Bottleneck** | Spill to disk/remote, high `execution_time` | Large result sets, large joins, large aggregations | `QUERY_PROFILE.spill_to_disk`, `QUERY_PROFILE.spill_to_remote` | Increase warehouse size, reduce data volume, use `LIMIT` |
| **Network Bottleneck** | High latency, low throughput | Client-server distance, small result sets | `QUERY_HISTORY.execution_time`, client-side metrics | Use larger result sets, reduce query frequency, use caching |
| **Concurrency Bottleneck** | High `queue_time`, low `running_queries` | Warehouse overloaded, too many concurrent queries | `QUERY_HISTORY.queue_time`, `WAREHOUSE_LOAD_HISTORY.queued_queries` | Use larger warehouse, use multi-cluster warehouse, use query prioritization |
| **Compilation Bottleneck** | High `compilation_time`, low `execution_time` | Complex queries, many subqueries, large query plans | `QUERY_HISTORY.compilation_time` | Simplify queries, use cached plans, avoid dynamic SQL |
| **Storage Bottleneck** | High `bytes_scanned`, high `execution_time` | Large tables, poor clustering, external tables | `QUERY_HISTORY.bytes_scanned`, `TABLE_STORAGE_METRICS.storage_bytes` | Use clustering, use materialized views, partition large tables |

### **B. Bottleneck Diagnosis Flowchart**

```mermaid
%% Bottleneck Diagnosis Flowchart
flowchart TD
    A[("Slow Query")] --> B{High bytes_scanned?}
    B -->|Yes| C[("I/O Bottleneck")]
    B -->|No| D{High execution_time?}
    D -->|Yes| E{High CPU usage?}
    D -->|No| F[("Network/Compilation Bottleneck")]
    E -->|Yes| G[("CPU Bottleneck")]
    E -->|No| H{Spill to disk/remote?}
    H -->|Yes| I[("Memory Bottleneck")]
    H -->|No| J{High queue_time?}
    J -->|Yes| K[("Concurrency Bottleneck")]
    J -->|No| L[("Other Bottleneck")]

    C --> M[("Add filters\nUse clustering\nUse partition pruning")]
    G --> N[("Optimize joins\nUse approximate functions\nReduce data volume")]
    I --> O[("Increase warehouse size\nReduce data volume\nUse LIMIT")]
    K --> P[("Use larger warehouse\nUse multi-cluster warehouse\nUse query prioritization")]
    F --> Q[("Use result caching\nReduce query frequency\nSimplify queries")]
    L --> R[("Check external tables\nCheck cloud storage latency\nCheck network connectivity")]

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef io fill:#4285f4,stroke:#1976d2;
    classDef cpu fill:#ff9800,stroke:#f57c00;
    classDef memory fill:#e91e63,stroke:#c2185b;
    classDef concurrency fill:#009688,stroke:#00796b;
    classDef network fill:#9c27b0,stroke:#7b1fa2;
    classDef other fill:#607d8b,stroke:#455a64;
    classDef solution fill:#8bc34a,stroke:#689f38;
    class A default;
    class B,D,E,F,H,J io;
    class C io;
    class D,E,G cpu;
    class G cpu;
    class H,I memory;
    class I memory;
    class J,K concurrency;
    class K concurrency;
    class F network;
    class L other;
    class M,N,O,P,Q,R solution;
```

### **C. Bottleneck-Specific Diagnosis**

#### **1. I/O Bottleneck**
**Symptoms**:
- High `bytes_scanned` in `QUERY_HISTORY` or `QUERY_PROFILE`.
- High `execution_time` but low CPU usage.
- High `partitions_scanned` (scanning many micro-partitions).

**Root Causes**:
- **Full table scans**: Missing or ineffective `WHERE` clauses.
- **Poor clustering**: Data not clustered on frequently filtered columns.
- **Missing partition pruning**: Filters not applied to partitioned columns.
- **Column pruning**: Selecting all columns (`SELECT *`) when only a few are needed.
- **External tables**: Slow cloud storage access (S3, Azure Blob, GCS).

**Diagnosis**:
```sql
-- Check for high bytes_scanned
SELECT
    query_id,
    query_text,
    bytes_scanned,
    partitions_scanned,
    execution_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    bytes_scanned > 1000000000  -- >1GB
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    bytes_scanned DESC;

-- Check for full table scans
SELECT
    query_id,
    query_text,
    operation,
    options
FROM
    TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('01a2b3c4-d5e6-78f9'))
WHERE
    operation = 'TableScan'
    AND options NOT LIKE '%filters%';
```

**Solutions**:
| **Solution** | **Description** | **When to Use** | **Example** |
|--------------|-----------------|-----------------|-------------|
| **Add Filters** | Add `WHERE` clauses to reduce scanned data | Always | `SELECT * FROM my_table WHERE date > '2023-01-01'` |
| **Use Clustering** | Cluster tables on frequently filtered columns | Large tables, repetitive queries | `ALTER TABLE my_table CLUSTER BY (date, region)` |
| **Use Partition Pruning** | Filter on partitioned columns | Partitioned tables | `SELECT * FROM my_table WHERE partition_column = '2023-01'` |
| **Use Column Pruning** | Select only needed columns | Queries with `SELECT *` | `SELECT col1, col2 FROM my_table` |
| **Use Materialized Views** | Pre-compute and store results | Repetitive queries on large tables | `CREATE MATERIALIZED VIEW my_mv AS SELECT * FROM my_table WHERE date > CURRENT_DATE()` |
| **Use External Table Caching** | Cache external table metadata | Frequent queries on external tables | `ALTER EXTERNAL TABLE my_external_table SET AUTO_REFRESH = FALSE` |

#### **2. CPU Bottleneck**
**Symptoms**:
- High `execution_time` with high CPU usage (check `WAREHOUSE_LOAD_HISTORY`).
- High `execution_time` for **joins**, **aggregations**, or **window functions**.
- Low `bytes_scanned` but high `execution_time`.

**Root Causes**:
- **Complex joins**: Large or inefficient joins (e.g., Cartesian products).
- **Complex aggregations**: `GROUP BY` with many groups or expensive functions.
- **Window functions**: `OVER()` clauses with large partitions.
- **User-Defined Functions (UDFs)**: Custom JavaScript or Python UDFs.
- **Regular expressions**: Expensive `RLIKE` or `REGEXP` operations.

**Diagnosis**:
```sql
-- Check for high CPU usage
SELECT
    query_id,
    query_text,
    execution_time,
    warehouse_size,
    running_queries
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE
    running_queries > 0
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY
    running_queries DESC;

-- Check for expensive operators
SELECT
    query_id,
    step_id,
    operation,
    execution_time,
    rows_produced
FROM
    TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('01a2b3c4-d5e6-78f9'))
WHERE
    operation IN ('HashJoin', 'Aggregate', 'WindowFunction', 'NestedLoopJoin')
ORDER BY
    execution_time DESC;
```

**Solutions**:
| **Solution** | **Description** | **When to Use** | **Example** |
|--------------|-----------------|-----------------|-------------|
| **Optimize Joins** | Use proper join type, reduce join size | Large joins | `SELECT * FROM table1 INNER JOIN table2 ON table1.id = table2.id` |
| **Use Approximate Functions** | Use `APPROX_COUNT_DISTINCT`, `APPROX_QUANTILE` | Large aggregations | `SELECT APPROX_COUNT_DISTINCT(user_id) FROM my_table` |
| **Reduce Join Size** | Filter tables before joining | Joins with large tables | `SELECT * FROM (SELECT * FROM table1 WHERE date > '2023-01-01') t1 JOIN table2 t2 ON t1.id = t2.id` |
| **Avoid Cartesian Products** | Ensure join conditions are correct | Joins without `ON` clause | `SELECT * FROM table1, table2 WHERE table1.id = table2.id` |
| **Avoid UDFs** | Replace UDFs with built-in functions | Queries with UDFs | Use `DATE_PART` instead of JavaScript UDF |
| **Avoid Regex** | Replace `RLIKE` with simpler filters | Queries with regex | `SELECT * FROM my_table WHERE col LIKE '%pattern%'` |
| **Use Materialized Views** | Pre-compute expensive operations | Repetitive aggregations | `CREATE MATERIALIZED VIEW my_mv AS SELECT col1, COUNT(*) FROM my_table GROUP BY col1` |

#### **3. Memory Bottleneck**
**Symptoms**:
- **Spill to disk** (`spill_to_disk > 0` in `QUERY_PROFILE`).
- **Spill to remote** (`spill_to_remote > 0` in `QUERY_PROFILE`).
- High `execution_time` with high memory usage.

**Root Causes**:
- **Large result sets**: Queries returning millions of rows.
- **Large joins**: Joins producing large intermediate results.
- **Large aggregations**: `GROUP BY` with many groups.
- **Large window functions**: `OVER()` with large partitions.
- **Small warehouse**: Insufficient memory for the query.

**Diagnosis**:
```sql
-- Check for spills
SELECT
    query_id,
    query_text,
    spill_to_disk,
    spill_to_remote,
    execution_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    spill_to_disk > 0 OR spill_to_remote > 0
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    spill_to_remote DESC, spill_to_disk DESC;

-- Check query profile for spills
SELECT
    step_id,
    operation,
    spill_to_disk,
    spill_to_remote,
    rows_produced
FROM
    TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('01a2b3c4-d5e6-78f9'))
WHERE
    spill_to_disk > 0 OR spill_to_remote > 0
ORDER BY
    spill_to_remote DESC, spill_to_disk DESC;
```

**Solutions**:
| **Solution** | **Description** | **When to Use** | **Example** |
|--------------|-----------------|-----------------|-------------|
| **Increase Warehouse Size** | Use a larger warehouse for memory-intensive queries | Queries with spills | `ALTER WAREHOUSE MY_WH SET WAREHOUSE_SIZE = 'LARGE'` |
| **Use LIMIT** | Limit the number of rows returned | Queries returning large result sets | `SELECT * FROM my_table LIMIT 1000` |
| **Use Pagination** | Use `LIMIT` and `OFFSET` or keyset pagination | Queries with large result sets | `SELECT * FROM my_table ORDER BY id LIMIT 1000 OFFSET 0` |
| **Reduce Join Size** | Filter tables before joining | Joins with large intermediate results | `SELECT * FROM (SELECT * FROM table1 WHERE date > '2023-01-01') t1 JOIN table2 t2 ON t1.id = t2.id` |
| **Reduce Aggregation Groups** | Use `HAVING` to filter groups | Aggregations with many groups | `SELECT col1, COUNT(*) FROM my_table GROUP BY col1 HAVING COUNT(*) > 100` |
| **Use Approximate Functions** | Use `APPROX_COUNT_DISTINCT`, `APPROX_QUANTILE` | Large aggregations | `SELECT APPROX_COUNT_DISTINCT(user_id) FROM my_table` |
| **Avoid ORDER BY** | Remove `ORDER BY` if not needed | Queries with large result sets | `SELECT * FROM my_table` (instead of `ORDER BY`) |
| **Use Materialized Views** | Pre-compute and store results | Repetitive queries with large results | `CREATE MATERIALIZED VIEW my_mv AS SELECT * FROM my_table WHERE date > CURRENT_DATE()` |

#### **4. Concurrency Bottleneck**
**Symptoms**:
- High `queue_time` in `QUERY_HISTORY`.
- High `queued_queries` in `WAREHOUSE_LOAD_HISTORY`.
- Low `running_queries` but high `queued_queries`.

**Root Causes**:
- **Warehouse overloaded**: Too many concurrent queries for the warehouse size.
- **Multi-cluster warehouse at max**: All clusters are busy.
- **Query prioritization**: Low-priority queries waiting for high-priority queries.
- **Resource contention**: Other workloads (e.g., ETL, reporting) using the warehouse.

**Diagnosis**:
```sql
-- Check for high queue_time
SELECT
    query_id,
    query_text,
    queue_time,
    execution_time,
    warehouse_name,
    warehouse_size
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    queue_time > 1000  -- >1 second
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    queue_time DESC;

-- Check warehouse load
SELECT
    warehouse_name,
    warehouse_size,
    start_time,
    running_queries,
    queued_queries
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE
    queued_queries > 0
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY
    queued_queries DESC;
```

**Solutions**:
| **Solution** | **Description** | **When to Use** | **Example** |
|--------------|-----------------|-----------------|-------------|
| **Use Larger Warehouse** | Increase warehouse size to handle more concurrent queries | Warehouse overloaded | `ALTER WAREHOUSE MY_WH SET WAREHOUSE_SIZE = 'LARGE'` |
| **Use Multi-Cluster Warehouse** | Use multi-cluster warehouse for high concurrency | High concurrency workloads | `ALTER WAREHOUSE MY_WH SET MAX_CLUSTER_COUNT = 4` |
| **Use Query Prioritization** | Prioritize important queries | Mixed workloads | `ALTER WAREHOUSE MY_WH SET QUERY_PRIORITY = 'HIGH'` for important queries |
| **Use Separate Warehouses** | Use separate warehouses for different workloads | Resource contention | `CREATE WAREHOUSE ETL_WH`, `CREATE WAREHOUSE REPORTING_WH` |
| **Use Query Timeouts** | Set timeouts for long-running queries | Prevent warehouse hogging | `ALTER WAREHOUSE MY_WH SET STATEMENT_TIMEOUT_IN_SECONDS = 300` |
| **Use Resource Monitors** | Set limits on warehouse usage | Prevent runaway queries | `CREATE RESOURCE MONITOR MY_MONITOR WITH CREDIT_QUOTA = 1000` |

#### **5. Network Bottleneck**
**Symptoms**:
- High `execution_time` but low `bytes_scanned` and low CPU usage.
- High latency between client and Snowflake.
- Slow result retrieval.

**Root Causes**:
- **Client-server distance**: Client is far from Snowflake region.
- **Small result sets**: Many small queries with high latency.
- **Slow client**: Client application is slow to process results.
- **Network issues**: Network latency or packet loss.

**Diagnosis**:
```sql
-- Check for high execution_time with low bytes_scanned
SELECT
    query_id,
    query_text,
    execution_time,
    bytes_scanned,
    rows_produced
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    execution_time > 1000  -- >1 second
    AND bytes_scanned < 1000000  -- <1MB
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    execution_time DESC;

-- Client-side: Check network latency
ping myaccount.us-east-1.snowflakecomputing.com
traceroute myaccount.us-east-1.snowflakecomputing.com
```

**Solutions**:
| **Solution** | **Description** | **When to Use** | **Example** |
|--------------|-----------------|-----------------|-------------|
| **Use Larger Result Sets** | Reduce the number of queries by fetching more data per query | Many small queries | `SELECT * FROM my_table LIMIT 10000` (instead of 1000) |
| **Use Caching** | Cache query results to avoid re-execution | Repetitive queries | Use `RESULT_SCAN` or application-level caching |
| **Use Compression** | Compress results to reduce network transfer | Large result sets | `ALTER SESSION SET CLIENT_RESULT_COMPRESSION = TRUE` |
| **Use Asynchronous Queries** | Use `ASYNC` queries to avoid blocking | Client applications | `CALL SYSTEM$SUBMIT_ASYNC_QUERY('SELECT * FROM my_table')` |
| **Use PrivateLink/PSC** | Use private connectivity to reduce latency | Production environments | Configure AWS PrivateLink, Azure Private Link, or GCP PSC |
| **Use Regional Snowflake** | Use Snowflake in the same region as your clients | Global clients | Deploy Snowflake in `us-west-2` for US West clients |

#### **6. Compilation Bottleneck**
**Symptoms**:
- High `compilation_time` in `QUERY_HISTORY`.
- Low `execution_time` but high total duration.
- Complex queries with many subqueries or dynamic SQL.

**Root Causes**:
- **Complex queries**: Many subqueries, CTEs, or dynamic SQL.
- **Large query plans**: Queries with many operators.
- **Schema changes**: Recent changes to tables or views.
- **Cached plan invalidation**: Query plan cache was invalidated.

**Diagnosis**:
```sql
-- Check for high compilation_time
SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time,
    total_elapsed_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    compilation_time > 1000  -- >1 second
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    compilation_time DESC;
```

**Solutions**:
| **Solution** | **Description** | **When to Use** | **Example** |
|--------------|-----------------|-----------------|-------------|
| **Simplify Queries** | Break complex queries into simpler ones | Queries with many subqueries | Split into multiple CTEs or temporary tables |
| **Use Cached Plans** | Ensure query plan cache is used | Repetitive queries | Avoid dynamic SQL, use parameterized queries |
| **Avoid Dynamic SQL** | Use static SQL instead of dynamic SQL | Queries with dynamic SQL | Use prepared statements instead of `EXECUTE IMMEDIATE` |
| **Use Materialized Views** | Pre-compute complex queries | Repetitive complex queries | `CREATE MATERIALIZED VIEW my_mv AS SELECT ...` |
| **Use Stored Procedures** | Encapsulate complex logic in stored procedures | Complex workflows | `CREATE PROCEDURE my_proc() AS BEGIN ... END` |

#### **7. Storage Bottleneck**
**Symptoms**:
- High `bytes_scanned` for external tables.
- High `execution_time` for queries on external tables.
- Slow cloud storage access (S3, Azure Blob, GCS).

**Root Causes**:
- **Slow cloud storage**: High latency or low throughput from cloud storage.
- **Large external tables**: External tables with billions of rows.
- **Frequent external table queries**: Many queries on the same external table.
- **No caching**: External table data is not cached.

**Diagnosis**:
```sql
-- Check for slow external table queries
SELECT
    query_id,
    query_text,
    bytes_scanned,
    execution_time,
    tables_referenced
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    bytes_scanned > 1000000000  -- >1GB
    AND query_text LIKE '%@%'
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    execution_time DESC;

-- Check external table storage metrics
SELECT
    table_name,
    storage_bytes,
    row_count,
    last_altered
FROM
    SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE
    table_name LIKE '%EXTERNAL%'
ORDER BY
    storage_bytes DESC;
```

**Solutions**:
| **Solution** | **Description** | **When to Use** | **Example** |
|--------------|-----------------|-----------------|-------------|
| **Use Internal Tables** | Load external data into Snowflake tables | Frequently queried external tables | `COPY INTO my_table FROM @my_external_stage` |
| **Use Materialized Views** | Pre-compute and store results from external tables | Repetitive queries on external tables | `CREATE MATERIALIZED VIEW my_mv AS SELECT * FROM @my_external_stage` |
| **Use Caching** | Cache external table metadata | Frequently queried external tables | `ALTER EXTERNAL TABLE my_external_table SET AUTO_REFRESH = FALSE` |
| **Use PrivateLink/PSC** | Use private connectivity to cloud storage | Production environments | Configure AWS PrivateLink, Azure Private Link, or GCP PSC |
| **Optimize File Format** | Use efficient file formats (Parquet, ORC) | External tables with large files | `CREATE FILE FORMAT my_format TYPE = 'PARQUET'` |
| **Partition External Tables** | Partition external tables by date or key | Large external tables | `CREATE EXTERNAL TABLE my_table PARTITION BY (date)` |

## **5. Performance Optimization Techniques**

### **A. Clustering**

#### **1. Definition and Architecture**
**Clustering** in Snowflake is a **data organization technique** that **co-locates related data** within the same **micro-partitions** to **minimize I/O** and **improve query performance**. Unlike traditional databases, Snowflake's clustering is **not a physical index** but rather a **logical organization** of data that guides the **query optimizer**.

```mermaid
%% Clustering Architecture
flowchart TD
    subgraph Unclustered["Unclustered Table"]
        A1[("Partition 1\n(Rows 1-1000)")] -->|Scan All| B[("Query: WHERE date = '2023-01-01'")]
        A2[("Partition 2\n(Rows 1001-2000)")] --> B
        A3[("Partition 3\n(Rows 2001-3000)")] --> B
        A4[("Partition N\n(Rows N-1000 to N)")] --> B
    end

    subgraph Clustered["Clustered Table"]
        B1[("Partition 1\n(date='2023-01-01')")] -->|Scan 1| C[("Query: WHERE date = '2023-01-01'")]
        B2[("Partition 2\n(date='2023-01-02')")] -->|Skip| C
        B3[("Partition 3\n(date='2023-01-03')")] -->|Skip| C
        B4[("Partition N\n(date='2023-01-N')")] -->|Skip| C
    end

    subgraph ClusteringKeys["Clustering Keys"]
        D[("Single-Column\n(date)")]
        E[("Multi-Column\n(region, date)")]
        F[("Automatic\n(Snowflake-managed)")]
    end

    subgraph Metadata["Metadata"]
        G[("Clustering Metadata\n(Statistics)")]
        H[("Query Optimizer\n(Pruning)")]
    end
    D --> G
    E --> G
    F --> G
    G --> H
    H --> C

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef unclustered fill:#ffebee,stroke:#ef9a9a;
    classDef clustered fill:#e8f5e9,stroke:#2e7d32;
    classDef keys fill:#fff3e0,stroke:#ef6c00;
    classDef metadata fill:#2196f3,stroke:#03a9f4;
    class A1,A2,A3,A4 unclustered;
    class B1,B2,B3,B4 clustered;
    class B,C unclustered;
    class C clustered;
    class D,E,F keys;
    class G,H metadata;
```

#### **2. How Clustering Works**
1. **Clustering Keys**:
   - Define **1-4 columns** as clustering keys.
   - Snowflake **reorganizes data** within micro-partitions to **group rows with similar clustering key values**.
   - Example: Clustering on `date` groups all rows with the same date in the same micro-partitions.

2. **Partition Pruning**:
   - The **query optimizer** uses clustering metadata to **skip irrelevant micro-partitions**.
   - Example: A query with `WHERE date = '2023-01-01'` will only scan micro-partitions containing data for that date.

3. **Automatic Reclustering**:
   - Snowflake **automatically reclusters** data in the background as new data is loaded.
   - Reclustering is **free** (included in Snowflake credits).

4. **Clustering Depth**:
   - **Depth 1**: Data is clustered on the first clustering key.
   - **Depth 2**: Data is clustered on the first two clustering keys.
   - **Depth 3**: Data is clustered on the first three clustering keys.
   - **Depth 4**: Data is clustered on all four clustering keys.
   - Higher depth = **better pruning** but **higher maintenance overhead**.

#### **3. When to Use Clustering**
✅ **Large tables** (>1TB) with **repetitive queries** on specific columns.
✅ **Frequently filtered columns** (e.g., `date`, `region`, `customer_id`).
✅ **Range queries** (e.g., `WHERE date BETWEEN '2023-01-01' AND '2023-01-31'`).
✅ **Join columns** (clustering on join keys can improve join performance).
✅ **High-cardinality columns** (many distinct values) for **point queries**.
✅ **Time-series data** (cluster on `date` or `timestamp`).

#### **4. When NOT to Use Clustering**
❌ **Small tables** (<1GB; clustering overhead outweighs benefits).
❌ **Ad-hoc queries** (no repetitive patterns to optimize for).
❌ **Low-cardinality columns** (few distinct values; e.g., `gender`, `status`).
❌ **Columns not used in filters** (clustering has no effect).
❌ **Frequently updated tables** (reclustering overhead may impact performance).

#### **5. Clustering Types**
| **Type** | **Description** | **Use Case** | **Example** |
|----------|-----------------|--------------|-------------|
| **Single-Column** | Cluster on one column | Simple filtering | `ALTER TABLE my_table CLUSTER BY (date)` |
| **Multi-Column** | Cluster on 2-4 columns | Complex filtering | `ALTER TABLE my_table CLUSTER BY (region, date)` |
| **Automatic** | Snowflake manages clustering | Hands-off optimization | `ALTER TABLE my_table CLUSTER BY AUTO` |
| **None** | No clustering | Default | `ALTER TABLE my_table CLUSTER BY NONE` |

#### **6. Clustering Performance Impact**
| **Clustering Depth** | **Pruning Effectiveness** | **Reclustering Overhead** | **Best For** |
|----------------------|---------------------------|----------------------------|--------------|
| **Depth 0 (None)** | None | None | Small tables, ad-hoc queries |
| **Depth 1** | Low | Low | Single-column filtering |
| **Depth 2** | Medium | Medium | Two-column filtering |
| **Depth 3** | High | High | Three-column filtering |
| **Depth 4** | Very High | Very High | Four-column filtering |

#### **7. Clustering Monitoring**
```sql
-- Check clustering information for a table
SELECT
    table_name,
    clustering_information
FROM
    INFORMATION_SCHEMA.TABLES
WHERE
    table_name = 'MY_TABLE';

-- Check clustering depth
SELECT
    table_name,
    clustering_information:'CLUSTERING_DEPTH' AS clustering_depth
FROM
    INFORMATION_SCHEMA.TABLES
WHERE
    table_name = 'MY_TABLE';

-- Check clustering keys
SELECT
    table_name,
    clustering_information:'CLUSTER_BY' AS cluster_by
FROM
    INFORMATION_SCHEMA.TABLES
WHERE
    table_name = 'MY_TABLE';

-- Check reclustering status
SELECT
    table_name,
    last_reclustered,
    recluster_reason,
    clustering_information
FROM
    SNOWFLAKE.ACCOUNT_USAGE.TABLE_CLUSTERING_HISTORY
WHERE
    table_name = 'MY_TABLE'
ORDER BY
    last_reclustered DESC;
```

#### **8. Clustering Best Practices**
| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Cluster on Frequently Filtered Columns** | Choose columns used in `WHERE` clauses | `CLUSTER BY (date, region)` |
| **Prioritize High-Cardinality Columns** | Cluster on columns with many distinct values | `CLUSTER BY (customer_id, date)` |
| **Avoid Low-Cardinality Columns** | Avoid clustering on columns with few distinct values | Avoid `CLUSTER BY (gender)` |
| **Use Multi-Column Clustering for Complex Queries** | Cluster on multiple columns for complex filters | `CLUSTER BY (region, date, product_id)` |
| **Monitor Clustering Effectiveness** | Check `CLUSTERING_DEPTH` and `PARTITIONS_SCANNED` | `SELECT CLUSTERING_INFORMATION FROM INFORMATION_SCHEMA.TABLES` |
| **Recluster Manually if Needed** | Force reclustering for critical queries | `ALTER TABLE my_table RECLUSTER` |
| **Use Automatic Clustering for Hands-Off Optimization** | Let Snowflake manage clustering | `ALTER TABLE my_table CLUSTER BY AUTO` |
| **Combine with Partitioning** | Use clustering with partitioning for large tables | `CREATE TABLE my_table (id INT, date DATE) PARTITION BY (date) CLUSTER BY (region)` |

### **B. Materialized Views**

#### **1. Definition and Architecture**
**Materialized Views (MVs)** in Snowflake are **pre-computed query results** that are **stored as tables** and **automatically refreshed** when the underlying data changes. MVs are ideal for **repetitive, expensive queries** that can benefit from **pre-computation**.

```mermaid
%% Materialized Views Architecture
flowchart TD
    subgraph SourceTables["Source Tables"]
        A[("Table 1")] -->|Data| B[("Materialized View")]
        C[("Table 2")] -->|Data| B
    end

    subgraph MaterializedView["Materialized View"]
        B -->|Refresh| D[("Refresh Process")]
        D -->|Query| E[("Query Engine")]
        E -->|Update| B
    end

    subgraph Queries["Queries"]
        F[("Query on MV")] --> B
        G[("Query on Source Tables")] --> A
        G --> C
    end

    subgraph Refresh["Refresh Strategies"]
        H[("Automatic\n(On Data Change)")]
        I[("Manual\n(On Demand)")]
        J[("Scheduled\n(CRON)")]
    end
    D --> H
    D --> I
    D --> J

    subgraph Monitoring["Monitoring"]
        K[("REFRESH_HISTORY")]
        L[("QUERY_HISTORY")]
    end
    D --> K
    F --> L

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef source fill:#4285f4,stroke:#1976d2;
    classDef mv fill:#009688,stroke:#00796b;
    classDef queries fill:#ff9800,stroke:#f57c00;
    classDef refresh fill:#e91e63,stroke:#c2185b;
    classDef monitoring fill:#9c27b0,stroke:#7b1fa2;
    class A,C source;
    class B,D,E mv;
    class F,G queries;
    class H,I,J refresh;
    class K,L monitoring;
```

#### **2. How Materialized Views Work**
1. **Creation**:
   - Define a **materialized view** with a **query** (e.g., `SELECT * FROM table1 JOIN table2 ON ...`).
   - Snowflake **executes the query** and **stores the results** as a table.

2. **Refresh**:
   - **Automatic Refresh**: Snowflake **automatically refreshes** the MV when the underlying data changes.
   - **Manual Refresh**: Use `ALTER MATERIALIZED VIEW ... REFRESH` to refresh on demand.
   - **Scheduled Refresh**: Use **Tasks** to refresh on a schedule.

3. **Querying**:
   - Query the MV **like a regular table** (e.g., `SELECT * FROM my_mv`).
   - Snowflake **automatically uses the MV** for queries that match its definition.

4. **Optimization**:
   - Snowflake **rewrites queries** to use the MV when possible.
   - **Query folding**: Combines MV query with the original query for better performance.

#### **3. When to Use Materialized Views**
✅ **Repetitive, expensive queries** (e.g., daily aggregations).
✅ **Complex joins or aggregations** that are queried frequently.
✅ **Pre-computed reports** (e.g., dashboards, KPIs).
✅ **Data warehousing** (pre-compute star schema fact tables).
✅ **ETL pipelines** (pre-compute intermediate results).
✅ **Real-time analytics** (pre-compute frequently accessed data).

#### **4. When NOT to Use Materialized Views**
❌ **Ad-hoc queries** (no repetitive patterns).
❌ **Small tables** (<1GB; MV overhead outweighs benefits).
❌ **Frequently updated underlying data** (refresh overhead may impact performance).
❌ **Queries with highly variable filters** (MV may not cover all filter combinations).
❌ **Storage constraints** (MVs consume storage space).

#### **5. Materialized View Types**
| **Type** | **Description** | **Refresh Strategy** | **Use Case** |
|----------|-----------------|----------------------|--------------|
| **Standard MV** | Pre-computed query results | Automatic, Manual, Scheduled | Repetitive queries |
| **Secure MV** | MV with **row-level security** (RLS) and **data masking** | Automatic, Manual, Scheduled | Secure data access |
| **Non-Secure MV** | MV without RLS or data masking | Automatic, Manual, Scheduled | General use |

#### **6. Materialized View Performance Impact**
| **Metric** | **Impact** | **Notes** |
|------------|------------|-----------|
| **Query Performance** | ⬆️ **10-100x faster** | Pre-computed results |
| **Storage Usage** | ⬆️ **Increases** | MV stores a copy of the data |
| **Refresh Overhead** | ⬆️ **Increases** | Automatic refresh consumes credits |
| **Concurrency** | ⬆️ **Improves** | Reduces load on source tables |
| **Cost** | ⬆️ **Increases** | Storage + refresh credits |

#### **7. Materialized View Monitoring**
```sql
-- Check materialized view refresh history
SELECT
    view_name,
    refresh_time,
    status,
    rows_refreshed,
    bytes_refreshed,
    error_message
FROM
    SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_REFRESH_HISTORY
WHERE
    view_name = 'MY_MV'
ORDER BY
    refresh_time DESC;

-- Check materialized view storage usage
SELECT
    view_name,
    storage_bytes,
    row_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_STORAGE
WHERE
    view_name = 'MY_MV';

-- Check materialized view dependencies
SELECT
    view_name,
    dependent_object_name,
    dependent_object_type
FROM
    SNOWFLAKE.ACCOUNT_USAGE.OBJECT_DEPENDENCIES
WHERE
    object_name = 'MY_MV'
    AND object_type = 'MATERIALIZED VIEW';
```

#### **8. Materialized View Best Practices**
| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use for Repetitive Queries** | Create MVs for queries run frequently | `CREATE MATERIALIZED VIEW my_mv AS SELECT * FROM my_table WHERE date > CURRENT_DATE()` |
| **Use Automatic Refresh** | Let Snowflake refresh the MV automatically | `CREATE MATERIALIZED VIEW my_mv AS SELECT ...` (default) |
| **Use Manual Refresh for Large MVs** | Refresh large MVs on a schedule | `ALTER MATERIALIZED VIEW my_mv REFRESH` |
| **Use Scheduled Refresh for Batch Workloads** | Refresh MVs during off-peak hours | `CREATE TASK refresh_my_mv AS ALTER MATERIALIZED VIEW my_mv REFRESH` |
| **Monitor Refresh Performance** | Check `MATERIALIZED_VIEW_REFRESH_HISTORY` for errors | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_REFRESH_HISTORY` |
| **Use Secure MVs for Sensitive Data** | Apply RLS and data masking to MVs | `CREATE MATERIALIZED VIEW my_secure_mv SECURE AS SELECT ...` |
| **Avoid Overlapping MVs** | Avoid creating MVs with redundant data | Use a single MV for multiple queries |
| **Drop Unused MVs** | Remove MVs that are no longer needed | `DROP MATERIALIZED VIEW my_mv` |
| **Use MV for Star Schema Fact Tables** | Pre-compute fact tables for BI tools | `CREATE MATERIALIZED VIEW fact_sales AS SELECT * FROM sales JOIN dim_product ON ...` |
| **Combine with Clustering** | Cluster MVs for better query performance | `CREATE MATERIALIZED VIEW my_mv CLUSTER BY (date) AS SELECT ...` |

### **C. Query Rewriting**

#### **1. Definition**
**Query rewriting** is the process of **transforming a query** into an **equivalent but more efficient form**. Snowflake's **query optimizer** automatically rewrites queries, but you can also **manually rewrite queries** for better performance.

#### **2. Automatic Query Rewriting**
Snowflake automatically applies the following rewrites:
- **Predicate Pushdown**: Moves `WHERE` clauses closer to the data source.
- **Column Pruning**: Removes unreferenced columns from scans.
- **Partition Pruning**: Skips irrelevant micro-partitions.
- **Join Reordering**: Reorders joins to minimize intermediate results.
- **Common Subexpression Elimination (CSE)**: Reuses subqueries or expressions.
- **Constant Folding**: Evaluates constant expressions at compile time.
- **Query Folding**: Combines nested views or subqueries into a single query.

#### **3. Manual Query Rewriting Techniques**

| **Technique** | **Description** | **Before** | **After** | **Performance Impact** |
|---------------|-----------------|------------|-----------|-------------------------|
| **Predicate Pushdown** | Move filters early in the query | `SELECT * FROM (SELECT * FROM my_table) WHERE date > '2023-01-01'` | `SELECT * FROM my_table WHERE date > '2023-01-01'` | ⬆️ Reduces data scanned |
| **Column Pruning** | Select only needed columns | `SELECT * FROM my_table` | `SELECT col1, col2 FROM my_table` | ⬆️ Reduces I/O and memory |
| **Join Reordering** | Reorder joins to minimize intermediate results | `SELECT * FROM large_table JOIN small_table ON ...` | `SELECT * FROM small_table JOIN large_table ON ...` | ⬆️ Reduces join size |
| **Subquery to Join** | Replace correlated subqueries with joins | `SELECT * FROM table1 WHERE id IN (SELECT id FROM table2)` | `SELECT * FROM table1 JOIN table2 ON table1.id = table2.id` | ⬆️ Improves performance |
| **NOT IN to NOT EXISTS** | Replace `NOT IN` with `NOT EXISTS` | `SELECT * FROM table1 WHERE id NOT IN (SELECT id FROM table2)` | `SELECT * FROM table1 WHERE NOT EXISTS (SELECT 1 FROM table2 WHERE table2.id = table1.id)` | ⬆️ Avoids NULL handling issues |
| **EXISTS to IN** | Replace `EXISTS` with `IN` for small subqueries | `SELECT * FROM table1 WHERE EXISTS (SELECT 1 FROM table2 WHERE table2.id = table1.id)` | `SELECT * FROM table1 WHERE id IN (SELECT id FROM table2)` | ⬆️ Simpler execution plan |
| **Avoid SELECT *** | Select only needed columns | `SELECT * FROM my_table` | `SELECT col1, col2 FROM my_table` | ⬆️ Reduces I/O and memory |
| **Use WHERE Instead of HAVING** | Filter early with `WHERE` | `SELECT col1, COUNT(*) FROM my_table GROUP BY col1 HAVING COUNT(*) > 100` | `SELECT col1, COUNT(*) FROM (SELECT col1 FROM my_table WHERE ...) GROUP BY col1` | ⬆️ Reduces data scanned |
| **Use Semi-Join** | Use `EXISTS` or `IN` for filtering | `SELECT DISTINCT t1.* FROM table1 t1 JOIN table2 t2 ON t1.id = t2.id` | `SELECT * FROM table1 WHERE id IN (SELECT id FROM table2)` | ⬆️ Reduces duplicate rows |
| **Use Anti-Join** | Use `NOT EXISTS` or `NOT IN` for exclusion | `SELECT t1.* FROM table1 t1 LEFT JOIN table2 t2 ON t1.id = t2.id WHERE t2.id IS NULL` | `SELECT * FROM table1 WHERE id NOT IN (SELECT id FROM table2)` | ⬆️ Simpler execution plan |

### **D. Warehouse Optimization**

#### **1. Warehouse Sizing**
Snowflake warehouses come in **9 sizes** (X-Small to 4X-Large), each with **different compute and memory resources**. Choosing the right size is critical for **performance** and **cost optimization**.

| **Warehouse Size** | **Compute (Credits/Hour)** | **Memory (GB)** | **Max Concurrent Queries** | **Max Threads** | **Best For** | **Credit Cost/Hour** |
|--------------------|----------------------------|-----------------|-----------------------------|-----------------|--------------|----------------------|
| **X-Small** | 1 | 16 | 1 | 8 | Small queries, development | 0.28 |
| **Small** | 2 | 32 | 2 | 16 | Medium queries, small workloads | 0.56 |
| **Medium** | 4 | 64 | 4 | 32 | Large queries, medium workloads | 1.12 |
| **Large** | 8 | 128 | 8 | 64 | Very large queries, high workloads | 2.24 |
| **X-Large** | 16 | 256 | 16 | 128 | Complex queries, high concurrency | 4.48 |
| **2X-Large** | 32 | 512 | 32 | 256 | Very complex queries, very high concurrency | 8.96 |
| **3X-Large** | 64 | 1024 | 64 | 512 | Extremely complex queries, massive workloads | 17.92 |
| **4X-Large** | 128 | 2048 | 128 | 1024 | Largest workloads, highest concurrency | 35.84 |

#### **2. Warehouse Types**
| **Type** | **Description** | **Use Case** | **Example** |
|----------|-----------------|--------------|-------------|
| **Standard** | Single-cluster warehouse | General-purpose queries | `CREATE WAREHOUSE MY_WH WAREHOUSE_SIZE = 'MEDIUM'` |
| **Multi-Cluster** | Multi-cluster warehouse for high concurrency | High concurrency workloads | `CREATE WAREHOUSE MY_WH WAREHOUSE_SIZE = 'MEDIUM' MAX_CLUSTER_COUNT = 4` |
| **Serverless** | Serverless compute for specific operations (e.g., Snowpipe, Ingestion Service) | Serverless workloads | `CREATE INGESTION STREAM MY_STREAM` (uses serverless compute) |

#### **3. Warehouse Auto-Suspend and Auto-Resume**
- **Auto-Suspend**: Warehouse **suspends** after a period of inactivity (default: **10 minutes**).
- **Auto-Resume**: Warehouse **resumes** when a new query is submitted.
- **Cost Impact**: Suspended warehouses **do not consume credits**.

```sql
-- Set auto-suspend time (in seconds)
ALTER WAREHOUSE MY_WH SET AUTO_SUSPEND = 300;  -- 5 minutes

-- Disable auto-suspend
ALTER WAREHOUSE MY_WH SET AUTO_SUSPEND = NULL;
```

#### **4. Warehouse Scaling**
- **Multi-Cluster Warehouses**:
  - **Max Cluster Count**: Number of clusters (1-10 for Standard, 2-10 for Enterprise).
  - **Min Cluster Count**: Minimum number of clusters (1-10).
  - **Scaling Policy**: `STANDARD` (default) or `ECONOMY`.
    - **STANDARD**: Aggressively adds clusters to meet demand.
    - **ECONOMY**: Conservatively adds clusters to save costs.

```sql
-- Create a multi-cluster warehouse
CREATE WAREHOUSE MY_MC_WH
  WAREHOUSE_SIZE = 'MEDIUM'
  MAX_CLUSTER_COUNT = 4
  MIN_CLUSTER_COUNT = 1
  SCALING_POLICY = 'STANDARD'
  AUTO_SUSPEND = 300;

-- Alter a multi-cluster warehouse
ALTER WAREHOUSE MY_MC_WH SET MAX_CLUSTER_COUNT = 8;
```

#### **5. Warehouse Optimization Best Practices**
| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Right-Size Warehouses** | Use the smallest warehouse that meets performance requirements | `CREATE WAREHOUSE MY_WH WAREHOUSE_SIZE = 'SMALL'` |
| **Use Auto-Suspend** | Suspend warehouses when not in use to save credits | `ALTER WAREHOUSE MY_WH SET AUTO_SUSPEND = 300` |
| **Use Multi-Cluster for High Concurrency** | Use multi-cluster warehouses for high concurrency workloads | `CREATE WAREHOUSE MY_MC_WH MAX_CLUSTER_COUNT = 4` |
| **Use STANDARD Scaling for Critical Workloads** | Use `STANDARD` scaling for performance-critical workloads | `ALTER WAREHOUSE MY_WH SET SCALING_POLICY = 'STANDARD'` |
| **Use ECONOMY Scaling for Cost-Sensitive Workloads** | Use `ECONOMY` scaling for cost-sensitive workloads | `ALTER WAREHOUSE MY_WH SET SCALING_POLICY = 'ECONOMY'` |
| **Monitor Warehouse Utilization** | Check `WAREHOUSE_LOAD_HISTORY` for utilization | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY` |
| **Use Separate Warehouses for Different Workloads** | Avoid resource contention | `CREATE WAREHOUSE ETL_WH`, `CREATE WAREHOUSE REPORTING_WH` |
| **Use Query Timeouts** | Prevent long-running queries from hogging resources | `ALTER WAREHOUSE MY_WH SET STATEMENT_TIMEOUT_IN_SECONDS = 300` |
| **Use Resource Monitors** | Set limits on warehouse usage | `CREATE RESOURCE MONITOR MY_MONITOR WITH CREDIT_QUOTA = 1000` |

### **E. Result Caching**

#### **1. Definition**
Snowflake **automatically caches query results** for **24 hours** (configurable) if:
- The **query text** is identical.
- The **underlying data** has not changed.
- The **user's permissions** are the same.

#### **2. How Result Caching Works**
1. **Query Execution**:
   - Snowflake executes the query and **stores the results in cache**.
2. **Cache Lookup**:
   - For subsequent identical queries, Snowflake **checks the cache**.
   - If the cache is **valid**, Snowflake **returns the cached results** without re-executing the query.
3. **Cache Invalidation**:
   - Cache is **invalidated** if:
     - The **underlying data changes** (e.g., `INSERT`, `UPDATE`, `DELETE`).
     - The **query text changes**.
     - The **user's permissions change**.
     - The **cache TTL expires** (default: 24 hours).

#### **3. Result Cache Types**
| **Cache Type** | **Description** | **TTL** | **Scope** |
|----------------|-----------------|---------|-----------|
| **Result Cache** | Caches query results | 24 hours (configurable) | Per-user, per-query |
| **Metadata Cache** | Caches table metadata (e.g., statistics, schema) | Session | Per-session |
| **Local Disk Cache** | Caches frequently accessed data in SSD | Session | Per-warehouse |

#### **4. When to Use Result Caching**
✅ **Repetitive queries** (e.g., dashboards, reports).
✅ **Expensive queries** (e.g., complex joins, aggregations).
✅ **Read-only workloads** (no data changes).
✅ **Interactive queries** (e.g., BI tools, ad-hoc analysis).

#### **5. When NOT to Use Result Caching**
❌ **Frequently changing data** (cache is frequently invalidated).
❌ **Unique queries** (each query is different).
❌ **Write-heavy workloads** (cache is frequently invalidated).
❌ **Real-time data** (cache TTL is too long).

#### **6. Result Cache Configuration**
```sql
-- Enable/disable result caching (default: enabled)
ALTER SESSION SET USE_CACHED_RESULTS = TRUE;

-- Set result cache TTL (in seconds, default: 86400 = 24 hours)
ALTER SESSION SET RESULT_CACHE_TTL = 3600;  -- 1 hour

-- Check if a query used cached results
SELECT
    query_id,
    used_cached_result
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_id = '01a2b3c4-d5e6-78f9';
```

#### **7. Result Cache Best Practices**
| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use for Repetitive Queries** | Cache results for queries run frequently | Dashboards, reports |
| **Set Appropriate TTL** | Adjust TTL based on data freshness requirements | `ALTER SESSION SET RESULT_CACHE_TTL = 3600` |
| **Monitor Cache Usage** | Check `used_cached_result` in `QUERY_HISTORY` | `SELECT used_cached_result FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` |
| **Avoid Cache Invalidation** | Minimize changes to underlying data | Batch updates instead of frequent small updates |
| **Use for Read-Only Workloads** | Cache works best for read-only workloads | BI tools, reporting |
| **Disable for Unique Queries** | Disable caching for unique queries | `ALTER SESSION SET USE_CACHED_RESULTS = FALSE` |

### **F. Query Tagging**

#### **1. Definition**
**Query Tagging** allows you to **label queries** with custom metadata (e.g., `application`, `user`, `query_type`) for **monitoring**, **cost allocation**, and **performance analysis**.

#### **2. How Query Tagging Works**
1. **Set Query Tags**:
   - Use `ALTER SESSION SET QUERY_TAG = 'value'` to set a tag for the session.
   - Use `/*+ QUERY_TAG('value') */` to set a tag for a specific query.
2. **View Query Tags**:
   - Query tags are visible in `QUERY_HISTORY` and `ACCOUNT_USAGE.QUERY_HISTORY`.
3. **Filter by Query Tags**:
   - Use query tags to **filter** and **analyze** queries in monitoring views.

#### **3. When to Use Query Tagging**
✅ **Cost allocation** (track credits used by application/team).
✅ **Performance monitoring** (track performance by query type).
✅ **Debugging** (identify queries from specific applications).
✅ **Audit trails** (track query origins).

#### **4. Query Tagging Examples**
```sql
-- Set a query tag for the session
ALTER SESSION SET QUERY_TAG = 'my_application';

-- Set a query tag for a specific query
SELECT * FROM my_table /*+ QUERY_TAG('my_query_type') */;

-- Set multiple query tags
ALTER SESSION SET QUERY_TAG = 'application=my_app,user=my_user,query_type=report';

-- View query tags in QUERY_HISTORY
SELECT
    query_id,
    query_text,
    query_tag
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_tag LIKE '%my_application%'
ORDER BY
    start_time DESC;
```

#### **5. Query Tagging Best Practices**
| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use Consistent Tagging** | Use a consistent format for query tags | `application=my_app,user=my_user` |
| **Tag by Application** | Tag queries by the application that generated them | `ALTER SESSION SET QUERY_TAG = 'application=my_app'` |
| **Tag by User** | Tag queries by the user who ran them | `ALTER SESSION SET QUERY_TAG = 'user=my_user'` |
| **Tag by Query Type** | Tag queries by type (e.g., report, ETL) | `ALTER SESSION SET QUERY_TAG = 'query_type=report'` |
| **Use for Cost Allocation** | Track credits used by application/team | `SELECT query_tag, SUM(credits_used) FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY GROUP BY query_tag` |
| **Use for Performance Monitoring** | Track performance by query type | `SELECT query_tag, AVG(execution_time) FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY GROUP BY query_tag` |

## **6. Best Practices for Query Performance**

### **A. SQL Style Guide for Performance**

| **Guideline** | **Description** | **Bad Example** | **Good Example** | **Performance Impact** |
|---------------|-----------------|-----------------|------------------|-------------------------|
| **Use Explicit JOIN Syntax** | Use `JOIN` instead of comma-separated tables | `SELECT * FROM table1, table2 WHERE table1.id = table2.id` | `SELECT * FROM table1 JOIN table2 ON table1.id = table2.id` | ⬆️ Clearer intent, better optimization |
| **Avoid SELECT *** | Select only needed columns | `SELECT * FROM my_table` | `SELECT col1, col2 FROM my_table` | ⬆️ Reduces I/O and memory |
| **Use WHERE Before GROUP BY/HAVING** | Filter data early | `SELECT col1, COUNT(*) FROM my_table GROUP BY col1 HAVING COUNT(*) > 100` | `SELECT col1, COUNT(*) FROM (SELECT col1 FROM my_table WHERE date > '2023-01-01') GROUP BY col1` | ⬆️ Reduces data scanned |
| **Use INNER JOIN Instead of WHERE** | Use explicit `INNER JOIN` for joins | `SELECT * FROM table1, table2 WHERE table1.id = table2.id` | `SELECT * FROM table1 INNER JOIN table2 ON table1.id = table2.id` | ⬆️ Better optimization |
| **Use EXISTS Instead of IN for Large Subqueries** | `EXISTS` is more efficient for large subqueries | `SELECT * FROM table1 WHERE id IN (SELECT id FROM table2)` | `SELECT * FROM table1 WHERE EXISTS (SELECT 1 FROM table2 WHERE table2.id = table1.id)` | ⬆️ Stops at first match |
| **Use NOT EXISTS Instead of NOT IN** | `NOT EXISTS` handles NULLs better | `SELECT * FROM table1 WHERE id NOT IN (SELECT id FROM table2)` | `SELECT * FROM table1 WHERE NOT EXISTS (SELECT 1 FROM table2 WHERE table2.id = table1.id)` | ⬆️ Avoids NULL handling issues |
| **Use LIMIT for Pagination** | Use `LIMIT` and `OFFSET` for pagination | `SELECT * FROM my_table WHERE id > 1000 AND id <= 2000` | `SELECT * FROM my_table ORDER BY id LIMIT 1000 OFFSET 1000` | ⬆️ More efficient |
| **Use Keyset Pagination** | Use keyset pagination for large datasets | `SELECT * FROM my_table ORDER BY id LIMIT 1000 OFFSET 1000000` | `SELECT * FROM my_table WHERE id > last_id ORDER BY id LIMIT 1000` | ⬆️ Faster for large offsets |
| **Avoid Nested Subqueries** | Use joins or CTEs instead of nested subqueries | `SELECT * FROM (SELECT * FROM (SELECT * FROM my_table))` | `WITH cte AS (SELECT * FROM my_table) SELECT * FROM cte` | ⬆️ Easier to optimize |
| **Use CTEs for Readability** | Use CTEs (`WITH` clause) for complex queries | `SELECT * FROM (SELECT * FROM table1 JOIN table2 ON ...) WHERE ...` | `WITH cte AS (SELECT * FROM table1 JOIN table2 ON ...) SELECT * FROM cte WHERE ...` | ⬆️ More readable, same performance |
| **Avoid Functions on Filtered Columns** | Apply functions after filtering | `SELECT * FROM my_table WHERE UPPER(name) = 'ALICE'` | `SELECT * FROM my_table WHERE name = 'Alice'` | ⬆️ Enables index usage (if available) |
| **Use Parameterized Queries** | Use parameters instead of literals | `SELECT * FROM my_table WHERE id = 1` | `SELECT * FROM my_table WHERE id = ?` | ⬆️ Enables query plan caching |
| **Use Standard SQL** | Avoid non-standard SQL constructs | `SELECT * FROM my_table WHERE col LIKE '%pattern%'` | `SELECT * FROM my_table WHERE col ILIKE '%pattern%'` | ⬆️ Better portability |
| **Use Approximate Functions** | Use `APPROX_COUNT_DISTINCT`, `APPROX_QUANTILE` | `SELECT COUNT(DISTINCT user_id) FROM my_table` | `SELECT APPROX_COUNT_DISTINCT(user_id) FROM my_table` | ⬆️ Faster for large datasets |
| **Avoid OR Conditions** | Use `UNION ALL` instead of `OR` for indexed columns | `SELECT * FROM my_table WHERE col1 = 'A' OR col1 = 'B'` | `SELECT * FROM my_table WHERE col1 = 'A' UNION ALL SELECT * FROM my_table WHERE col1 = 'B'` | ⬆️ Better optimization |

### **B. Indexing Strategies (Snowflake-Specific)**

Snowflake **does not use traditional indexes** (B-tree, hash). Instead, it relies on:
1. **Micro-Partitioning**: Data is divided into **micro-partitions** (50MB-500MB).
2. **Clustering**: Data is **logically organized** within micro-partitions.
3. **Partition Pruning**: Only **relevant micro-partitions** are scanned.
4. **Column Pruning**: Only **relevant columns** are read.
5. **Metadata**: **Statistics** and **histograms** are stored for query optimization.

#### **1. Clustering as "Indexing"**
- **Single-Column Clustering**: Like a **single-column index**.
  ```sql
  ALTER TABLE my_table CLUSTER BY (date);
  ```
- **Multi-Column Clustering**: Like a **composite index**.
  ```sql
  ALTER TABLE my_table CLUSTER BY (region, date);
  ```
- **Automatic Clustering**: Like **automatic index management**.
  ```sql
  ALTER TABLE my_table CLUSTER BY AUTO;
  ```

#### **2. When to Use Clustering**
| **Scenario** | **Clustering Strategy** | **Example** |
|--------------|--------------------------|-------------|
| **Time-Series Data** | Cluster on `date` or `timestamp` | `CLUSTER BY (date)` |
| **Geospatial Data** | Cluster on `region`, `country`, `zip_code` | `CLUSTER BY (region, country)` |
| **High-Cardinality Columns** | Cluster on columns with many distinct values | `CLUSTER BY (customer_id)` |
| **Join Columns** | Cluster on columns used in joins | `CLUSTER BY (user_id)` |
| **Filter Columns** | Cluster on columns used in `WHERE` clauses | `CLUSTER BY (status, date)` |

#### **3. Clustering Anti-Patterns**
| **Anti-Pattern** | **Description** | **Example** | **Solution** |
|------------------|-----------------|-------------|--------------|
| **Clustering on Low-Cardinality Columns** | Clustering on columns with few distinct values | `CLUSTER BY (gender)` | Use high-cardinality columns |
| **Clustering on Non-Filtered Columns** | Clustering on columns not used in filters | `CLUSTER BY (unrelated_col)` | Cluster on filtered columns |
| **Over-Clustering** | Clustering on too many columns | `CLUSTER BY (col1, col2, col3, col4, col5)` | Limit to 1-4 columns |
| **Clustering on Frequently Updated Columns** | Clustering on columns that change often | `CLUSTER BY (last_updated)` | Cluster on static columns |
| **Clustering on Non-Selective Columns** | Clustering on columns with low selectivity | `CLUSTER BY (flag)` | Use selective columns |

### **C. Join Optimization**

#### **1. Join Types in Snowflake**
| **Join Type** | **Description** | **Syntax** | **Performance** | **When to Use** |
|---------------|-----------------|------------|-----------------|-----------------|
| **Inner Join** | Returns rows with matches in both tables | `SELECT * FROM t1 INNER JOIN t2 ON t1.id = t2.id` | ⭐⭐⭐⭐⭐ | Default join type, most efficient |
| **Left Join** | Returns all rows from left table + matches from right | `SELECT * FROM t1 LEFT JOIN t2 ON t1.id = t2.id` | ⭐⭐⭐⭐ | When you need all rows from left table |
| **Right Join** | Returns all rows from right table + matches from left | `SELECT * FROM t1 RIGHT JOIN t2 ON t1.id = t2.id` | ⭐⭐⭐ | Rarely used (use `LEFT JOIN` with swapped tables) |
| **Full Outer Join** | Returns all rows from both tables | `SELECT * FROM t1 FULL OUTER JOIN t2 ON t1.id = t2.id` | ⭐⭐ | When you need all rows from both tables |
| **Cross Join** | Returns Cartesian product of both tables | `SELECT * FROM t1 CROSS JOIN t2` | ⭐ | Only for small tables (avoid for large tables) |
| **Natural Join** | Joins on columns with the same name | `SELECT * FROM t1 NATURAL JOIN t2` | ⭐ | Avoid (ambiguous, not recommended) |
| **Self Join** | Joins a table to itself | `SELECT * FROM t1 a JOIN t1 b ON a.id = b.parent_id` | ⭐⭐⭐⭐ | Hierarchical data (e.g., org charts) |
| **Semi Join** | Returns rows from left table with matches in right | `SELECT * FROM t1 WHERE id IN (SELECT id FROM t2)` | ⭐⭐⭐⭐⭐ | Filtering (use `EXISTS` or `IN`) |
| **Anti Join** | Returns rows from left table without matches in right | `SELECT * FROM t1 WHERE id NOT IN (SELECT id FROM t2)` | ⭐⭐⭐⭐ | Exclusion (use `NOT EXISTS` or `NOT IN`) |

#### **2. Join Algorithms in Snowflake**
| **Algorithm** | **Description** | **When Used** | **Performance** | **Memory Usage** |
|---------------|-----------------|---------------|-----------------|------------------|
| **Hash Join** | Builds a hash table on one table and probes with the other | Default for most joins | ⭐⭐⭐⭐ | Medium |
| **Sort-Merge Join** | Sorts both tables and merges them | Used for large sorted tables | ⭐⭐⭐ | Low |
| **Nested Loop Join** | Nested loop over rows (rare in Snowflake) | Small tables, specific cases | ⭐ | High |
| **Broadcast Join** | Broadcasts the smaller table to all nodes | Small dimension tables | ⭐⭐⭐⭐⭐ | Low |

#### **3. Join Optimization Best Practices**
| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use Inner Join as Default** | Use `INNER JOIN` unless you need outer joins | `SELECT * FROM t1 INNER JOIN t2 ON t1.id = t2.id` |
| **Filter Before Joining** | Apply `WHERE` clauses before joining | `SELECT * FROM (SELECT * FROM t1 WHERE date > '2023-01-01') a JOIN t2 b ON a.id = b.id` |
| **Join on Indexed Columns** | Join on clustered or partitioned columns | `SELECT * FROM t1 CLUSTER BY (id) JOIN t2 ON t1.id = t2.id` |
| **Avoid Cartesian Products** | Ensure join conditions are specified | `SELECT * FROM t1 JOIN t2 ON t1.id = t2.id` (not `SELECT * FROM t1, t2`) |
| **Use Semi-Joins for Filtering** | Use `EXISTS` or `IN` for filtering | `SELECT * FROM t1 WHERE EXISTS (SELECT 1 FROM t2 WHERE t2.id = t1.id)` |
| **Use Anti-Joins for Exclusion** | Use `NOT EXISTS` or `NOT IN` for exclusion | `SELECT * FROM t1 WHERE NOT EXISTS (SELECT 1 FROM t2 WHERE t2.id = t1.id)` |
| **Join Small Tables First** | Join small tables before large tables | `SELECT * FROM small_table JOIN medium_table ON ... JOIN large_table ON ...` |
| **Use Broadcast Join for Small Tables** | Snowflake automatically uses broadcast join for small tables | `SELECT * FROM large_table JOIN small_table ON ...` |
| **Avoid Redundant Joins** | Remove unnecessary joins | `SELECT * FROM t1 JOIN t2 ON t1.id = t2.id JOIN t3 ON t1.id = t3.id` (if `t3` is not needed) |
| **Use CTEs for Complex Joins** | Use CTEs to break down complex joins | `WITH cte AS (SELECT * FROM t1 JOIN t2 ON ...) SELECT * FROM cte JOIN t3 ON ...` |

### **D. Aggregation Optimization**

#### **1. Aggregation Functions in Snowflake**
| **Function** | **Description** | **Performance** | **When to Use** |
|--------------|-----------------|-----------------|-----------------|
| `COUNT(*)` | Counts all rows | ⭐⭐⭐⭐⭐ | Always |
| `COUNT(col)` | Counts non-NULL values in a column | ⭐⭐⭐⭐ | When you need non-NULL counts |
| `SUM(col)` | Sums values in a column | ⭐⭐⭐⭐ | Numeric aggregations |
| `AVG(col)` | Averages values in a column | ⭐⭐⭐ | Numeric aggregations |
| `MIN(col)` / `MAX(col)` | Finds minimum/maximum value | ⭐⭐⭐⭐⭐ | Always |
| `APPROX_COUNT_DISTINCT(col)` | Approximate count of distinct values | ⭐⭐⭐⭐⭐ | Large datasets (faster than `COUNT(DISTINCT)`) |
| `APPROX_QUANTILE(col, p)` | Approximate quantile (e.g., median) | ⭐⭐⭐⭐ | Large datasets |
| `APPROX_TOP_K(col, k)` | Approximate top K values | ⭐⭐⭐⭐ | Large datasets |
| `APPROX_TOP_SUM(col, k)` | Approximate top K values by sum | ⭐⭐⭐⭐ | Large datasets |
| `STDDEV(col)` / `VARIANCE(col)` | Standard deviation / variance | ⭐⭐ | Statistical analysis |
| `LISTAGG(col, delimiter)` | Concatenates values into a list | ⭐⭐ | String aggregations |
| `ARRAY_AGG(col)` | Aggregates values into an array | ⭐⭐ | Array aggregations |

#### **2. Aggregation Optimization Best Practices**
| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Filter Before Aggregating** | Apply `WHERE` before `GROUP BY` | `SELECT col1, COUNT(*) FROM (SELECT col1 FROM my_table WHERE date > '2023-01-01') GROUP BY col1` |
| **Use Approximate Functions** | Use `APPROX_COUNT_DISTINCT` for large datasets | `SELECT APPROX_COUNT_DISTINCT(user_id) FROM my_table` |
| **Avoid COUNT(DISTINCT) on Large Tables** | `COUNT(DISTINCT)` is expensive for large tables | `SELECT APPROX_COUNT_DISTINCT(user_id) FROM my_table` |
| **Use GROUP BY on Clustered Columns** | Group by clustered columns for better performance | `SELECT region, COUNT(*) FROM my_table CLUSTER BY (region) GROUP BY region` |
| **Limit GROUP BY Groups** | Use `HAVING` to filter groups | `SELECT col1, COUNT(*) FROM my_table GROUP BY col1 HAVING COUNT(*) > 100` |
| **Use ROLLUP/CUBE for Hierarchical Aggregations** | Use `ROLLUP` or `CUBE` for multi-level aggregations | `SELECT region, product, SUM(sales) FROM my_table GROUP BY ROLLUP(region, product)` |
| **Use GROUPING SETS for Multiple Aggregations** | Use `GROUPING SETS` for multiple aggregations in one query | `SELECT region, SUM(sales) FROM my_table GROUP BY GROUPING SETS ((region), ())` |
| **Avoid Unnecessary Aggregations** | Remove aggregations that are not needed | `SELECT col1, col2 FROM my_table` (instead of `SELECT col1, COUNT(*) FROM my_table GROUP BY col1, col2`) |
| **Use Materialized Views for Aggregations** | Pre-compute aggregations | `CREATE MATERIALIZED VIEW my_mv AS SELECT region, COUNT(*) FROM my_table GROUP BY region` |
| **Use Window Functions for Running Aggregations** | Use `OVER()` for running totals, averages, etc. | `SELECT date, sales, SUM(sales) OVER (ORDER BY date) AS running_total FROM my_table` |

### **E. Subquery Optimization**

#### **1. Subquery Types**
| **Type** | **Description** | **Performance** | **Optimization** |
|----------|-----------------|-----------------|------------------|
| **Scalar Subquery** | Returns a single value | ⭐⭐⭐ | Use joins or CTEs |
| **Row Subquery** | Returns a single row | ⭐⭐⭐⭐ | Use joins or CTEs |
| **Table Subquery** | Returns multiple rows | ⭐⭐⭐ | Use joins or CTEs |
| **Correlated Subquery** | References columns from outer query | ⭐ | Use joins or `EXISTS` |
| **Uncorrelated Subquery** | Does not reference outer query | ⭐⭐⭐⭐ | Use CTEs or joins |

#### **2. Subquery Optimization Best Practices**
| **Best Practice** | **Description** | **Before** | **After** |
|-------------------|-----------------|------------|-----------|
| **Replace Correlated Subqueries with Joins** | Correlated subqueries are slow | `SELECT * FROM t1 WHERE id IN (SELECT id FROM t2 WHERE t2.col = t1.col)` | `SELECT * FROM t1 JOIN t2 ON t1.id = t2.id AND t2.col = t1.col` |
| **Replace EXISTS with IN for Small Subqueries** | `IN` is faster for small subqueries | `SELECT * FROM t1 WHERE EXISTS (SELECT 1 FROM t2 WHERE t2.id = t1.id)` | `SELECT * FROM t1 WHERE id IN (SELECT id FROM t2)` |
| **Replace NOT IN with NOT EXISTS** | `NOT EXISTS` handles NULLs better | `SELECT * FROM t1 WHERE id NOT IN (SELECT id FROM t2)` | `SELECT * FROM t1 WHERE NOT EXISTS (SELECT 1 FROM t2 WHERE t2.id = t1.id)` |
| **Use CTEs for Readability and Performance** | CTEs are optimized and easier to read | `SELECT * FROM (SELECT * FROM t1 JOIN t2 ON ...) WHERE ...` | `WITH cte AS (SELECT * FROM t1 JOIN t2 ON ...) SELECT * FROM cte WHERE ...` |
| **Avoid Nested Subqueries** | Nested subqueries are hard to optimize | `SELECT * FROM (SELECT * FROM (SELECT * FROM t1))` | `WITH cte1 AS (SELECT * FROM t1) SELECT * FROM cte1` |
| **Use Materialized Subqueries** | Use `MATERIALIZED` hint for subqueries | `SELECT * FROM (SELECT * FROM t1) WHERE ...` | `SELECT * FROM MATERIALIZE((SELECT * FROM t1)) WHERE ...` |
| **Limit Subquery Results** | Limit the number of rows returned by subqueries | `SELECT * FROM t1 WHERE id IN (SELECT id FROM t2)` | `SELECT * FROM t1 WHERE id IN (SELECT id FROM t2 LIMIT 1000)` |

### **F. Window Function Optimization**

#### **1. Window Function Types**
| **Type** | **Description** | **Performance** | **Example** |
|----------|-----------------|-----------------|-------------|
| **Ranking** | `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()` | ⭐⭐⭐⭐ | `ROW_NUMBER() OVER (ORDER BY sales)` |
| **Aggregate** | `SUM()`, `AVG()`, `COUNT()`, `MIN()`, `MAX()` | ⭐⭐⭐ | `SUM(sales) OVER (PARTITION BY region)` |
| **Value** | `FIRST_VALUE()`, `LAST_VALUE()`, `LAG()`, `LEAD()` | ⭐⭐⭐⭐ | `LAG(sales) OVER (ORDER BY date)` |
| **Analytic** | `PERCENT_RANK()`, `CUME_DIST()`, `NTILE()` | ⭐⭐⭐ | `PERCENT_RANK() OVER (ORDER BY sales)` |

#### **2. Window Function Optimization Best Practices**
| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Partition by Clustered Columns** | Partition by clustered columns for better performance | `SELECT *, SUM(sales) OVER (PARTITION BY region ORDER BY date) FROM my_table CLUSTER BY (region)` |
| **Limit Window Frame** | Use `RANGE` or `ROWS` to limit the window frame | `SUM(sales) OVER (PARTITION BY region ORDER BY date ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING)` |
| **Avoid Unbounded Windows** | Unbounded windows can be expensive | `SUM(sales) OVER (PARTITION BY region ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` |
| **Use Materialized Views for Window Functions** | Pre-compute window functions | `CREATE MATERIALIZED VIEW my_mv AS SELECT *, SUM(sales) OVER (PARTITION BY region ORDER BY date) FROM my_table` |
| **Filter Before Window Functions** | Apply `WHERE` before window functions | `SELECT *, SUM(sales) OVER (PARTITION BY region) FROM (SELECT * FROM my_table WHERE date > '2023-01-01')` |
| **Use QUALIFY for Filtering Window Results** | Use `QUALIFY` to filter window function results | `SELECT *, ROW_NUMBER() OVER (PARTITION BY region ORDER BY sales DESC) AS rn FROM my_table QUALIFY rn <= 10` |
| **Avoid Redundant Window Functions** | Remove window functions that are not needed | `SELECT *, ROW_NUMBER() OVER (ORDER BY id) AS rn FROM my_table` (if `rn` is not used) |
| **Use INDEX OFF for Large Windows** | Disable index usage for large windows (Snowflake-specific) | `SELECT *, SUM(sales) OVER (PARTITION BY region ORDER BY date) FROM my_table /*+ INDEX(OFF) */` |

### **G. Sorting and Ordering Optimization**

#### **1. Sorting Best Practices**
| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Avoid ORDER BY on Large Result Sets** | Sorting large result sets is expensive | `SELECT * FROM my_table LIMIT 1000` (instead of `ORDER BY`) |
| **Use LIMIT with ORDER BY** | Limit the number of rows sorted | `SELECT * FROM my_table ORDER BY sales DESC LIMIT 100` |
| **Sort by Clustered Columns** | Sort by clustered columns for better performance | `SELECT * FROM my_table CLUSTER BY (date) ORDER BY date` |
| **Use Keyset Pagination** | Use keyset pagination for large datasets | `SELECT * FROM my_table WHERE id > last_id ORDER BY id LIMIT 1000` |
| **Avoid Unnecessary Sorting** | Remove `ORDER BY` if not needed | `SELECT * FROM my_table` (instead of `ORDER BY id`) |
| **Use Materialized Views for Sorted Data** | Pre-compute sorted results | `CREATE MATERIALIZED VIEW my_mv AS SELECT * FROM my_table ORDER BY sales DESC` |

#### **2. ORDER BY vs. SORT BY**
- **`ORDER BY`**: Guarantees **global ordering** of the result set.
- **`SORT BY`**: Only **sorts within partitions** (faster but less precise).

```sql
-- ORDER BY (global sorting)
SELECT * FROM my_table ORDER BY sales DESC LIMIT 100;

-- SORT BY (partition-level sorting)
SELECT * FROM my_table SORT BY sales DESC LIMIT 100;
```

### **H. Pivoting and Unpivoting Optimization**

#### **1. PIVOT**
- **PIVOT** transforms **rows into columns**.
- Use `PIVOT` for **cross-tab reports** (e.g., sales by region and product).

```sql
-- Example: Pivot sales by region
SELECT *
FROM (
    SELECT
        product,
        region,
        sales
    FROM sales
)
PIVOT (
    SUM(sales)
    FOR region IN ('US', 'EU', 'APAC')
) AS p;
```

#### **2. UNPIVOT**
- **UNPIVOT** transforms **columns into rows**.
- Use `UNPIVOT` to **normalize data** (e.g., convert wide tables to long tables).

```sql
-- Example: Unpivot sales by region
SELECT
    product,
    region,
    sales
FROM sales_pivot
UNPIVOT (
    sales
    FOR region IN (us_sales, eu_sales, apac_sales)
);
```

#### **3. Pivot/Unpivot Optimization Best Practices**
| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Filter Before Pivoting** | Apply `WHERE` before `PIVOT` | `SELECT * FROM (SELECT * FROM sales WHERE date > '2023-01-01') PIVOT ...` |
| **Use PIVOT for Cross-Tab Reports** | Use `PIVOT` for reports with columns as categories | `SELECT * FROM sales PIVOT (SUM(sales) FOR region IN (...))` |
| **Use UNPIVOT for Normalization** | Use `UNPIVOT` to convert wide tables to long tables | `SELECT * FROM wide_table UNPIVOT (...)` |
| **Avoid Pivoting Large Tables** | Pivoting large tables can be expensive | Use `LIMIT` or filter before pivoting |
| **Use Materialized Views for Pivoted Data** | Pre-compute pivoted results | `CREATE MATERIALIZED VIEW my_mv AS SELECT * FROM sales PIVOT ...` |

## **7. Advanced Performance Topics**

### **A. Micro-Partitioning Deep Dive**

#### **1. What Are Micro-Partitions?**
- Snowflake **automatically divides** tables into **micro-partitions** (50MB-500MB each).
- Micro-partitions are **immutable** (once written, they cannot be modified).
- Each micro-partition contains:
  - **Data files** (columnar storage in Parquet format).
  - **Metadata** (min/max values for each column, row count, etc.).

#### **2. Micro-Partition Properties**
| **Property** | **Description** | **Impact on Performance** |
|--------------|-----------------|---------------------------|
| **Size** | 50MB-500MB | Smaller partitions = better pruning |
| **Immutability** | Cannot be modified after creation | Enables efficient updates (via COPY ON WRITE) |
| **Columnar Storage** | Data stored in Parquet format | Enables column pruning, vectorized execution |
| **Metadata** | Min/max values for each column | Enables partition pruning |
| **Compression** | Data is compressed (Snappy, Zstd) | Reduces I/O and storage costs |

#### **3. How Micro-Partitioning Improves Performance**
1. **Partition Pruning**:
   - Snowflake **skips irrelevant micro-partitions** based on query filters.
   - Example: `WHERE date = '2023-01-01'` only scans partitions containing data for that date.

2. **Column Pruning**:
   - Snowflake **only reads the columns** referenced in the query.
   - Example: `SELECT col1, col2 FROM my_table` only reads `col1` and `col2`.

3. **Parallel Processing**:
   - Each micro-partition is **processed in parallel** by a separate thread.
   - Example: A query scanning 100 partitions uses 100 threads.

4. **Vectorized Execution**:
   - Data is processed in **batches** (vectors) using SIMD instructions.
   - Example: 1000 rows processed in a single CPU instruction.

#### **4. Micro-Partitioning Best Practices**
| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Cluster Tables for Pruning** | Cluster tables on frequently filtered columns | `ALTER TABLE my_table CLUSTER BY (date)` |
| **Use Partition Pruning** | Filter on clustered or partitioned columns | `SELECT * FROM my_table WHERE date = '2023-01-01'` |
| **Use Column Pruning** | Select only needed columns | `SELECT col1, col2 FROM my_table` |
| **Avoid Full Table Scans** | Use filters to reduce scanned partitions | `SELECT * FROM my_table WHERE region = 'US'` |
| **Monitor Partition Scans** | Check `PARTITIONS_SCANNED` in `QUERY_HISTORY` | `SELECT PARTITIONS_SCANNED FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` |

### **B. Columnar Storage Deep Dive**

#### **1. What Is Columnar Storage?**
- Data is stored **by column** (not by row).
- Each column is stored in a **separate file** (Parquet format).
- Enables:
  - **Column pruning**: Only read the columns you need.
  - **Compression**: Columns with similar values compress well.
  - **Vectorized execution**: Process columns as vectors.

#### **2. Columnar Storage vs. Row-Based Storage**
| **Feature** | **Columnar Storage** | **Row-Based Storage** |
|-------------|----------------------|-----------------------|
| **Storage Format** | Parquet | CSV, JSON, etc. |
| **Compression** | High (columns with similar values) | Low |
| **Read Performance** | Fast for analytical queries | Fast for transactional queries |
| **Write Performance** | Slower (requires reorganization) | Faster |
| **Column Pruning** | ✅ Yes | ❌ No |
| **Vectorized Execution** | ✅ Yes | ❌ No |
| **Use Case** | Analytics, OLAP | Transactions, OLTP |

#### **3. How Columnar Storage Improves Performance**
1. **Column Pruning**:
   - Only **read the columns** referenced in the query.
   - Example: `SELECT col1, col2 FROM my_table` only reads `col1` and `col2`.

2. **Compression**:
   - Columns with **similar values** (e.g., `date`, `region`) compress well.
   - Example: A column with 10 distinct values compresses better than a column with 1M distinct values.

3. **Vectorized Execution**:
   - Data is processed in **batches** (vectors) using SIMD instructions.
   - Example: 1000 rows processed in a single CPU instruction.

4. **Predicate Pushdown**:
   - Filters are **pushed down** to the storage layer.
   - Example: `WHERE date = '2023-01-01'` is applied during scanning.

#### **4. Columnar Storage Best Practices**
| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Select Only Needed Columns** | Avoid `SELECT *` | `SELECT col1, col2 FROM my_table` |
| **Use Columnar File Formats** | Use Parquet, ORC for external tables | `CREATE FILE FORMAT my_format TYPE = 'PARQUET'` |
| **Compress Data** | Use compression for external tables | `CREATE FILE FORMAT my_format TYPE = 'PARQUET' COMPRESSION = 'SNAPPY'` |
| **Partition Data** | Partition external tables for pruning | `CREATE EXTERNAL TABLE my_table PARTITION BY (date)` |
| **Use Columnar Aggregations** | Use `APPROX_COUNT_DISTINCT`, `APPROX_QUANTILE` | `SELECT APPROX_COUNT_DISTINCT(col1) FROM my_table` |

### **C. Vectorized Execution Deep Dive**

#### **1. What Is Vectorized Execution?**
- Data is processed in **batches** (vectors) of **1000-10000 rows** (configurable).
- Each vector is processed using **SIMD (Single Instruction, Multiple Data)** instructions.
- Enables **high throughput** and **low CPU usage**.

#### **2. How Vectorized Execution Works**
1. **Vector Creation**:
   - Snowflake **reads data in batches** (vectors) from micro-partitions.
2. **Vector Processing**:
   - **SIMD instructions** process the entire vector in a single CPU instruction.
   - Example: Adding 1000 values in a single instruction.
3. **Result Materialization**:
   - Results are **materialized** and returned to the client.

#### **3. Vectorized Execution Best Practices**
| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use Simple Expressions** | Avoid complex expressions in vectors | `SELECT col1 + col2 FROM my_table` (instead of `SELECT complex_function(col1) FROM my_table`) |
| **Use Built-in Functions** | Built-in functions are optimized for vectorization | `SELECT SUM(col1) FROM my_table` |
| **Avoid UDFs** | UDFs disable vectorization | Use built-in functions instead of JavaScript/Python UDFs |
| **Use Approximate Functions** | Approximate functions are vectorized | `SELECT APPROX_COUNT_DISTINCT(col1) FROM my_table` |
| **Monitor Vectorization** | Check `QUERY_PROFILE` for vectorized operators | `SELECT operation, rows_produced FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('...'))` |

### **D. Query Compilation Deep Dive**

#### **1. What Is Query Compilation?**
- Snowflake **compiles SQL queries** into **executable code** (LLVM IR).
- Compilation happens **once per query** (cached for repeated queries).
- Includes:
  - **Parsing**: Validate SQL syntax and semantics.
  - **Optimization**: Apply query transformations (e.g., predicate pushdown).
  - **Code Generation**: Generate executable code (LLVM IR).

#### **2. Query Compilation Phases**
| **Phase** | **Description** | **Duration** | **Optimization Opportunities** |
|-----------|-----------------|--------------|---------------------------------|
| **Parsing** | Validate SQL syntax and semantics | 10-100ms | Use valid SQL, avoid complex expressions |
| **Logical Optimization** | Apply logical transformations (e.g., predicate pushdown) | 10-500ms | Use simple queries, avoid nested subqueries |
| **Physical Optimization** | Choose physical operators (e.g., HashJoin vs. SortMergeJoin) | 10-200ms | Use proper join types, avoid expensive operators |
| **Code Generation** | Generate executable code (LLVM IR) | 10-100ms | Use cached plans where possible |

#### **3. Query Plan Cache**
- Snowflake **caches query plans** for **repeated queries**.
- Cache is **invalidated** if:
  - The **query text changes**.
  - The **schema changes** (e.g., table/column added/dropped).
  - The **data changes** (for some optimizations).
- **Cache TTL**: 24 hours (configurable).

```sql
-- Check if a query used a cached plan
SELECT
    query_id,
    used_cached_result,
    used_cached_plan
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_id = '01a2b3c4-d5e6-78f9';
```

#### **4. Query Compilation Best Practices**
| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use Parameterized Queries** | Parameterized queries can reuse cached plans | `SELECT * FROM my_table WHERE id = ?` |
| **Avoid Dynamic SQL** | Dynamic SQL cannot reuse cached plans | Use prepared statements instead of `EXECUTE IMMEDIATE` |
| **Use Simple Queries** | Simple queries compile faster | `SELECT col1, col2 FROM my_table WHERE date > '2023-01-01'` |
| **Avoid Nested Subqueries** | Nested subqueries are hard to optimize | Use CTEs or joins instead |
| **Monitor Compilation Time** | Check `COMPILATION_TIME` in `QUERY_HISTORY` | `SELECT COMPILATION_TIME FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` |

### **E. Statistics and Metadata**

#### **1. Statistics in Snowflake**
Snowflake **automatically collects statistics** for:
- **Table row counts**.
- **Column min/max values**.
- **Column histograms** (for selective columns).
- **Micro-partition metadata** (min/max for each column).

#### **2. How Statistics Improve Performance**
1. **Partition Pruning**:
   - Snowflake uses **min/max statistics** to skip irrelevant micro-partitions.
   - Example: `WHERE date = '2023-01-01'` skips partitions with `date` outside that range.

2. **Join Optimization**:
   - Snowflake uses **histograms** to estimate join sizes and choose the best join algorithm.

3. **Query Rewriting**:
   - Snowflake uses statistics to **rewrite queries** (e.g., predicate pushdown).

#### **3. Statistics Best Practices**
| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Update Statistics Manually** | Manually update statistics for critical tables | `ALTER TABLE my_table UPDATE STATISTICS` |
| **Use ANALYZE TABLE** | Collect statistics for a table | `ANALYZE TABLE my_table` |
| **Monitor Statistics** | Check `TABLE_STATISTICS` for missing statistics | `SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.TABLE_STATISTICS('MY_TABLE'))` |
| **Use Clustering** | Clustering improves partition pruning | `ALTER TABLE my_table CLUSTER BY (date)` |
| **Avoid Skewed Data** | Skewed data can lead to uneven partition sizes | Use `RECLUSTER` to rebalance data |

### **F. Workload Management**

#### **1. Resource Monitors**
- **Resource Monitors** allow you to **set limits** on **credit usage** for warehouses, users, or roles.
- **Credit Quotas**: Limit the number of credits used per day/month.
- **Warehouse Limits**: Limit the number of warehouses a user/role can use.

```sql
-- Create a resource monitor
CREATE RESOURCE MONITOR my_monitor
  WITH CREDIT_QUOTA = 1000  -- 1000 credits per day
  FREQUENCY = DAILY
  START_TIMESTAMP = DATEADD('day', -1, CURRENT_TIMESTAMP())
  END_TIMESTAMP = DATEADD('day', 1, CURRENT_TIMESTAMP())
  NOTIFY_USERS = (SELECT user_name FROM SNOWFLAKE.ACCOUNT_USAGE.USERS);

-- Assign resource monitor to a warehouse
ALTER WAREHOUSE MY_WH SET RESOURCE_MONITOR = my_monitor;

-- Assign resource monitor to a user
ALTER USER my_user SET RESOURCE_MONITOR = my_monitor;

-- Assign resource monitor to a role
ALTER ROLE my_role SET RESOURCE_MONITOR = my_monitor;
```

#### **2. Query Queues**
- **Query Queues** manage **concurrent queries** in a warehouse.
- **Priority**: Queries can be **prioritized** (HIGH, MEDIUM, LOW).
- **Timeout**: Queries can be **timed out** if they run too long.

```sql
-- Set query priority
ALTER WAREHOUSE MY_WH SET QUERY_PRIORITY = 'HIGH' FOR CURRENT_SESSION();

-- Set statement timeout
ALTER WAREHOUSE MY_WH SET STATEMENT_TIMEOUT_IN_SECONDS = 300;

-- Set query queue timeout
ALTER WAREHOUSE MY_WH SET QUERY_QUEUE_TIMEOUT_IN_SECONDS = 60;
```

#### **3. Multi-Cluster Warehouses**
- **Multi-Cluster Warehouses** can **scale out** to handle **concurrent queries**.
- **Max Cluster Count**: Number of clusters (1-10).
- **Min Cluster Count**: Minimum number of clusters (1-10).
- **Scaling Policy**: `STANDARD` (aggressive) or `ECONOMY` (conservative).

```sql
-- Create a multi-cluster warehouse
CREATE WAREHOUSE MY_MC_WH
  WAREHOUSE_SIZE = 'MEDIUM'
  MAX_CLUSTER_COUNT = 4
  MIN_CLUSTER_COUNT = 1
  SCALING_POLICY = 'STANDARD'
  AUTO_SUSPEND = 300;

-- Monitor multi-cluster warehouse
SELECT
    warehouse_name,
    cluster_number,
    start_time,
    end_time,
    running_queries,
    queued_queries
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE
    warehouse_name = 'MY_MC_WH'
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP());
```

## **8. Case Studies: Real-World Performance Tuning**

### **A. Case Study 1: Slow Dashboard Queries**

#### **Problem**
- A **dashboard** with 10 queries takes **30 seconds** to load.
- Queries scan **100GB+** of data each.
- **User complaints** about slow performance.

#### **Diagnosis**
1. **Identify Slow Queries**:
   ```sql
   SELECT
       query_id,
       query_text,
       execution_time,
       bytes_scanned,
       warehouse_size
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       query_text LIKE '%dashboard%'
       AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY
       execution_time DESC;
   ```
   - **Finding**: Queries scan **100GB+** and take **5-10 seconds** each.

2. **Check Query Profiles**:
   ```sql
   SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('01a2b3c4-d5e6-78f9'));
   ```
   - **Finding**: Most time is spent in **`TableScan`** (90% of execution time).

3. **Check Table Clustering**:
   ```sql
   SELECT
       table_name,
       clustering_information
   FROM
       INFORMATION_SCHEMA.TABLES
   WHERE
       table_name IN ('SALES', 'CUSTOMERS', 'PRODUCTS');
   ```
   - **Finding**: Tables are **not clustered**.

#### **Solution**
1. **Add Clustering**:
   ```sql
   ALTER TABLE SALES CLUSTER BY (date, region);
   ALTER TABLE CUSTOMERS CLUSTER BY (customer_id);
   ALTER TABLE PRODUCTS CLUSTER BY (product_id);
   ```

2. **Add Filters to Queries**:
   - Original:
     ```sql
     SELECT * FROM SALES JOIN CUSTOMERS ON SALES.customer_id = CUSTOMERS.customer_id;
     ```
   - Optimized:
     ```sql
     SELECT * FROM SALES
     JOIN CUSTOMERS ON SALES.customer_id = CUSTOMERS.customer_id
     WHERE SALES.date > CURRENT_DATE() - 30;
     ```

3. **Use Materialized Views**:
   ```sql
   CREATE MATERIALIZED VIEW DASHBOARD_SALES AS
   SELECT
       date,
       region,
       product_category,
       SUM(sales) AS total_sales,
       COUNT(*) AS transaction_count
   FROM
       SALES JOIN PRODUCTS ON SALES.product_id = PRODUCTS.product_id
   WHERE
       date > CURRENT_DATE() - 30
   GROUP BY
       date, region, product_category;

   CREATE MATERIALIZED VIEW DASHBOARD_CUSTOMERS AS
   SELECT
       region,
       COUNT(*) AS customer_count,
       SUM(total_spend) AS total_spend
   FROM
       CUSTOMERS
   GROUP BY
       region;
   ```

4. **Use Result Caching**:
   - Enable result caching for dashboard queries:
     ```sql
     ALTER SESSION SET USE_CACHED_RESULTS = TRUE;
     ```

#### **Results**
| **Metric** | **Before** | **After** | **Improvement** |
|------------|------------|-----------|-----------------|
| **Dashboard Load Time** | 30 seconds | 3 seconds | **10x faster** |
| **Bytes Scanned** | 100GB+ | 10GB | **90% reduction** |
| **Credit Usage** | 50 credits | 5 credits | **90% reduction** |
| **User Satisfaction** | ❌ Poor | ✅ Excellent | **Significant improvement** |

### **B. Case Study 2: High Credit Usage**

#### **Problem**
- **Monthly credit usage** increased from **10,000** to **50,000** credits.
- **No obvious increase** in query volume or data size.
- **Budget overruns** causing cost concerns.

#### **Diagnosis**
1. **Check Credit Usage by Warehouse**:
   ```sql
   SELECT
       warehouse_name,
       SUM(credits_used) AS total_credits,
       COUNT(*) AS query_count,
       AVG(credits_used) AS avg_credits_per_query
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       start_time > DATEADD('month', -1, CURRENT_TIMESTAMP())
   GROUP BY
       warehouse_name
   ORDER BY
       total_credits DESC;
   ```
   - **Finding**: **ETL_WH** warehouse used **40,000 credits** (80% of total).

2. **Check Credit Usage by Query**:
   ```sql
   SELECT
       query_text,
       SUM(credits_used) AS total_credits,
       COUNT(*) AS query_count,
       AVG(execution_time) AS avg_execution_time
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       warehouse_name = 'ETL_WH'
       AND start_time > DATEADD('month', -1, CURRENT_TIMESTAMP())
   GROUP BY
       query_text
   ORDER BY
       total_credits DESC;
   ```
   - **Finding**: A **daily ETL job** used **30,000 credits** (75% of ETL_WH usage).

3. **Check Query Profile**:
   ```sql
   SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('01a2b3c4-d5e6-78f9'));
   ```
   - **Finding**: Query **spilled to remote storage** (10GB spilled).

#### **Solution**
1. **Increase Warehouse Size**:
   ```sql
   ALTER WAREHOUSE ETL_WH SET WAREHOUSE_SIZE = 'X-LARGE';
   ```

2. **Optimize ETL Query**:
   - Original:
     ```sql
     INSERT INTO TARGET_TABLE
     SELECT * FROM SOURCE_TABLE
     JOIN LOOKUP_TABLE ON SOURCE_TABLE.id = LOOKUP_TABLE.id;
     ```
   - Optimized:
     ```sql
     INSERT INTO TARGET_TABLE
     SELECT
         s.col1, s.col2, l.col3
     FROM
         SOURCE_TABLE s
     JOIN LOOKUP_TABLE l ON s.id = l.id
     WHERE
         s.date > CURRENT_DATE() - 1;
     ```

3. **Use Batch Processing**:
   - Break the ETL job into **smaller batches**:
     ```sql
     -- Process data in batches of 1 day
     FOR day IN (SELECT DISTINCT date FROM SOURCE_TABLE WHERE date > CURRENT_DATE() - 30) DO
         INSERT INTO TARGET_TABLE
         SELECT
             s.col1, s.col2, l.col3
         FROM
             SOURCE_TABLE s
         JOIN LOOKUP_TABLE l ON s.id = l.id
         WHERE
             s.date = day.date;
     END FOR;
     ```

4. **Use Materialized Views**:
   ```sql
   CREATE MATERIALIZED VIEW ETL_SOURCE AS
   SELECT
       id, col1, col2, date
   FROM
       SOURCE_TABLE
   WHERE
       date > CURRENT_DATE() - 30;
   ```

5. **Set Query Timeout**:
   ```sql
   ALTER WAREHOUSE ETL_WH SET STATEMENT_TIMEOUT_IN_SECONDS = 300;
   ```

#### **Results**
| **Metric** | **Before** | **After** | **Improvement** |
|------------|------------|-----------|-----------------|
| **Monthly Credit Usage** | 50,000 | 15,000 | **70% reduction** |
| **ETL Job Credit Usage** | 30,000 | 5,000 | **83% reduction** |
| **Spill to Remote** | 10GB | 0 | **Eliminated** |
| **ETL Job Duration** | 2 hours | 30 minutes | **4x faster** |

### **C. Case Study 3: Slow JOIN Performance**

#### **Problem**
- A **JOIN query** between **SALES** (100M rows) and **CUSTOMERS** (10M rows) takes **5 minutes**.
- **User complaints** about slow report generation.

#### **Diagnosis**
1. **Check Query Profile**:
   ```sql
   SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('01a2b3c4-d5e6-78f9'));
   ```
   - **Finding**: **HashJoin** step takes **4 minutes** (80% of total time).

2. **Check Join Size**:
   ```sql
   SELECT
       COUNT(*) AS join_size
   FROM
       SALES JOIN CUSTOMERS ON SALES.customer_id = CUSTOMERS.customer_id;
   ```
   - **Finding**: Join produces **100M rows** (1:1 join).

3. **Check Join Columns**:
   ```sql
   SELECT
       DISTINCT customer_id
   FROM
       SALES;
   ```
   - **Finding**: **10M distinct `customer_id`** in SALES (matches CUSTOMERS table size).

#### **Solution**
1. **Use Broadcast Join**:
   - Snowflake **automatically uses broadcast join** for small tables.
   - Ensure **CUSTOMERS** is the smaller table:
     ```sql
     SELECT * FROM SALES s JOIN CUSTOMERS c ON s.customer_id = c.customer_id;
     ```

2. **Filter Before Joining**:
   - Original:
     ```sql
     SELECT * FROM SALES JOIN CUSTOMERS ON SALES.customer_id = CUSTOMERS.customer_id
     WHERE SALES.date > '2023-01-01';
     ```
   - Optimized:
     ```sql
     SELECT * FROM
         (SELECT * FROM SALES WHERE date > '2023-01-01') s
     JOIN CUSTOMERS c ON s.customer_id = c.customer_id;
     ```

3. **Use Semi-Join**:
   - If you only need rows from SALES with matches in CUSTOMERS:
     ```sql
     SELECT * FROM SALES
     WHERE customer_id IN (SELECT customer_id FROM CUSTOMERS);
     ```

4. **Cluster Tables on Join Columns**:
   ```sql
   ALTER TABLE SALES CLUSTER BY (customer_id);
   ALTER TABLE CUSTOMERS CLUSTER BY (customer_id);
   ```

5. **Use Materialized View**:
   ```sql
   CREATE MATERIALIZED VIEW SALES_WITH_CUSTOMERS AS
   SELECT
       s.*,
       c.name,
       c.region
   FROM
       SALES s
   JOIN CUSTOMERS c ON s.customer_id = c.customer_id;
   ```

#### **Results**
| **Metric** | **Before** | **After** | **Improvement** |
|------------|------------|-----------|-----------------|
| **Join Query Time** | 5 minutes | 30 seconds | **10x faster** |
| **Bytes Scanned
