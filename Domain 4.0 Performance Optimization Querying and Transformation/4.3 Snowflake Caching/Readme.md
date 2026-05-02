# **Snowflake Caching: Production-Grade Technical Deep Dive**


## **1. Overview of Snowflake Caching**

### **Mermaid: Snowflake Caching Architecture**
```mermaid
%% Snowflake Caching Architecture
flowchart TD
    subgraph Client["Client Layer"]
        A[("Query")] --> B[("Query Parser")]
    end

    subgraph Caching["Caching Layer"]
        B --> C[("Result Cache")]
        B --> D[("Metadata Cache")]
        B --> E[("Local Disk Cache")]
        C --> F[("Cache Lookup")]
        D --> F
        E --> F
        F -->|Cache Hit| G[("Return Cached Data")]
        F -->|Cache Miss| H[("Execute Query")]
        H --> I[("Cache Results")]
        I --> C
        I --> E
    end

    subgraph Execution["Execution Layer"]
        H --> J[("Query Execution Engine")]
        J --> K[("Virtual Warehouse")]
    end

    subgraph Storage["Storage Layer"]
        K --> L[("Cloud Storage")]
        C --> M[("Result Cache Storage\n(SSD)")]
        D --> N[("Metadata Service")]
        E --> O[("Local Disk\n(SSD)")]
    end

    subgraph Monitoring["Monitoring Layer"]
        P[("QUERY_HISTORY")]
        Q[("ACCOUNT_USAGE")]
    end
    J --> P
    G --> P
    H --> P
    C --> Q
    D --> Q
    E --> Q

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef client fill:#4285f4,stroke:#1976d2;
    classDef caching fill:#ff9800,stroke:#f57c00;
    classDef execution fill:#009688,stroke:#00796b;
    classDef storage fill:#29abe2,stroke:#1a8fb8;
    classDef monitoring fill:#9c27b0,stroke:#7b1fa2;
    class A client;
    class B,C,D,E,F,G,H,I caching;
    class J,K execution;
    class L,M,N,O storage;
    class P,Q monitoring;
```


### **What is Caching in Snowflake?**

Caching in Snowflake is a **performance optimization technique** that **stores frequently accessed data or results** in **fast storage layers** (SSD, memory) to **reduce query execution time** and **lower credit usage**. Snowflake implements **multiple caching layers**, each designed to optimize different aspects of query performance:

1. **Result Cache**: Caches **query results** for identical queries.
2. **Metadata Cache**: Caches **table metadata** (schema, statistics, partition info).
3. **Local Disk Cache**: Caches **frequently accessed data** in local SSD for each warehouse.
4. **Query Cache**: Caches **query plans** and **compiled queries** for reuse.
5. **File Metadata Cache**: Caches **external table metadata** (file formats, stages).
6. **Warehouse Cache**: Caches **warehouse-specific data** (e.g., temporary tables).


### **Why Caching Matters in Snowflake**

| **Benefit** | **Description** | **Impact** |
|-------------|-----------------|------------|
| **Reduced Execution Time** | Cached results are returned instantly (typically <10ms) | ⭐⭐⭐⭐⭐ |
| **Lower Credit Usage** | Cached queries consume **0 credits** (except for cache maintenance) | ⭐⭐⭐⭐⭐ |
| **Improved Concurrency** | Reduces warehouse load, enabling more concurrent queries | ⭐⭐⭐⭐ |
| **Better User Experience** | Faster response times for dashboards, reports, and ad-hoc queries | ⭐⭐⭐⭐⭐ |
| **Cost Savings** | Reduces overall Snowflake spend by avoiding redundant computations | ⭐⭐⭐⭐⭐ |
| **Scalability** | Enables high-performance workloads without proportional cost increases | ⭐⭐⭐⭐ |


### **Snowflake Caching Layers Overview**

| **Cache Type** | **Scope** | **TTL** | **Storage** | **What It Caches** | **Performance Impact** | **Cost** | **Configuration** |
|----------------|-----------|---------|-------------|--------------------|-------------------------|----------|------------------|
| **Result Cache** | Per-user, per-query | 24 hours (configurable) | SSD | Query results | ⭐⭐⭐⭐⭐ (10-1000x faster) | Included | `USE_CACHED_RESULTS`, `RESULT_CACHE_TTL` |
| **Metadata Cache** | Per-session | Session | Memory | Table metadata (schema, statistics) | ⭐⭐ (1.1-2x faster) | Included | Automatic |
| **Local Disk Cache** | Per-warehouse | Session | SSD | Frequently accessed data | ⭐⭐⭐ (2-10x faster) | Included | Automatic |
| **Query Cache** | Per-session | Session | Memory | Query plans, compiled queries | ⭐⭐ (1.1-2x faster) | Included | Automatic |
| **File Metadata Cache** | Per-session | Session | Memory | External table metadata | ⭐⭐ (1.1-2x faster) | Included | Automatic |
| **Warehouse Cache** | Per-warehouse | Session | SSD | Temporary tables, intermediate results | ⭐⭐⭐ (2-10x faster) | Included | Automatic |


### **When to Use Caching in Snowflake**

| **Use Case** | **Recommended Cache Type** | **Example** |
|-------------|----------------------------|-------------|
| **Repetitive Queries** | Result Cache | Dashboard queries, reports |
| **Identical Queries** | Result Cache | BI tool queries, ad-hoc analysis |
| **Frequently Accessed Tables** | Local Disk Cache | Hot tables (e.g., `customers`, `products`) |
| **Complex Queries** | Query Cache | Queries with many joins/aggregations |
| **External Tables** | File Metadata Cache | Queries on S3, Azure Blob, GCS |
| **Metadata Access** | Metadata Cache | All queries (automatic) |
| **Temporary Tables** | Warehouse Cache | Session-specific temporary tables |
| **High Concurrency Workloads** | All Caches | Production workloads with many users |


## **2. Result Cache Deep Dive**

### **A. Definition and Architecture**

**Result Cache** in Snowflake **automatically caches query results** for **24 hours** (configurable) if:
- The **query text** is identical (including whitespace, case, and comments).
- The **underlying data** has not changed.
- The **user's permissions** are the same.
- The **session parameters** (e.g., time zone, role) are the same.

```mermaid
%% Result Cache Architecture
flowchart TD
    subgraph Query["Query Layer"]
        A[("Query")] --> B[("Cache Lookup")]
    end

    subgraph Cache["Result Cache Layer"]
        B --> C{Cache Hit?}
        C -->|Yes| D[("Return Cached Results\n(<10ms)")]
        C -->|No| E[("Execute Query")]
        E --> F[("Cache Results\n(SSD)")]
    end

    subgraph Execution["Execution Layer"]
        E --> G[("Query Execution Engine")]
    end

    subgraph Storage["Storage Layer"]
        F --> H[("Result Cache Storage\n(SSD)")]
    end

    subgraph Monitoring["Monitoring Layer"]
        I[("QUERY_HISTORY.used_cached_result")]
    end
    D --> I
    E --> I

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef query fill:#4285f4,stroke:#1976d2;
    classDef cache fill:#ff9800,stroke:#f57c00;
    classDef execution fill:#009688,stroke:#00796b;
    classDef storage fill:#29abe2,stroke:#1a8fb8;
    classDef monitoring fill:#9c27b0,stroke:#7b1fa2;
    class A query;
    class B,C,D,E,F cache;
    class G execution;
    class H storage;
    class I monitoring;
```

#### **1. How Result Cache Works**
1. **Cache Lookup**:
   - When a query is submitted, Snowflake **checks the result cache** for a matching entry.
   - A **cache hit** occurs if:
     - The **query text** is identical (including whitespace, case, and comments).
     - The **underlying data** has not changed since the result was cached.
     - The **user's permissions** are the same.
     - The **session parameters** (e.g., time zone, role) are the same.

2. **Cache Hit**:
   - If a cache hit occurs, Snowflake **returns the cached results immediately**.
   - The **execution_time** in `QUERY_HISTORY` will be **very low** (typically <10ms).
   - **Credit usage**: **0 credits** (no compute used).

3. **Cache Miss**:
   - If no cache hit occurs, Snowflake **executes the query normally**.
   - After execution, Snowflake **caches the results** for future use.
   - **Credit usage**: Normal credits for query execution.

4. **Cache Invalidation**:
   - The result cache is **automatically invalidated** if:
     - The **underlying data changes** (e.g., INSERT, UPDATE, DELETE on source tables).
     - The **query text changes** (even slightly).
     - The **user's permissions change**.
     - The **session parameters change** (e.g., time zone, role).
     - The **cache TTL expires** (default: 24 hours).

5. **Cache Storage**:
   - Cached results are stored in **SSD** (local disk cache for the warehouse).
   - **Storage cost**: Included in Snowflake's storage pricing.

### **B. Result Cache Configuration**

#### **1. Enable/Disable Result Caching**
```sql
-- Enable result caching (default)
ALTER SESSION SET USE_CACHED_RESULTS = TRUE;

-- Disable result caching
ALTER SESSION SET USE_CACHED_RESULTS = FALSE;

-- Enable for all sessions in a user
ALTER USER my_user SET USE_CACHED_RESULTS = TRUE;

-- Enable for all sessions in a role
ALTER ROLE my_role SET USE_CACHED_RESULTS = TRUE;
```

#### **2. Set Result Cache TTL**
```sql
-- Set result cache TTL to 1 hour (default: 24 hours)
ALTER SESSION SET RESULT_CACHE_TTL = 3600;

-- Set result cache TTL to 12 hours
ALTER SESSION SET RESULT_CACHE_TTL = 43200;

-- Set result cache TTL to 0 (disable caching)
ALTER SESSION SET RESULT_CACHE_TTL = 0;
```

**Note**: The maximum `RESULT_CACHE_TTL` is **86400 seconds (24 hours)**.

### **C. Result Cache Performance**

#### **1. Performance Impact**
| **Metric** | **Without Result Cache** | **With Result Cache (Hit)** | **With Result Cache (Miss)** | **Improvement (Hit)** |
|------------|----------------------------|-------------------------------|--------------------------------|-------------------------|
| **Execution Time** | 1-10 seconds | 1-10 ms | 1-10 seconds | 100-1000x faster |
| **Bytes Scanned** | 1-100 GB | 0 | 1-100 GB | Infinite reduction |
| **Credit Usage** | 1-100 credits | 0 | 1-100 credits | Infinite reduction |
| **Concurrency** | Limited by warehouse | Improved | Limited by warehouse | Better resource utilization |
| **Storage Usage** | N/A | 0.1-10 GB | N/A | Additional storage |

#### **2. Performance Example**
**Query**:
```sql
-- Dashboard query (run every 5 minutes)
SELECT
    region,
    SUM(sales) AS total_sales,
    COUNT(*) AS transaction_count
FROM
    sales
WHERE
    date > CURRENT_DATE() - 30
GROUP BY
    region;
```

**Performance Comparison**:
| **Execution** | **Execution Time** | **Bytes Scanned** | **Credit Usage** | **Notes** |
|---------------|--------------------|-------------------|------------------|-----------|
| **First Run (Cache Miss)** | 5 seconds | 20 GB | 2.5 credits | Full query execution |
| **Subsequent Runs (Cache Hit)** | 5 ms | 0 | 0 | Results returned from cache |

### **D. Result Cache Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use for Repetitive Queries** | Cache results for queries run frequently | Dashboards, reports, BI tools |
| **Set Appropriate TTL** | Adjust TTL based on data freshness requirements | `ALTER SESSION SET RESULT_CACHE_TTL = 3600` (1 hour) |
| **Monitor Cache Usage** | Check `used_cached_result` in `QUERY_HISTORY` | `SELECT used_cached_result FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` |
| **Avoid Cache Invalidation** | Minimize changes to underlying data | Batch updates instead of frequent small updates |
| **Use for Read-Only Workloads** | Cache works best for read-only workloads | BI tools, reporting, analytics |
| **Disable for Unique Queries** | Disable caching for unique queries | `ALTER SESSION SET USE_CACHED_RESULTS = FALSE` |
| **Use Consistent Query Text** | Ensure query text is identical for cache hits | Avoid dynamic SQL, use parameterized queries |
| **Use Consistent Session Parameters** | Ensure session parameters (e.g., time zone) are consistent | Set session parameters explicitly |
| **Test Cache Performance** | Compare performance with and without caching | Run queries and check `used_cached_result` |
| **Document Cacheable Queries** | Document which queries benefit from caching | Internal wiki or Confluence page |
| **Use for Expensive Queries** | Cache results for queries with high credit usage | Queries using >10 credits |
| **Avoid for DML Queries** | DML queries (INSERT, UPDATE, DELETE) are not cached | N/A |
| **Avoid for DDL Queries** | DDL queries (CREATE, ALTER, DROP) are not cached | N/A |

### **E. Result Cache Limitations**

| **Limitation** | **Description** | **Workaround** |
|----------------|-----------------|---------------|
| **Query Text Must Match Exactly** | Cache is keyed by exact query text (including whitespace, case, comments) | Use consistent query formatting |
| **Data Changes Invalidate Cache** | Cache is invalidated when underlying data changes | Batch data changes |
| **Session Parameters Affect Cache** | Cache is keyed by session parameters (e.g., time zone, role) | Set session parameters explicitly |
| **Permissions Affect Cache** | Cache is keyed by user permissions | Use consistent roles |
| **TTL Limited to 24 Hours** | Maximum cache TTL is 24 hours | Use materialized views for longer caching |
| **No Partial Caching** | Cannot cache partial results | Use materialized views for partial results |
| **No Caching for DML** | DML queries (INSERT, UPDATE, DELETE) are not cached | N/A |
| **No Caching for DDL** | DDL queries (CREATE, ALTER, DROP) are not cached | N/A |
| **No Caching for Some Queries** | Some complex queries cannot be cached | Use materialized views |
| **Storage Overhead** | Cached results consume storage | Monitor storage usage |
| **Memory Overhead** | Cache metadata consumes memory | Monitor warehouse memory usage |
| **No Caching for External Tables** | Limited caching for external tables | Use internal tables or materialized views |
| **No Caching for Views** | Views are not cached (but their underlying queries may be) | Query the underlying tables directly |

### **F. Result Cache Examples**

