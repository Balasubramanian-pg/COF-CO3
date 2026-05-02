# **Snowflake: Structured, Semi-Structured, and Unstructured Data Handling**

*Production-Grade Technical Deep Dive for Platform Engineers, SREs, and Architects*

---

## **1. Mermaid Architecture & Execution Flow Diagram**

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#ffd700', 'edgeLabelBackground':'#fff'}}}%%
flowchart TD
    subgraph "Ingestion Layer"
        A[Structured: CSV/TSV/Parquet] -->|Bulk Load| B[COPY INTO]
        C[Semi-Structured: JSON/Parquet/Avro/XML] -->|Bulk Load| B
        D[Unstructured: PDF/Log/Images] -->|Stage Load| E[External Stage\n(S3/Azure Blob/GCS)]
        E -->|File Metadata| F[File Format Object]
    end

    subgraph "Storage Layer"
        B -->|Columnar Storage| G[Micro-Partitions\n(16-128MB)]
        C -->|VARIANT Type| G
        D -->|BLOB Type| G
        G -->|Metadata| H[Metadata Store\n(Snowflake Internal)]
    end

    subgraph "Processing Layer"
        G -->|Query Execution| I[Execution Engine\n(Vectorized)]
        I -->|Pruning| J[Partition Pruning\nColumnar Pruning]
        I -->|Parsing| K[JSON/XML Parsing\n(FastPath)]
        I -->|UDFs| L[Java/JS/Python UDFs]
        K -->|Extraction| M[FLATTEN\nJSON_EXTRACT_PATH_TEXT]
        L -->|Transformation| N[Custom Logic]
    end

    subgraph "Failure Paths"
        I -->|OOM| O[Spill-to-Disk\n(Local SSD)]
        O -->|Spill Overflow| P[Query Abort\n(Error: 2003)]
        K -->|Malformed JSON| Q[Error: 1204\nON_ERROR=CONTINUE]
        D -->|Unsupported Format| R[Error: 100072\nPermission Denied]
    end

    subgraph "Output Layer"
        I -->|Result Set| S[Structured Output\n(Tables/Views)]
        M -->|Structured Output| S
        N -->|Structured Output| S
        D -->|BLOB Output| T[Unstructured Output\n(External Stage)]
    end

    style A fill:#9f9,stroke:#333
    style C fill:#ff9,stroke:#333
    style D fill:#f99,stroke:#333
    style G fill:#bbf,stroke:#333
    style I fill:#f96,stroke:#333
    style P fill:#f99,stroke:#333
    style Q fill:#ff9,stroke:#333
    style R fill:#f99,stroke:#333