#### **Example 1: Dashboard Queries**
```sql
-- Enable result caching for dashboard queries
ALTER SESSION SET USE_CACHED_RESULTS = TRUE;

-- Run dashboard query (first execution)
SELECT
    region,
    SUM(sales) AS total_sales
FROM
    sales
WHERE
    date > CURRENT_DATE() - 30
GROUP BY
    region;

-- Run the same query again (cache hit)
SELECT
    region,
    SUM(sales) AS total_sales
FROM
    sales
WHERE
    date > CURRENT_DATE() - 30
GROUP BY
    region;

-- Check if the second query used cached results
SELECT
    query_id,
    query_text,
    used_cached_result,
    execution_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%region%SUM(sales)%'
ORDER BY
    start_time DESC;
```

#### **Example 2: Set Cache TTL for Time-Sensitive Data**
```sql
-- Set cache TTL to 1 hour for time-sensitive data
ALTER SESSION SET RESULT_CACHE_TTL = 3600;

-- Run query
SELECT * FROM my_table WHERE date > CURRENT_DATE() - 1;

-- Run the same query within 1 hour (cache hit)
SELECT * FROM my_table WHERE date > CURRENT_DATE() - 1;

-- Run the same query after 1 hour (cache miss)
-- (Wait 1 hour)
SELECT * FROM my_table WHERE date > CURRENT_DATE() - 1;
```

#### **Example 3: Disable Caching for Unique Queries**
```sql
-- Disable result caching for a session with unique queries
ALTER SESSION SET USE_CACHED_RESULTS = FALSE;

-- Run unique queries (no caching)
SELECT * FROM my_table WHERE id = 1;
SELECT * FROM my_table WHERE id = 2;
SELECT * FROM my_table WHERE id = 3;
```

#### **Example 4: Monitor Cache Usage**
```sql
-- Check cache usage for recent queries
SELECT
    query_id,
    query_text,
    used_cached_result,
    execution_time,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY
    used_cached_result DESC, execution_time;

-- Check cache hit rate
SELECT
    used_cached_result,
    COUNT(*) AS query_count,
    AVG(execution_time) AS avg_execution_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    used_cached_result;
```

#### **Example 5: Compare Performance with and without Caching**
```sql
-- Disable caching and run query
ALTER SESSION SET USE_CACHED_RESULTS = FALSE;
SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7;
SELECT LAST_QUERY_ID() AS query_id_no_cache;

-- Enable caching and run query
ALTER SESSION SET USE_CACHED_RESULTS = TRUE;
SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7;
SELECT LAST_QUERY_ID() AS query_id_with_cache;

-- Compare performance
SELECT
    query_id,
    query_text,
    used_cached_result,
    execution_time,
    credits_used
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_id IN ('query_id_no_cache', 'query_id_with_cache')
ORDER BY
    query_id;
```

## **3. Metadata Cache Deep Dive**

### **A. Definition and Architecture**

**Metadata Cache** in Snowflake **caches table metadata** (e.g., statistics, schema, partition information) to **improve query performance**. Metadata caching is **automatic** and **transparent to users**.

```mermaid
%% Metadata Cache Architecture
flowchart TD
    subgraph Query["Query Layer"]
        A[("Query")] --> B[("Metadata Lookup")]
    end

    subgraph Cache["Metadata Cache Layer"]
        B --> C{Cache Hit?}
        C -->|Yes| D[("Return Cached Metadata")]
        C -->|No| E[("Fetch Metadata from Storage")]
        E --> F[("Cache Metadata")]
    end

    subgraph Execution["Execution Layer"]
        D --> G[("Query Execution Engine")]
        F --> G
    end

    subgraph Storage["Storage Layer"]
        E --> H[("Metadata Service")]
        H --> I[("Cloud Storage")]
    end

    subgraph Monitoring["Monitoring Layer"]
        J[("QUERY_HISTORY.compilation_time")]
    end
    G --> J

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef query fill:#4285f4,stroke:#1976d2;
    classDef cache fill:#ff9800,stroke:#f57c00;
    classDef execution fill:#009688,stroke:#00796b;
    classDef storage fill:#29abe2,stroke:#1a8fb8;
    classDef monitoring fill:#9c27b0,stroke:#7b1fa2;
    class A query;
    class B,C,D,E,F cache;
    class G execution;
    class H,I storage;
    class J monitoring;
```

#### **1. How Metadata Cache Works**
1. **Metadata Lookup**:
   - When a query is submitted, Snowflake **checks the metadata cache** for the required metadata (e.g., table statistics, schema, partition information).

2. **Cache Hit**:
   - If the metadata is **cached**, Snowflake **uses the cached metadata** for query optimization.

3. **Cache Miss**:
   - If the metadata is **not cached**, Snowflake **fetches the metadata** from the **metadata service** and **caches it** for future use.

4. **Cache Invalidation**:
   - The metadata cache is **automatically invalidated** when:
     - The **table schema changes** (e.g., ALTER TABLE).
     - The **table data changes significantly** (e.g., bulk load).
     - The **session ends**.
     - The **cache TTL expires** (session duration).

#### **2. Metadata Cached by Snowflake**
| **Metadata Type** | **Description** | **Usage** | **Cache Duration** |
|-------------------|-----------------|-----------|-------------------|
| **Table Statistics** | Row count, byte count, partition count | Query optimization, partition pruning | Session |
| **Column Statistics** | Min/max values, histograms, distinct counts | Query optimization, predicate pushdown | Session |
| **Schema Information** | Table schema, column data types | Query validation, optimization | Session |
| **Partition Information** | Micro-partition metadata (min/max values) | Partition pruning | Session |
| **Clustering Information** | Clustering keys, clustering depth | Partition pruning | Session |
| **Storage Information** | Storage usage, file formats | Query optimization | Session |
| **Index Information** | Search optimization indexes | Query optimization | Session |
| **Materialized View Information** | MV definitions, refresh status | Query rewriting | Session |

### **B. Metadata Cache Configuration**

**Note**: Metadata caching is **automatic** and **requires no configuration**. However, you can **influence** the metadata cache by:
- **Updating statistics** manually for large tables.
- **Avoiding frequent DDL changes** (which invalidate the cache).

**Update Statistics Manually**:
```sql
-- Update statistics for a table
ALTER TABLE my_table UPDATE STATISTICS;

-- Update statistics for specific columns
ALTER TABLE my_table UPDATE STATISTICS (col1, col2);

-- Update statistics for all tables in a schema
FOR table IN (
    SELECT table_name
    FROM INFORMATION_SCHEMA.TABLES
    WHERE table_schema = 'MY_SCHEMA'
) DO
    EXECUTE IMMEDIATE 'ALTER TABLE ' || table || ' UPDATE STATISTICS';
END FOR;
```

**Check Table Statistics**:
```sql
-- Check statistics for a table
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.TABLE_STATISTICS('MY_TABLE'));

-- Check statistics for specific columns
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.COLUMN_STATISTICS('MY_TABLE', 'COL1'));
```

### **C. Metadata Cache Performance**

#### **1. Performance Impact**
| **Metric** | **Without Metadata Cache** | **With Metadata Cache** | **Improvement** | **Notes** |
|------------|-------------------------------|----------------------------|-----------------|-----------|
| **Compilation Time** | 100-500 ms | 10-50 ms | 2-10x faster | Depends on table size and complexity |
| **Query Optimization** | Limited | Improved | Better execution plans | Enables partition pruning, predicate pushdown |
| **First Query Performance** | Slow | Fast | Faster subsequent queries | Metadata is cached after first query |
| **Concurrency** | Limited | Improved | Better resource utilization | Reduces metadata service load |

#### **2. Performance Example**
**Query**:
```sql
-- First query (metadata cache miss)
SELECT * FROM my_table WHERE date > '2023-01-01';

-- Subsequent queries (metadata cache hit)
SELECT * FROM my_table WHERE date > '2023-01-01';
```

**Performance Comparison**:
| **Execution** | **Compilation Time** | **Execution Time** | **Bytes Scanned** | **Credit Usage** |
|---------------|-----------------------|--------------------|-------------------|------------------|
| **First Query** | 300 ms | 5 seconds | 10 GB | 2.5 credits |
| **Subsequent Queries** | 50 ms | 5 seconds | 1 GB | 0.5 credits |

**Note**: The **execution time** remains the same, but the **compilation time** is reduced due to cached metadata. The **bytes scanned** may also be reduced due to **partition pruning** enabled by cached metadata.

### **D. Metadata Cache Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Update Statistics for Large Tables** | Ensure statistics are up-to-date for large tables | `ALTER TABLE my_table UPDATE STATISTICS` |
| **Update Statistics After Data Changes** | Update statistics after significant data changes | `ALTER TABLE my_table UPDATE STATISTICS` after bulk loads |
| **Avoid Frequent DDL Changes** | DDL changes invalidate metadata cache | Batch DDL changes |
| **Use Clustering for Better Metadata** | Clustering improves partition pruning | `ALTER TABLE my_table CLUSTER BY (date)` |
| **Monitor Metadata Cache Hits** | Check metadata cache usage (indirectly via query performance) | Improved query performance |
| **Use Consistent Table Schemas** | Avoid frequent schema changes | Design schemas carefully |
| **Test Query Performance** | Compare performance with and without metadata caching | Run queries and compare compilation times |
| **Document Metadata Strategies** | Document metadata management strategies | Internal wiki or Confluence page |
| **Use Automatic Statistics** | Let Snowflake update statistics automatically | No action needed (default) |
| **Monitor Statistics Freshness** | Check when statistics were last updated | `SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.TABLE_STATISTICS('MY_TABLE'))` |

### **E. Metadata Cache Limitations**

| **Limitation** | **Description** | **Workaround** |
|----------------|-----------------|---------------|
| **Session-Scoped Cache** | Metadata cache is session-scoped | Use connection pooling to reuse sessions |
| **Invalidated by DDL** | DDL changes invalidate metadata cache | Batch DDL changes |
| **Invalidated by Data Changes** | Significant data changes may invalidate cache | Update statistics manually |
| **No Partial Metadata Caching** | Cannot cache partial metadata | Use filtered tables or views |
| **No Custom Metadata** | Cannot customize metadata | Use Snowflake's built-in metadata |
| **No Metadata for External Tables** | Limited metadata for external tables | Use internal tables or materialized views |
| **No Direct Monitoring** | Metadata cache usage is not directly visible | Monitor compilation time and query performance |
| **Storage Overhead** | Metadata consumes storage | Monitor storage usage |

### **F. Metadata Cache Examples**

#### **Example 1: Update Statistics for Large Table**
```sql
-- Update statistics for a large table
ALTER TABLE large_table UPDATE STATISTICS;

-- Check table statistics
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.TABLE_STATISTICS('LARGE_TABLE'));

-- Check column statistics
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.COLUMN_STATISTICS('LARGE_TABLE', 'COL1'));
```

#### **Example 2: Monitor Metadata Usage**
```sql
-- Check if queries are using metadata for partition pruning
SELECT
    query_id,
    query_text,
    partitions_scanned,
    bytes_scanned
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%large_table%'
    AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY
    partitions_scanned;

-- Check compilation time (indirect indicator of metadata cache usage)
SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    compilation_time DESC;
```

#### **Example 3: Batch DDL Changes**
```sql
-- Bad: Frequent DDL changes (invalidates metadata cache)
ALTER TABLE my_table ADD COLUMN col1 INT;
ALTER TABLE my_table ADD COLUMN col2 INT;
ALTER TABLE my_table ADD COLUMN col3 INT;

-- Good: Batch DDL changes
ALTER TABLE my_table
ADD COLUMN col1 INT,
ADD COLUMN col2 INT,
ADD COLUMN col3 INT;
```

#### **Example 4: Update Statistics After Bulk Load**
```sql
-- Load data into a table
COPY INTO my_table FROM @my_stage;

-- Update statistics after bulk load
ALTER TABLE my_table UPDATE STATISTICS;

-- Check statistics
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.TABLE_STATISTICS('MY_TABLE'));
```

## **4. Local Disk Cache Deep Dive**

### **A. Definition and Architecture**

**Local Disk Cache** in Snowflake **caches frequently accessed data** in **local SSD storage** for each **virtual warehouse**. This cache is **automatically managed** by Snowflake and **transparent to users**.

```mermaid
%% Local Disk Cache Architecture
flowchart TD
    subgraph Warehouse["Warehouse Layer"]
        A[("Virtual Warehouse")] --> B[("Local Disk Cache\n(SSD)")]
    end

    subgraph Data["Data Layer"]
        C[("Cloud Storage")] -->|Data| D[("Micro-Partitions")]
    end

    subgraph Queries["Query Layer"]
        E[("Query")] --> A
        A --> F[("Cache Lookup")]
        F -->|Cache Hit| G[("Return Cached Data")]
        F -->|Cache Miss| H[("Fetch from Cloud Storage")]
        H --> B
    end

    subgraph Execution["Execution Layer"]
        E --> A
    end

    subgraph Monitoring["Monitoring Layer"]
        I[("QUERY_PROFILE")]
    end
    A --> I

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef warehouse fill:#4285f4,stroke:#1976d2;
    classDef data fill:#009688,stroke:#00796b;
    classDef queries fill:#ff9800,stroke:#f57c00;
    classDef execution fill:#29abe2,stroke:#1a8fb8;
    classDef monitoring fill:#9c27b0,stroke:#7b1fa2;
    class A warehouse;
    class B data;
    class C,D data;
    class E,F,G,H queries;
    class I monitoring;
```

#### **1. How Local Disk Cache Works**
1. **Data Fetching**:
   - When a query is executed, Snowflake **fetches data** from **cloud storage** (S3, Azure Blob, GCS) into the **warehouse's local SSD cache**.

2. **Cache Population**:
   - Frequently accessed **micro-partitions** are **cached in local SSD** for faster access.

3. **Cache Lookup**:
   - For subsequent queries, Snowflake **checks the local disk cache** first.
   - If the data is **cached**, it is **returned from SSD** (faster than cloud storage).
   - If the data is **not cached**, it is **fetched from cloud storage**.

4. **Cache Management**:
   - The local disk cache is **automatically managed** by Snowflake.
   - **Cache size**: Limited by the **warehouse size** (larger warehouses have more cache).
   - **Cache eviction**: Least recently used (LRU) data is **evicted** when the cache is full.

5. **Cache Scope**:
   - **Per-warehouse**: Each warehouse has its own local disk cache.
   - **Per-session**: Data cached during a session may be reused for subsequent queries in the same session.

### **B. Local Disk Cache Configuration**