```

---


## **2. Execution Internals & Transactional Boundaries**


### **2.1 Data Type Classification & Storage Internals**


| **Data Type**              | **Snowflake Representation**        | **Storage Format**     | **Internal Encoding**       | **Query Access Method**             | **Transactional Guarantees**          |
| -------------------------- | ----------------------------------- | ---------------------- | --------------------------- | ----------------------------------- | ------------------------------------- |
| **Structured**             | Native SQL Types (`INT`, `VARCHAR`) | Columnar (Parquet)     | **Snappy/Zstd Compression** | Direct column access                | Full ACID (MVCC)                      |
| **Semi-Structured**        | `VARIANT`                           | Columnar (Parquet)     | **Binary JSON (BSON-like)** | `JSON_EXTRACT_*`, `FLATTEN`         | Full ACID (stored as BLOB + metadata) |
| **Unstructured**           | `BLOB`, `TEXT`                      | Raw bytes (no parsing) | **Base64/UTF-8**            | `BLOB_*`, `TEXT_*` functions        | **No ACID** (treated as opaque bytes) |
| **Semi-Structured (Avro)** | `VARIANT`                           | Columnar (Parquet)     | **Avro Binary**             | `AVRO_EXTRACT_*` (custom UDF)       | Full ACID                             |
| **Semi-Structured (XML)**  | `VARIANT` or `TEXT`                 | Raw or Parsed          | **UTF-8**                   | `XMLGET`, `XMLEXTRACT` (custom UDF) | Full ACID (if stored as `VARIANT`)    |



### **2.2 Ingestion Internals by Data Type**

#### **2.2.1 Structured Data (CSV/TSV/Parquet)**

- **Ingestion Path**:  
`COPY INTO <table> FROM {internalStage|externalStage} [FILE_FORMAT = (TYPE = 'CSV' | 'PARQUET')]`
- **Execution Flow**:
  1. **Stage Scanning**:
    - **External Stages**: Pre-signed URLs (15-min TTL) → **HTTPS GET** → **GCS/S3/Azure Blob**.
    - **Internal Stages**: Direct Snowflake-managed storage access.
  2. **File Chunking**:
    - Files split into **16MB-1GB chunks** (default: **128MB**).
    - **Parallelism**: 1 thread per chunk (max **8 threads/core**).
  3. **Parsing & Validation**:
    - **CSV/TSV**: Row-by-row parsing with `FIELD_OPTIONALLY_ENCLOSED_BY`.
    - **Parquet**: Columnar decoding (Snappy/Zstd).
    - **Validation**: `VALIDATION_MODE = RETURN_ERRORS` (default: `RETURN_NONE`).
  4. **Type Coercion**:
    - Automatic casting (e.g., `'123'` → `INT`).
    - **Error Handling**: `ON_ERROR = CONTINUE | ABORT_STATEMENT | SKIP_FILE`.
  5. **Transaction Boundaries**:
    - **Atomicity**: All-or-nothing per `COPY` statement.
    - **Isolation**: **Snapshot Isolation** (reads see pre-COPY state until commit).
- **Credit Math**:
  - **Cloud Services**: `$0.005/TB scanned` (external stages).
  - **Compute**: `1 credit = 1 core-second` (warehouse-dependent).
  - **Example**: 1TB CSV load on `X-LARGE` (4 cores) in 10 min:  
  `4 cores * 600 sec / 3600 = 0.67 credits (compute) + 5 credits (cloud) = 5.67 credits total`.


#### **2.2.2 Semi-Structured Data (JSON/Parquet/Avro/XML)**

- **Ingestion Path**:
  - **JSON/Parquet/Avro**: `COPY INTO <table> FROM @stage FILE_FORMAT = (TYPE = 'JSON' | 'PARQUET' | 'AVRO')`
  - **XML**: Requires **custom UDF** (e.g., `XML_TO_VARIANT`).
- **Execution Flow**:
  1. **Stage Scanning**: Same as structured.
  2. **File Parsing**:
    - **JSON**: Parsed into `VARIANT` (binary BSON-like format).
    - **Parquet/Avro**: Decoded into `VARIANT` or native types (if schema enforced).
    - **XML**: Parsed via UDF (e.g., `XMLGET`).
  3. **Schema Inference**:
    - **Infer Schema**: `INFER_SCHEMA = TRUE` (default: `FALSE`).
    - **Schema Evolution**: New fields are **automatically added** to `VARIANT`.
  4. **Type Handling**:
    - **JSON**: Nested objects → `VARIANT`, arrays → `ARRAY`.
    - **Avro**: Complex types (e.g., `RECORD`, `UNION`) → `VARIANT`.
  5. **Validation**:
    - **JSON**: `STRICT = TRUE` (fails on malformed JSON).
    - **Parquet/Avro**: Schema validation (if `ENFORCE_SCHEMA = TRUE`).
  6. **Transaction Boundaries**:
    - **Atomicity**: Per-file (if `ON_ERROR = SKIP_FILE`).
    - **Isolation**: **Snapshot Isolation** (metadata updates are atomic).
- **Credit Math**:
  - **JSON Parsing Overhead**: **+10-20% credits** (vs. structured).
  - **Parquet/Avro**: **+5% credits** (columnar decoding).
  - **Example**: 1TB JSON load on `X-LARGE`:  
  `4 * 600 / 3600 = 0.67 (compute) + 5 (cloud) + 1.34 (parsing) = 7.01 credits`.


#### **2.2.3 Unstructured Data (PDF/Logs/Images)**

- **Ingestion Path**:
  - **BLOB**: `COPY INTO <table> FROM @stage FILE_FORMAT = (TYPE = 'BINARY')`
  - **TEXT**: `COPY INTO <table> FROM @stage FILE_FORMAT = (TYPE = 'TEXT')`
- **Execution Flow**:
  1. **Stage Scanning**: Same as structured.
  2. **No Parsing**:
    - **BLOB**: Stored as raw bytes (no validation).
    - **TEXT**: Stored as `VARCHAR` (max **16MB/row**).
  3. **Metadata Extraction**:
    - `METADATA$FILE_NAME`, `METADATA$FILE_SIZE`, `METADATA$FILE_LAST_MODIFIED`.
  4. **Transaction Boundaries**:
    - **Atomicity**: Per-file (no row-level atomicity).
    - **Isolation**: **No ACID** (BLOB/TEXT are opaque to Snowflake).
- **Credit Math**:
  - **No Parsing Overhead**: Same as structured for **BLOB**.
  - **TEXT**: **+5% credits** (UTF-8 validation).
  - **Example**: 1TB PDF load on `X-LARGE`:  
  `4 * 600 / 3600 + 5 = 5.67 credits`.


### **2.3 Transactional Semantics by Data Type**


| **Operation**   | **Structured**              | **Semi-Structured**         | **Unstructured**            |
| --------------- | --------------------------- | --------------------------- | --------------------------- |
| **Atomicity**   | Statement-level             | File-level (if `SKIP_FILE`) | File-level                  |
| **Consistency** | Snapshot Isolation          | Snapshot Isolation          | None (opaque bytes)         |
| **Isolation**   | MVCC                        | MVCC                        | None                        |
| **Durability**  | Metadata + Data (S3-backed) | Metadata + Data (S3-backed) | Metadata + Data (S3-backed) |
| **Rollback**    | Full (metadata + data)      | Full (metadata + data)      | Metadata only               |
| **Time Travel** | 90 days                     | 90 days                     | 90 days (metadata only)     |



### **2.4 Error Handling & Recovery Paths**


| **Error Code** | **Data Type**   | **Root Cause**                           | **Recovery Path**                               | **Credit Impact**      |
| -------------- | --------------- | ---------------------------------------- | ----------------------------------------------- | ---------------------- |
| `1204`         | Semi-Structured | Malformed JSON/XML                       | `ON_ERROR = CONTINUE` + DLQ routing.            | +2% (retry overhead)   |
| `100072`       | All             | Permission denied (stage/table)          | Grant `READ` on stage, `INSERT` on table.       | None                   |
| `100083`       | Structured      | Column count mismatch                    | Use `TRUNCATECOLUMNS = TRUE` or align source.   | None                   |
| `1219`         | Semi-Structured | File format mismatch (e.g., CSV as JSON) | Validate `FILE_FORMAT`.                         | None                   |
| `2003`         | All             | Memory limit exceeded                    | Increase warehouse size or optimize query.      | +2x (spill overhead)   |
| `1049`         | All             | Disk full (spill overflow)               | Increase warehouse size or reduce data scanned. | +3x (abort + retry)    |
| `2012`         | All             | Stage I/O error (network/permissions)    | Retry with exponential backoff.                 | +1.5x (retry overhead) |




## **3. Parameter/Configuration Deep Dive**


### **3.1 Structured Data Parameters**


| **Parameter**        | **Internal Behavior**                                                      | **Performance Impact**                                                               | **Compliance/Edge Cases**                                                                | **Production Default** |
| -------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------- | ---------------------- |
| `FILE_FORMAT` (TYPE) | `CSV`, `TSV`, `PARQUET`, `JSON`, `AVRO`, `XML`.                            | `PARQUET` = **+10% faster** than CSV (columnar). `JSON` = **+15% slower** (parsing). | `XML` requires custom UDF. `PARQUET` requires `ENFORCE_SCHEMA = TRUE` for strict typing. | `CSV`                  |
| `FIELD_DELIMITER`    | Delimiter for CSV/TSV (default: `,`).                                      | No impact.                                                                           | `TAB` for TSV. `PIPE` for PSV.                                                           | `,`                    |
| `SKIP_HEADER`        | Skip N rows (default: `0`).                                                | No impact.                                                                           | Set to `1` for header rows.                                                              | `0`                    |
| `NULL_IF`            | Treat strings as `NULL` (e.g., `NULL_IF = ('NULL', 'N/A')`).               | Reduces **storage by ~30%** (if many NULLs).                                         | Case-sensitive.                                                                          | `()` (empty)           |
| `COMPRESSION`        | `AUTO`, `GZIP`, `BZ2`, `BROTLI`, `ZSTD`, `DEFLATE`, `RAW_DEFLATE`, `NONE`. | `ZSTD` = **best compression** (+20% smaller files). `GZIP` = **+10% CPU overhead**.  | `AUTO` selects `ZSTD` for Parquet, `GZIP` for CSV.                                       | `AUTO`                 |
| `ENFORCE_SCHEMA`     | Reject rows with extra/missing columns.                                    | **+5% credits** (validation overhead).                                               | Required for **strict schema enforcement**.                                              | `FALSE`                |
| `TRUNCATECOLUMNS`    | Truncate strings to target column length.                                  | No impact.                                                                           | **Silent truncation** (use `VALIDATION_MODE=RETURN_ERRORS` to detect).                   | `FALSE`                |
| `VALIDATION_MODE`    | `RETURN_ERRORS`, `RETURN_1_ROWS`, `RETURN_ALL_ERRORS`.                     | `RETURN_ALL_ERRORS` = **+10% memory** (buffers errors).                              | **Production**: Use `RETURN_ERRORS` + DLQ.                                               | `RETURN_ERRORS`        |
| `ON_ERROR`           | `ABORT_STATEMENT`, `CONTINUE`, `SKIP_FILE`, `SKIP_FILE_<n>`.               | `CONTINUE` = **+2% credits** (partial load).                                         | `SKIP_FILE` logs to `COPY_HISTORY` but **no DLQ**.                                       | `ABORT_STATEMENT`      |



### **3.2 Semi-Structured Data Parameters**


| **Parameter**                 | **Internal Behavior**                                               | **Performance Impact**                                                               | **Compliance/Edge Cases**                                              | **Production Default**  |
| ----------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------- | ----------------------- |
| `STRIP_OUTER_ARRAY`           | Flatten outer JSON array (e.g., `[{...}, {...}]` → rows).           | **+10% faster** (avoids nested `FLATTEN`).                                           | Required for **Newline-Delimited JSON (NDJSON)**.                      | `FALSE`                 |
| `IGNORE_UTF8_ERRORS`          | Skip invalid UTF-8 sequences.                                       | No impact.                                                                           | **Silent corruption** (use `VALIDATION_MODE=RETURN_ERRORS` to detect). | `FALSE`                 |
| `INFER_SCHEMA`                | Automatically detect schema from first N files (default: `100`).    | **+5-10% credits** (schema inference overhead).                                      | **Schema drift**: New fields are **automatically added** to `VARIANT`. | `FALSE`                 |
| `ENFORCE_SCHEMA`              | Reject files with schema mismatches.                                | **+5% credits** (validation overhead).                                               | Required for **strict typing** (e.g., `INT` vs. `STRING`).             | `FALSE`                 |
| `FILE_FORMAT` (TYPE=JSON)     | `STRICT` (fail on malformed JSON), `RELAXED` (best-effort parsing). | `STRICT` = **+10% credits** (validation). `RELAXED` = **faster but risky**.          | **Production**: Use `STRICT` + DLQ.                                    | `RELAXED`               |
| `BINARY_FORMAT`               | `UBJSON`, `BSON`, `AVRO`, `PARQUET`.                                | `UBJSON` = **+20% faster** than JSON (binary). `AVRO` = **+5% faster** than Parquet. | `AVRO` requires `ENFORCE_SCHEMA = TRUE`.                               | `JSON` (for JSON files) |
| `GENERATE_COLUMN_DESCRIPTION` | Generate column descriptions from JSON keys.                        | **+5% credits** (metadata generation).                                               | Useful for **self-describing data**.                                   | `FALSE`                 |



### **3.3 Unstructured Data Parameters**


| **Parameter**               | **Internal Behavior**                       | **Performance Impact**                       | **Compliance/Edge Cases**                       | **Production Default** |
| --------------------------- | ------------------------------------------- | -------------------------------------------- | ----------------------------------------------- | ---------------------- |
| `FILE_FORMAT` (TYPE=BINARY) | Treat files as raw bytes.                   | **No parsing overhead** (fastest ingestion). | **No validation** (use for PDF, images, etc.).  | `BINARY`               |
| `FILE_FORMAT` (TYPE=TEXT)   | Treat files as UTF-8 strings.               | **+5% credits** (UTF-8 validation).          | **Max 16MB/row** (use `BLOB` for larger files). | `TEXT`                 |
| `PRESERVE_SPACE`            | Preserve leading/trailing spaces in `TEXT`. | No impact.                                   | **Default**: `FALSE` (trims spaces).            | `FALSE`                |
| `COMPRESSION`               | `AUTO`, `GZIP`, `BZ2`, `ZSTD`, `NONE`.      | `ZSTD` = **best for text** (30% smaller).    | `NONE` for pre-compressed files (e.g., `.gz`).  | `AUTO`                 |
| `BINARY_SIZE_LIMIT`         | Max size for `BLOB` (default: **16MB**).    | **No impact** (enforced at load time).       | **Workaround**: Split large files into chunks.  | `16777216` (16MB)      |




## **4. Performance & Resource Implications**


### **4.1 Memory & CPU Behavior by Data Type**


| **Data Type**                 | **Memory Allocation**     | **CPU Intensity** | **Spill Trigger**    | **Spill Overhead** | **Vectorized?**    |
| ----------------------------- | ------------------------- | ----------------- | -------------------- | ------------------ | ------------------ |
| **Structured (CSV)**          | Low (row-based)           | Medium (parsing)  | 80% heap utilization | +2x credits        | Yes                |
| **Structured (Parquet)**      | Low (columnar)            | Low (pre-decoded) | 80% heap utilization | +1.5x credits      | Yes                |
| **Semi-Structured (JSON)**    | High (nested objects)     | High (parsing)    | 70% heap utilization | +2.5x credits      | Partial (FastPath) |
| **Semi-Structured (Parquet)** | Medium (columnar)         | Low (pre-decoded) | 80% heap utilization | +1.5x credits      | Yes                |
| **Unstructured (BLOB)**       | Low (raw bytes)           | Low (no parsing)  | N/A (no spill)       | N/A                | No                 |
| **Unstructured (TEXT)**       | Medium (UTF-8 validation) | Low               | 80% heap utilization | +2x credits        | No                 |



### **4.2 I/O Patterns & Stage Interactions**


| **Operation**              | **Structured**       | **Semi-Structured**       | **Unstructured**        | **Network Overhead**          |
| -------------------------- | -------------------- | ------------------------- | ----------------------- | ----------------------------- |
| **Bulk Load (COPY)**       | Parallel chunk reads | Parallel chunk reads      | Parallel chunk reads    | **15-min pre-signed URL TTL** |
| **Query (SELECT)**         | Columnar scans       | `VARIANT` scans + parsing | Full scans (no pruning) | **None** (internal)           |
| **FLATTEN**                | N/A                  | Recursive descent         | N/A                     | **None**                      |
| **JSON_EXTRACT_PATH_TEXT** | N/A                  | Path-based extraction     | N/A                     | **None**                      |
| **External Function**      | N/A                  | UDF (JS/Python)           | UDF (JS/Python)         | **HTTP egress costs**         |



### **4.3 Spill-to-Disk Triggers & Credit Math**


| **Scenario**                        | **Spill Threshold** | **Credit Overhead** | **Recovery**            |
| ----------------------------------- | ------------------- | ------------------- | ----------------------- |
| **Structured (CSV) Join**           | 80% heap            | +2x credits         | Automatic (transparent) |
| **Semi-Structured (JSON) Parsing**  | 70% heap            | +2.5x credits       | Automatic               |
| **Unstructured (TEXT) Aggregation** | 80% heap            | +2x credits         | Automatic               |
| **Structured (Parquet) Sort**       | 85% heap            | +1.8x credits       | Automatic               |
| **Semi-Structured (JSON) FLATTEN**  | 75% heap            | +3x credits         | Manual retry required   |


**Spill Credit Formula**:

```
Spill Overhead (credits) =
  (Spilled Data Size in GB / Warehouse Memory in GB) *
  2 *
  Query Duration (sec) /
  3600
```

**Example**:

- Warehouse: `X-LARGE` (128GB RAM).
- Spilled Data: 200GB (JSON parsing).
- Query Duration: 600 sec.
- **Overhead**: `(200 / 128) * 2 * (600 / 3600) = 0.52 credits`.


### **4.4 Warehouse Sizing Rules by Data Type**


| **Data Type**                 | **Recommended Warehouse** | **Max Data Scanned** | **Concurrency** | **Credit Math**                      |
| ----------------------------- | ------------------------- | -------------------- | --------------- | ------------------------------------ |
| **Structured (CSV)**          | `MEDIUM`                  | 1TB                  | 4               | `1 credit = 1 core-second`           |
| **Structured (Parquet)**      | `LARGE`                   | 5TB                  | 8               | `1 credit = 1 core-second`           |
| **Semi-Structured (JSON)**    | `X-LARGE`                 | 2TB                  | 16              | `1 credit = 1.2 core-seconds` (+20%) |
| **Semi-Structured (Parquet)** | `LARGE`                   | 5TB                  | 8               | `1 credit = 1.1 core-seconds` (+10%) |
| **Unstructured (BLOB)**       | `MEDIUM`                  | 10TB                 | 4               | `1 credit = 1 core-second`           |
| **Unstructured (TEXT)**       | `LARGE`                   | 5TB                  | 8               | `1 credit = 1.1 core-seconds` (+10%) |



### **4.5 Concurrency Scaling & Multi-Cluster Impact**


| **Workload Type**               | **Recommended Warehouse** | **Max Clusters** | **Scaling Strategy** | **Credit Overhead** |
| ------------------------------- | ------------------------- | ---------------- | -------------------- | ------------------- |
| **Structured ETL**              | `X-LARGE`                 | 4                | `MULTI_CLUSTER=TRUE` | +10% per cluster    |
| **Semi-Structured (JSON) Load** | `2X-LARGE`                | 2                | `MULTI_CLUSTER=TRUE` | +15% per cluster    |
| **Unstructured (PDF) Load**     | `LARGE`                   | 1                | Single-cluster       | 0%                  |
| **Ad-Hoc Queries (JSON)**       | `MEDIUM`                  | 1                | `AUTO_RESUME=TRUE`   | +5% (idle overhead) |
| **Real-Time (JSON Streams)**    | `3X-LARGE`                | 3                | `MULTI_CLUSTER=TRUE` | +20% per cluster    |




## **5. Monitoring, Observability & Troubleshooting**


### **5.1 Key Monitoring Views by Data Type**


| **View**                                   | **Structured** | **Semi-Structured** | **Unstructured** | **Critical Columns**                                            |
| ------------------------------------------ | -------------- | ------------------- | ---------------- | --------------------------------------------------------------- |
| `QUERY_HISTORY`                            | ✅              | ✅                   | ✅                | `QUERY_ID`, `CREDITS_USED`, `MEMORY_USAGE`, `ERROR_CODE`        |
| `COPY_HISTORY`                             | ✅              | ✅                   | ✅                | `FILE_NAME`, `ROW_PARSED`, `ERROR_COUNT`, `FIRST_ERROR_MESSAGE` |
| `STAGE_FILE_METADATA`                      | ✅              | ✅                   | ✅                | `FILE_NAME`, `FILE_SIZE`, `LAST_MODIFIED`                       |
| `INFORMATION_SCHEMA.TABLE_STORAGE_METRICS` | ✅              | ✅                   | ❌                | `TABLE_NAME`, `STORAGE_BYTES`, `ROW_COUNT`                      |
| `INFORMATION_SCHEMA.VARIANT_SCHEMA`        | ❌              | ✅                   | ❌                | `TABLE_NAME`, `COLUMN_NAME`, `SCHEMA` (for `VARIANT` columns)   |
| `ACCOUNT_USAGE.QUERY_HISTORY`              | ✅              | ✅                   | ✅                | `USER_NAME`, `WAREHOUSE_NAME`, `EXECUTION_STATUS`               |
| `INFORMATION_SCHEMA.EXTERNAL_TABLE_FILES`  | ❌              | ✅                   | ✅                | `TABLE_NAME`, `FILE_NAME`, `FILE_SIZE`                          |



### **5.2 Production-Grade Monitoring Queries**


#### **5.2.1 Structured Data Monitoring**

```sql
-- Top 10 longest-running COPY commands (last 24h)
SELECT
    QUERY_ID,
    USER_NAME,
    WAREHOUSE_NAME,
    QUERY_TEXT,
    TOTAL_ELAPSED_TIME / 1000 AS duration_sec,
    CREDITS_USED,
    ROWS_LOADED,
    ERROR_COUNT,
    FIRST_ERROR_MESSAGE
FROM
    TABLE(INFORMATION_SCHEMA.QUERY_HISTORY(
        DATEADD('HOUR', -24, CURRENT_TIMESTAMP()),
        CURRENT_TIMESTAMP()
    ))
WHERE
    QUERY_TEXT LIKE '%COPY INTO%'
    AND QUERY_TEXT LIKE '%CSV%'
ORDER BY
    TOTAL_ELAPSED_TIME DESC
LIMIT 10;

-- Storage growth by table (structured)
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    PARTITION_ID,
    STORAGE_BYTES / POWER(1024, 3) AS storage_gb,
    ROW_COUNT
FROM
    INFORMATION_SCHEMA.TABLE_STORAGE_METRICS
WHERE
    TABLE_NAME IN (SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_TYPE = 'BASE TABLE')
ORDER BY
    STORAGE_BYTES DESC;
```


#### **5.2.2 Semi-Structured Data Monitoring**

```sql
-- JSON parsing errors (last 7 days)
SELECT
    FILE_NAME,
    ERROR_COUNT,
    FIRST_ERROR_MESSAGE,
    ROW_PARSED,
    ROW_LOADED
FROM
    INFORMATION_SCHEMA.COPY_HISTORY(
        TABLE_NAME => 'my_json_table',
        START_TIMESTAMP => DATEADD('DAY', -7, CURRENT_TIMESTAMP())
    )
WHERE
    ERROR_COUNT > 0
    AND FILE_FORMAT_TYPE = 'JSON'
ORDER BY
    ERROR_COUNT DESC;

-- VARIANT column schema evolution
SELECT
    TABLE_NAME,
    COLUMN_NAME,
    SCHEMA
FROM
    TABLE(INFORMATION_SCHEMA.VARIANT_SCHEMA(
        TABLE_NAME => 'my_json_table',
        COLUMN_NAME => 'json_data'
    ));
```


#### **5.2.3 Unstructured Data Monitoring**

```sql
-- BLOB storage growth (last 30 days)
SELECT
    TABLE_NAME,
    COUNT(*) AS blob_count,
    SUM(FILE_SIZE) / POWER(1024, 3) AS total_size_gb,
    MAX(FILE_SIZE) / POWER(1024, 2) AS max_file_size_mb
FROM
    INFORMATION_SCHEMA.EXTERNAL_TABLE_FILES
WHERE
    TABLE_NAME = 'my_blob_table'
    AND LAST_MODIFIED > DATEADD('DAY', -30, CURRENT_TIMESTAMP())
GROUP BY
    TABLE_NAME;

-- Failed BLOB loads
SELECT
    FILE_NAME,
    ERROR_COUNT,
    FIRST_ERROR_MESSAGE
FROM
    INFORMATION_SCHEMA.COPY_HISTORY(
        TABLE_NAME => 'my_blob_table',
        START_TIMESTAMP => DATEADD('DAY', -7, CURRENT_TIMESTAMP())
    )
WHERE
    ERROR_COUNT > 0
    AND FILE_FORMAT_TYPE = 'BINARY';
```


#### **5.2.4 Cross-Data-Type Monitoring**

```sql
-- Credit usage by data type (last 30 days)
WITH copy_credits AS (
    SELECT
        CASE
            WHEN FILE_FORMAT_TYPE IN ('CSV', 'TSV', 'PARQUET') THEN 'Structured'
            WHEN FILE_FORMAT_TYPE IN ('JSON', 'AVRO') THEN 'Semi-Structured'
            WHEN FILE_FORMAT_TYPE IN ('BINARY', 'TEXT') THEN 'Unstructured'
        END AS data_type,
        SUM(CREDITS_USED) AS credits
    FROM
        SNOWFLAKE.ACCOUNT_USAGE.COPY_HISTORY
    WHERE
        START_TIME > DATEADD('DAY', -30, CURRENT_TIMESTAMP())
    GROUP BY
        data_type
)
SELECT
    data_type,
    credits,
    ROUND(credits * 3, 2) AS cost_usd -- $3/credit
FROM
    copy_credits
ORDER BY
    credits DESC;