**Note**: Local disk cache is **automatic** and **requires no configuration**. However, you can **influence** the cache by:
- **Using larger warehouses** (more cache capacity).
- **Querying the same data repeatedly** (populates the cache).

**Check Warehouse Cache Size**:
```sql
-- Check warehouse size (indirect indicator of cache size)
SELECT
    warehouse_name,
    warehouse_size,
    state
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSES
WHERE
    warehouse_name = 'MY_WH';
```

**Monitor Cache Usage**:
```sql
-- Check if queries are using local disk cache (indirectly via QUERY_PROFILE)
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'));
-- Look for cached data in the profile
```

### **C. Local Disk Cache Performance**

#### **1. Performance Impact**
| **Metric** | **Without Local Disk Cache** | **With Local Disk Cache** | **Improvement** | **Notes** |
|------------|--------------------------------|-----------------------------|-----------------|-----------|
| **I/O Latency** | 100-500 ms (cloud storage) | 1-10 ms (SSD) | 10-100x faster | Depends on cloud storage latency |
| **Throughput** | Limited by cloud storage | Limited by SSD | 2-10x higher | Depends on warehouse size |
| **Credit Usage** | Higher (more time spent waiting for I/O) | Lower (less time waiting for I/O) | 1.1-2x reduction | Indirect impact via faster queries |
| **Concurrency** | Limited by I/O | Improved | Better resource utilization | Reduces I/O bottlenecks |

#### **2. Performance Example**
**Query**:
```sql
-- First query (cache miss)
SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7;

-- Subsequent queries (cache hit)
SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7;
```

**Performance Comparison**:
| **Execution** | **Execution Time** | **I/O Latency** | **Credit Usage** | **Notes** |
|---------------|--------------------|-----------------|------------------|-----------|
| **First Query** | 5 seconds | 200 ms | 2.5 credits | Data fetched from cloud storage |
| **Subsequent Queries** | 1 second | 5 ms | 0.5 credits | Data returned from local SSD cache |

### **D. Local Disk Cache Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use Larger Warehouses for Hot Data** | Larger warehouses have more local disk cache | `CREATE WAREHOUSE my_wh WAREHOUSE_SIZE = 'X-LARGE'` |
| **Query the Same Data Repeatedly** | Populate the cache with frequently accessed data | Dashboards, reports, BI tools |
| **Use Clustering for Cache Efficiency** | Cluster tables to improve cache hit rates | `ALTER TABLE my_table CLUSTER BY (date)` |
| **Monitor Cache Performance** | Check QUERY_PROFILE for cache hits | `SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))` |
| **Avoid Frequent Warehouse Restarts** | Cache is lost when warehouse is restarted | Use auto-suspend/resume instead of stopping/starting |
| **Use Multi-Cluster Warehouses for High Concurrency** | Distribute cache across multiple clusters | `CREATE WAREHOUSE my_wh MAX_CLUSTER_COUNT = 4` |
| **Combine with Result Caching** | Use both local disk cache and result caching | `ALTER SESSION SET USE_CACHED_RESULTS = TRUE` |
| **Monitor Warehouse Utilization** | Check WAREHOUSE_LOAD_HISTORY for cache efficiency | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY` |
| **Use Auto-Suspend for Idle Warehouses** | Suspend warehouses when idle to save costs | `ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = 600` |
| **Document Cacheable Data** | Document which tables benefit from caching | Internal wiki or Confluence page |

### **E. Local Disk Cache Limitations**

| **Limitation** | **Description** | **Workaround** |
|----------------|-----------------|---------------|
| **Per-Warehouse Cache** | Each warehouse has its own cache | Use larger warehouses for hot data |
| **Limited by Warehouse Size** | Cache size is limited by warehouse size | Use larger warehouses for more cache |
| **LRU Eviction** | Least recently used data is evicted when cache is full | Query important data frequently to keep it in cache |
| **No Direct Control** | Cannot manually manage cache | Use clustering and query patterns to influence cache |
| **Cache Lost on Warehouse Restart** | Cache is lost when warehouse is stopped or restarted | Use auto-suspend/resume instead of stop/start |
| **No Monitoring for Cache Hits** | Cache hits are not directly visible | Monitor query performance and I/O latency |
| **No Caching for External Tables** | Limited caching for external tables | Use internal tables or materialized views |
| **Storage Overhead** | Cache consumes SSD storage | Monitor warehouse storage usage |

### **F. Local Disk Cache Examples**

#### **Example 1: Cache Hot Tables**
```sql
-- Create a large warehouse for hot tables
CREATE WAREHOUSE hot_data_wh
  WAREHOUSE_SIZE = 'X-LARGE'
  AUTO_SUSPEND = 1800
  AUTO_RESUME = TRUE;

-- Query hot tables repeatedly to populate cache
SELECT * FROM customers WHERE region = 'US';
SELECT * FROM products WHERE category = 'Electronics';
SELECT * FROM orders WHERE date > CURRENT_DATE() - 7;

-- Subsequent queries will benefit from local disk cache
SELECT * FROM customers WHERE region = 'US';
```

#### **Example 2: Monitor Cache Performance**
```sql
-- Check query profile for cache usage
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'));

-- Check for cached data in the profile
-- Look for steps with low execution_time and bytes_scanned

-- Compare performance with and without cache
-- First query (cache miss)
SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7;

-- Subsequent queries (cache hit)
SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7;
```

#### **Example 3: Use Clustering for Cache Efficiency**
```sql
-- Cluster a table to improve cache hit rates
ALTER TABLE my_table CLUSTER BY (date, region);

-- Query the table with filters on clustered columns
SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7 AND region = 'US';

-- Subsequent queries will benefit from both clustering and local disk cache
SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7 AND region = 'US';
```

#### **Example 4: Multi-Cluster Warehouse for High Concurrency**
```sql
-- Create a multi-cluster warehouse for high concurrency
CREATE WAREHOUSE high_concurrency_wh
  WAREHOUSE_SIZE = 'LARGE'
  MAX_CLUSTER_COUNT = 4
  MIN_CLUSTER_COUNT = 1
  SCALING_POLICY = 'STANDARD'
  AUTO_SUSPEND = 1800
  AUTO_RESUME = TRUE;

-- Query the warehouse with multiple concurrent queries
-- Each cluster will have its own local disk cache
SELECT * FROM table1 WHERE date > CURRENT_DATE() - 7;
SELECT * FROM table2 WHERE region = 'US';
SELECT * FROM table3 WHERE category = 'Electronics';
```

## **5. Query Cache Deep Dive**

### **A. Definition and Architecture**

**Query Cache** in Snowflake **caches query plans and compiled queries** to **improve query performance** for **repetitive or similar queries**. Query caching is **automatic** and **transparent to users**.

```mermaid
%% Query Cache Architecture
flowchart TD
    subgraph Query["Query Layer"]
        A[("Query")] --> B[("Query Parser")]
    end

    subgraph Cache["Query Cache Layer"]
        B --> C[("Query Plan Cache")]
        B --> D[("Compiled Query Cache")]
        C --> E[("Cache Lookup")]
        D --> E
        E -->|Cache Hit| F[("Reuse Cached Plan")]
        E -->|Cache Miss| G[("Compile Query")]
        G --> C
        G --> D
    end

    subgraph Execution["Execution Layer"]
        F --> H[("Query Execution Engine")]
        G --> H
    end

    subgraph Monitoring["Monitoring Layer"]
        I[("QUERY_HISTORY.compilation_time")]
    end
    H --> I

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef query fill:#4285f4,stroke:#1976d2;
    classDef cache fill:#ff9800,stroke:#f57c00;
    classDef execution fill:#009688,stroke:#00796b;
    classDef monitoring fill:#9c27b0,stroke:#7b1fa2;
    class A query;
    class B,C,D,E,F,G cache;
    class H execution;
    class I monitoring;
```

#### **1. How Query Cache Works**
1. **Query Parsing**:
   - When a query is submitted, Snowflake **parses the query** and **generates a query plan**.

2. **Cache Lookup**:
   - Snowflake **checks the query cache** for a matching query plan or compiled query.
   - A **cache hit** occurs if:
     - The **query text** is identical or similar to a cached query.
     - The **table schemas** are the same.
     - The **session parameters** are the same.

3. **Cache Hit**:
   - If a cache hit occurs, Snowflake **reuses the cached query plan or compiled query**.
   - The **compilation_time** in `QUERY_HISTORY` will be **very low** (typically <10ms).

4. **Cache Miss**:
   - If no cache hit occurs, Snowflake **compiles the query** normally.
   - After compilation, Snowflake **caches the query plan and compiled query** for future use.

5. **Cache Invalidation**:
   - The query cache is **automatically invalidated** if:
     - The **table schema changes** (e.g., ALTER TABLE).
     - The **query text changes significantly**.
     - The **session parameters change** (e.g., time zone, role).
     - The **cache TTL expires** (session duration).

#### **2. Query Cache Types**
| **Cache Type** | **Description** | **Scope** | **TTL** | **Performance Impact** |
|----------------|-----------------|-----------|---------|-------------------------|
| **Query Plan Cache** | Caches the **query execution plan** | Per-session | Session | ⭐⭐ (1.1-2x faster) |
| **Compiled Query Cache** | Caches the **compiled query** (ready to execute) | Per-session | Session | ⭐⭐⭐ (2-5x faster) |
| **Parameterized Query Cache** | Caches **parameterized queries** (e.g., `SELECT * FROM my_table WHERE id = ?`) | Per-session | Session | ⭐⭐⭐ (2-5x faster) |

### **B. Query Cache Configuration**

**Note**: Query caching is **automatic** and **requires no configuration**. However, you can **influence** the query cache by:
- **Using parameterized queries** (enables parameterized query caching).
- **Avoiding frequent DDL changes** (which invalidate the cache).

**Use Parameterized Queries**:
```sql
-- Use parameterized queries for better cache reuse
-- In your application code, use:
-- PreparedStatement stmt = connection.prepareStatement("SELECT * FROM my_table WHERE id = ?");
-- stmt.setInt(1, 123);
-- ResultSet rs = stmt.executeQuery();

-- In Snowsight or other tools, use bind variables
SELECT * FROM my_table WHERE id = ?;
```

### **C. Query Cache Performance**

#### **1. Performance Impact**
| **Metric** | **Without Query Cache** | **With Query Cache (Hit)** | **With Query Cache (Miss)** | **Improvement (Hit)** |
|------------|---------------------------|-----------------------------|-------------------------------|-------------------------|
| **Compilation Time** | 100-500 ms | 1-10 ms | 100-500 ms | 10-100x faster |
| **Execution Time** | Unchanged | Unchanged | Unchanged | No change |
| **Credit Usage** | Unchanged | Unchanged | Unchanged | No change |
| **Concurrency** | Limited by compilation | Improved | Limited by compilation | Better resource utilization |

#### **2. Performance Example**
**Query**:
```sql
-- First query (cache miss)
SELECT * FROM my_table WHERE date > '2023-01-01' AND region = 'US';

-- Subsequent queries (cache hit)
SELECT * FROM my_table WHERE date > '2023-01-01' AND region = 'US';
SELECT * FROM my_table WHERE date > '2023-01-02' AND region = 'EU';
```

**Performance Comparison**:
| **Execution** | **Compilation Time** | **Execution Time** | **Credit Usage** | **Notes** |
|---------------|-----------------------|--------------------|------------------|-----------|
| **First Query** | 300 ms | 5 seconds | 2.5 credits | Query compiled and executed |
| **Subsequent Queries** | 10 ms | 5 seconds | 2.5 credits | Reused cached query plan |

### **D. Query Cache Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use Parameterized Queries** | Enables parameterized query caching | `SELECT * FROM my_table WHERE id = ?` |
| **Use Consistent Query Structure** | Use similar query structures for better cache reuse | `SELECT col1, col2 FROM my_table WHERE ...` |
| **Avoid Dynamic SQL** | Dynamic SQL prevents query caching | Use parameterized queries instead |
| **Use Stored Procedures** | Stored procedures can reuse cached plans | `CREATE PROCEDURE my_proc() AS SELECT * FROM my_table;` |
| **Monitor Compilation Time** | Check `compilation_time` in `QUERY_HISTORY` | `SELECT compilation_time FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` |
| **Avoid Frequent DDL Changes** | DDL changes invalidate query cache | Batch DDL changes |
| **Use Query Tags** | Tag queries for better cache management | `ALTER SESSION SET QUERY_TAG = 'dashboard=main'` |
| **Document Query Patterns** | Document repetitive query patterns for caching | Internal wiki or Confluence page |
| **Test Query Performance** | Compare performance with and without query caching | Run queries and compare compilation times |
| **Use Connection Pooling** | Reuse connections to reuse cached query plans | Configure connection pooling in your application |

### **E. Query Cache Limitations**

| **Limitation** | **Description** | **Workaround** |
|----------------|-----------------|---------------|
| **Session-Scoped Cache** | Query cache is session-scoped | Use connection pooling to reuse sessions |
| **Invalidated by DDL** | DDL changes invalidate query cache | Batch DDL changes |
| **Invalidated by Schema Changes** | Schema changes invalidate query cache | Avoid frequent schema changes |
| **No Direct Monitoring** | Query cache usage is not directly visible | Monitor compilation time |
| **No Caching for All Queries** | Some complex queries cannot be cached | Use materialized views or result caching |
| **No Caching for External Tables** | Limited caching for external tables | Use internal tables or materialized views |
| **No Custom Cache Size** | Cannot configure query cache size | Use parameterized queries for better reuse |
| **Memory Overhead** | Query cache consumes memory | Monitor warehouse memory usage |

### **F. Query Cache Examples**

#### **Example 1: Parameterized Queries**
```sql
-- Use parameterized queries in your application code
-- Java example:
PreparedStatement stmt = connection.prepareStatement(
    "SELECT * FROM my_table WHERE id = ? AND date > ?"
);
stmt.setInt(1, 123);
stmt.setDate(2, Date.valueOf("2023-01-01"));
ResultSet rs = stmt.executeQuery();

-- Python example (using Snowflake Connector):
cursor = conn.cursor()
cursor.execute("SELECT * FROM my_table WHERE id = %s AND date > %s", (123, '2023-01-01'))
results = cursor.fetchall()
```

#### **Example 2: Monitor Compilation Time**
```sql
-- Check compilation time for recent queries
SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    compilation_time DESC;