-- Query performance by data type
SELECT
    CASE
        WHEN REGEXP_LIKE(QUERY_TEXT, 'COPY INTO.*JSON|VARIANT') THEN 'Semi-Structured'
        WHEN REGEXP_LIKE(QUERY_TEXT, 'COPY INTO.*CSV|PARQUET') THEN 'Structured'
        WHEN REGEXP_LIKE(QUERY_TEXT, 'COPY INTO.*BLOB|TEXT') THEN 'Unstructured'
        ELSE 'Unknown'
    END AS data_type,
    AVG(TOTAL_ELAPSED_TIME / 1000) AS avg_duration_sec,
    AVG(CREDITS_USED) AS avg_credits,
    COUNT(*) AS query_count
FROM
    TABLE(INFORMATION_SCHEMA.QUERY_HISTORY(
        DATEADD('DAY', -7, CURRENT_TIMESTAMP()),
        CURRENT_TIMESTAMP()
    ))
WHERE
    QUERY_TEXT LIKE '%COPY INTO%'
GROUP BY
    data_type;
```


### **5.3 Error Categorization & Incident Runbooks**


#### **5.3.1 Structured Data Errors**


| **Error Code** | **Error**             | **Root Cause**                                | **Runbook**                                                         | **Credit Impact** |
| -------------- | --------------------- | --------------------------------------------- | ------------------------------------------------------------------- | ----------------- |
| `100083`       | Column count mismatch | Source file has more/less columns than table. | Use `TRUNCATECOLUMNS = TRUE` or align schemas.                      | None              |
| `100072`       | Permission denied     | Missing `READ` on stage or `INSERT` on table. | Grant permissions: `GRANT READ ON STAGE my_stage TO ROLE my_role;`. | None              |
| `1219`         | File format mismatch  | CSV file loaded as JSON.                      | Validate `FILE_FORMAT` type.                                        | None              |
| `2003`         | Memory limit exceeded | Large JOIN or aggregation.                    | Increase warehouse size or optimize query.                          | +2x credits       |
| `1049`         | Disk full             | Spill-to-disk limit reached.                  | Increase warehouse size or reduce data scanned.                     | +3x credits       |



#### **5.3.2 Semi-Structured Data Errors**


| **Error Code** | **Error**             | **Root Cause**                                | **Runbook**                                                        | **Credit Impact**  |
| -------------- | --------------------- | --------------------------------------------- | ------------------------------------------------------------------ | ------------------ |
| `1204`         | Malformed JSON        | Invalid JSON syntax.                          | Use `ON_ERROR = CONTINUE` + DLQ. Validate with `IS_VALID_JSON`.    | +2% credits        |
| `1203`         | JSON parsing error    | Nested JSON exceeds depth limit.              | Flatten JSON before loading or increase `MAX_JSON_DEPTH` (UDF).    | +5% credits        |
| `1219`         | File format mismatch  | JSON file loaded as Parquet.                  | Validate `FILE_FORMAT` type.                                       | None               |
| `1020`         | Transaction conflict  | Concurrent writes to same table.              | Use `ABORT` in multi-statement transactions or retry with backoff. | +1x credit (retry) |
| `2003`         | Memory limit exceeded | Large `VARIANT` operations (e.g., `FLATTEN`). | Increase warehouse size or break into smaller batches.             | +2.5x credits      |



#### **5.3.3 Unstructured Data Errors**


| **Error Code** | **Error**         | **Root Cause**                    | **Runbook**                                                             | **Credit Impact** |
| -------------- | ----------------- | --------------------------------- | ----------------------------------------------------------------------- | ----------------- |
| `100072`       | Permission denied | Missing `READ` on stage.          | Grant `READ` on stage: `GRANT READ ON STAGE my_stage TO ROLE my_role;`. | None              |
| `1049`         | Disk full         | BLOB/TEXT spill limit reached.    | Increase warehouse size or split large files.                           | +3x credits       |
| `2012`         | Stage I/O error   | Network timeout or invalid URL.   | Retry with exponential backoff. Validate stage URL and IAM permissions. | +1.5x credits     |
| `100023`       | File too large    | File exceeds `BINARY_SIZE_LIMIT`. | Split files into chunks (<16MB for `BLOB`).                             | None              |



#### **5.3.4 Incident Recovery Procedures**


##### **Runbook: Malformed JSON (`1204`)**

1. **Diagnose**:
  ```sql
   -- Identify problematic files
   SELECT
       FILE_NAME,
       ERROR_COUNT,
       FIRST_ERROR_MESSAGE
   FROM
       INFORMATION_SCHEMA.COPY_HISTORY(
           TABLE_NAME => 'my_json_table',
           START_TIMESTAMP => DATEADD('HOUR', -1, CURRENT_TIMESTAMP())
       )
   WHERE
       ERROR_CODE = 1204;
  ```
2. **Mitigate**:
  - **Short-term**: Use `ON_ERROR = CONTINUE` to skip bad rows.
  - **Long-term**: Validate JSON before loading:
    ```sql
    -- Create a DLQ for malformed JSON
    CREATE TABLE json_dlq AS
    SELECT
        $1 AS file_name,
        $2 AS row_number,
        $3 AS error_message,
        $4 AS raw_line
    FROM @my_stage
    WHERE NOT IS_VALID_JSON($4);

    -- Reprocess DLQ after fixing
    INSERT INTO my_json_table
    SELECT
        raw_line:field1::STRING,
        raw_line:field2::INT
    FROM json_dlq
    WHERE raw_line IS NOT NULL;
    ```
3. **Prevent**:
  - Use **schema validation** (`ENFORCE_SCHEMA = TRUE`).
  - Implement **pre-load validation** (e.g., AWS Lambda + `IS_VALID_JSON`).


##### **Runbook: Memory Limit Exceeded (`2003`) for Semi-Structured Data**

1. **Diagnose**:
  ```sql
   -- Check memory usage for failed queries
   SELECT
       QUERY_ID,
       MEMORY_USAGE,
       PARTITION_ID,
       QUERY_TEXT
   FROM
       TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
   WHERE
       ERROR_CODE = 2003
       AND QUERY_TEXT LIKE '%JSON%';
  ```
2. **Mitigate**:
  - **Short-term**: Increase warehouse size (e.g., `X-LARGE` → `2X-LARGE`).
  - **Long-term**:
    - **Optimize `FLATTEN**`: Break into smaller batches.
      ```sql
      -- Batch FLATTEN
      WITH batch_1 AS (
          SELECT
              f.value:field1::STRING AS field1,
              f.value:field2::INT AS field2
          FROM
              my_json_table,
              TABLE(FLATTEN(json_data)) f
          WHERE
              f.key = 'items'
          LIMIT 10000
      )
      SELECT * FROM batch_1;
      ```
    - **Use `VARIANT` sparingly**: Extract only needed fields.
      ```sql
      -- Avoid SELECT * on VARIANT
      SELECT
          json_data:field1::STRING,
          json_data:field2::INT
      FROM my_json_table;
      ```
3. **Prevent**:
  - **Monitor `MEMORY_USAGE**` in `QUERY_HISTORY`.
  - **Set `STATEMENT_TIMEOUT_IN_SECONDS**` to kill runaway queries.


##### **Runbook: Stage I/O Error (`2012`) for Unstructured Data**

1. **Diagnose**:
  ```sql
   -- Check stage accessibility
   SELECT
       STAGE_NAME,
       STAGE_URL,
       LAST_ALTERED
   FROM
       INFORMATION_SCHEMA.STAGES
   WHERE
       STAGE_NAME = 'my_blob_stage';
  ```
2. **Mitigate**:
  - **Retry with backoff**:
  - **Validate IAM**:
    - For S3: Ensure IAM role has `s3:GetObject`.
    - For Azure: Ensure SAS token is valid.
3. **Prevent**:
  - Use **Snowflake Internal Stages** for critical loads.
  - **Monitor `STAGE_FILE_METADATA**` for accessibility.



## **6. Advanced Production Patterns**


### **6.1 Structured Data Patterns**


| **Pattern**            | **Use Case**                       | **Implementation**                                                                     | **Pros**                                           | **Cons**                              |
| ---------------------- | ---------------------------------- | -------------------------------------------------------------------------------------- | -------------------------------------------------- | ------------------------------------- |
| **Partitioned Tables** | Large tables with time-series data | `CREATE TABLE ... PARTITION BY RANGE (date_column)`.                                   | **10-100x faster** queries on partitioned columns. | **Storage overhead** (+10%).          |
| **Clustering Keys**    | High-cardinality filter columns    | `CREATE CLUSTERING KEY (col1, col2) ON TABLE my_table`.                                | **90% pruning** for filtered queries.              | **Reclustering cost** (+1 credit/TB). |
| **Materialized Views** | Repeated aggregations              | `CREATE MATERIALIZED VIEW mv AS SELECT col1, COUNT(*) FROM my_table GROUP BY col1;`.   | **100x faster** for repeated queries.              | **Refresh cost** (1 credit/TB).       |
| **Zero-Copy Cloning**  | Dev/Test environments              | `CREATE TABLE dev_table CLONE prod_table;`.                                            | **Instant**, **0 credits**.                        | **Storage shared** (not independent). |
| **Incremental Loads**  | Daily ETL pipelines                | `MERGE INTO target USING source ON target.id = source.id WHEN MATCHED THEN UPDATE...`. | **Idempotent**, **low credit cost**.               | **Complex logic**.                    |



**Example: Partitioned + Clustered Table**

```sql
-- Create a partitioned and clustered table
CREATE TABLE sales (
    sale_id INT,
    sale_date DATE,
    customer_id INT,
    amount DECIMAL(18, 2),
    region STRING
)
PARTITION BY RANGE (sale_date) (
    START ('2020-01-01') INCLUSIVE,
    END ('2021-01-01') EXCLUSIVE,
    EVERY (INTERVAL '1 MONTH')
)
CLUSTER BY (region, customer_id);

-- Query with partition pruning
SELECT
    region,
    SUM(amount) AS total_sales
FROM
    sales
WHERE
    sale_date BETWEEN '2020-06-01' AND '2020-06-30'
GROUP BY
    region;