-- Check for queries with high compilation time
SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    compilation_time > 100  -- >100ms
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    compilation_time DESC;
```

#### **Example 3: Use Stored Procedures for Caching**
```sql
-- Create a stored procedure
CREATE PROCEDURE get_customer_orders(customer_id INT)
RETURNS TABLE ()
AS
$$
  SELECT * FROM orders WHERE customer_id = customer_id;
$$;

-- Call the stored procedure (reuses cached plan)
CALL get_customer_orders(123);
CALL get_customer_orders(456);
```

#### **Example 4: Compare Performance with and without Query Cache**
```sql
-- First query (cache miss)
SELECT * FROM my_table WHERE date > '2023-01-01';

-- Subsequent queries (cache hit)
SELECT * FROM my_table WHERE date > '2023-01-01';
SELECT * FROM my_table WHERE date > '2023-01-02';

-- Compare compilation times
SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%my_table%date%'
ORDER BY
    compilation_time;
```

## **6. File Metadata Cache Deep Dive**

### **A. Definition and Architecture**

**File Metadata Cache** in Snowflake **caches metadata for external tables** (e.g., file formats, stages, external locations) to **improve query performance**. File metadata caching is **automatic** and **transparent to users**.

```mermaid
%% File Metadata Cache Architecture
flowchart TD
    subgraph Query["Query Layer"]
        A[("Query on External Table")] --> B[("File Metadata Lookup")]
    end

    subgraph Cache["File Metadata Cache Layer"]
        B --> C{Cache Hit?}
        C -->|Yes| D[("Return Cached Metadata")]
        C -->|No| E[("Fetch Metadata from Cloud Storage")]
        E --> F[("Cache Metadata")]
    end

    subgraph Execution["Execution Layer"]
        D --> G[("Query Execution Engine")]
        F --> G
    end

    subgraph Storage["Storage Layer"]
        E --> H[("Cloud Storage\n(S3, Azure Blob, GCS)")]
    end

    subgraph Monitoring["Monitoring Layer"]
        I[("QUERY_HISTORY.compilation_time")]
    end
    G --> I

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef query fill:#4285f4,stroke:#1976d2;
    classDef cache fill:#ff9800,stroke:#f57c00;
    classDef execution fill:#009688,stroke:#00796b;
    classDef storage fill:#29abe2,stroke:#1a8fb8;
    classDef monitoring fill:#9c27b0,stroke:#7b1fa2;
    class A query;
    class B,C,D,E,F cache;
    class G execution;
    class H storage;
    class I monitoring;
```

#### **1. How File Metadata Cache Works**
1. **Metadata Lookup**:
   - When a query on an **external table** is submitted, Snowflake **checks the file metadata cache** for the required metadata (e.g., file formats, stages, external locations).

2. **Cache Hit**:
   - If the metadata is **cached**, Snowflake **uses the cached metadata** for query optimization.

3. **Cache Miss**:
   - If the metadata is **not cached**, Snowflake **fetches the metadata** from **cloud storage** and **caches it** for future use.

4. **Cache Invalidation**:
   - The file metadata cache is **automatically invalidated** when:
     - The **external table definition changes** (e.g., ALTER EXTERNAL TABLE).
     - The **stage definition changes** (e.g., ALTER STAGE).
     - The **file format changes** (e.g., ALTER FILE FORMAT).
     - The **session ends**.
     - The **cache TTL expires** (session duration).

#### **2. File Metadata Cached by Snowflake**
| **Metadata Type** | **Description** | **Usage** | **Cache Duration** |
|-------------------|-----------------|-----------|-------------------|
| **Stage Metadata** | Stage URL, credentials, file formats | Query optimization, file access | Session |
| **File Format Metadata** | File format type, options (e.g., compression, field delimiter) | Query optimization, file parsing | Session |
| **External Table Metadata** | External table definition, column mappings | Query optimization, file access | Session |
| **File Metadata** | File sizes, last modified times, partitions | Partition pruning, file access | Session |
| **Partition Metadata** | Partition columns, partition values | Partition pruning | Session |

### **B. File Metadata Cache Configuration**

**Note**: File metadata caching is **automatic** and **requires no configuration**. However, you can **influence** the cache by:
- **Using consistent stage and file format definitions**.
- **Avoiding frequent changes to external table definitions**.

**Check External Table Metadata**:
```sql
-- Check external table metadata
SELECT * FROM INFORMATION_SCHEMA.EXTERNAL_TABLES
WHERE table_name = 'MY_EXTERNAL_TABLE';

-- Check stage metadata
SELECT * FROM INFORMATION_SCHEMA.STAGES
WHERE stage_name = 'MY_STAGE';

-- Check file format metadata
SELECT * FROM INFORMATION_SCHEMA.FILE_FORMATS
WHERE file_format_name = 'MY_FILE_FORMAT';
```

### **C. File Metadata Cache Performance**

#### **1. Performance Impact**
| **Metric** | **Without File Metadata Cache** | **With File Metadata Cache** | **Improvement** | **Notes** |
|------------|-------------------------------------|----------------------------------|-----------------|-----------|
| **Compilation Time** | 500-2000 ms | 50-200 ms | 5-10x faster | Depends on number of files |
| **Query Optimization** | Limited | Improved | Better execution plans | Enables partition pruning |
| **First Query Performance** | Slow | Fast | Faster subsequent queries | Metadata is cached after first query |
| **Concurrency** | Limited | Improved | Better resource utilization | Reduces cloud storage metadata lookups |

#### **2. Performance Example**
**Query**:
```sql
-- First query on external table (cache miss)
SELECT * FROM my_external_table WHERE date > '2023-01-01';

-- Subsequent queries (cache hit)
SELECT * FROM my_external_table WHERE date > '2023-01-01';
```

**Performance Comparison**:
| **Execution** | **Compilation Time** | **Execution Time** | **Bytes Scanned** | **Credit Usage** |
|---------------|-----------------------|--------------------|-------------------|------------------|
| **First Query** | 1000 ms | 10 seconds | 100 GB | 5 credits |
| **Subsequent Queries** | 100 ms | 10 seconds | 10 GB | 0.5 credits |

**Note**: The **execution time** remains the same, but the **compilation time** is reduced due to cached file metadata. The **bytes scanned** may also be reduced due to **partition pruning** enabled by cached metadata.

### **D. File Metadata Cache Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use Consistent Stage Definitions** | Avoid changing stage definitions frequently | `CREATE STAGE my_stage URL = 's3://my-bucket/'` |
| **Use Consistent File Formats** | Avoid changing file format definitions frequently | `CREATE FILE FORMAT my_format TYPE = 'PARQUET'` |
| **Use Partitioned External Tables** | Partition external tables for better performance | `CREATE EXTERNAL TABLE my_table PARTITION BY (date)` |
| **Monitor File Metadata Cache Hits** | Check compilation time for external table queries | `SELECT compilation_time FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` |
| **Avoid Frequent External Table Changes** | Changes to external tables invalidate the cache | Use consistent external table definitions |
| **Use External Table Clustering** | Cluster external tables for better performance | `ALTER EXTERNAL TABLE my_table CLUSTER BY (date)` |
| **Document External Table Configurations** | Document stage, file format, and external table configurations | Internal wiki or Confluence page |
| **Test Query Performance** | Compare performance with and without file metadata caching | Run queries and compare compilation times |

### **E. File Metadata Cache Limitations**

| **Limitation** | **Description** | **Workaround** |
|----------------|-----------------|---------------|
| **Session-Scoped Cache** | File metadata cache is session-scoped | Use connection pooling to reuse sessions |
| **Invalidated by DDL** | DDL changes invalidate file metadata cache | Batch DDL changes |
| **Invalidated by External Changes** | Changes to external files may not be reflected | Use `REFRESH` for external tables |
| **No Direct Monitoring** | File metadata cache usage is not directly visible | Monitor compilation time and query performance |
| **No Caching for All External Tables** | Some external table configurations may not be cached | Use internal tables or materialized views |
| **Cloud Storage Latency** | File metadata cache does not eliminate cloud storage latency | Use local stages or internal tables |
| **Storage Overhead** | File metadata consumes storage | Monitor storage usage |

### **F. File Metadata Cache Examples**

#### **Example 1: Partitioned External Table**
```sql
-- Create a stage
CREATE STAGE my_s3_stage URL = 's3://my-bucket/sales/';

-- Create a file format
CREATE FILE FORMAT my_parquet_format TYPE = 'PARQUET';

-- Create a partitioned external table
CREATE EXTERNAL TABLE my_partitioned_table (
    sale_id INTEGER,
    sale_date DATE,
    amount FLOAT
)
WITH LOCATION = @my_s3_stage
FILE_FORMAT = (TYPE = 'PARQUET')
PARTITION BY (YEAR(sale_date), MONTH(sale_date));

-- Query the external table (first query populates cache)
SELECT * FROM my_partitioned_table
WHERE YEAR(sale_date) = 2023 AND MONTH(sale_date) = 1;

-- Subsequent queries benefit from cached file metadata
SELECT * FROM my_partitioned_table
WHERE YEAR(sale_date) = 2023 AND MONTH(sale_date) = 2;
```

#### **Example 2: Refresh External Table Metadata**
```sql
-- Refresh external table metadata (if external files have changed)
ALTER EXTERNAL TABLE my_external_table REFRESH;

-- Check external table metadata
SELECT * FROM INFORMATION_SCHEMA.EXTERNAL_TABLES
WHERE table_name = 'MY_EXTERNAL_TABLE';
```

#### **Example 3: Monitor File Metadata Cache Performance**
```sql
-- Check compilation time for external table queries
SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%my_external_table%'
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    compilation_time DESC;
```

#### **Example 4: Use Clustering with External Tables**
```sql
-- Create a clustered external table
CREATE EXTERNAL TABLE my_clustered_external_table (
    sale_id INTEGER,
    sale_date DATE,
    amount FLOAT,
    region STRING
)
WITH LOCATION = @my_s3_stage
FILE_FORMAT = (TYPE = 'PARQUET')
CLUSTER BY (sale_date, region);

-- Query the external table with filters on clustered columns
SELECT * FROM my_clustered_external_table
WHERE sale_date > '2023-01-01' AND region = 'US';
```

## **7. Warehouse Cache Deep Dive**

### **A. Definition and Architecture**

**Warehouse Cache** in Snowflake **caches warehouse-specific data** (e.g., temporary tables, intermediate results) to **improve query performance**. Warehouse caching is **automatic** and **transparent to users**.

```mermaid
%% Warehouse Cache Architecture
flowgraph TD
    subgraph Warehouse["Warehouse Layer"]
        A[("Virtual Warehouse")] --> B[("Warehouse Cache\n(SSD)")]
    end

    subgraph Data["Data Layer"]
        C[("Temporary Tables")] --> B
        D[("Intermediate Results")] --> B
        E[("Spill Data")] --> B
    end

    subgraph Queries["Query Layer"]
        F[("Query")] --> A
        A --> G[("Cache Lookup")]
        G -->|Cache Hit| H[("Return Cached Data")]
        G -->|Cache Miss| I[("Execute Query")]
        I --> B
    end

    subgraph Execution["Execution Layer"]
        F --> A
    end

    subgraph Monitoring["Monitoring Layer"]
        J[("QUERY_PROFILE")]
    end
    A --> J

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef warehouse fill:#4285f4,stroke:#1976d2;
    classDef data fill:#009688,stroke:#00796b;
    classDef queries fill:#ff9800,stroke:#f57c00;
    classDef execution fill:#29abe2,stroke:#1a8fb8;
    classDef monitoring fill:#9c27b0,stroke:#7b1fa2;
    class A warehouse;
    class B data;
    class C,D,E data;
    class F,G,H,I queries;
    class J monitoring;
```

#### **1. How Warehouse Cache Works**
1. **Data Caching**:
   - When a query is executed, Snowflake **caches warehouse-specific data** in **local SSD**, including:
     - **Temporary tables**: Tables created with `CREATE TEMPORARY TABLE`.
     - **Intermediate results**: Results from subqueries or CTEs.
     - **Spill data**: Data spilled to disk during query execution.

2. **Cache Lookup**:
   - For subsequent queries, Snowflake **checks the warehouse cache** first.
   - If the data is **cached**, it is **returned from SSD** (faster than recomputing).

3. **Cache Management**:
   - The warehouse cache is **automatically managed** by Snowflake.
   - **Cache size**: Limited by the **warehouse size** (larger warehouses have more cache).
   - **Cache eviction**: Least recently used (LRU) data is **evicted** when the cache is full.

4. **Cache Scope**:
   - **Per-warehouse**: Each warehouse has its own cache.
   - **Per-session**: Data cached during a session may be reused for subsequent queries in the same session.

### **B. Warehouse Cache Configuration**

**Note**: Warehouse cache is **automatic** and **requires no configuration**. However, you can **influence** the cache by:
- **Using larger warehouses** (more cache capacity).
- **Using temporary tables** for intermediate results.

**Create a Temporary Table**:
```sql
-- Create a temporary table (cached in warehouse cache)
CREATE TEMPORARY TABLE temp_sales AS
SELECT * FROM sales WHERE date > CURRENT_DATE() - 7;

-- Query the temporary table (benefits from warehouse cache)
SELECT * FROM temp_sales WHERE region = 'US';
```

**Use Warehouse-Specific Data**:
```sql
-- Use warehouse cache for intermediate results
WITH cte AS (
    SELECT * FROM sales WHERE date > CURRENT_DATE() - 7
)
SELECT * FROM cte WHERE region = 'US';
-- The CTE results may be cached in the warehouse cache
```

### **C. Warehouse Cache Performance**

#### **1. Performance Impact**
| **Metric** | **Without Warehouse Cache** | **With Warehouse Cache** | **Improvement** | **Notes** |
|------------|--------------------------------|----------------------------|-----------------|-----------|
| **I/O Latency** | 100-500 ms (cloud storage) | 1-10 ms (SSD) | 10-100x faster | Depends on warehouse size |
| **Throughput** | Limited by cloud storage | Limited by SSD | 2-10x higher | Depends on warehouse size |
| **Credit Usage** | Higher (more time spent waiting for I/O) | Lower (less time waiting for I/O) | 1.1-2x reduction | Indirect impact via faster queries |
| **Concurrency** | Limited by I/O | Improved | Better resource utilization | Reduces I/O bottlenecks |

#### **2. Performance Example**
**Query**:
```sql
-- First query (cache miss)
CREATE TEMPORARY TABLE temp_sales AS
SELECT * FROM sales WHERE date > CURRENT_DATE() - 7;