```


**Example: Materialized View for Aggregations**

```sql
-- Create a materialized view
CREATE MATERIALIZED VIEW daily_sales AS
SELECT
    sale_date,
    region,
    SUM(amount) AS total_sales,
    COUNT(*) AS transaction_count
FROM
    sales
GROUP BY
    sale_date, region;

-- Query the MV (automatically refreshed)
SELECT * FROM daily_sales WHERE sale_date = '2020-06-01';
```


### **6.2 Semi-Structured Data Patterns**


| **Pattern**                  | **Use Case**                 | **Implementation**                                                                                  | **Pros**                                | **Cons**                                  |
| ---------------------------- | ---------------------------- | --------------------------------------------------------------------------------------------------- | --------------------------------------- | ----------------------------------------- |
| **Schema-on-Read**           | Flexible data ingestion      | Load JSON as `VARIANT`, extract fields at query time.                                               | **No schema changes** needed.           | **Slower queries** (+10-20%).             |
| **FLATTEN + LATERAL JOIN**   | Nested JSON arrays           | `SELECT f.value FROM my_table, TABLE(FLATTEN(json_data)) f`.                                        | **Handles nested data** easily.         | **Memory-intensive** (+2.5x credits).     |
| **JSON Path Extraction**     | Selective field extraction   | `SELECT JSON_EXTRACT_PATH_TEXT(json_data, '$.user.name') FROM my_table;`.                           | **Fast**, **no parsing overhead**.      | **Brittle** (path changes break queries). |
| **VARIANT to Structured**    | Performance-critical queries | `CREATE TABLE structured AS SELECT json_data:field1::STRING, json_data:field2::INT FROM my_table;`. | **10x faster** for repeated queries.    | **Storage overhead** (+20%).              |
| **Avro/Parquet with Schema** | Strict typing                | `COPY INTO my_table FROM @stage FILE_FORMAT = (TYPE = 'AVRO' ENFORCE_SCHEMA = TRUE)`.               | **Type safety**, **faster parsing**.    | **Schema evolution** requires DDL.        |
| **Custom UDFs for XML**      | XML parsing                  | `CREATE FUNCTION xml_to_variant(xml_string STRING) RETURNS VARIANT LANGUAGE JAVASCRIPT AS ...`.     | **Flexible**, **supports complex XML**. | **Slower** (+30% credits).                |



**Example: Schema-on-Read with VARIANT**

```sql
-- Load JSON as VARIANT
CREATE TABLE user_events (
    event_id STRING,
    event_data VARIANT,
    event_time TIMESTAMP
);

-- Query nested fields
SELECT
    event_id,
    event_data:user:id::STRING AS user_id,
    event_data:user:name::STRING AS user_name,
    event_data:action::STRING AS action
FROM
    user_events
WHERE
    event_data:user:country::STRING = 'US';

-- Extract and flatten nested arrays
SELECT
    u.event_id,
    i.item_id::STRING AS item_id,
    i.price::DECIMAL(10, 2) AS price
FROM
    user_events u,
    TABLE(FLATTEN(u.event_data:items)) i;
```


**Example: VARIANT to Structured Conversion**

```sql
-- Convert VARIANT to structured table for performance
CREATE TABLE user_events_structured AS
SELECT
    event_id,
    event_data:user:id::STRING AS user_id,
    event_data:user:name::STRING AS user_name,
    event_data:action::STRING AS action,
    event_data:timestamp::TIMESTAMP AS event_timestamp,
    event_time
FROM
    user_events;

-- Query the structured table (faster)
SELECT
    user_id,
    COUNT(*) AS event_count
FROM
    user_events_structured
WHERE
    action = 'purchase'
GROUP BY
    user_id;
```


**Example: Custom UDF for XML Parsing**

```sql
-- Create a JavaScript UDF to parse XML
CREATE OR REPLACE FUNCTION parse_xml(xml_string STRING)
RETURNS VARIANT
LANGUAGE JAVASCRIPT
AS
$$
    // Simple XML to JSON parser (for demonstration)
    const parser = new DOMParser();
    const xmlDoc = parser.parseFromString(XML_STRING, "text/xml");
    const result = {};

    // Extract root element
    const root = xmlDoc.documentElement;
    result[root.tagName] = {};

    // Extract child nodes
    for (let i = 0; i < root.childNodes.length; i++) {
        const child = root.childNodes[i];
        if (child.nodeType === 1) { // Element node
            result[root.tagName][child.tagName] = child.textContent;
        }
    }

    return result;
$$;

-- Use the UDF to load XML
CREATE TABLE xml_data AS
SELECT
    METADATA$FILE_NAME AS file_name,
    parse_xml($1) AS parsed_data