SELECT * FROM temp_sales WHERE region = 'US';

-- Subsequent queries (cache hit)
SELECT * FROM temp_sales WHERE region = 'US';
```

**Performance Comparison**:
| **Execution** | **Execution Time** | **I/O Latency** | **Credit Usage** | **Notes** |
|---------------|--------------------|-----------------|------------------|-----------|
| **First Query** | 5 seconds | 200 ms | 2.5 credits | Data fetched from cloud storage |
| **Subsequent Queries** | 100 ms | 5 ms | 0.1 credits | Data returned from warehouse cache |

### **D. Warehouse Cache Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use Temporary Tables for Intermediate Results** | Temporary tables are cached in warehouse cache | `CREATE TEMPORARY TABLE temp_results AS SELECT ...` |
| **Use CTEs for Complex Queries** | CTE results may be cached in warehouse cache | `WITH cte AS (SELECT ...) SELECT * FROM cte` |
| **Use Larger Warehouses for Cache-Intensive Workloads** | Larger warehouses have more cache capacity | `CREATE WAREHOUSE my_wh WAREHOUSE_SIZE = 'X-LARGE'` |
| **Reuse Temporary Tables** | Reuse temporary tables to benefit from caching | `SELECT * FROM temp_table WHERE ...` |
| **Avoid Frequent Warehouse Restarts** | Cache is lost when warehouse is restarted | Use auto-suspend/resume instead of stopping/starting |
| **Use Multi-Cluster Warehouses for High Concurrency** | Distribute cache across multiple clusters | `CREATE WAREHOUSE my_wh MAX_CLUSTER_COUNT = 4` |
| **Monitor Cache Performance** | Check QUERY_PROFILE for cache hits | `SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))` |
| **Use Auto-Suspend for Idle Warehouses** | Suspend warehouses when idle to save costs | `ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = 600` |
| **Document Cacheable Data** | Document which data benefits from caching | Internal wiki or Confluence page |

### **E. Warehouse Cache Limitations**

| **Limitation** | **Description** | **Workaround** |
|----------------|-----------------|---------------|
| **Per-Warehouse Cache** | Each warehouse has its own cache | Use larger warehouses for hot data |
| **Limited by Warehouse Size** | Cache size is limited by warehouse size | Use larger warehouses for more cache |
| **LRU Eviction** | Least recently used data is evicted when cache is full | Query important data frequently to keep it in cache |
| **No Direct Control** | Cannot manually manage cache | Use temporary tables and CTEs to influence cache |
| **Cache Lost on Warehouse Restart** | Cache is lost when warehouse is stopped or restarted | Use auto-suspend/resume instead of stop/start |
| **No Monitoring for Cache Hits** | Cache hits are not directly visible | Monitor query performance and I/O latency |
| **Temporary Tables Only** | Only temporary tables and intermediate results are cached | Use temporary tables for intermediate results |
| **Session-Scoped** | Temporary tables are session-scoped | Use global temporary tables for cross-session caching |
| **Storage Overhead** | Cache consumes SSD storage | Monitor warehouse storage usage |

### **F. Warehouse Cache Examples**

#### **Example 1: Temporary Tables**
```sql
-- Create a temporary table (cached in warehouse cache)
CREATE TEMPORARY TABLE temp_sales AS
SELECT * FROM sales WHERE date > CURRENT_DATE() - 7;

-- Query the temporary table (benefits from warehouse cache)
SELECT * FROM temp_sales WHERE region = 'US';

-- Reuse the temporary table in subsequent queries
SELECT region, SUM(amount) AS total_sales
FROM temp_sales
GROUP BY region;
```

#### **Example 2: CTEs for Intermediate Results**
```sql
-- Use CTEs for intermediate results (may be cached in warehouse cache)
WITH sales_cte AS (
    SELECT * FROM sales WHERE date > CURRENT_DATE() - 7
)
SELECT
    region,
    SUM(amount) AS total_sales
FROM
    sales_cte
GROUP BY
    region;

-- Reuse the CTE in subsequent queries
WITH sales_cte AS (
    SELECT * FROM sales WHERE date > CURRENT_DATE() - 7
)
SELECT
    product_category,
    SUM(amount) AS total_sales
FROM
    sales_cte
GROUP BY
    product_category;
```

#### **Example 3: Monitor Warehouse Cache Performance**
```sql
-- Check query profile for warehouse cache usage
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'));

-- Look for cached data in the profile (e.g., low execution_time for temporary tables)

-- Compare performance with and without warehouse cache
-- First query (cache miss)
CREATE TEMPORARY TABLE temp_sales AS SELECT * FROM sales WHERE date > CURRENT_DATE() - 7;
SELECT * FROM temp_sales WHERE region = 'US';

-- Subsequent queries (cache hit)
SELECT * FROM temp_sales WHERE region = 'US';
```

#### **Example 4: Multi-Cluster Warehouse for High Concurrency**
```sql
-- Create a multi-cluster warehouse for high concurrency
CREATE WAREHOUSE high_concurrency_wh
  WAREHOUSE_SIZE = 'LARGE'
  MAX_CLUSTER_COUNT = 4
  MIN_CLUSTER_COUNT = 1
  SCALING_POLICY = 'STANDARD'
  AUTO_SUSPEND = 1800
  AUTO_RESUME = TRUE;

-- Create temporary tables in the warehouse
CREATE TEMPORARY TABLE temp_table1 AS SELECT * FROM table1;
CREATE TEMPORARY TABLE temp_table2 AS SELECT * FROM table2;

-- Query the temporary tables (each cluster has its own cache)
SELECT * FROM temp_table1 WHERE col1 = 'value';
SELECT * FROM temp_table2 WHERE col2 = 'value';
```

## **8. Caching Decision Matrix**

### **Mermaid: Snowflake Caching Decision Tree**
```mermaid
%% Snowflake Caching Decision Tree
flowchart TD
    A[("Query Performance\nIssue")] --> B{Query Type?}
    B -->|Repetitive, Identical| C[("Result Cache")]
    B -->|Frequently Accessed Data| D[("Local Disk Cache")]
    B -->|Complex Queries| E[("Query Cache")]
    B -->|External Tables| F[("File Metadata Cache")]
    B -->|Temporary Data| G[("Warehouse Cache")]
    B -->|Metadata Access| H[("Metadata Cache")]

    C --> I[("Enable Result Caching\n(USE_CACHED_RESULTS)")]
    D --> J[("Use Larger Warehouses\n(CREATE WAREHOUSE ...)")]
    E --> K[("Use Parameterized Queries\n(SELECT * FROM table WHERE id = ?)")]
    F --> L[("Use Partitioned External Tables\n(CREATE EXTERNAL TABLE ... PARTITION BY)")]
    G --> M[("Use Temporary Tables\n(CREATE TEMPORARY TABLE)")]
    H --> N[("Update Statistics\n(ALTER TABLE ... UPDATE STATISTICS)")]

    I --> O[("Set Appropriate TTL\n(RESULT_CACHE_TTL)")]
    J --> P[("Query Hot Data Repeatedly")]
    K --> Q[("Use Connection Pooling")]
    L --> R[("Refresh External Table Metadata\n(ALTER EXTERNAL TABLE ... REFRESH)")]
    M --> S[("Reuse Temporary Tables")]
    N --> T[("Avoid Frequent DDL Changes")]

    O --> U[("Use for Dashboards, Reports")]
    P --> V[("Use for BI Tools, Analytics")]
    Q --> W[("Use for Application Queries")]
    R --> X[("Use for External Table Queries")]
    S --> Y[("Use for ETL, Intermediate Results")]
    T --> Z[("Use for All Queries")]

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef query fill:#4285f4,stroke:#1976d2;
    classDef result fill:#ff9800,stroke:#f57c00;
    classDef local fill:#009688,stroke:#00796b;
    classDef querycache fill:#e91e63,stroke:#c2185b;
    classDef file fill:#9c27b0,stroke:#7b1fa2;
    classDef warehouse fill:#3f51b5,stroke:#303f9f;
    classDef metadata fill:#795548,stroke:#5d4037;
    class A query;
    class B query;
    class C result;
    class D local;
    class E querycache;
    class F file;
    class G warehouse;
    class H metadata;
    class I,O result;
    class J,P local;
    class K,Q querycache;
    class L,R file;
    class M,S warehouse;
    class N,T metadata;
    class U,V,W,X,Y,Z default;