FROM @my_xml_stage;
```


### **6.3 Unstructured Data Patterns**


| **Pattern**            | **Use Case**           | **Implementation**                                                                      | **Pros**                                                       | **Cons**                           |
| ---------------------- | ---------------------- | --------------------------------------------------------------------------------------- | -------------------------------------------------------------- | ---------------------------------- |
| **BLOB Storage**       | PDFs, images, binaries | `CREATE TABLE documents (id INT, content BLOB);`.                                       | **No parsing**, **fast ingestion**.                            | **No queryable metadata**.         |
| **TEXT with Regex**    | Log file analysis      | `SELECT REGEXP_SUBSTR(log_text, 'ERROR: .*') FROM logs;`.                               | **Flexible**, **no schema needed**.                            | **Slow** (+50% credits).           |
| **External Functions** | Image/PDF processing   | `CREATE EXTERNAL FUNCTION process_pdf(BLOB) RETURNS VARIANT API_INTEGRATION = my_api;`. | **Leverage cloud services** (AWS Lambda, GCP Cloud Functions). | **Network latency** (+100ms/call). |
| **Stage Metadata**     | File-level operations  | `SELECT METADATA$FILE_NAME, METADATA$FILE_SIZE FROM @my_stage;`.                        | **No loading required**.                                       | **No row-level access**.           |
| **Chunked BLOBs**      | Large files (>16MB)    | Split files into 16MB chunks before loading.                                            | **Avoids `BINARY_SIZE_LIMIT**`.                                | **Complex ETL**.                   |



**Example: BLOB Storage with Metadata**

```sql
-- Create a table for PDFs
CREATE TABLE documents (
    doc_id INT AUTOINCREMENT,
    doc_name STRING,
    doc_content BLOB,
    doc_size INT,
    upload_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

-- Load PDFs from stage
COPY INTO documents (doc_name, doc_content, doc_size)
FROM (
    SELECT
        METADATA$FILE_NAME,
        $1,
        METADATA$FILE_SIZE
    FROM @pdf_stage
)
FILE_FORMAT = (TYPE = 'BINARY');

-- Query metadata (no content parsing)
SELECT
    doc_id,
    doc_name,
    doc_size / POWER(1024, 2) AS size_mb,
    upload_time
FROM
    documents
WHERE
    doc_size > 10 * POWER(1024, 2); -- >10MB
```


**Example: Log Analysis with TEXT and Regex**

```sql
-- Create a table for logs
CREATE TABLE app_logs (
    log_id INT AUTOINCREMENT,
    log_time TIMESTAMP,
    log_level STRING,
    log_message STRING,
    raw_log TEXT
);

-- Load logs from stage
COPY INTO app_logs (log_time, log_level, log_message, raw_log)
FROM (
    SELECT
        REGEXP_SUBSTR($1, '^(\\d{4}-\\d{2}-\\d{2} \\d{2}:\\d{2}:\\d{2})')::TIMESTAMP AS log_time,
        REGEXP_SUBSTR($1, '\\[([A-Z]+)\\]') AS log_level,
        REGEXP_SUBSTR($1, ': (.*)$') AS log_message,
        $1 AS raw_log
    FROM @log_stage
)
FILE_FORMAT = (TYPE = 'TEXT');

-- Query errors
SELECT
    log_time,
    log_level,
    log_message
FROM
    app_logs
WHERE
    log_level = 'ERROR'
    AND log_time > DATEADD('HOUR', -1, CURRENT_TIMESTAMP());
```


**Example: External Function for Image Processing**

```sql
-- Step 1: Create an API integration
CREATE API INTEGRATION my_image_api
    ENABLED = TRUE
    API_PROVIDER = AWS_LAMBDA
    API_AWS_ROLE_ARN = 'arn:aws:iam::123456789012:role/snowflake-lambda-role'
    API_ALLOWED_PREFIXES = ('https://my-lambda.execute-api.us-west-2.amazonaws.com')
    ENABLED = TRUE;

-- Step 2: Create an external function
CREATE EXTERNAL FUNCTION process_image(image BLOB)
RETURNS VARIANT
API_INTEGRATION = my_image_api
AS 'https://my-lambda.execute-api.us-west-2.amazonaws.com/default/processImage';

-- Step 3: Use the function
SELECT
    doc_id,
    doc_name,
    process_image(doc_content) AS image_metadata
FROM
    documents
WHERE
    doc_name LIKE '%.jpg';
```


### **6.4 Idempotency & DLQ Patterns**


| **Pattern**                | **Data Type**   | **Implementation**                                                                     | **Example**                                         |
| -------------------------- | --------------- | -------------------------------------------------------------------------------------- | --------------------------------------------------- |
| **Idempotent COPY**        | All             | `FORCE=TRUE` + `TRUNCATECOLUMNS=TRUE` + DLQ.                                           | [COPY with DLQ](#copy-with-dlq)                     |
| **Merge for UPSERT**       | Structured      | `MERGE INTO target USING source ON target.id = source.id WHEN MATCHED THEN UPDATE...`. | [Merge Example](#merge-example)                     |
| **JSON Schema Validation** | Semi-Structured | Custom UDF to validate JSON against a schema.                                          | [JSON Validation UDF](#json-validation-udf)         |
| **BLOB Checksum**          | Unstructured    | `SELECT SHA2_BINARY(BINARY_LOAD_FILE('file.pdf'))`. Store checksums in metadata table. | [BLOB Checksum Example](#blob-checksum-example)     |
| **Transaction Log**        | All             | Log `QUERY_ID` + `FILE_NAME` in a control table. Replay only unprocessed files.        | [Transaction Log Example](#transaction-log-example) |



**Example: COPY with DLQ for JSON**

```sql
-- Step 1: Create DLQ table
CREATE TABLE json_dlq (
    file_name STRING,
    row_number INTEGER,
    error_message STRING,
    raw_line VARIANT,
    load_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

-- Step 2: COPY with DLQ routing
COPY INTO my_json_table
FROM @my_json_stage
FILE_FORMAT = (TYPE = 'JSON' STRICT = FALSE)
ON_ERROR = CONTINUE
VALIDATION_MODE = RETURN_ALL_ERRORS;

-- Step 3: Capture errors in DLQ
INSERT INTO json_dlq
SELECT
    METADATA$FILE_NAME,
    METADATA$ROW_NUMBER,
    METADATA$ERROR_MESSAGE,
    $1
FROM @my_json_stage
WHERE METADATA$FILE_NAME IN (
    SELECT FILE_NAME
    FROM INFORMATION_SCHEMA.COPY_HISTORY(
        TABLE_NAME => 'my_json_table',
        START_TIMESTAMP => DATEADD('HOUR', -1, CURRENT_TIMESTAMP())
    )
    WHERE ERROR_COUNT > 0
);

-- Step 4: Reprocess DLQ
INSERT INTO my_json_table
SELECT
    raw_line:field1::STRING,
    raw_line:field2::INT
FROM json_dlq
WHERE retry_count < 3;

-- Step 5: Update retry count
UPDATE json_dlq
SET retry_count = retry_count + 1
WHERE file_name IN (
    SELECT file_name FROM json_dlq WHERE retry_count < 3
);
```


**Example: Merge for UPSERT (Structured)**

```sql
-- UPSERT with MERGE
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


**Example: JSON Schema Validation UDF**

```sql
-- Create a UDF to validate JSON against a schema
CREATE OR REPLACE FUNCTION validate_json_schema(json_data VARIANT, schema VARIANT)
RETURNS BOOLEAN
LANGUAGE JAVASCRIPT
AS
$$
    // Simple schema validation (for demonstration)
    const ajv = new Ajv(); // Hypothetical JSON Schema validator
    const validate = ajv.compile(SCHEMA);
    return validate(JSON_DATA) ? true : false;
$$;

-- Use the UDF in COPY
COPY INTO my_json_table
FROM (
    SELECT
        $1 AS raw_json
    FROM @my_json_stage
    WHERE validate_json_schema($1, PARSE_JSON('{"type": "object", "properties": {"id": {"type": "string"}}}'))
)
FILE_FORMAT = (TYPE = 'JSON');
```


**Example: BLOB Checksum for Idempotency**

```sql
-- Step 1: Create a checksum table
CREATE TABLE blob_checksums (
    file_name STRING PRIMARY KEY,
    sha256_hash STRING,
    load_time TIMESTAMP,
    load_status STRING
);

-- Step 2: Load BLOBs with checksum validation
COPY INTO my_blob_table
FROM (
    SELECT
        $1 AS blob_content,
        METADATA$FILE_NAME AS file_name
    FROM @my_blob_stage
    WHERE NOT EXISTS (
        SELECT 1
        FROM blob_checksums
        WHERE file_name = METADATA$FILE_NAME
          AND sha256_hash = SHA2_HEX($1)
    )
)
FILE_FORMAT = (TYPE = 'BINARY');

-- Step 3: Update checksums
MERGE INTO blob_checksums AS target
USING (
    SELECT
        METADATA$FILE_NAME AS file_name,
        SHA2_HEX($1) AS sha256_hash,
        CURRENT_TIMESTAMP() AS load_time,
        'SUCCESS' AS load_status
    FROM @my_blob_stage
) AS source
ON target.file_name = source.file_name
WHEN MATCHED THEN
    UPDATE SET
        sha256_hash = source.sha256_hash,
        load_time = source.load_time,
        load_status = source.load_status
WHEN NOT MATCHED THEN
    INSERT (file_name, sha256_hash, load_time, load_status)
    VALUES (source.file_name, source.sha256_hash, source.load_time, source.load_status);
```


**Example: Transaction Log for Idempotency**

```sql
-- Step 1: Create a transaction log
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


### **6.5 CI/CD Validation Patterns**


| **Validation Type**        | **Data Type**   | **Tool/Method**                         | **Example**                                               |
| -------------------------- | --------------- | --------------------------------------- | --------------------------------------------------------- |
| **Schema Validation**      | Structured      | `INFORMATION_SCHEMA.COLUMNS` + Git diff | [Schema Comparison Query](#schema-comparison-query)       |
| **JSON Schema Validation** | Semi-Structured | Custom UDF + Ajv                        | [JSON Schema UDF](#json-schema-validation-udf)            |
| **BLOB Integrity**         | Unstructured    | `SHA2_HEX` + Metadata Table             | [BLOB Checksum Example](#blob-checksum-example)           |
| **Performance Regression** | All             | `QUERY_HISTORY` + Baseline Comparison   | [Performance Test Query](#performance-test-query)         |
| **Data Quality**           | All             | Great Expectations + Snowflake          | [Great Expectations Example](#great-expectations-example) |



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


### **6.6 Retry & Backpressure Patterns**


| **Scenario**            | **Data Type**   | **Retry Strategy**                | **Implementation**                            |
| ----------------------- | --------------- | --------------------------------- | --------------------------------------------- |
| **Transient Errors**    | All             | Exponential backoff (3x, 2^n sec) | [Python Retry Example](#python-retry-example) |
| **Warehouse Busy**      | All             | Queue + Auto-resume               | Set `AUTO_RESUME=TRUE` + `AUTO_SUSPEND=60`.   |
| **Stage I/O Timeouts**  | All             | Retry with jitter                 | [Stage Retry Example](#stage-retry-example)   |
| **Memory Errors**       | Semi-Structured | Increase warehouse size + Retry   | [Memory Retry Example](#memory-retry-example) |
| **JSON Parsing Errors** | Semi-Structured | Skip bad rows + DLQ               | [JSON DLQ Example](#json-dlq-example)         |
| **BLOB Size Limit**     | Unstructured    | Split files + Retry               | [Chunked BLOB Example](#chunked-blob-example) |



**Example: Python Retry Logic for COPY**

```python
import time
import snowflake.connector
from snowflake.connector.errors import OperationalError

def copy_with_retry(query, max_retries=3, initial_delay=1):
    delay = initial_delay
    for attempt in range(max_retries):
        try:
            conn = snowflake.connector.connect(...)
            cursor = conn.cursor()
            cursor.execute(query)
            return "SUCCESS"
        except OperationalError as e:
            if attempt == max_retries - 1:
                raise
            if e.errno in [2003, 2012, 1020, 002008]:  # Retryable errors
                time.sleep(delay)
                delay *= 2  # Exponential backoff
            else:
                raise
    raise Exception("Max retries exceeded")

# Usage
copy_with_retry("""
    COPY INTO my_table
    FROM @my_stage
    FILE_FORMAT = (TYPE = 'JSON')
    ON_ERROR = CONTINUE
""")
```


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


**Example: Memory Retry for JSON (Increase Warehouse Size)**

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
    EXECUTE IMMEDIATE 'COPY INTO my_json_table FROM @my_stage FILE_FORMAT = (TYPE = ''JSON'') ON_ERROR = CONTINUE';

    -- Revert to original size (optional)
    ALTER WAREHOUSE my_warehouse SET WAREHOUSE_SIZE = LARGE;
END;
```



## **7. Decision Matrix / Quick Reference Flowchart**


```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#ffd700', 'edgeLabelBackground':'#fff'}}}%%
flowchart TD
    A[Data Ingestion Task] --> B{Data Type?}
    B -->|Structured (CSV/Parquet)| C[Use COPY INTO]
    B -->|Semi-Structured (JSON/Avro)| D[Use COPY INTO with VARIANT]
    B -->|Unstructured (PDF/Logs)| E[Use COPY INTO with BLOB/TEXT]

    C --> F{Volume?}
    D --> F
    E --> F
    F -->|< 100GB| G[Warehouse: MEDIUM]
    F -->|100GB - 1TB| H[Warehouse: LARGE]
    F -->|> 1TB| I[Warehouse: X-LARGE+]

    G --> J{Concurrency?}
    H --> J
    I --> J
    J -->|Single User| K[MULTI_CLUSTER=FALSE]
    J -->|Team (10-50)| L[MULTI_CLUSTER=TRUE\nMAX_CLUSTERS=2]
    J -->|Enterprise (50+)| M[MULTI_CLUSTER=TRUE\nMAX_CLUSTERS=4-10]

    K --> N{Transformation?}
    L --> N
    M --> N
    N -->|Simple (Filter/Project)| O[Standard SQL]
    N -->|Complex (Joins/Aggregates)| P[CTEs/Stored Procedures]
    N -->|Nested Data| Q[FLATTEN + LATERAL JOIN]

    O --> R{Performance Critical?}
    P --> R
    Q --> R
    R -->|Yes| S[Clustering Keys\n+ Materialized Views]
    R -->|No| T[Proceed]

    D --> U{Schema Known?}
    U -->|Yes| V[Enforce Schema\nENFORCE_SCHEMA=TRUE]
    U -->|No| W[Schema-on-Read\nVARIANT + JSON_EXTRACT]

    E --> X{Processing Needed?}
    X -->|No| Y[Store as BLOB/TEXT]
    X -->|Yes| Z[External Function\n(AWS Lambda/GCP)]

    S --> T
    V --> T
    W --> T
    Y --> T
    Z --> T

    T --> AA{Idempotency Required?}
    AA -->|Yes| AB[DLQ + Retry Logic]
    AA -->|No| AC[Standard Execution]

    AB --> AD[Production-Ready]
    AC --> AD
```



## **8. Key Engineering Principles & Bottom Line**


### **8.1 Core Principles for Data Handling**


| **Principle**                     | **Application in Snowflake**                                          | **Impact**                                                 |
| --------------------------------- | --------------------------------------------------------------------- | ---------------------------------------------------------- |
| **Separation of Compute/Storage** | **Compute**: Warehouses. **Storage**: S3/GCS/Azure Blob.              | **Independent scaling** (e.g., query without moving data). |
| **Columnar Storage**              | **Parquet** for structured/semi-structured. **No row-based storage**. | **10-100x faster** for analytical queries.                 |
| **Late Materialization**          | **Pruning** (partition, columnar) + **vectorized execution**.         | **90% less I/O** for filtered queries.                     |
| **Immutable Data**                | **No in-place updates** (MVCC).                                       | **Time travel** (up to 90 days) + **snapshot isolation**.  |
| **Schema Evolution**              | **VARIANT** for semi-structured. **ALTER TABLE** for structured.      | **No DDL needed** for JSON/Avro.                           |
| **Cost-Proportional Scaling**     | **Credits = f(Warehouse Size × Time)**.                               | **Pay-per-use** (no over-provisioning).                    |
| **Failure Isolation**             | **Micro-partitioning** (16-128MB chunks) + **automatic retry**.       | **No single point of failure**.                            |



### **8.2 Bottom Line for Production Engineers**


#### **8.2.1 Structured Data**

1. **Ingestion**:
  - **Best Format**: **Parquet** (columnar, compressed, fast).
  - **Avoid CSV** for large datasets (+30% slower than Parquet).
  - **Use `COMPRESSION = ZSTD**` for best compression ratio.
2. **Query Performance**:
  - **Cluster on high-cardinality filter columns** (e.g., `date`, `region`).
  - **Partition by time** for time-series data (prunes 90% of data).
  - **Materialize aggregations** for repeated queries (100x faster).
3. **Cost Control**:
  - **Biggest Cost Drivers**:
    - **Warehouse Idle Time** (set `AUTO_SUSPEND=60`).
    - **Full Table Scans** (add `WHERE` clauses).
  - **Savings Strategies**:
    - **Use `USE_CACHED_RESULT=TRUE**` (90-95% latency reduction, 0 credits).
    - **Right-size warehouses** (avoid `4X-LARGE` for ad-hoc queries).


#### **8.2.2 Semi-Structured Data**

1. **Ingestion**:
  - **Best Format**: **Parquet > JSON > Avro** (performance-wise).
  - **Use `VARIANT**` for flexible schema (schema-on-read).
  - **Avoid `STRICT = TRUE**` unless schema is guaranteed (adds +10% credits).
2. **Query Performance**:
  - **Extract only needed fields** (avoid `SELECT *` on `VARIANT`).
  - **Use `FLATTEN` sparingly** (memory-intensive, +2.5x credits).
  - **Convert `VARIANT` to structured** for performance-critical queries.
3. **Cost Control**:
  - **JSON Parsing Overhead**: **+10-20% credits** vs. structured.
  - **Avro/Parquet**: **+5% credits** (faster parsing).
  - **Monitor `MEMORY_USAGE**` for `FLATTEN` queries.


#### **8.2.3 Unstructured Data**

1. **Ingestion**:
  - **Use `BLOB` for binaries** (PDF, images, etc.).
  - **Use `TEXT` for logs** (max 16MB/row).
  - **Split large files** (>16MB) into chunks.
2. **Query Performance**:
  - **No native parsing** for `BLOB` (use **external functions**).
  - **Regex on `TEXT**` is slow (+50% credits).
3. **Cost Control**:
  - **No parsing overhead** for `BLOB` (fastest ingestion).
  - **Storage Cost**: Same as structured/semi-structured ($23/TB/month).


### **8.3 Performance Cheat Sheet**


| **Operation**          | **Structured** | **Semi-Structured** | **Unstructured** | **Optimization**                        |
| ---------------------- | -------------- | ------------------- | ---------------- | --------------------------------------- |
| **Bulk Load**          | ⚡⚡⚡⚡⚡          | ⚡⚡⚡⚡                | ⚡⚡⚡⚡⚡            | Use **Parquet** for all types.          |
| **Filter Queries**     | ⚡⚡⚡⚡⚡          | ⚡⚡⚡                 | ⚡                | **Cluster/partition** structured data.  |
| **JOINs**              | ⚡⚡⚡⚡⚡          | ⚡⚡                  | N/A              | **Pre-aggregate** semi-structured data. |
| **Aggregations**       | ⚡⚡⚡⚡⚡          | ⚡⚡⚡                 | N/A              | **Materialize** results.                |
| **FLATTEN**            | N/A            | ⚡⚡                  | N/A              | **Batch processing** to avoid OOM.      |
| **Regex**              | N/A            | N/A                 | ⚡                | **Pre-process** logs before loading.    |
| **External Functions** | N/A            | ⚡⚡⚡                 | ⚡⚡⚡              | **Cache results** to reduce calls.      |



### **8.4 Cost Cheat Sheet**


| **Operation**       | **Structured**    | **Semi-Structured**  | **Unstructured**  | **Cost Driver**                        |
| ------------------- | ----------------- | -------------------- | ----------------- | -------------------------------------- |
| **Storage**         | $23/TB/month      | $23/TB/month         | $23/TB/month      | **Same for all types**.                |
| **Compute (COPY)**  | 1 credit/core-sec | 1.2 credits/core-sec | 1 credit/core-sec | **JSON parsing overhead**.             |
| **Compute (Query)** | 1 credit/core-sec | 1.1 credits/core-sec | 1 credit/core-sec | **FLATTEN overhead**.                  |
| **Cloud Services**  | $0.005/TB         | $0.005/TB            | $0.005/TB         | **Data scanned from external stages**. |
| **Spill-to-Disk**   | +2x credits       | +2.5x credits        | +2x credits       | **Memory limits exceeded**.            |
| **Multi-Cluster**   | +10%/cluster      | +15%/cluster         | +10%/cluster      | **Concurrency scaling**.               |



### **8.5 Reliability Cheat Sheet**


| **Risk**                  | **Structured** | **Semi-Structured** | **Unstructured**  | **Mitigation**                         |
| ------------------------- | -------------- | ------------------- | ----------------- | -------------------------------------- |
| **Schema Drift**          | ❌              | ✅                   | ❌                 | **Use `VARIANT` + schema validation**. |
| **Malformed Data**        | ❌              | ✅                   | ❌                 | **DLQ + `ON_ERROR=CONTINUE**`.         |
| **Memory Limits**         | ✅              | ✅✅                  | ❌                 | **Increase warehouse size**.           |
| **Stage I/O Errors**      | ✅              | ✅                   | ✅                 | **Retry with backoff**.                |
| **Time Travel**           | ✅              | ✅                   | ❌ (metadata only) | **Use structured/semi-structured**.    |
| **Transaction Conflicts** | ✅              | ✅                   | ❌                 | **Use `ABORT` + retry**.               |




## **9. Production Checklist**


### **9.1 Structured Data**

- **Use Parquet** for bulk loads (faster than CSV).
- **Cluster tables** on high-cardinality filter columns.
- **Partition by time** for time-series data.
- **Materialize aggregations** for repeated queries.
- **Set `AUTO_SUSPEND=60`** to reduce idle costs.
- **Monitor `QUERY_HISTORY`** for slow queries.
- **Validate schemas** before loading (avoid `100083`).
- **Use `TRUNCATECOLUMNS=TRUE`** for flexible loads.
- **Enable `USE_CACHED_RESULT=TRUE`** for repeated queries.
- **Right-size warehouses** (avoid `X-SMALL` for production).


### **9.2 Semi-Structured Data**

- **Use `VARIANT`** for schema-on-read flexibility.
- **Avoid `STRICT = TRUE`** unless schema is guaranteed.
- **Extract only needed fields** (avoid `SELECT *` on `VARIANT`).
- **Batch `FLATTEN` operations** to avoid OOM.
- **Convert `VARIANT` to structured** for performance-critical queries.
- **Use `INFER_SCHEMA=TRUE`** for initial loads.
- **Monitor `MEMORY_USAGE`** for `FLATTEN` queries.
- **Implement DLQ** for malformed JSON.
- **Use Parquet/Avro** for better performance than JSON.
- **Set `ON_ERROR=CONTINUE`** for partial loads.


### **9.3 Unstructured Data**

- **Use `BLOB` for binaries** (PDF, images, etc.).
- **Use `TEXT` for logs** (max 16MB/row).
- **Split large files** (>16MB) into chunks.
- **Store metadata** (e.g., `SHA256` hashes) for idempotency.
- **Use external functions** for processing (e.g., AWS Lambda).
- **Monitor `STAGE_FILE_METADATA`** for accessibility.
- **Avoid regex on `TEXT`** (use pre-processing).
- **Set `COMPRESSION=AUTO`** for text files.
- **Use internal stages** for critical data.
- **Implement retry logic** for stage I/O errors.



## **10. Quick Reference Commands**


### **10.1 Structured Data**


| **Task**              | **Command**                                                                            |
| --------------------- | -------------------------------------------------------------------------------------- |
| **Load CSV**          | `COPY INTO my_table FROM @my_stage FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1);`      |
| **Load Parquet**      | `COPY INTO my_table FROM @my_stage FILE_FORMAT = (TYPE = 'PARQUET');`                  |
| **Cluster Table**     | `CREATE CLUSTERING KEY (col1, col2) ON TABLE my_table;`                                |
| **Partition Table**   | `CREATE TABLE my_table (...) PARTITION BY RANGE (date_column);`                        |
| **Materialized View** | `CREATE MATERIALIZED VIEW mv AS SELECT col1, COUNT(*) FROM my_table GROUP BY col1;`    |
| **Check Schema**      | `SELECT * FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME = 'my_table';`              |
| **Monitor Loads**     | `SELECT * FROM INFORMATION_SCHEMA.COPY_HISTORY WHERE TABLE_NAME = 'my_table';`         |
| **Optimize Table**    | `ALTER TABLE my_table OPTIMIZE;` (reclusters data)                                     |
| **Zero-Copy Clone**   | `CREATE TABLE dev_table CLONE prod_table;`                                             |
| **Incremental Load**  | `MERGE INTO target USING source ON target.id = source.id WHEN MATCHED THEN UPDATE...;` |



### **10.2 Semi-Structured Data**


| **Task**                          | **Command**                                                                                 |
| --------------------------------- | ------------------------------------------------------------------------------------------- |
| **Load JSON**                     | `COPY INTO my_table FROM @my_stage FILE_FORMAT = (TYPE = 'JSON' STRIP_OUTER_ARRAY = TRUE);` |
| **Load Avro**                     | `COPY INTO my_table FROM @my_stage FILE_FORMAT = (TYPE = 'AVRO' ENFORCE_SCHEMA = TRUE);`    |
| **Extract JSON Field**            | `SELECT JSON_EXTRACT_PATH_TEXT(json_data, '$.user.name') FROM my_table;`                    |
| **Flatten JSON Array**            | `SELECT f.value FROM my_table, TABLE(FLATTEN(json_data:items)) f;`                          |
| **Convert VARIANT to Structured** | `CREATE TABLE structured AS SELECT json_data:field1::STRING FROM my_table;`                 |
| **Check VARIANT Schema**          | `SELECT * FROM TABLE(INFORMATION_SCHEMA.VARIANT_SCHEMA('my_table', 'json_data'));`          |
| **Monitor JSON Loads**            | `SELECT * FROM INFORMATION_SCHEMA.COPY_HISTORY WHERE FILE_FORMAT_TYPE = 'JSON';`            |
| **Infer Schema**                  | `COPY INTO my_table FROM @my_stage FILE_FORMAT = (TYPE = 'JSON' INFER_SCHEMA = TRUE);`      |
| **Validate JSON**                 | `SELECT * FROM @my_stage WHERE IS_VALID_JSON($1);`                                          |
| **DLQ for JSON**                  | `COPY INTO my_table FROM @my_stage ON_ERROR = CONTINUE;`                                    |



### **10.3 Unstructured Data**


| **Task**                      | **Command**                                                                        |
| ----------------------------- | ---------------------------------------------------------------------------------- |
| **Load BLOB**                 | `COPY INTO my_table FROM @my_stage FILE_FORMAT = (TYPE = 'BINARY');`               |
| **Load TEXT**                 | `COPY INTO my_table FROM @my_stage FILE_FORMAT = (TYPE = 'TEXT');`                 |
| **Check BLOB Size**           | `SELECT OCTET_LENGTH(blob_column) FROM my_table;`                                  |
| **Compute SHA256**            | `SELECT SHA2_HEX(blob_column) FROM my_table;`                                      |
| **External Function (PDF)**   | `SELECT process_pdf(blob_column) FROM my_table;`                                   |
| **Stage Metadata**            | `SELECT * FROM @my_stage (FILE_NAME, FILE_SIZE, LAST_MODIFIED);`                   |
| **Monitor BLOB Loads**        | `SELECT * FROM INFORMATION_SCHEMA.COPY_HISTORY WHERE FILE_FORMAT_TYPE = 'BINARY';` |
| **Split Large Files**         | Use `SPLIT` in Python or `aws s3 cp` with `--multipart-chunksize`.                 |
| **Check Stage Accessibility** | `SELECT * FROM INFORMATION_SCHEMA.STAGES WHERE STAGE_NAME = 'my_stage';`           |
| **Retry Failed Loads**        | `COPY INTO my_table FROM @my_stage ON_ERROR = CONTINUE;`                           |




## **11. Anti-Patterns to Avoid**


### **11.1 Structured Data**


| **Anti-Pattern**           | **Why It’s Bad**                                         | **Fix**                                                                  |
| -------------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------ |
| **SELECT ***               | Scans all columns (no columnar pruning).                 | Explicitly list columns.                                                 |
| **No WHERE Clause**        | Full table scan (no partition pruning).                  | Add filters on **clustering/partition keys**.                            |
| **Large JOINs**            | Spills to disk (memory limit: **80% of warehouse RAM**). | **Pre-aggregate** or use **broadcast joins** (`/*+ LEADING(table1) */`). |
| **No AUTO_SUSPEND**        | Warehouse runs idle (burns credits).                     | Set `AUTO_SUSPEND=60`.                                                   |
| **X-SMALL Warehouse**      | Fails on **>1TB scans** (OOM).                           | Use **at least `MEDIUM**` for production.                                |
| **No Clustering**          | Queries scan **entire table** (no pruning).              | Cluster on **high-cardinality filter columns**.                          |
| **CSV for Large Datasets** | **+30% slower** than Parquet.                            | Use **Parquet**.                                                         |
| **No Schema Validation**   | **Silent data corruption** (e.g., `VARCHAR` truncation). | Use `ENFORCE_SCHEMA=TRUE` + `VALIDATION_MODE=RETURN_ERRORS`.             |
| **Manual Retries**         | No backoff → **thundering herd**.                        | Use **exponential backoff** + jitter.                                    |
| **No Monitoring**          | **Undetected failures** (e.g., silent truncation).       | Monitor `QUERY_HISTORY`, `COPY_HISTORY`, `WAREHOUSE_METERING_HISTORY`.   |



### **11.2 Semi-Structured Data**


| **Anti-Pattern**            | **Why It’s Bad**                                 | **Fix**                                                   |
| --------------------------- | ------------------------------------------------ | --------------------------------------------------------- |
| **SELECT * on VARIANT**     | Parses **entire JSON** (slow, memory-intensive). | Extract **only needed fields**.                           |
| **FLATTEN on Large Arrays** | **+2.5x credits** (memory-intensive).            | **Batch processing** (e.g., `LIMIT 10000`).               |
| **STRICT = TRUE**           | **Fails on malformed JSON** (no partial loads).  | Use `STRICT = FALSE` + **DLQ**.                           |
| **No Schema Inference**     | **Manual schema management** (error-prone).      | Use `INFER_SCHEMA=TRUE` for initial loads.                |
| **JSON as TEXT**            | **No queryable structure** (regex only).         | Use `VARIANT` + `JSON_EXTRACT_*`.                         |
| **No DLQ**                  | **Failed rows are silently dropped**.            | Always use `ON_ERROR=CONTINUE` + DLQ.                     |
| **Deeply Nested JSON**      | **Path extraction is slow** (e.g., `$.a.b.c.d`). | **Flatten before loading** or use **materialized views**. |
| **No Memory Monitoring**    | **OOM crashes** for `FLATTEN` queries.           | Monitor `MEMORY_USAGE` in `QUERY_HISTORY`.                |
| **Avro without Schema**     | **No type safety** (all fields as `VARIANT`).    | Use `ENFORCE_SCHEMA=TRUE`.                                |
| **XML as TEXT**             | **No native parsing** (regex only).              | Use **custom UDF** (e.g., `XML_TO_VARIANT`).              |



### **11.3 Unstructured Data**


| **Anti-Pattern**                    | **Why It’s Bad**                                           | **Fix**                                         |
| ----------------------------------- | ---------------------------------------------------------- | ----------------------------------------------- |
| **BLOB > 16MB**                     | **Fails with `100023**` (size limit).                      | **Split files** into 16MB chunks.               |
| **Regex on TEXT**                   | **+50% credits** (slow).                                   | **Pre-process logs** before loading.            |
| **No External Functions**           | **No processing** (BLOB/TEXT are opaque).                  | Use **AWS Lambda/GCP Cloud Functions**.         |
| **No Metadata Tracking**            | **No idempotency** (duplicate loads).                      | Store **SHA256 hashes** in a metadata table.    |
| **Public Stage for Sensitive Data** | **Security risk** (unrestricted access).                   | Use **private internal stages**.                |
| **No Stage Monitoring**             | **Undetected I/O errors** (e.g., expired pre-signed URLs). | Monitor `STAGE_FILE_METADATA` + `COPY_HISTORY`. |
| **TEXT for Binaries**               | **Corruption risk** (UTF-8 validation).                    | Use **BLOB** for binaries.                      |
| **No Retry Logic**                  | **Failed loads are not retried**.                          | Implement **exponential backoff** + jitter.     |
| **Large TEXT Rows**                 | **Fails with `100023**` (16MB limit).                      | **Split into multiple rows** or use **BLOB**.   |
| **No Compression**                  | **+30% storage costs**.                                    | Use `COMPRESSION=AUTO` (ZSTD for text).         |




## **12. Further Reading**

- [Snowflake Documentation: Loading Data](https://docs.snowflake.com/en/user-guide/data-load)
- [Snowflake Semi-Structured Data Guide](https://docs.snowflake.com/en/user-guide/semi-structured)
- [Snowflake JSON Data Handling](https://docs.snowflake.com/en/user-guide/json)
- [Snowflake VARIANT Type](https://docs.snowflake.com/en/sql-reference/data-types/semi-structured)
- [Snowflake BLOB Type](https://docs.snowflake.com/en/sql-reference/data-types/binary)
- [Snowflake External Functions](https://docs.snowflake.com/en/user-guide/external-functions)
- [Snowflake Performance Best Practices](https://docs.snowflake.com/en/user-guide/performance)
- [Snowflake Credit Usage Guide](https://www.snowflake.com/blog/understanding-snowflake-credits/)
- [Snowflake Internal Architecture (Sigmod 2020)](https://dl.acm.org/doi/10.1145/3318464.3386134)
- [Great Expectations for Snowflake](https://docs.greatexpectations.io/docs/guides/connecting_to_data/other_databases/snowflake)