```

### **Caching Selection Matrix**

| **Cache Type** | **Best For** | **When to Use** | **Performance Impact** | **Cost** | **Configuration Effort** | **TTL** | **Scope** |
|----------------|-------------|-----------------|-------------------------|----------|--------------------------|---------|-----------|
| **Result Cache** | Repetitive, identical queries | Dashboards, reports, BI tools | ⭐⭐⭐⭐⭐ (10-1000x faster) | Included | Low | 24 hours (configurable) | Per-user, per-query |
| **Metadata Cache** | All queries | All queries (automatic) | ⭐⭐ (1.1-2x faster) | Included | None | Session | Per-session |
| **Local Disk Cache** | Frequently accessed data | Hot tables, BI tools, analytics | ⭐⭐⭐ (2-10x faster) | Included | None | Session | Per-warehouse |
| **Query Cache** | Complex queries, parameterized queries | Application queries, stored procedures | ⭐⭐ (1.1-2x faster) | Included | None | Session | Per-session |
| **File Metadata Cache** | External table queries | Queries on S3, Azure Blob, GCS | ⭐⭐ (1.1-2x faster) | Included | None | Session | Per-session |
| **Warehouse Cache** | Temporary data, intermediate results | ETL, intermediate results, temporary tables | ⭐⭐⭐ (2-10x faster) | Included | None | Session | Per-warehouse |

### **When to Use Each Cache Type**

| **Use Case** | **Recommended Cache Types** | **Example** |
|-------------|-------------------------------|-------------|
| **Dashboard Queries** | Result Cache, Local Disk Cache, Query Cache | `SELECT region, SUM(sales) FROM sales GROUP BY region` |
| **Repetitive Aggregations** | Result Cache, Materialized Views | `SELECT date, SUM(sales) FROM sales GROUP BY date` |
| **Hot Tables** | Local Disk Cache | `SELECT * FROM customers WHERE region = 'US'` |
| **Complex Joins** | Query Cache, Local Disk Cache | `SELECT * FROM table1 JOIN table2 ON table1.id = table2.id` |
| **Parameterized Queries** | Query Cache | `SELECT * FROM my_table WHERE id = ?` |
| **External Table Queries** | File Metadata Cache, Local Disk Cache | `SELECT * FROM my_external_table WHERE date > '2023-01-01'` |
| **Temporary Tables** | Warehouse Cache | `CREATE TEMPORARY TABLE temp AS SELECT * FROM my_table` |
| **Metadata Access** | Metadata Cache | All queries (automatic) |
| **ETL Pipelines** | Warehouse Cache, Local Disk Cache | `CREATE TEMPORARY TABLE temp AS SELECT * FROM source_table` |
| **Ad-Hoc Analysis** | Result Cache, Query Cache | `SELECT * FROM my_table WHERE col1 = 'value'` |

### **Cache Type Comparison**

| **Feature** | **Result Cache** | **Metadata Cache** | **Local Disk Cache** | **Query Cache** | **File Metadata Cache** | **Warehouse Cache** |
|-------------|------------------|--------------------|----------------------|-----------------|--------------------------|---------------------|
| **Purpose** | Cache query results | Cache table metadata | Cache frequently accessed data | Cache query plans | Cache external table metadata | Cache temporary data |
| **Scope** | Per-user, per-query | Per-session | Per-warehouse | Per-session | Per-session | Per-warehouse |
| **TTL** | 24 hours (configurable) | Session | Session | Session | Session | Session |
| **Storage** | SSD | Memory | SSD | Memory | Memory | SSD |
| **Performance Impact** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| **Cost** | Included | Included | Included | Included | Included | Included |
| **Configuration** | `USE_CACHED_RESULTS`, `RESULT_CACHE_TTL` | Automatic | Automatic | Automatic | Automatic | Automatic |
| **Best For** | Repetitive queries | All queries | Hot tables | Parameterized queries | External tables | Temporary data |
| **Monitoring** | `QUERY_HISTORY.used_cached_result` | `QUERY_HISTORY.compilation_time` | `QUERY_PROFILE` | `QUERY_HISTORY.compilation_time` | `QUERY_HISTORY.compilation_time` | `QUERY_PROFILE` |
| **Invalidated By** | Data changes, query text changes, permissions, TTL | DDL, data changes, TTL | Warehouse restart, TTL | DDL, schema changes, TTL | DDL, external changes, TTL | Warehouse restart, TTL |
| **Limitations** | Query text must match exactly | Session-scoped | Per-warehouse, limited by size | Session-scoped | Session-scoped | Per-warehouse, temporary data only |

## **9. Caching Best Practices**

### **A. General Caching Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Enable Result Caching** | Enable result caching for repetitive queries | `ALTER SESSION SET USE_CACHED_RESULTS = TRUE` |
| **Use Appropriate Cache TTL** | Set TTL based on data freshness requirements | `ALTER SESSION SET RESULT_CACHE_TTL = 3600` |
| **Monitor Cache Usage** | Monitor cache hits and misses | `SELECT used_cached_result FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` |
| **Use Consistent Query Text** | Ensure query text is identical for cache hits | Avoid dynamic SQL, use parameterized queries |
| **Use Parameterized Queries** | Enable parameterized query caching | `SELECT * FROM my_table WHERE id = ?` |
| **Use Connection Pooling** | Reuse connections to reuse cached query plans | Configure connection pooling in your application |
| **Use Larger Warehouses for Hot Data** | Larger warehouses have more cache capacity | `CREATE WAREHOUSE my_wh WAREHOUSE_SIZE = 'X-LARGE'` |
| **Use Temporary Tables for Intermediate Results** | Temporary tables are cached in warehouse cache | `CREATE TEMPORARY TABLE temp AS SELECT ...` |
| **Update Statistics for Large Tables** | Ensure statistics are up-to-date for large tables | `ALTER TABLE my_table UPDATE STATISTICS` |
| **Avoid Frequent DDL Changes** | DDL changes invalidate caches | Batch DDL changes |
| **Use Clustering for Cache Efficiency** | Cluster tables to improve cache hit rates | `ALTER TABLE my_table CLUSTER BY (date)` |
| **Combine Caching with Other Optimizations** | Use caching with clustering, materialized views, etc. | `CLUSTER BY (date) + USE_CACHED_RESULTS = TRUE` |
| **Document Caching Strategies** | Document which caches are used and why | Internal wiki or Confluence page |
| **Set Up Alerts for Cache Issues** | Alert on cache misses, high compilation times, etc. | `CREATE ALERT ...` |

### **B. Cache-Specific Best Practices**

#### **1. Result Cache Best Practices**
- **Use for Repetitive Queries**: Cache results for queries run frequently (e.g., dashboards, reports).
- **Set Appropriate TTL**: Adjust TTL based on data freshness requirements (e.g., 1 hour for time-sensitive data).
- **Monitor Cache Usage**: Check `used_cached_result` in `QUERY_HISTORY`.
- **Avoid Cache Invalidation**: Minimize changes to underlying data (e.g., batch updates instead of frequent small updates).
- **Use for Read-Only Workloads**: Cache works best for read-only workloads (e.g., BI tools, reporting).
- **Disable for Unique Queries**: Disable caching for unique queries (e.g., ad-hoc analysis with varying filters).
- **Use Consistent Session Parameters**: Ensure session parameters (e.g., time zone, role) are consistent for cache hits.
- **Test Cache Performance**: Compare performance with and without caching.

#### **2. Metadata Cache Best Practices**
- **Update Statistics for Large Tables**: Ensure statistics are up-to-date for large tables.
- **Update Statistics After Data Changes**: Update statistics after significant data changes (e.g., bulk loads).
- **Avoid Frequent DDL Changes**: DDL changes invalidate metadata cache.
- **Use Clustering for Better Metadata**: Clustering improves partition pruning.
- **Monitor Metadata Cache Hits**: Check compilation time and query performance.
- **Use Consistent Table Schemas**: Avoid frequent schema changes.

#### **3. Local Disk Cache Best Practices**
- **Use Larger Warehouses for Hot Data**: Larger warehouses have more local disk cache.
- **Query the Same Data Repeatedly**: Populate the cache with frequently accessed data.
- **Use Clustering for Cache Efficiency**: Cluster tables to improve cache hit rates.
- **Monitor Cache Performance**: Check `QUERY_PROFILE` for cache hits.
- **Avoid Frequent Warehouse Restarts**: Cache is lost when warehouse is restarted.
- **Use Multi-Cluster Warehouses for High Concurrency**: Distribute cache across multiple clusters.
- **Combine with Result Caching**: Use both local disk cache and result caching.

#### **4. Query Cache Best Practices**
- **Use Parameterized Queries**: Enables parameterized query caching.
- **Use Consistent Query Structure**: Use similar query structures for better cache reuse.
- **Avoid Dynamic SQL**: Dynamic SQL prevents query caching.
- **Use Stored Procedures**: Stored procedures can reuse cached plans.
- **Monitor Compilation Time**: Check `compilation_time` in `QUERY_HISTORY`.
- **Avoid Frequent DDL Changes**: DDL changes invalidate query cache.
- **Use Connection Pooling**: Reuse connections to reuse cached query plans.

#### **5. File Metadata Cache Best Practices**
- **Use Consistent Stage Definitions**: Avoid changing stage definitions frequently.
- **Use Consistent File Formats**: Avoid changing file format definitions frequently.
- **Use Partitioned External Tables**: Partition external tables for better performance.
- **Monitor File Metadata Cache Hits**: Check compilation time for external table queries.
- **Avoid Frequent External Table Changes**: Changes to external tables invalidate the cache.
- **Use External Table Clustering**: Cluster external tables for better performance.

#### **6. Warehouse Cache Best Practices**
- **Use Temporary Tables for Intermediate Results**: Temporary tables are cached in warehouse cache.
- **Use CTEs for Complex Queries**: CTE results may be cached in warehouse cache.
- **Use Larger Warehouses for Cache-Intensive Workloads**: Larger warehouses have more cache capacity.
- **Reuse Temporary Tables**: Reuse temporary tables to benefit from caching.
- **Avoid Frequent Warehouse Restarts**: Cache is lost when warehouse is restarted.
- **Use Multi-Cluster Warehouses for High Concurrency**: Distribute cache across multiple clusters.

### **C. Caching Anti-Patterns**

| **Anti-Pattern** | **Description** | **Impact** | **Solution** |
|------------------|-----------------|------------|--------------|
| **No Caching** | Not using any caching | ❌ Poor performance, ❌ High costs | Enable result caching, use larger warehouses |
| **Over-Caching** | Using caching for all queries | ❌ High storage costs, ❌ Cache inefficiency | Use caching selectively for high-impact queries |
| **Not Monitoring Cache** | Not monitoring cache usage | ❌ No visibility into performance, ❌ Issues not detected | Monitor cache hits, misses, and performance |
| **Frequent Cache Invalidation** | Frequent data or schema changes | ❌ Cache inefficiency, ❌ High costs | Batch changes, use appropriate TTL |
| **Inconsistent Query Text** | Using different query text for the same query | ❌ Cache misses, ❌ Poor performance | Use consistent query formatting, parameterized queries |
| **Not Using Parameterized Queries** | Using dynamic SQL instead of parameterized queries | ❌ No query cache reuse, ❌ Poor performance | Use parameterized queries |
| **Using Small Warehouses for Hot Data** | Using small warehouses for frequently accessed data | ❌ Limited cache capacity, ❌ Poor performance | Use larger warehouses for hot data |
| **Not Updating Statistics** | Not updating statistics for large tables | ❌ Poor query optimization, ❌ Cache inefficiency | Update statistics manually or automatically |
| **Frequent DDL Changes** | Making frequent DDL changes | ❌ Cache invalidation, ❌ Poor performance | Batch DDL changes |
| **Not Using Temporary Tables** | Not using temporary tables for intermediate results | ❌ No warehouse cache benefits | Use temporary tables for intermediate results |
| **Not Combining Caching with Other Optimizations** | Using caching in isolation | ❌ Suboptimal performance | Combine caching with clustering, materialized views, etc. |

## **10. Caching Monitoring and Alerting**

### **A. Monitoring Caching Performance**

#### **1. Result Cache Monitoring**
```sql
-- Check if queries are using cached results
SELECT
    query_id,
    query_text,
    used_cached_result,
    execution_time,
    bytes_scanned,
    credits_used,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    used_cached_result DESC, execution_time;

-- Check cache hit rate
SELECT
    used_cached_result,
    COUNT(*) AS query_count,
    AVG(execution_time) AS avg_execution_time,
    AVG(credits_used) AS avg_credits_used
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    used_cached_result;

-- Check cache usage by user
SELECT
    user_name,
    COUNT(*) AS query_count,
    SUM(CASE WHEN used_cached_result = TRUE THEN 1 ELSE 0 END) AS cache_hits,
    SUM(CASE WHEN used_cached_result = FALSE THEN 1 ELSE 0 END) AS cache_misses,
    SUM(CASE WHEN used_cached_result = TRUE THEN 1 ELSE 0 END) * 100.0 /
        NULLIF(COUNT(*), 0) AS cache_hit_rate_percent
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    user_name
ORDER BY
    cache_hit_rate_percent DESC;

-- Check cache usage by warehouse
SELECT
    warehouse_name,
    COUNT(*) AS query_count,
    SUM(CASE WHEN used_cached_result = TRUE THEN 1 ELSE 0 END) AS cache_hits,
    SUM(CASE WHEN used_cached_result = FALSE THEN 1 ELSE 0 END) AS cache_misses,
    SUM(CASE WHEN used_cached_result = TRUE THEN 1 ELSE 0 END) * 100.0 /
        NULLIF(COUNT(*), 0) AS cache_hit_rate_percent
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    warehouse_name
ORDER BY
    cache_hit_rate_percent DESC;
```

#### **2. Metadata Cache Monitoring**
```sql
-- Check compilation time (indirect indicator of metadata cache usage)
SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    compilation_time DESC;

-- Check average compilation time by table
SELECT
    table_name,
    AVG(compilation_time) AS avg_compilation_time,
    COUNT(*) AS query_count
FROM (
    SELECT
        query_id,
        query_text,
        compilation_time,
        REGEXP_SUBSTR(query_text, 'FROM\\s+([^\\s,;]+)', 1, 1, '', 1) AS table_name
    FROM
        SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE
        start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
        AND compilation_time > 0
)
GROUP BY
    table_name
ORDER BY
    avg_compilation_time DESC;
```

#### **3. Local Disk Cache Monitoring**
```sql
-- Check query profile for local disk cache usage
SELECT
    step_id,
    operation,
    execution_time,
    bytes_scanned,
    rows_produced
FROM
    TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))
WHERE
    execution_time < 100  -- Fast steps may indicate cache hits
ORDER BY
    execution_time;

-- Compare performance with and without local disk cache
-- First query (cache miss)
SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7;

-- Subsequent queries (cache hit)
SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7;

-- Check execution times
SELECT
    query_id,
    query_text,
    execution_time,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%my_table%date%'
ORDER BY
    start_time;
```

#### **4. Query Cache Monitoring**
```sql
-- Check compilation time for query cache usage
SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    compilation_time > 0
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    compilation_time DESC;

-- Check average compilation time by query pattern
SELECT
    REGEXP_SUBSTR(query_text, '^[^\\s]+', 1, 1) AS query_type,
    AVG(compilation_time) AS avg_compilation_time,
    COUNT(*) AS query_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    compilation_time > 0
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    query_type
ORDER BY
    avg_compilation_time DESC;
```

#### **5. File Metadata Cache Monitoring**
```sql
-- Check compilation time for external table queries
SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%my_external_table%'
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    compilation_time DESC;

-- Check average compilation time for external tables
SELECT
    table_name,
    AVG(compilation_time) AS avg_compilation_time,
    COUNT(*) AS query_count
FROM (
    SELECT
        query_id,
        query_text,
        compilation_time,
        REGEXP_SUBSTR(query_text, 'FROM\\s+([^\\s,;]+)', 1, 1, '', 1) AS table_name
    FROM
        SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE
        query_text LIKE '%FROM%'
        AND compilation_time > 0
        AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
)
WHERE
    table_name IN (
        SELECT table_name
        FROM INFORMATION_SCHEMA.EXTERNAL_TABLES
    )
GROUP BY
    table_name
ORDER BY
    avg_compilation_time DESC;
```

#### **6. Warehouse Cache Monitoring**
```sql
-- Check query profile for warehouse cache usage
SELECT
    step_id,
    operation,
    execution_time,
    bytes_scanned,
    rows_produced
FROM
    TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))
WHERE
    operation LIKE '%Temp%'
    OR operation LIKE '%CTE%'
ORDER BY
    execution_time;

-- Compare performance with and without warehouse cache
-- First query (cache miss)
CREATE TEMPORARY TABLE temp_table AS SELECT * FROM my_table;
SELECT * FROM temp_table WHERE col1 = 'value';

-- Subsequent queries (cache hit)
SELECT * FROM temp_table WHERE col1 = 'value';

-- Check execution times
SELECT
    query_id,
    query_text,
    execution_time,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%temp_table%'
ORDER BY
    start_time;
```

### **B. Caching Alerts**

#### **1. Low Cache Hit Rate Alert**
```sql
CREATE OR REPLACE ALERT low_cache_hit_rate_alert
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON 0 * * * * America/Los_Angeles'  -- Every hour
AS
  SELECT
    'Low Cache Hit Rate' AS alert_type,
    COUNT(*) AS total_queries,
    SUM(CASE WHEN used_cached_result = TRUE THEN 1 ELSE 0 END) AS cache_hits,
    SUM(CASE WHEN used_cached_result = FALSE THEN 1 ELSE 0 END) AS cache_misses,
    SUM(CASE WHEN used_cached_result = TRUE THEN 1 ELSE 0 END) * 100.0 /
        NULLIF(COUNT(*), 0) AS cache_hit_rate_percent,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE
    start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
  HAVING
    SUM(CASE WHEN used_cached_result = TRUE THEN 1 ELSE 0 END) * 100.0 /
        NULLIF(COUNT(*), 0) < 50;  -- Cache hit rate < 50%
```

#### **2. High Compilation Time Alert**
```sql
CREATE OR REPLACE ALERT high_compilation_time_alert
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'  -- Every 5 minutes
AS
  SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time,
    warehouse_name,
    user_name,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE
    compilation_time > 500  -- >500ms
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
  ORDER BY
    compilation_time DESC;
```

#### **3. Cache Not Used for Repetitive Queries Alert**
```sql
CREATE OR REPLACE ALERT cache_not_used_for_repetitive_queries_alert
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON 0 * * * * America/Los_Angeles'  -- Every hour
AS
  WITH query_counts AS (
    SELECT
        query_text,
        COUNT(*) AS query_count,
        SUM(CASE WHEN used_cached_result = TRUE THEN 1 ELSE 0 END) AS cache_hits
    FROM
        SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE
        start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
    GROUP BY
        query_text
    HAVING
        COUNT(*) > 5  -- Queries run more than 5 times in the last hour
  )
  SELECT
    query_text,
    query_count,
    cache_hits,
    query_count - cache_hits AS cache_misses,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    query_counts
  WHERE
    cache_hits = 0;  -- No cache hits for repetitive queries
```

#### **4. High Cache Invalidation Alert**
```sql
CREATE OR REPLACE ALERT high_cache_invalidation_alert
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'  -- Every 5 minutes
AS
  SELECT
    table_name,
    COUNT(*) AS ddl_changes,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE
    query_text LIKE '%ALTER TABLE%'
    OR query_text LIKE '%CREATE TABLE%'
    OR query_text LIKE '%DROP TABLE%'
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
  GROUP BY
    table_name
  HAVING
    COUNT(*) > 5;  -- More than 5 DDL changes in the last hour
```

#### **5. Warehouse Cache Performance Alert**
```sql
CREATE OR REPLACE ALERT warehouse_cache_performance_alert
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'  -- Every 5 minutes
AS
  SELECT
    warehouse_name,
    query_id,
    query_text,
    execution_time,
    bytes_scanned,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE
    query_text LIKE '%TEMPORARY TABLE%'
    OR query_text LIKE '%WITH%'
    AND execution_time > 1000  -- >1 second
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
  ORDER BY
    execution_time DESC;
```

## **11. Caching Troubleshooting**

### **A. Result Cache Troubleshooting**

#### **Symptom 1: Result Cache Not Working**
**Diagnosis**:
1. **Check if result caching is enabled**:
   ```sql
   SELECT
       CURRENT_SESSION() AS session_id,
       USE_CACHED_RESULTS
   FROM
       TABLE(SNOWFLAKE.INFORMATION_SCHEMA.CURRENT_SESSION());
   ```

2. **Check if query used cached results**:
   ```sql
   SELECT
       query_id,
       query_text,
       used_cached_result,
       execution_time
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       query_text LIKE '%MY_QUERY%'
       AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
   ORDER BY
       start_time DESC;
   ```

3. **Check for data changes**:
   - Result cache is **invalidated** when underlying data changes.

**Solutions**:
1. **Enable result caching**:
   ```sql
   ALTER SESSION SET USE_CACHED_RESULTS = TRUE;
   ```

2. **Ensure query text matches exactly**:
   - Result cache is **keyed by exact query text** (including whitespace, case, and comments).

3. **Ensure session parameters are consistent**:
   - Result cache is **keyed by session parameters** (e.g., time zone, role).

4. **Avoid data changes between queries**:
   - Batch data changes to minimize cache invalidation.

5. **Set appropriate TTL**:
   ```sql
   ALTER SESSION SET RESULT_CACHE_TTL = 3600;  -- 1 hour
   ```

#### **Symptom 2: Low Cache Hit Rate**
**Diagnosis**:
1. **Check cache hit rate**:
   ```sql
   SELECT
       used_cached_result,
       COUNT(*) AS query_count
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   GROUP BY
       used_cached_result;
   ```

2. **Check for unique queries**:
   ```sql
   SELECT
       query_text,
       COUNT(*) AS query_count
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   GROUP BY
       query_text
   HAVING
       COUNT(*) = 1  -- Unique queries
   ORDER BY
       query_count DESC;
   ```

**Solutions**:
1. **Use consistent query text**:
   - Ensure queries are **formatted consistently** (e.g., same whitespace, case).

2. **Use parameterized queries**:
   - Use **bind variables** instead of dynamic SQL.

3. **Use materialized views for non-cacheable queries**:
   ```sql
   CREATE MATERIALIZED VIEW my_mv AS SELECT ...;
   ```

4. **Increase TTL for time-sensitive data**:
   ```sql
   ALTER SESSION SET RESULT_CACHE_TTL = 7200;  -- 2 hours
   ```

#### **Symptom 3: Cache Invalidation Due to Data Changes**
**Diagnosis**:
1. **Check for data changes between queries**:
   ```sql
   SELECT
       query_id,
       query_text,
       start_time,
       used_cached_result
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       query_text LIKE '%MY_QUERY%'
       AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
   ORDER BY
       start_time;
   ```

2. **Check for DML on source tables**:
   ```sql
   SELECT
       query_id,
       query_text,
       start_time
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       query_text LIKE '%INSERT%'
       OR query_text LIKE '%UPDATE%'
       OR query_text LIKE '%DELETE%'
       AND query_text LIKE '%MY_TABLE%'
       AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
   ORDER BY
       start_time;
   ```

**Solutions**:
1. **Batch data changes**:
   - Use **batch updates** instead of frequent small updates.

2. **Use materialized views for time-sensitive data**:
   ```sql
   CREATE MATERIALIZED VIEW my_mv AS SELECT ...;
   ```

3. **Disable caching for frequently updated tables**:
   ```sql
   ALTER SESSION SET USE_CACHED_RESULTS = FALSE;
   ```

4. **Use shorter TTL for volatile data**:
   ```sql
   ALTER SESSION SET RESULT_CACHE_TTL = 300;  -- 5 minutes
   ```

### **B. Metadata Cache Troubleshooting**

#### **Symptom 1: High Compilation Time**
**Diagnosis**:
1. **Check compilation time for queries**:
   ```sql
   SELECT
       query_id,
       query_text,
       compilation_time,
       execution_time
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       compilation_time > 500  -- >500ms
       AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY
       compilation_time DESC;
   ```

2. **Check for DDL changes**:
   ```sql
   SELECT
       query_id,
       query_text,
       start_time
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       query_text LIKE '%ALTER TABLE%'
       OR query_text LIKE '%CREATE TABLE%'
       OR query_text LIKE '%DROP TABLE%'
       AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY
       start_time DESC;
   ```

**Solutions**:
1. **Update statistics for large tables**:
   ```sql
   ALTER TABLE my_table UPDATE STATISTICS;
   ```

2. **Avoid frequent DDL changes**:
   - Batch DDL changes to minimize metadata cache invalidation.

3. **Use clustering for better metadata**:
   ```sql
   ALTER TABLE my_table CLUSTER BY (date);
   ```

4. **Monitor metadata cache hits**:
   - Check compilation time for queries on the same table.

#### **Symptom 2: Metadata Cache Not Improving Performance**
**Diagnosis**:
1. **Check if metadata is being cached**:
   - Metadata cache usage is **not directly visible**, but you can infer it from **compilation time**.

2. **Check for schema changes**:
   ```sql
   SELECT
       query_id,
       query_text,
       start_time
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       query_text LIKE '%ALTER TABLE%'
       AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY
       start_time DESC;
   ```

**Solutions**:
1. **Update statistics manually**:
   ```sql
   ALTER TABLE my_table UPDATE STATISTICS;
   ```

2. **Use automatic statistics**:
   - Snowflake **automatically updates statistics** for most tables.

3. **Check for data skew**:
   - Skewed data can reduce the effectiveness of metadata cache.

### **C. Local Disk Cache Troubleshooting**

#### **Symptom 1: Local Disk Cache Not Improving Performance**
**Diagnosis**:
1. **Check warehouse size**:
   ```sql
   SELECT
       warehouse_name,
       warehouse_size
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSES
   WHERE
       warehouse_name = 'MY_WH';
   ```

2. **Check query profile for cache hits**:
   ```sql
   SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'));
   -- Look for steps with low execution_time and bytes_scanned
   ```

**Solutions**:
1. **Use larger warehouses for hot data**:
   ```sql
   ALTER WAREHOUSE my_wh SET WAREHOUSE_SIZE = 'X-LARGE';
   ```

2. **Query the same data repeatedly**:
   - Populate the cache by querying the same data multiple times.

3. **Use clustering for cache efficiency**:
   ```sql
   ALTER TABLE my_table CLUSTER BY (date);
   ```

4. **Avoid frequent warehouse restarts**:
   - Cache is lost when warehouse is restarted.

#### **Symptom 2: High I/O Latency**
**Diagnosis**:
1. **Check query profile for I/O latency**:
   ```sql
   SELECT
       step_id,
       operation,
       execution_time,
       bytes_scanned
   FROM
       TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))
   ORDER BY
       execution_time DESC;
   ```

2. **Check warehouse load**:
   ```sql
   SELECT
       warehouse_name,
       running_queries,
       queued_queries,
       credit_usage
   FROM
       SNOWFLAKE.INFORMATION_SCHEMA.WAREHOUSE_MONITOR
   WHERE
       warehouse_name = 'MY_WH';
   ```

**Solutions**:
1. **Use larger warehouses**:
   ```sql
   ALTER WAREHOUSE my_wh SET WAREHOUSE_SIZE = 'X-LARGE';
   ```

2. **Use multi-cluster warehouses for high concurrency**:
   ```sql
   ALTER WAREHOUSE my_wh SET MAX_CLUSTER_COUNT = 4;
   ```

3. **Use clustering for better I/O efficiency**:
   ```sql
   ALTER TABLE my_table CLUSTER BY (date);
   ```

4. **Use local disk cache for hot data**:
   - Query the same data repeatedly to populate the cache.

### **D. Query Cache Troubleshooting**

#### **Symptom 1: High Compilation Time for Parameterized Queries**
**Diagnosis**:
1. **Check compilation time for parameterized queries**:
   ```sql
   SELECT
       query_id,
       query_text,
       compilation_time,
       execution_time
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       query_text LIKE '%?%'
       AND compilation_time > 100  -- >100ms
       AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY
       compilation_time DESC;
   ```

2. **Check for dynamic SQL**:
   ```sql
   SELECT
       query_id,
       query_text,
       compilation_time
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       query_text LIKE '%EXECUTE IMMEDIATE%'
       AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY
       compilation_time DESC;
   ```

**Solutions**:
1. **Use parameterized queries**:
   - Use **bind variables** instead of dynamic SQL.

2. **Use stored procedures**:
   ```sql
   CREATE PROCEDURE my_proc(param1 INT) AS
   BEGIN
       SELECT * FROM my_table WHERE col1 = param1;
   END;
   ```

3. **Use connection pooling**:
   - Reuse connections to reuse cached query plans.

4. **Avoid frequent DDL changes**:
   - DDL changes invalidate query cache.

#### **Symptom 2: Query Cache Not Reusing Plans**
**Diagnosis**:
1. **Check for schema changes**:
   ```sql
   SELECT
       query_id,
       query_text,
       start_time
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       query_text LIKE '%ALTER TABLE%'
       AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY
       start_time DESC;
   ```

2. **Check for different session parameters**:
   ```sql
   SELECT
       query_id,
       query_text,
       session_parameters
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       query_text LIKE '%MY_QUERY%'
       AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY
       start_time DESC;
   ```

**Solutions**:
1. **Use consistent session parameters**:
   - Set session parameters explicitly (e.g., time zone, role).

2. **Avoid frequent schema changes**:
   - Batch schema changes to minimize query cache invalidation.

3. **Use stored procedures**:
   ```sql
   CREATE PROCEDURE my_proc() AS SELECT * FROM my_table;
   ```

### **E. File Metadata Cache Troubleshooting**

#### **Symptom 1: High Compilation Time for External Tables**
**Diagnosis**:
1. **Check compilation time for external table queries**:
   ```sql
   SELECT
       query_id,
       query_text,
       compilation_time,
       execution_time
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       query_text LIKE '%my_external_table%'
       AND compilation_time > 500  -- >500ms
       AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY
       compilation_time DESC;
   ```

2. **Check for external table changes**:
   ```sql
   SELECT
       query_id,
       query_text,
       start_time
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       query_text LIKE '%ALTER EXTERNAL TABLE%'
       AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY
       start_time DESC;
   ```

**Solutions**:
1. **Use consistent external table definitions**:
   - Avoid changing external table definitions frequently.

2. **Use partitioned external tables**:
   ```sql
   CREATE EXTERNAL TABLE my_table PARTITION BY (date);
   ```

3. **Refresh external table metadata**:
   ```sql
   ALTER EXTERNAL TABLE my_table REFRESH;
   ```

4. **Use internal tables for hot data**:
   - Copy external data to internal tables for better performance.

#### **Symptom 2: File Metadata Cache Not Improving Performance**
**Diagnosis**:
1. **Check if external table metadata is being cached**:
   - File metadata cache usage is **not directly visible**, but you can infer it from **compilation time**.

2. **Check for external file changes**:
   - If external files have changed, the metadata cache may be stale.

**Solutions**:
1. **Refresh external table metadata**:
   ```sql
   ALTER EXTERNAL TABLE my_table REFRESH;
   ```

2. **Use consistent stage definitions**:
   - Avoid changing stage definitions frequently.

3. **Use file formats with caching in mind**:
   - Use **Parquet** or **ORC** for better performance.

### **F. Warehouse Cache Troubleshooting**

#### **Symptom 1: Warehouse Cache Not Improving Performance**
**Diagnosis**:
1. **Check warehouse size**:
   ```sql
   SELECT
       warehouse_name,
       warehouse_size
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSES
   WHERE
       warehouse_name = 'MY_WH';
   ```

2. **Check query profile for warehouse cache usage**:
   ```sql
   SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'));
   -- Look for steps with temporary tables or CTEs
   ```

**Solutions**:
1. **Use larger warehouses for cache-intensive workloads**:
   ```sql
   ALTER WAREHOUSE my_wh SET WAREHOUSE_SIZE = 'X-LARGE';
   ```

2. **Use temporary tables for intermediate results**:
   ```sql
   CREATE TEMPORARY TABLE temp AS SELECT * FROM my_table;
   ```

3. **Reuse temporary tables**:
   - Query the same temporary table multiple times to benefit from caching.

4. **Avoid frequent warehouse restarts**:
   - Cache is lost when warehouse is restarted.

#### **Symptom 2: High Execution Time for Temporary Tables**
**Diagnosis**:
1. **Check query profile for temporary table performance**:
   ```sql
   SELECT
       step_id,
       operation,
       execution_time,
       bytes_scanned
   FROM
       TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))
   WHERE
       operation LIKE '%Temp%'
   ORDER BY
       execution_time DESC;
   ```

2. **Check for spill to disk**:
   ```sql
   SELECT
       step_id,
       operation,
       spill_to_disk,
       spill_to_remote
   FROM
       TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))
   WHERE
       spill_to_disk > 0 OR spill_to_remote > 0;
   ```

**Solutions**:
1. **Use larger warehouses**:
   ```sql
   ALTER WAREHOUSE my_wh SET WAREHOUSE_SIZE = 'X-LARGE';
   ```

2. **Use clustering for temporary tables**:
   ```sql
   CREATE TEMPORARY TABLE temp CLUSTER BY (date) AS SELECT * FROM my_table;
   ```

3. **Reduce data volume in temporary tables**:
   ```sql
   CREATE TEMPORARY TABLE temp AS SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7;
   ```

4. **Use CTEs instead of temporary tables for small datasets**:
   ```sql
   WITH cte AS (SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7)
   SELECT * FROM cte WHERE col1 = 'value';
   ```

## **12. Production Checklist for Snowflake Caching**

### **A. Result Cache Checklist**

#### **1. Configuration**
- [ ] **Enable Result Caching**:
  ```sql
  ALTER SESSION SET USE_CACHED_RESULTS = TRUE;
  ```
- [ ] **Set Appropriate TTL**:
  ```sql
  ALTER SESSION SET RESULT_CACHE_TTL = 3600;  -- 1 hour
  ```
- [ ] **Enable for Users/Roles**:
  ```sql
  ALTER USER my_user SET USE_CACHED_RESULTS = TRUE;
  ALTER ROLE my_role SET USE_CACHED_RESULTS = TRUE;
  ```

#### **2. Monitoring**
- [ ] **Monitor Cache Usage**:
  ```sql
  SELECT used_cached_result FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY;
  ```
- [ ] **Monitor Cache Hit Rate**:
  ```sql
  SELECT used_cached_result, COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY GROUP BY used_cached_result;
  ```
- [ ] **Monitor Cache Performance**:
  ```sql
  SELECT query_id, query_text, execution_time, credits_used FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY WHERE used_cached_result = TRUE;
  ```
- [ ] **Set Up Alerts for Low Cache Hit Rate**:
  ```sql
  CREATE ALERT low_cache_hit_rate_alert ...;
  ```

#### **3. Optimization**
- [ ] **Use for Repetitive Queries**: Dashboards, reports, BI tools.
- [ ] **Use Consistent Query Text**: Ensure queries are formatted consistently.
- [ ] **Use Consistent Session Parameters**: Set time zone, role, etc. explicitly.
- [ ] **Avoid Data Changes Between Queries**: Batch updates instead of frequent small updates.
- [ ] **Disable for Unique Queries**: Ad-hoc analysis with varying filters.
- [ ] **Use for Expensive Queries**: Queries using >10 credits.
- [ ] **Document Cacheable Queries**: Maintain a list of queries that benefit from caching.

### **B. Metadata Cache Checklist**

#### **1. Configuration**
- [ ] **Update Statistics for Large Tables**:
  ```sql
  ALTER TABLE my_table UPDATE STATISTICS;
  ```
- [ ] **Use Automatic Statistics**: Snowflake automatically updates statistics for most tables.

#### **2. Monitoring**
- [ ] **Monitor Compilation Time**:
  ```sql
  SELECT compilation_time FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY;
  ```
- [ ] **Monitor Query Performance**:
  ```sql
  SELECT query_id, query_text, compilation_time, execution_time FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY;
  ```
- [ ] **Monitor Statistics Freshness**:
  ```sql
  SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.TABLE_STATISTICS('MY_TABLE'));
  ```

#### **3. Optimization**
- [ ] **Update Statistics After Data Changes**: Bulk loads, significant updates.
- [ ] **Avoid Frequent DDL Changes**: Batch DDL changes to minimize cache invalidation.
- [ ] **Use Clustering for Better Metadata**: Improves partition pruning.
- [ ] **Use Consistent Table Schemas**: Avoid frequent schema changes.

### **C. Local Disk Cache Checklist**

#### **1. Configuration**
- [ ] **Use Larger Warehouses for Hot Data**:
  ```sql
  CREATE WAREHOUSE my_wh WAREHOUSE_SIZE = 'X-LARGE';
  ```
- [ ] **Use Multi-Cluster Warehouses for High Concurrency**:
  ```sql
  CREATE WAREHOUSE my_wh MAX_CLUSTER_COUNT = 4;
  ```

#### **2. Monitoring**
- [ ] **Monitor Query Profile**:
  ```sql
  SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'));
  ```
- [ ] **Monitor Warehouse Utilization**:
  ```sql
  SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY;
  ```
- [ ] **Monitor Warehouse Events**:
  ```sql
  SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_EVENTS_HISTORY;
  ```

#### **3. Optimization**
- [ ] **Query the Same Data Repeatedly**: Populate the cache with frequently accessed data.
- [ ] **Use Clustering for Cache Efficiency**: Improves cache hit rates.
- [ ] **Avoid Frequent Warehouse Restarts**: Cache is lost when warehouse is restarted.
- [ ] **Use Auto-Suspend for Idle Warehouses**: Suspend warehouses when idle to save costs.

### **D. Query Cache Checklist**

#### **1. Configuration**
- [ ] **Use Parameterized Queries**: Enables parameterized query caching.
- [ ] **Use Stored Procedures**: Stored procedures can reuse cached plans.
- [ ] **Use Connection Pooling**: Reuse connections to reuse cached query plans.

#### **2. Monitoring**
- [ ] **Monitor Compilation Time**:
  ```sql
  SELECT compilation_time FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY;
  ```
- [ ] **Monitor Query Performance**:
  ```sql
  SELECT query_id, query_text, compilation_time, execution_time FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY;
  ```

#### **3. Optimization**
- [ ] **Use Consistent Query Structure**: Similar queries can reuse cached plans.
- [ ] **Avoid Dynamic SQL**: Dynamic SQL prevents query caching.
- [ ] **Avoid Frequent DDL Changes**: DDL changes invalidate query cache.
- [ ] **Use Consistent Session Parameters**: Set time zone, role, etc. explicitly.

### **E. File Metadata Cache Checklist**

#### **1. Configuration**
- [ ] **Use Consistent Stage Definitions**: Avoid changing stage definitions frequently.
- [ ] **Use Consistent File Formats**: Avoid changing file format definitions frequently.
- [ ] **Use Partitioned External Tables**: Improves partition pruning.

#### **2. Monitoring**
- [ ] **Monitor Compilation Time for External Tables**:
  ```sql
  SELECT compilation_time FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY WHERE query_text LIKE '%my_external_table%';
  ```
- [ ] **Monitor External Table Performance**:
  ```sql
  SELECT query_id, query_text, execution_time FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY WHERE query_text LIKE '%my_external_table%';
  ```

#### **3. Optimization**
- [ ] **Refresh External Table Metadata**:
  ```sql
  ALTER EXTERNAL TABLE my_table REFRESH;
  ```
- [ ] **Use Clustering for External Tables**: Improves query performance.
- [ ] **Use Internal Tables for Hot Data**: Copy external data to internal tables for better performance.

### **F. Warehouse Cache Checklist**

#### **1. Configuration**
- [ ] **Use Temporary Tables for Intermediate Results**:
  ```sql
  CREATE TEMPORARY TABLE temp AS SELECT * FROM my_table;
  ```
- [ ] **Use CTEs for Complex Queries**:
  ```sql
  WITH cte AS (SELECT * FROM my_table) SELECT * FROM cte;
  ```

#### **2. Monitoring**
- [ ] **Monitor Query Profile**:
  ```sql
  SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'));
  ```
- [ ] **Monitor Temporary Table Performance**:
  ```sql
  SELECT query_id, query_text, execution_time FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY WHERE query_text LIKE '%TEMPORARY TABLE%';
  ```

#### **3. Optimization**
- [ ] **Reuse Temporary Tables**: Query the same temporary table multiple times.
- [ ] **Use Larger Warehouses for Cache-Intensive Workloads**: More cache capacity.
- [ ] **Avoid Frequent Warehouse Restarts**: Cache is lost when warehouse is restarted.
- [ ] **Use Multi-Cluster Warehouses for High Concurrency**: Distribute cache across multiple clusters.

## **13. Final Recommendations**

### **A. Caching Strategy Summary**

| **Cache Type** | **When to Use** | **Performance Impact** | **Cost** | **Effort** | **Best For** |
|----------------|-----------------|-------------------------|----------|-------------|--------------|
| **Result Cache** | Repetitive, identical queries | ⭐⭐⭐⭐⭐ (10-1000x faster) | Included | Low | Dashboards, reports, BI tools |
| **Metadata Cache** | All queries | ⭐⭐ (1.1-2x faster) | Included | None | All queries (automatic) |
| **Local Disk Cache** | Frequently accessed data | ⭐⭐⭐ (2-10x faster) | Included | None | Hot tables, BI tools, analytics |
| **Query Cache** | Complex queries, parameterized queries | ⭐⭐ (1.1-2x faster) | Included | None | Application queries, stored procedures |
| **File Metadata Cache** | External table queries | ⭐⭐ (1.1-2x faster) | Included | None | Queries on S3, Azure Blob, GCS |
| **Warehouse Cache** | Temporary data, intermediate results | ⭐⭐⭐ (2-10x faster) | Included | None | ETL, intermediate results, temporary tables |

### **B. Caching Implementation Roadmap**

#### **Phase 1: Enable Basic Caching**
1. **Enable Result Caching**:
   ```sql
   ALTER SESSION SET USE_CACHED_RESULTS = TRUE;
   ```
2. **Set Appropriate TTL**:
   ```sql
   ALTER SESSION SET RESULT_CACHE_TTL = 3600;  -- 1 hour
   ```
3. **Monitor Cache Usage**:
   ```sql
   SELECT used_cached_result FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY;
   ```

#### **Phase 2: Optimize Metadata Cache**
1. **Update Statistics for Large Tables**:
   ```sql
   ALTER TABLE my_table UPDATE STATISTICS;
   ```
2. **Use Clustering for Better Metadata**:
   ```sql
   ALTER TABLE my_table CLUSTER BY (date);
   ```
3. **Monitor Compilation Time**:
   ```sql
   SELECT compilation_time FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY;
   ```

#### **Phase 3: Optimize Local Disk Cache**
1. **Use Larger Warehouses for Hot Data**:
   ```sql
   CREATE WAREHOUSE hot_data_wh WAREHOUSE_SIZE = 'X-LARGE';
   ```
2. **Query the Same Data Repeatedly**:
   - Populate the cache with frequently accessed data.
3. **Monitor Query Profile**:
   ```sql
   SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'));
   ```

#### **Phase 4: Optimize Query Cache**
1. **Use Parameterized Queries**:
   - Use bind variables in your application code.
2. **Use Stored Procedures**:
   ```sql
   CREATE PROCEDURE my_proc() AS SELECT * FROM my_table;
   ```
3. **Monitor Compilation Time**:
   ```sql
   SELECT compilation_time FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY;
   ```

#### **Phase 5: Optimize File Metadata Cache**
1. **Use Partitioned External Tables**:
   ```sql
   CREATE EXTERNAL TABLE my_table PARTITION BY (date);
   ```
2. **Refresh External Table Metadata**:
   ```sql
   ALTER EXTERNAL TABLE my_table REFRESH;
   ```
3. **Monitor Compilation Time for External Tables**:
   ```sql
   SELECT compilation_time FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY WHERE query_text LIKE '%my_external_table%';
   ```

#### **Phase 6: Optimize Warehouse Cache**
1. **Use Temporary Tables for Intermediate Results**:
   ```sql
   CREATE TEMPORARY TABLE temp AS SELECT * FROM my_table;
   ```
2. **Reuse Temporary Tables**:
   - Query the same temporary table multiple times.
3. **Monitor Query Profile**:
   ```sql
   SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'));
   ```

### **C. Caching Best Practices Summary**

1. **Start with Result Caching**:
   - Enable result caching for **repetitive queries** (dashboards, reports, BI tools).
   - Set **appropriate TTL** based on data freshness requirements.

2. **Use Metadata Cache Automatically**:
   - Metadata cache is **automatic** and requires no configuration.
   - Update **statistics** for large tables to improve cache effectiveness.

3. **Optimize Local Disk Cache**:
   - Use **larger warehouses** for hot data.
   - **Query the same data repeatedly** to populate the cache.
   - Use **clustering** to improve cache hit rates.

4. **Use Parameterized Queries**:
   - Enable **parameterized query caching** for application queries.
   - Use **stored procedures** to reuse cached plans.

5. **Optimize File Metadata Cache**:
   - Use **partitioned external tables** for better performance.
   - **Refresh external table metadata** when external files change.

6. **Use Temporary Tables for Intermediate Results**:
   - Temporary tables are **cached in warehouse cache**.
   - **Reuse temporary tables** to benefit from caching.

7. **Monitor and Adjust**:
   - **Monitor cache usage** (cache hits, misses, performance).
   - **Adjust configurations** based on usage patterns (e.g., TTL, warehouse size).

8. **Combine Caching with Other Optimizations**:
   - Use caching with **clustering**, **materialized views**, and **query optimization**.
   - Example: `CLUSTER BY (date) + USE_CACHED_RESULTS = TRUE`.

9. **Document Caching Strategies**:
   - Document **which caches are used** and **why**.
   - Maintain a **runbook** for troubleshooting and maintenance.

10. **Set Up Alerts for Cache Issues**:
    - Alert on **low cache hit rates**, **high compilation times**, etc.
    - Example: `CREATE ALERT low_cache_hit_rate_alert ...`.

### **D. Caching Anti-Patterns to Avoid**

1. **Not Using Caching at All**:
   - **Impact**: Poor performance, high costs.
   - **Solution**: Enable result caching for repetitive queries.

2. **Using Caching for All Queries**:
   - **Impact**: High storage costs, cache inefficiency.
   - **Solution**: Use caching **selectively** for high-impact queries.

3. **Not Monitoring Cache Usage**:
   - **Impact**: No visibility into performance, issues not detected.
   - **Solution**: Monitor cache hits, misses, and performance.

4. **Frequent Cache Invalidation**:
   - **Impact**: Cache inefficiency, high costs.
   - **Solution**: Batch changes, use appropriate TTL.

5. **Inconsistent Query Text**:
   - **Impact**: Cache misses, poor performance.
   - **Solution**: Use consistent query formatting, parameterized queries.

6. **Not Using Parameterized Queries**:
   - **Impact**: No query cache reuse, poor performance.
   - **Solution**: Use bind variables instead of dynamic SQL.

7. **Using Small Warehouses for Hot Data**:
   - **Impact**: Limited cache capacity, poor performance.
   - **Solution**: Use larger warehouses for hot data.

8. **Not Updating Statistics**:
   - **Impact**: Poor query optimization, cache inefficiency.
   - **Solution**: Update statistics manually or automatically.

9. **Frequent DDL Changes**:
   - **Impact**: Cache invalidation, poor performance.
   - **Solution**: Batch DDL changes.

10. **Not Using Temporary Tables**:
    - **Impact**: No warehouse cache benefits.
    - **Solution**: Use temporary tables for intermediate results.

### **E. Bottom Line**

Snowflake's **multi-layered caching** provides **powerful performance optimizations** for a wide range of workloads. By **understanding the different cache types**, **when to use each**, and **how to monitor and optimize** them, you can **dramatically improve query performance** while **reducing costs** and **enhancing the user experience**.

**Key Takeaways**:
1. **Result Cache** is the **most impactful** for repetitive queries (10-1000x faster).
2. **Metadata Cache** and **Query Cache** provide **automatic optimizations** for all queries (1.1-2x faster).
3. **Local Disk Cache** and **Warehouse Cache** improve performance for **frequently accessed data** (2-10x faster).
4. **File Metadata Cache** optimizes **external table queries** (1.1-2x faster).
5. **Combine caching with other optimizations** (clustering, materialized views) for **maximum performance**.
6. **Monitor and adjust** caching configurations based on **usage patterns** and **performance data**.

By following these **best practices** and **implementation guidelines**, you can **build a high-performance, cost-effective Snowflake environment** that **scales with your workloads**.
