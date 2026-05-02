# **Query Result Cache and Warehouse Cache: Production-Grade Technical Deep Dive**

---

## **1. Overview of Query Result Cache and Warehouse Cache**

Snowflake implements **multiple caching layers** to optimize query performance. This document focuses on two critical caching mechanisms:

1. **Query Result Cache**: Caches **complete query result sets** to eliminate redundant query execution for identical requests
2. **Warehouse Cache**: Caches **warehouse-specific data** including temporary tables, CTE results, and intermediate query data

These caches work in **concert** with other Snowflake optimization features to deliver **sub-second response times** for repetitive operations while **minimizing credit consumption**.

---

### **Mermaid: Query Result Cache and Warehouse Cache Architecture**
```mermaid
%% Query Result Cache and Warehouse Cache Architecture
flowchart TD
    subgraph Client["Client Layer"]
        A[("Query")] --> B[("Query Parser")]
    end

    subgraph QueryResultCache["Query Result Cache"]
        B --> C[("Cache Key Generation")]
        C --> D[("Cache Lookup")]
        D --> E{Cache Hit?}
        E -->|Yes| F[("Return Cached Results\n<10ms")]
        E -->|No| G[("Execute Query")]
        G --> H[("Cache Results\n(SSD)")]
        H --> F
    end

    subgraph WarehouseCache["Warehouse Cache"]
        G --> I[("Warehouse Cache Lookup")]
        I --> J{Cache Hit?}
        J -->|Yes| K[("Return Cached Data\n<10ms")]
        J -->|No| L[("Fetch from Cloud Storage\n100-500ms")]
        L --> M[("Cache in SSD")]
        M --> K
    end

    subgraph Execution["Execution Layer"]
        F --> N[("Client")]
        K --> N
        G --> O[("Query Execution Engine")]
        L --> O
    end

    subgraph Storage["Storage Layer"]
        H --> P[("Result Cache Storage\n(SSD)")]
        M --> Q[("Warehouse Cache Storage\n(SSD)")]
        L --> R[("Cloud Storage\n(S3/Azure/GCS)")]
    end

    subgraph CacheDetails["Cache Details"]
        C --> S[("Cache Key Components:")]
        S --> T[("Query Text")]
        S --> U[("User Permissions")]
        S --> V[("Session Parameters")]
        S --> W[("Data Version")]
        I --> X[("Cache Key Components:")]
        X --> Y[("Micro-Partition ID")]
        X --> Z[("Temporary Table Name")]
        X --> AA[("CTE Name")]
    end

    subgraph Monitoring["Monitoring Layer"]
        AB[("QUERY_HISTORY")]
        AC[("QUERY_PROFILE")]
        AD[("WAREHOUSE_LOAD_HISTORY")]
    end
    O --> AB
    F --> AB
    K --> AB
    O --> AC
    L --> AD

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef client fill:#4285f4,stroke:#1976d2;
    classDef qrc fill:#ff9800,stroke:#f57c00;
    classDef wc fill:#009688,stroke:#00796b;
    classDef execution fill:#29abe2,stroke:#1a8fb8;
    classDef storage fill:#e91e63,stroke:#c2185b;
    classDef details fill:#9c27b0,stroke:#7b1fa2;
    classDef monitoring fill:#3f51b5,stroke:#303f9f;
    class A client;
    class B,C,D,E,F,G,H qrc;
    class I,J,K,L,M wc;
    class N,O execution;
    class P,Q,R storage;
    class S,T,U,V,W,X,Y,Z,AA details;
    class AB,AC,AD monitoring;
```


### **Key Concepts and Definitions**

#### **1. Query Result Cache**
**Definition**: A **per-user, per-query caching mechanism** that stores the **complete result sets** of executed queries. When an **identical query** is submitted, Snowflake returns the cached results **without re-executing the query**, providing **sub-second response times** for repetitive operations.

**How It Works**:
- Snowflake generates a **cache key** based on query text, user permissions, session parameters, and data version
- If a matching key exists in cache, results are returned immediately (typically **<10ms**)
- If no match exists, query executes normally and results are cached for future use
- Cache is **automatically invalidated** when underlying data changes

**Primary Use Cases**:
- Dashboard queries that run repeatedly
- BI tool refreshes
- Ad-hoc queries that are re-run with identical parameters
- Reporting queries with static filters

#### **2. Warehouse Cache**
**Definition**: A **per-warehouse caching mechanism** that stores **warehouse-specific data** in **local SSD storage** to **reduce I/O latency** and **improve query performance**. This includes caching of:
- **Temporary tables** (created with `CREATE TEMPORARY TABLE`)
- **CTE results** (Common Table Expressions)
- **Intermediate query results** (from subqueries, joins, aggregations)
- **Spill data** (data spilled to disk during query execution)

**How It Works**:
- When a query accesses data, Snowflake **fetches micro-partitions** from cloud storage
- Frequently accessed micro-partitions are **cached in local SSD** for the warehouse
- Subsequent queries check the warehouse cache first (typical latency: **1-10ms**)
- If data is cached, it's returned from SSD; otherwise, it's fetched from cloud storage (typical latency: **100-500ms**)
- Cache uses **LRU (Least Recently Used) eviction** when full

**Primary Use Cases**:
- Repeatedly accessed hot data
- ETL pipelines with temporary tables
- Complex queries with CTEs or subqueries
- Workloads with data locality patterns

### **Comparison: Query Result Cache vs. Warehouse Cache**

| **Feature** | **Query Result Cache** | **Warehouse Cache** | **Notes** |
|-------------|------------------------|--------------------|-----------|
| **Purpose** | Cache complete query results | Cache warehouse-specific data (temp tables, CTEs, intermediate results) | Different but complementary |
| **Scope** | Per-user, per-query | Per-warehouse | Result cache is user-specific; warehouse cache is warehouse-specific |
| **TTL** | 24 hours (configurable) | Session | Result cache persists for 24h; warehouse cache persists until warehouse restart |
| **Storage Medium** | SSD | SSD | Both use local SSD storage |
| **What It Caches** | Complete result sets | Micro-partitions, temp tables, CTE results, intermediate results, spill data | Result cache stores final results; warehouse cache stores intermediate data |
| **Performance Impact** | ⭐⭐⭐⭐⭐ (10-1000x faster) | ⭐⭐⭐ (2-10x faster) | Result cache has higher impact for repetitive queries |
| **Credit Usage** | 0 (for cache hits) | Reduced (faster I/O) | Result cache eliminates credit usage; warehouse cache reduces it |
| **Configuration** | `USE_CACHED_RESULTS`, `RESULT_CACHE_TTL` | Automatic | Result cache requires configuration; warehouse cache is automatic |
| **Invalidation Triggers** | Data changes, query text changes, permissions changes, TTL expiry | Warehouse restart, session end, LRU eviction | Different invalidation mechanisms |
| **Best For** | Repetitive identical queries | Hot data, temporary tables, CTEs | Use both for maximum performance |
| **Monitoring** | `QUERY_HISTORY.used_cached_result` | `QUERY_PROFILE` (indirect) | Different monitoring approaches |
| **Storage Overhead** | Medium (result sets) | High (micro-partitions) | Warehouse cache can consume significant SSD storage |

### **When to Use Each Cache Type**

| **Use Case** | **Query Result Cache** | **Warehouse Cache** | **Combined Approach** |
|-------------|------------------------|--------------------|----------------------|
| **Dashboard Queries** | ✅✅✅ Best | ✅✅ Good | ✅✅✅ Use both |
| **Repetitive Aggregations** | ✅✅✅ Best | ✅ Good | ✅✅✅ Use both |
| **Hot Tables** | ✅✅ Good | ✅✅✅ Best | ✅✅✅ Use both |
| **Complex Joins** | ✅✅ Good | ✅✅ Good | ✅✅✅ Use both |
| **Parameterized Queries** | ⚠️ Limited | ✅✅ Good | ✅✅ Use warehouse cache |
| **Temporary Tables** | ❌ No | ✅✅✅ Best | ✅✅✅ Use warehouse cache |
| **CTEs** | ❌ No | ✅✅✅ Best | ✅✅✅ Use warehouse cache |
| **ETL Pipelines** | ⚠️ Limited | ✅✅✅ Best | ✅✅✅ Use warehouse cache |
| **Ad-Hoc Analysis** | ⚠️ Limited | ✅✅ Good | ✅✅ Use warehouse cache |
| **Real-Time Analytics** | ❌ No | ✅✅ Good | ✅✅ Use warehouse cache |

**Key**:
- ✅✅✅ = Highly recommended
- ✅✅ = Recommended
- ✅ = Some benefit
- ⚠️ = Limited benefit
- ❌ = Not applicable

## **2. Query Result Cache Deep Dive**

### **A. Architecture and Workflow**

```mermaid
%% Query Result Cache Workflow
flowchart TD
    A[("Query Submission")] --> B[("Query Parser")]
    B --> C[("Generate Cache Key")]
    C --> D[("Cache Lookup\n(SSD)")]
    D --> E{Cache Hit?}
    E -->|Yes| F[("Return Cached Results\n1-10ms")]
    E -->|No| G[("Execute Query")]
    G --> H[("Store Results in Cache\n(SSD)")]
    H --> F
    C --> I[("Cache Key Components")]
    I --> J[("Query Text\n(Exact Match)")]
    I --> K[("User Permissions\n(Roles, Privileges)")]
    I --> L[("Session Parameters\n(Timezone, Formats)")]
    I --> M[("Underlying Data\n(Version)")]

    subgraph CacheStorage["Cache Storage"]
        D --> N[("Result Cache\n(SSD)")]
        H --> N
    end

    subgraph Execution["Execution"]
        G --> O[("Query Execution Engine")]
    end

    subgraph Monitoring["Monitoring"]
        P[("QUERY_HISTORY.used_cached_result")]
    end
    F --> P
    G --> P

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef process fill:#4285f4,stroke:#1976d2;
    classDef decision fill:#ff9800,stroke:#f57c00;
    classDef cache fill:#009688,stroke:#00796b;
    classDef storage fill:#29abe2,stroke:#1a8fb8;
    classDef execution fill:#e91e63,stroke:#c2185b;
    classDef monitoring fill:#9c27b0,stroke:#7b1fa2;
    class A,B,C,G,H,O process;
    class D,E decision;
    class F,N cache;
    class J,K,L,M default;
    class P monitoring;
```

### **B. Technical Deep Dive**

#### **1. Cache Key Generation**
The cache key is a **hash** generated from multiple components:

| **Component** | **Description** | **Example** | **Impact on Cache** |
|---------------|-----------------|-------------|---------------------|
| **Query Text** | Exact SQL text (including whitespace, case, comments) | `SELECT * FROM my_table WHERE id = 1` | Different text = different cache entry |
| **User Permissions** | User's roles and privileges | `ROLE1, ROLE2` | Different permissions = different cache entry |
| **Session Parameters** | Session-specific settings (timezone, date formats, etc.) | `TIMEZONE='UTC', DATE_FORMAT='YYYY-MM-DD'` | Different parameters = different cache entry |
| **Underlying Data Version** | Version of the source tables at query time | `v12345` | Data changes = cache invalidation |

**Important**: Even **minor differences** in any of these components will result in a **cache miss**.

**Example of Cache Key Differences**:
```sql
-- These will NOT share the same cache entry:
SELECT * FROM my_table WHERE id = 1;
select * from my_table where id = 1;  -- Different case
SELECT * FROM my_table WHERE id = 1; -- Extra space

-- These WILL share the same cache entry:
SELECT * FROM my_table WHERE id = 1;
SELECT * FROM my_table WHERE id = 1;  -- Identical
```

#### **2. Cache Storage**
- **Storage Medium**: SSD (local to the warehouse)
- **Storage Location**: Managed by Snowflake, not user-visible
- **Storage Format**: Compressed columnar format (same as Snowflake's internal storage)
- **Storage Cost**: Included in Snowflake's storage pricing
- **Compression**: Results are stored in compressed format to minimize storage usage

#### **3. Cache Invalidation**
The query result cache is **automatically invalidated** when:

| **Trigger** | **Description** | **Example** |
|-------------|-----------------|-------------|
| **Data Changes** | Any DML operation (INSERT, UPDATE, DELETE, MERGE, TRUNCATE) on source tables | `UPDATE my_table SET col1 = 1` |
| **Schema Changes** | Any DDL operation (ALTER, CREATE, DROP) on source tables | `ALTER TABLE my_table ADD COLUMN col2 INT` |
| **Query Text Changes** | Any modification to the query string | Changing `WHERE id = 1` to `WHERE id = 2` |
| **Permission Changes** | Changes to user's roles or privileges | `GRANT SELECT ON my_table TO ROLE1` |
| **Session Parameter Changes** | Changes to session parameters (timezone, date formats, etc.) | `ALTER SESSION SET TIMEZONE = 'America/New_York'` |
| **TTL Expiration** | Cache entry expires after configured TTL (default: 24 hours) | After 24 hours |
| **Warehouse Restart** | Warehouse is stopped and restarted (cache is retained during auto-suspend) | `ALTER WAREHOUSE my_wh SUSPEND; ALTER WAREHOUSE my_wh RESUME;` |

**Note**: Auto-suspend **does not invalidate** the cache. The cache is **retained** while the warehouse is suspended and **available immediately** when the warehouse resumes.

#### **4. Cache Behavior with Different Query Types**

| **Query Type** | **Cacheable?** | **Notes** | **Example** |
|----------------|----------------|-----------|-------------|
| **SELECT** | ✅ Yes | Primary use case | `SELECT * FROM my_table WHERE id = 1` |
| **SELECT with CTEs** | ✅ Yes | CTE results may be cached as part of the query | `WITH cte AS (SELECT * FROM my_table) SELECT * FROM cte` |
| **SELECT with Subqueries** | ✅ Yes | Subquery results may be cached separately | `SELECT * FROM (SELECT * FROM my_table) AS subq` |
| **INSERT** | ❌ No | DML operations are not cached | `INSERT INTO my_table VALUES (1, 'A')` |
| **UPDATE** | ❌ No | DML operations are not cached | `UPDATE my_table SET col1 = 1` |
| **DELETE** | ❌ No | DML operations are not cached | `DELETE FROM my_table WHERE id = 1` |
| **CREATE TABLE** | ❌ No | DDL operations are not cached | `CREATE TABLE my_table (id INT)` |
| **ALTER TABLE** | ❌ No | DDL operations are not cached | `ALTER TABLE my_table ADD COLUMN col1 INT` |
| **DROP TABLE** | ❌ No | DDL operations are not cached | `DROP TABLE my_table` |
| **CREATE VIEW** | ❌ No | Views themselves are not cached (but underlying queries may be) | `CREATE VIEW my_view AS SELECT * FROM my_table` |
| **CREATE MATERIALIZED VIEW** | ❌ No | MVs are not cached (but their queries may use other caches) | `CREATE MATERIALIZED VIEW my_mv AS SELECT * FROM my_table` |
| **SHOW commands** | ❌ No | Metadata queries are not cached | `SHOW TABLES` |
| **DESCRIBE commands** | ❌ No | Metadata queries are not cached | `DESCRIBE TABLE my_table` |

### **C. Configuration**

#### **1. Enable/Disable Query Result Cache**
```sql
-- Enable for current session (default)
ALTER SESSION SET USE_CACHED_RESULTS = TRUE;

-- Disable for current session
ALTER SESSION SET USE_CACHED_RESULTS = FALSE;

-- Enable for a specific user
ALTER USER my_user SET USE_CACHED_RESULTS = TRUE;

-- Enable for a specific role
ALTER ROLE my_role SET USE_CACHED_RESULTS = TRUE;

-- Disable for all sessions (not recommended)
ALTER ACCOUNT SET USE_CACHED_RESULTS = FALSE;
```

#### **2. Configure Cache TTL**
```sql
-- Set TTL to 1 hour (3600 seconds)
ALTER SESSION SET RESULT_CACHE_TTL = 3600;

-- Set TTL to 12 hours
ALTER SESSION SET RESULT_CACHE_TTL = 43200;

-- Set TTL to 24 hours (default)
ALTER SESSION SET RESULT_CACHE_TTL = 86400;

-- Set TTL to 0 (effectively disables caching)
ALTER SESSION SET RESULT_CACHE_TTL = 0;
```

**Note**: The maximum TTL is **86400 seconds (24 hours)**. Setting TTL to 0 disables caching for that session.

#### **3. Check Current Configuration**
```sql
-- Check current session settings
SELECT
    CURRENT_SESSION() AS session_id,
    USE_CACHED_RESULTS,
    RESULT_CACHE_TTL
FROM
    TABLE(SNOWFLAKE.INFORMATION_SCHEMA.CURRENT_SESSION());

-- Check settings for a specific user
SELECT
    user_name,
    default_use_cached_results,
    default_result_cache_ttl
FROM
    SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE
    user_name = 'MY_USER';

-- Check settings for a specific role
SELECT
    role_name,
    use_cached_results,
    result_cache_ttl
FROM
    SNOWFLAKE.ACCOUNT_USAGE.ROLES
WHERE
    role_name = 'MY_ROLE';
```

### **D. Monitoring and Metrics**

#### **1. Check Cache Usage in Query History**
```sql
-- Basic cache usage check
SELECT
    query_id,
    query_text,
    used_cached_result,
    execution_time,
    start_time,
    warehouse_name,
    user_name,
    credits_used
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;

-- Cache hit rate by user
SELECT
    user_name,
    COUNT(*) AS total_queries,
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

-- Cache hit rate by warehouse
SELECT
    warehouse_name,
    COUNT(*) AS total_queries,
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

#### **2. Performance Comparison**
```sql
-- Compare performance with and without caching
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
    query_text LIKE '%your_query_pattern%'
    AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY
    used_cached_result, execution_time;

-- Calculate average performance improvement
SELECT
    AVG(CASE WHEN used_cached_result = TRUE THEN execution_time ELSE NULL END) AS avg_cached_time,
    AVG(CASE WHEN used_cached_result = FALSE THEN execution_time ELSE NULL END) AS avg_uncached_time,
    AVG(CASE WHEN used_cached_result = FALSE THEN execution_time ELSE NULL END) /
        NULLIF(AVG(CASE WHEN used_cached_result = TRUE THEN execution_time ELSE NULL END), 0) AS speedup_factor
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%your_query_pattern%'
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP());
```

#### **3. Cache Size Estimation**
```sql
-- Estimate cache size for a specific query pattern
SELECT
    query_text,
    COUNT(*) AS execution_count,
    SUM(credits_used) AS total_credits_saved,
    SUM(bytes_scanned) / 1024 / 1024 AS estimated_cache_size_mb
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    used_cached_result = TRUE
    AND query_text LIKE '%your_query_pattern%'
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    query_text
ORDER BY
    total_credits_saved DESC;
```

### **E. Performance Characteristics**

#### **1. Performance Impact**

| **Metric** | **Without Cache** | **With Cache (Hit)** | **With Cache (Miss)** | **Improvement (Hit)** |
|------------|-------------------|---------------------|----------------------|------------------------|
| Execution Time | 100ms - 10s+ | 1ms - 10ms | 100ms - 10s+ | **10x - 1000x faster** |
| Bytes Scanned | Full query scan | 0 | Full query scan | **Infinite reduction** |
| Credits Used | Normal query cost | 0 | Normal query cost | **Infinite reduction** |
| Network Traffic | Full result set | Minimal | Full result set | **90-99% reduction** |
| Warehouse Load | Full query execution | None | Full query execution | **100% reduction** |
| Compilation Time | Normal | Normal | Normal | No change |

#### **2. Real-World Performance Examples**

**Example 1: Dashboard Query**
```sql
-- Query: Daily sales by region
SELECT
    region,
    DATE_TRUNC('DAY', sale_date) AS day,
    SUM(amount) AS total_sales,
    COUNT(*) AS transaction_count
FROM
    sales
WHERE
    sale_date > CURRENT_DATE() - 30
GROUP BY
    region, DATE_TRUNC('DAY', sale_date)
ORDER BY
    day, region;
```

| **Scenario** | **Execution Time** | **Bytes Scanned** | **Credits Used** | **Network Traffic** |
|--------------|--------------------|-------------------|------------------|---------------------|
| First Execution | 8.5 seconds | 45 GB | 4.25 credits | 450 MB |
| Subsequent (Cache Hit) | 8 ms | 0 | 0 credits | 1 MB |
| After Data Change | 8.5 seconds | 45 GB | 4.25 credits | 450 MB |

**Improvement**:
- **Execution Time**: 1000x faster
- **Bytes Scanned**: Infinite reduction
- **Credits Used**: Infinite reduction
- **Network Traffic**: 450x reduction


**Example 2: Complex Aggregation Query**
```sql
-- Query: Customer lifetime value
SELECT
    c.customer_id,
    c.name,
    c.region,
    COUNT(o.order_id) AS order_count,
    SUM(o.amount) AS total_spend,
    MAX(o.order_date) AS last_order_date,
    DATEDIFF('day', MIN(o.order_date), CURRENT_DATE()) AS days_as_customer
FROM
    customers c
LEFT JOIN
    orders o ON c.customer_id = o.customer_id
WHERE
    o.order_date > CURRENT_DATE() - 365
GROUP BY
    c.customer_id, c.name, c.region
ORDER BY
    total_spend DESC;
```

| **Scenario** | **Execution Time** | **Bytes Scanned** | **Credits Used** |
|--------------|--------------------|-------------------|------------------|
| First Execution | 12.3 seconds | 120 GB | 6.15 credits |
| Subsequent (Cache Hit) | 12 ms | 0 | 0 credits |
| After Schema Change | 12.3 seconds | 120 GB | 6.15 credits |

**Improvement**: **1000x faster** with cache hit

### **F. Best Practices**

#### **1. When to Use Query Result Cache**
**Good Candidates**:
✅ **Repetitive queries** (dashboards, reports, BI tools)
✅ **Identical query text** (same SQL, same parameters)
✅ **Read-only workloads** (no DML operations)
✅ **Expensive queries** (high credit usage, long execution time)
✅ **Static data** (data that doesn't change frequently)
✅ **Parameterized queries** with consistent parameters
✅ **Queries with high bytes_scanned**

**Poor Candidates**:
❌ **Unique queries** (each query is different)
❌ **Frequently changing data** (cache will be constantly invalidated)
❌ **Write-heavy workloads** (DML operations invalidate cache)
❌ **Real-time data** (cache TTL may be too long)
❌ **Ad-hoc analysis** (queries are typically unique)
❌ **DML/DDL operations** (not cacheable)

#### **2. Optimization Techniques**

| **Technique** | **Description** | **Example** |
|---------------|-----------------|-------------|
| **Use Consistent Query Formatting** | Ensure identical query text for cache hits | `SELECT col1, col2 FROM table1` (not `select  col1,col2 from table1`) |
| **Use Parameterized Queries** | Enables cache reuse for similar queries | `SELECT * FROM table1 WHERE id = ?` |
| **Set Appropriate TTL** | Balance freshness and performance | `ALTER SESSION SET RESULT_CACHE_TTL = 3600` (1 hour) |
| **Batch Data Changes** | Minimize cache invalidation | Run updates during off-peak hours |
| **Use for Expensive Queries** | Prioritize caching for high-cost queries | Queries using >10 credits |
| **Monitor Cache Hit Rate** | Track effectiveness of caching | `SELECT used_cached_result FROM QUERY_HISTORY` |
| **Disable for Unique Queries** | Avoid cache overhead for non-repetitive queries | `ALTER SESSION SET USE_CACHED_RESULTS = FALSE` |
| **Use Consistent Session Parameters** | Ensure same timezone, date formats, etc. | `ALTER SESSION SET TIMEZONE = 'UTC'` |
| **Combine with Other Optimizations** | Use with clustering, materialized views, warehouse cache | `CLUSTER BY (date) + USE_CACHED_RESULTS = TRUE + larger warehouse` |
| **Use for Dashboard Queries** | Cache dashboard queries that run frequently | BI tools, reporting dashboards |
| **Use for Batch Processing** | Cache intermediate results in ETL pipelines | Temporary tables, CTEs |

#### **3. Common Pitfalls and Solutions**

| **Pitfall** | **Symptom** | **Root Cause** | **Solution** |
|-------------|-------------|----------------|--------------|
| **Cache Misses Due to Query Text Differences** | Low cache hit rate | Whitespace, case, or comment differences | Standardize query formatting |
| **Cache Invalidation from Frequent Updates** | Poor performance despite caching | Data changes invalidating cache | Batch updates, use shorter TTL |
| **Cache Not Used for Parameterized Queries** | No cache hits for similar queries | Different parameter values | Use bind variables consistently, or implement application-level caching |
| **High Memory Usage from Large Result Sets** | Warehouse memory pressure | Large cached results | Use filtering to reduce result size, use LIMIT |
| **Cache Not Working for Views** | Views not cached | Views are not directly cached | Query the underlying tables directly |
| **Cache Not Working Across Sessions** | Cache misses in new sessions | Session-scoped cache | Use connection pooling |
| **Cache TTL Too Long for Volatile Data** | Stale results | TTL longer than data freshness | Reduce TTL or disable caching |
| **Cache TTL Too Short for Static Data** | Unnecessary re-executions | TTL shorter than data freshness | Increase TTL |
| **Cache Not Working with Different Roles** | Cache misses for same query with different roles | Different permissions = different cache entry | Use consistent roles for repetitive queries |
| **Cache Not Working with Different Timezones** | Cache misses for same query with different timezones | Different session parameters = different cache entry | Set timezone explicitly |

### **G. Limitations**

| **Limitation** | **Description** | **Workaround** |
|----------------|-----------------|---------------|
| **Exact Query Text Match Required** | Cache key includes exact query text (including whitespace, case, comments) | Standardize query formatting, use parameterized queries |
| **Data Changes Invalidate Cache** | Any DML on source tables invalidates cache | Batch updates, use materialized views for volatile data |
| **Schema Changes Invalidate Cache** | DDL on source tables invalidates cache | Batch DDL changes, use views to abstract schema changes |
| **Permission Changes Invalidate Cache** | Changes to user roles/privileges invalidate cache | Use consistent roles for repetitive queries |
| **Session Parameter Changes Invalidate Cache** | Changes to timezone, formats, etc. invalidate cache | Set session parameters explicitly |
| **Maximum TTL of 24 Hours** | Cannot cache results longer than 24 hours | Use materialized views for longer caching |
| **No Partial Result Caching** | Cannot cache partial result sets | Use LIMIT or pagination to reduce result size |
| **No Caching for DML/DDL** | INSERT, UPDATE, DELETE, CREATE, ALTER, DROP are not cached | N/A (expected behavior) |
| **No Caching for Views** | Views themselves are not cached (but underlying queries may be) | Query the underlying tables directly |
| **Storage Overhead** | Cached results consume SSD storage | Monitor storage usage, drop unused cached results |
| **Memory Overhead** | Cache metadata consumes memory | Monitor warehouse memory usage |
| **No Direct Cache Management** | Cannot manually manage cache contents | Use TTL and query patterns to influence cache |
| **No Cache for External Tables** | Limited caching for external tables | Use internal tables or materialized views for hot external data |

### **H. Practical Examples**

#### **Example 1: Dashboard Optimization**
```sql
-- Enable result caching for dashboard users
ALTER ROLE dashboard_role SET USE_CACHED_RESULTS = TRUE;

-- Set shorter TTL for frequently updated dashboards
ALTER ROLE dashboard_role SET RESULT_CACHE_TTL = 1800;  -- 30 minutes

-- Dashboard query (will be cached)
SELECT
    region,
    DATE_TRUNC('DAY', sale_date) AS day,
    SUM(amount) AS total_sales,
    COUNT(*) AS transaction_count
FROM
    sales
WHERE
    sale_date > CURRENT_DATE() - 30
GROUP BY
    region, DATE_TRUNC('DAY', sale_date)
ORDER BY
    day, region;

-- Check cache usage for dashboard queries
SELECT
    query_id,
    query_text,
    used_cached_result,
    execution_time,
    credits_used
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%total_sales%'
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;
```

#### **Example 2: BI Tool Optimization**
```sql
-- Create a role for BI tool users
CREATE ROLE bi_tool_role;
GRANT USAGE ON WAREHOUSE bi_wh TO ROLE bi_tool_role;
GRANT USAGE ON DATABASE my_db TO ROLE bi_tool_role;
GRANT USAGE ON SCHEMA my_schema TO ROLE bi_tool_role;
GRANT SELECT ON ALL TABLES IN SCHEMA my_schema TO ROLE bi_tool_role;

-- Enable result caching for BI tool role
ALTER ROLE bi_tool_role SET USE_CACHED_RESULTS = TRUE;
ALTER ROLE bi_tool_role SET RESULT_CACHE_TTL = 7200;  -- 2 hours

-- BI tool query (will be cached)
SELECT
    product_category,
    SUM(revenue) AS category_revenue,
    SUM(quantity) AS category_quantity
FROM
    sales
WHERE
    sale_date > CURRENT_DATE() - 7
GROUP BY
    product_category
ORDER BY
    category_revenue DESC;

-- Grant role to BI tool user
GRANT ROLE bi_tool_role TO USER bi_tool_user;

-- Check cache performance
SELECT
    query_id,
    query_text,
    used_cached_result,
    execution_time,
    credits_used
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    user_name = 'BI_TOOL_USER'
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY
    execution_time;
```

#### **Example 3: Parameterized Query Caching (Application-Level)**
```sql
-- Application code using parameterized queries and application-level caching
// Java example with Guava Cache
import com.google.common.cache.Cache;
import com.google.common.cache.CacheBuilder;
import java.util.concurrent.TimeUnit;

public class SnowflakeQueryService {
    private final Cache<String, ResultSet> queryCache;
    private final Connection connection;

    public SnowflakeQueryService(Connection connection) {
        // Create a cache with 1-hour expiration
        this.queryCache = CacheBuilder.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(1, TimeUnit.HOURS)
            .build();
        this.connection = connection;
    }

    public ResultSet executeQuery(String sql, Object... params) throws SQLException {
        // Generate a cache key from the SQL and parameters
        String cacheKey = sql + ":" + String.join(",", Arrays.stream(params).map(Object::toString).toArray(String[]::new));

        try {
            // Try to get from cache
            return queryCache.get(cacheKey, () -> {
                PreparedStatement stmt = connection.prepareStatement(sql);
                for (int i = 0; i < params.length; i++) {
                    stmt.setObject(i + 1, params[i]);
                }
                return stmt.executeQuery();
            });
        } catch (Exception e) {
            // Fallback to direct execution
            PreparedStatement stmt = connection.prepareStatement(sql);
            for (int i = 0; i < params.length; i++) {
                stmt.setObject(i + 1, params[i]);
            }
            return stmt.executeQuery();
        }
    }
}

// Usage
SnowflakeQueryService queryService = new SnowflakeQueryService(connection);

// First call (cache miss)
ResultSet rs1 = queryService.executeQuery(
    "SELECT * FROM products WHERE category = ? AND price > ?",
    "Electronics", 100.0
);

// Second call with same parameters (cache hit)
ResultSet rs2 = queryService.executeQuery(
    "SELECT * FROM products WHERE category = ? AND price > ?",
    "Electronics", 100.0
);

// Third call with different parameters (cache miss)
ResultSet rs3 = queryService.executeQuery(
    "SELECT * FROM products WHERE category = ? AND price > ?",
    "Clothing", 50.0
);
```

#### **Example 4: Cache TTL Tuning**
```sql
-- For a dashboard that updates every 15 minutes
ALTER SESSION SET RESULT_CACHE_TTL = 900;  -- 15 minutes

-- For a report that updates hourly
ALTER SESSION SET RESULT_CACHE_TTL = 3600;  -- 1 hour

-- For a static report that updates daily
ALTER SESSION SET RESULT_CACHE_TTL = 86400;  -- 24 hours (max)

-- For a real-time dashboard (disable caching)
ALTER SESSION SET USE_CACHED_RESULTS = FALSE;
```

#### **Example 5: Cache Monitoring and Alerting**
```sql
-- Create an alert for low cache hit rate
CREATE OR REPLACE ALERT low_cache_hit_rate_alert
  WAREHOUSE = monitoring_wh
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
    AND warehouse_name = 'DASHBOARD_WH'
  HAVING
    SUM(CASE WHEN used_cached_result = TRUE THEN 1 ELSE 0 END) * 100.0 /
        NULLIF(COUNT(*), 0) < 70;  -- Cache hit rate < 70%

-- Create an alert for cache not being used
CREATE OR REPLACE ALERT cache_not_used_alert
  WAREHOUSE = monitoring_wh
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'  -- Every 5 minutes
AS
  SELECT
    query_text,
    COUNT(*) AS execution_count,
    SUM(CASE WHEN used_cached_result = TRUE THEN 1 ELSE 0 END) AS cache_hits,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE
    start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
    AND warehouse_name = 'DASHBOARD_WH'
    AND query_text LIKE '%daily_sales%'  -- Specific query pattern
  GROUP BY
    query_text
  HAVING
    COUNT(*) > 5  -- Query executed more than 5 times
    AND SUM(CASE WHEN used_cached_result = TRUE THEN 1 ELSE 0 END) = 0;  -- No cache hits
```

## **3. Warehouse Cache Deep Dive**

### **A. Architecture and Workflow**

```mermaid
%% Warehouse Cache Workflow
flowchart TD
    A[("Query Submission")] --> B[("Query Parser")]
    B --> C[("Identify Required Data")]
    C --> D[("Warehouse Cache Lookup")]
    D --> E{Cache Hit?}
    E -->|Yes| F[("Return Cached Data\n1-10ms")]
    E -->|No| G[("Fetch from Cloud Storage\n100-500ms")]
    G --> H[("Cache Data in SSD")]
    H --> F
    C --> I[("Required Data Types")]
    I --> J[("Micro-Partitions")]
    I --> K[("Temporary Tables")]
    I --> L[("CTE Results")]
    I --> M[("Intermediate Results")]
    I --> N[("Spill Data")]

    subgraph CacheStorage["Cache Storage"]
        D --> O[("Warehouse Cache\n(SSD)")]
        H --> O
    end

    subgraph CloudStorage["Cloud Storage"]
        G --> P[("S3 / Azure Blob / GCS")]
    end

    subgraph Execution["Execution"]
        F --> Q[("Query Execution Engine")]
        G --> Q
    end

    subgraph CacheDetails["Cache Details"]
        D --> R[("Cache Key:")]
        R --> S[("Micro-Partition ID")]
        R --> T[("Temporary Table Name")]
        R --> U[("CTE Name")]
        R --> V[("Query ID")]
    end

    subgraph Monitoring["Monitoring"]
        W[("QUERY_PROFILE")]
        X[("WAREHOUSE_LOAD_HISTORY")]
    end
    Q --> W
    G --> X

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef process fill:#4285f4,stroke:#1976d2;
    classDef decision fill:#ff9800,stroke:#f57c00;
    classDef cache fill:#009688,stroke:#00796b;
    classDef storage fill:#29abe2,stroke:#1a8fb8;
    classDef cloud fill:#e91e63,stroke:#c2185b;
    classDef execution fill:#9c27b0,stroke:#7b1fa2;
    classDef details fill:#3f51b5,stroke:#303f9f;
    classDef monitoring fill:#795548,stroke:#5d4037;
    class A,B,C,G,H,Q process;
    class D,E decision;
    class F,O cache;
    class J,K,L,M,N default;
    class P cloud;
    class R,S,T,U,V details;
    class W,X monitoring;
```

### **B. Technical Deep Dive**

#### **1. How Warehouse Cache Works**
1. **Data Caching**:
   - When a query is executed, Snowflake **identifies the required data** (micro-partitions, temporary tables, CTEs, etc.)
   - If the data is **not in cache**, it's **fetched from cloud storage** (S3, Azure Blob, GCS)
   - If the data is **frequently accessed**, it's **cached in local SSD** for the warehouse

2. **Cache Lookup**:
   - For subsequent queries, Snowflake **checks the warehouse cache** first
   - If the data is **cached**, it's **returned from SSD** (typical latency: **1-10ms**)
   - If the data is **not cached**, it's **fetched from cloud storage** (typical latency: **100-500ms**)

3. **Cache Management**:
   - **Cache Size**: Limited by the **warehouse size** (larger warehouses have more cache capacity)
     - X-Small: ~16GB cache
     - Small: ~32GB cache
     - Medium: ~64GB cache
     - Large: ~128GB cache
     - X-Large: ~256GB cache
     - 2X-Large: ~512GB cache
     - 3X-Large: ~1TB cache
     - 4X-Large: ~2TB cache
   - **Eviction Policy**: **Least Recently Used (LRU)** - When the cache is full, the least recently accessed data is evicted
   - **Scope**: **Per-warehouse** - Each warehouse has its own cache

4. **Cache Population**:
   - The cache is **populated on-demand** as queries access data
   - **Hot data** (frequently accessed micro-partitions) stays in cache
   - **Cold data** (infrequently accessed micro-partitions) may be evicted

#### **2. Cached Data Types**
| **Data Type** | **Description** | **Scope** | **Invalidation Triggers** | **Example** |
|---------------|-----------------|-----------|---------------------------|-------------|
| **Micro-Partitions** | Data from internal tables | Per-warehouse | Warehouse restart, LRU eviction | `SELECT * FROM my_table WHERE date > '2023-01-01'` |
| **Temporary Tables** | Tables created with `CREATE TEMPORARY TABLE` | Session | Session end, warehouse restart | `CREATE TEMPORARY TABLE temp AS SELECT * FROM my_table` |
| **CTE Results** | Results from Common Table Expressions (WITH clauses) | Query | Warehouse restart, LRU eviction | `WITH cte AS (SELECT * FROM my_table) SELECT * FROM cte` |
| **Intermediate Results** | Results from subqueries or join operations | Query | Warehouse restart, LRU eviction | `SELECT * FROM (SELECT * FROM t1 JOIN t2 ON ...) AS subq` |
| **Spill Data** | Data spilled to disk during query execution | Query | Warehouse restart, LRU eviction | Large sorts, aggregations, joins |

#### **3. Cache Storage Details**
- **Storage Medium**: SSD (NVMe in most cases)
- **Data Format**: Compressed columnar format (same as Snowflake's internal storage)
- **Compression**: Data is stored in compressed format to minimize storage usage
- **Encryption**: Data is encrypted at rest
- **Storage Cost**: Included in Snowflake's storage pricing

#### **4. Cache Behavior with Different Operations**
| **Operation** | **Cacheable?** | **Notes** | **Example** |
|---------------|----------------|-----------|-------------|
| **SELECT from Internal Tables** | ✅ Yes | Micro-partitions are cached | `SELECT * FROM my_table` |
| **SELECT from External Tables** | ✅ Yes | Micro-partitions from external stages are cached | `SELECT * FROM my_external_table` |
| **CREATE TEMPORARY TABLE** | ✅ Yes | Temporary tables are cached | `CREATE TEMPORARY TABLE temp AS SELECT * FROM my_table` |
| **WITH clause (CTE)** | ✅ Yes | CTE results may be cached | `WITH cte AS (SELECT * FROM my_table) SELECT * FROM cte` |
| **Subqueries** | ✅ Yes | Intermediate results may be cached | `SELECT * FROM (SELECT * FROM my_table) AS subq` |
| **JOIN operations** | ✅ Yes | Intermediate join results may be cached | `SELECT * FROM t1 JOIN t2 ON t1.id = t2.id` |
| **Aggregations** | ✅ Yes | Intermediate aggregation results may be cached | `SELECT region, SUM(sales) FROM sales GROUP BY region` |
| **Spill to Disk** | ✅ Yes | Spill data is cached in warehouse cache | Large sorts, aggregations |
| **DML on Temporary Tables** | ✅ Yes | Changes to temporary tables are cached | `UPDATE temp_table SET col1 = 1` |
| **DDL on Temporary Tables** | ❌ No | DDL operations are not cached | `ALTER TABLE temp_table ADD COLUMN col1 INT` |
| **Session End** | ⚠️ Partial | Session-scoped data (temporary tables) is invalidated | Session termination |
| **Warehouse Restart** | ❌ No | All cached data is invalidated | `ALTER WAREHOUSE my_wh SUSPEND; ALTER WAREHOUSE my_wh RESUME;` |

**Note**: Auto-suspend **does not invalidate** the warehouse cache. The cache is **retained** while the warehouse is suspended and **available immediately** when the warehouse resumes.

#### **5. Cache Invalidation**
Warehouse cache is **automatically invalidated** when:
1. **Warehouse Restart**: When the warehouse is **stopped and restarted** (auto-suspend does not invalidate cache)
2. **Session End**: When the **session terminates** (session-scoped data like temporary tables may be evicted)
3. **LRU Eviction**: When the **cache is full**, least recently used data is evicted
4. **Data Changes**: **Not automatically invalidated** for DML operations (Snowflake uses MVCC, so queries see the latest version)

**Important**: Unlike result cache, warehouse cache is **not invalidated by DML operations** because Snowflake uses **Multi-Version Concurrency Control (MVCC)**. However, queries will always see the **latest committed version** of the data.

### **C. Configuration**

**Note**: Warehouse cache is **automatic** and **requires no configuration**. However, you can **influence** the cache by:

#### **1. Warehouse Sizing for Cache Capacity**
```sql
-- Create a warehouse with appropriate size for your cache needs
CREATE WAREHOUSE small_wh WAREHOUSE_SIZE = 'SMALL';      -- ~32GB cache
CREATE WAREHOUSE medium_wh WAREHOUSE_SIZE = 'MEDIUM';    -- ~64GB cache
CREATE WAREHOUSE large_wh WAREHOUSE_SIZE = 'LARGE';      -- ~128GB cache
CREATE WAREHOUSE xlarge_wh WAREHOUSE_SIZE = 'X-LARGE';    -- ~256GB cache
CREATE WAREHOUSE xxlarge_wh WAREHOUSE_SIZE = 'XX-LARGE';  -- ~512GB cache
CREATE WAREHOUSE xxxlarge_wh WAREHOUSE_SIZE = 'XXX-LARGE'; -- ~1TB cache
CREATE WAREHOUSE xxxxlarge_wh WAREHOUSE_SIZE = 'XXXX-LARGE'; -- ~2TB cache

-- Resize an existing warehouse
ALTER WAREHOUSE my_wh SET WAREHOUSE_SIZE = 'X-LARGE';
```

#### **2. Auto-Suspend Configuration**
```sql
-- Set auto-suspend to balance cost and cache retention
ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = 600;  -- 10 minutes (cache retained during suspend)
ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = 1800; -- 30 minutes
ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = NULL; -- Disable auto-suspend (cache always retained)
```

**Note**: Auto-suspend **does not invalidate** the warehouse cache. The cache is **retained** while the warehouse is suspended and **available immediately** when the warehouse resumes.

#### **3. Multi-Cluster Warehouse Configuration**
```sql
-- Create a multi-cluster warehouse for high concurrency
CREATE WAREHOUSE mc_wh
  WAREHOUSE_SIZE = 'LARGE'
  MAX_CLUSTER_COUNT = 4
  MIN_CLUSTER_COUNT = 1
  SCALING_POLICY = 'STANDARD';

-- Each cluster has its own warehouse cache
-- Data accessed by one cluster is not automatically cached in other clusters
```

#### **4. Temporary Table Configuration**
```sql
-- Create a temporary table (cached in warehouse cache)
CREATE TEMPORARY TABLE temp_sales AS
SELECT * FROM sales WHERE sale_date > CURRENT_DATE() - 7;

-- Query the temporary table (benefits from warehouse cache)
SELECT * FROM temp_sales WHERE region = 'US';

-- Reuse the temporary table in subsequent queries
SELECT region, SUM(amount) AS total_sales
FROM temp_sales
GROUP BY region;
```

### **D. Monitoring and Metrics**

#### **1. Monitor Temporary Table Performance**
```sql
-- Check query performance for temporary tables
SELECT
    query_id,
    query_text,
    execution_time,
    bytes_scanned,
    start_time,
    warehouse_name
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%TEMPORARY TABLE%'
    OR query_text LIKE '%temp_%'
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    execution_time;

-- Compare performance with and without temporary tables
-- Without temporary table
SELECT * FROM sales WHERE sale_date > CURRENT_DATE() - 7;
SELECT LAST_QUERY_ID() AS query_id_without_temp;

-- With temporary table
CREATE TEMPORARY TABLE temp_sales AS SELECT * FROM sales WHERE sale_date > CURRENT_DATE() - 7;
SELECT * FROM temp_sales;
SELECT LAST_QUERY_ID() AS query_id_with_temp;

-- Compare
SELECT
    query_id,
    query_text,
    execution_time,
    bytes_scanned
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_id IN ('query_id_without_temp', 'query_id_with_temp')
ORDER BY
    query_id;
```

#### **2. Monitor CTE Performance**
```sql
-- Check query profile for CTE performance
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'));

-- Look for CTE-related steps in the profile
SELECT
    step_id,
    operation,
    execution_time,
    bytes_scanned,
    rows_produced
FROM
    TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))
WHERE
    operation LIKE '%CTE%'
ORDER BY
    execution_time;

-- Compare performance with and without CTEs
-- Without CTE
SELECT
    region,
    SUM(amount) AS total_sales
FROM
    (SELECT * FROM sales WHERE sale_date > CURRENT_DATE() - 7) s
GROUP BY
    region;
SELECT LAST_QUERY_ID() AS query_id_without_cte;

-- With CTE
WITH sales_cte AS (
    SELECT * FROM sales WHERE sale_date > CURRENT_DATE() - 7
)
SELECT
    region,
    SUM(amount) AS total_sales
FROM
    sales_cte
GROUP BY
    region;
SELECT LAST_QUERY_ID() AS query_id_with_cte;

-- Compare
SELECT
    query_id,
    query_text,
    execution_time,
    compilation_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_id IN ('query_id_without_cte', 'query_id_with_cte')
ORDER BY
    query_id;
```

#### **3. Monitor Warehouse Cache Efficiency**
```sql
-- Check warehouse utilization (indirect indicator of cache usage)
SELECT
    warehouse_name,
    start_time,
    end_time,
    running_queries,
    queued_queries,
    credit_usage
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE
    warehouse_name = 'MY_WH'
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;

-- Check for queries with low execution time (may indicate cache hits)
SELECT
    query_id,
    query_text,
    warehouse_name,
    execution_time,
    bytes_scanned,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    warehouse_name = 'MY_WH'
    AND execution_time < 100  -- <100ms (likely cache hit)
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    execution_time;
```

#### **4. Monitor Cache Eviction**
```sql
-- Check for warehouse restarts (which invalidate cache)
SELECT
    warehouse_name,
    event_time,
    event_type,
    old_size,
    new_size
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_EVENTS_HISTORY
WHERE
    warehouse_name = 'MY_WH'
    AND event_type IN ('SUSPEND', 'RESUME', 'ALTER')
    AND event_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    event_time DESC;

-- Check for queries that may have caused cache eviction
-- (Look for queries with high bytes_scanned that may have filled the cache)
SELECT
    query_id,
    query_text,
    warehouse_name,
    bytes_scanned,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    warehouse_name = 'MY_WH'
    AND bytes_scanned > 100 * 1024 * 1024 * 1024  -- >100GB
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    bytes_scanned DESC;
```

### **E. Performance Characteristics**

#### **1. Performance Impact**

| **Metric** | **Without Warehouse Cache** | **With Warehouse Cache (Hit)** | **With Warehouse Cache (Miss)** | **Improvement (Hit)** |
|------------|--------------------------------|----------------------------------|------------------------------------|------------------------|
| I/O Latency | 100-500ms (cloud storage) | 1-10ms (SSD) | 100-500ms (cloud storage) | **10-100x faster** |
| Throughput | Limited by cloud storage | Limited by SSD | Limited by cloud storage | **2-10x higher** |
| Execution Time | Higher (waiting for I/O) | Lower (fast I/O) | Higher (waiting for I/O) | **2-10x faster** |
| Bytes Scanned | Full scan from cloud storage | Cached data from SSD | Full scan from cloud storage | **Same** (but faster) |
| Credit Usage | Higher (more time spent waiting for I/O) | Lower (less time waiting for I/O) | Higher (more time spent waiting for I/O) | **1.1-2x reduction** |
| Temporary Table Performance | Slow (recomputed each time) | Fast (cached) | Slow (recomputed) | **10-100x faster** |
| CTE Performance | Slow (recomputed each time) | Fast (cached) | Slow (recomputed) | **2-10x faster** |

#### **2. Real-World Performance Examples**

**Example 1: Temporary Table in ETL Pipeline**
```sql
-- ETL pipeline with temporary tables
-- Step 1: Extract data from source
CREATE TEMPORARY TABLE temp_customers AS
SELECT * FROM source_customers WHERE last_updated > CURRENT_DATE() - 1;

-- Step 2: Transform data
CREATE TEMPORARY TABLE temp_customer_orders AS
SELECT
    c.customer_id,
    c.name,
    o.order_id,
    o.order_date,
    o.amount
FROM
    temp_customers c
JOIN
    source_orders o ON c.customer_id = o.customer_id;

-- Step 3: Load to target
INSERT INTO target_sales
SELECT * FROM temp_customer_orders;
```

| **Step** | **Execution Time (Without Cache)** | **Execution Time (With Cache)** | **Improvement** |
|----------|--------------------------------------|----------------------------------|-----------------|
| Step 1 | 5.2s | 5.2s | No change (first access) |
| Step 2 | 8.7s | 1.1s | 8x faster (temp_customers cached) |
| Step 3 | 3.4s | 0.5s | 7x faster (temp_customer_orders cached) |
| **Total** | 17.3s | 6.8s | **2.5x faster** |

**Note**: The first access to each temporary table is not cached, but subsequent accesses benefit from the warehouse cache.


**Example 2: Complex Query with CTEs**
```sql
-- Complex query with multiple CTEs
WITH
sales_2023 AS (
    SELECT * FROM sales WHERE YEAR(sale_date) = 2023
),
customer_sales AS (
    SELECT
        customer_id,
        SUM(amount) AS total_spend
    FROM
        sales_2023
    GROUP BY
        customer_id
),
top_customers AS (
    SELECT * FROM customer_sales
    WHERE total_spend > 10000
    ORDER BY
        total_spend DESC
    LIMIT 100
)
SELECT
    c.customer_id,
    c.name,
    c.region,
    tc.total_spend
FROM
    customers c
JOIN
    top_customers tc ON c.customer_id = tc.customer_id;
```

| **Scenario** | **Execution Time** | **Compilation Time** | **Total Time** |
|--------------|--------------------|-----------------------|---------------|
| First Execution | 12.5s | 300ms | 12.8s |
| Subsequent Executions (Cache Hit) | 1.2s | 50ms | 1.25s |

**Improvement**: **10x faster** with warehouse cache

### **F. Best Practices**

#### **1. When to Use Warehouse Cache**
**Good Candidates**:
✅ **Frequently accessed data** (hot data)
✅ **Large tables** with repeated queries
✅ **BI tools and dashboards** with repetitive queries
✅ **Analytical workloads** with data locality
✅ **Multi-cluster warehouses** for high concurrency
✅ **Queries with filters** on clustered columns
✅ **Temporary tables** in ETL pipelines
✅ **CTEs (Common Table Expressions)** in complex queries
✅ **Intermediate results** in multi-step queries
✅ **Large queries** that spill to disk

**Poor Candidates**:
⚠️ **Ad-hoc queries** (data not likely to be re-accessed)
⚠️ **Write-heavy workloads** (cache may be frequently evicted)
⚠️ **Small warehouses** (limited cache capacity)
⚠️ **Cold data** (infrequently accessed data)
⚠️ **Very large scans** (may evict other cached data)

#### **2. Optimization Techniques**

| **Technique** | **Description** | **Example** |
|---------------|-----------------|-------------|
| **Use Larger Warehouses for Hot Data** | Larger warehouses have more cache capacity | `CREATE WAREHOUSE hot_data_wh WAREHOUSE_SIZE = 'X-LARGE'` |
| **Query the Same Data Repeatedly** | Populate the cache with frequently accessed data | Dashboards, reports, BI tools |
| **Use Clustering for Cache Efficiency** | Cluster tables to improve cache hit rates | `ALTER TABLE my_table CLUSTER BY (date)` |
| **Use Auto-Suspend for Cost Savings** | Suspend warehouses when idle to save costs | `ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = 600` |
| **Avoid Frequent Warehouse Restarts** | Cache is lost when warehouse is restarted | Use auto-suspend/resume instead of stop/start |
| **Use Multi-Cluster Warehouses for High Concurrency** | Distribute cache across multiple clusters | `CREATE WAREHOUSE mc_wh MAX_CLUSTER_COUNT = 4` |
| **Use Temporary Tables for Intermediate Results** | Temporary tables are cached in warehouse cache | `CREATE TEMPORARY TABLE temp AS SELECT ...` |
| **Reuse Temporary Tables** | Query the same temporary table multiple times | `SELECT * FROM temp_table WHERE ...` |
| **Use CTEs for Complex Queries** | CTE results may be cached in warehouse cache | `WITH cte AS (SELECT ...) SELECT * FROM cte` |
| **Monitor Cache Performance** | Check QUERY_PROFILE for cache hits | `SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))` |
| **Combine with Other Optimizations** | Use with result cache, local disk cache, etc. | `CREATE TEMPORARY TABLE temp AS SELECT ... + USE_CACHED_RESULTS = TRUE + larger warehouse` |
| **Document Cacheable Data** | Document which tables benefit from caching | Internal wiki or Confluence page |

#### **3. Common Pitfalls and Solutions**

| **Pitfall** | **Symptom** | **Root Cause** | **Solution** |
|-------------|-------------|----------------|--------------|
| **Cache Not Populated** | No performance improvement | Data not accessed repeatedly | Query hot data repeatedly to populate cache |
| **Cache Eviction Due to Large Scans** | Performance degrades after large queries | Large scans evict hot data from cache | Use smaller warehouses for large scans, larger warehouses for hot data |
| **Cache Lost on Warehouse Restart** | Performance degrades after warehouse restart | Cache is invalidated on restart | Use auto-suspend instead of stop/start |
| **Low Cache Hit Rate** | Poor performance despite caching | Data access patterns not cache-friendly | Use clustering, query hot data more frequently |
| **Cache Not Effective for Ad-Hoc Queries** | No improvement for ad-hoc queries | Ad-hoc queries access different data each time | Use result caching or materialized views for repetitive ad-hoc queries |
| **Cache Not Shared Across Clusters** | Performance varies across clusters | Each cluster has its own cache | Use single-cluster warehouses for cache-sensitive workloads |
| **Cache Not Effective for External Tables** | No improvement for external tables | External table data may not be cached effectively | Use internal tables or materialized views for hot external data |
| **High Memory Usage** | Warehouse memory pressure | Large cached data | Use filtering to reduce data volume, use smaller warehouses |
| **Temporary Tables Not Reused** | Temporary tables not providing benefit | Temporary tables not queried multiple times | Reuse temporary tables in subsequent queries |
| **CTEs Not Reused** | CTEs not providing benefit | CTEs not referenced multiple times | Reference CTEs multiple times in the same query |

### **G. Limitations**

| **Limitation** | **Description** | **Workaround** |
|----------------|-----------------|---------------|
| **Per-Warehouse Cache** | Each warehouse has its own cache | Use larger warehouses for hot data, multi-cluster for high concurrency |
| **Limited by Warehouse Size** | Cache size is limited by warehouse size | Use larger warehouses for more cache |
| **LRU Eviction** | Least recently used data is evicted when cache is full | Query important data frequently to keep it in cache |
| **No Direct Control** | Cannot manually manage cache contents | Use query patterns and warehouse sizing to influence cache |
| **Cache Lost on Warehouse Restart** | Cache is lost when warehouse is stopped and restarted | Use auto-suspend/resume instead of stop/start |
| **No Monitoring for Cache Hits** | Cache hits are not directly visible | Monitor query performance and I/O latency |
| **Session-Scoped for Some Data** | Temporary tables are session-scoped | Use global temporary tables or persistent tables for cross-session caching |
| **Storage Overhead** | Cache consumes SSD storage | Monitor warehouse storage usage |
| **Not Invalidated by DML** | Cache may contain stale data after DML on temporary tables | Snowflake uses MVCC, so queries see latest data (cache is eventually consistent) |
| **No Cache for All Data Types** | Some data types may not be cached effectively | Use internal tables for hot data |
| **No Direct Cache Size Monitoring** | Cannot directly monitor cache size | Estimate based on warehouse size and usage patterns |

### **H. Practical Examples**

#### **Example 1: Hot Table Caching**
```sql
-- Create a large warehouse for hot data
CREATE WAREHOUSE hot_data_wh
  WAREHOUSE_SIZE = 'X-LARGE'
  AUTO_SUSPEND = 1800
  AUTO_RESUME = TRUE;

-- Grant access to users
GRANT USAGE ON WAREHOUSE hot_data_wh TO ROLE bi_role;

-- Query hot tables repeatedly to populate cache
SELECT * FROM customers WHERE region = 'US' AND signup_date > CURRENT_DATE() - 30;
SELECT * FROM products WHERE category = 'Electronics' AND price > 100;
SELECT * FROM orders WHERE order_date > CURRENT_DATE() - 7;

-- Subsequent queries will benefit from warehouse cache
SELECT * FROM customers WHERE region = 'US' AND signup_date > CURRENT_DATE() - 30;
SELECT * FROM products WHERE category = 'Electronics' AND price > 100;
SELECT * FROM orders WHERE order_date > CURRENT_DATE() - 7;

-- Check query performance
SELECT
    query_id,
    query_text,
    execution_time,
    bytes_scanned,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    warehouse_name = 'HOT_DATA_WH'
    AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY
    execution_time;
```

#### **Example 2: Dashboard Optimization with Warehouse Cache**
```sql
-- Create a dedicated warehouse for dashboards
CREATE WAREHOUSE dashboard_wh
  WAREHOUSE_SIZE = 'LARGE'
  MAX_CLUSTER_COUNT = 2
  AUTO_SUSPEND = 3600
  AUTO_RESUME = TRUE;

-- Enable result caching for dashboards
ALTER WAREHOUSE dashboard_wh SET USE_CACHED_RESULTS = TRUE;
ALTER WAREHOUSE dashboard_wh SET RESULT_CACHE_TTL = 3600;  -- 1 hour

-- Create temporary tables for dashboard data
CREATE TEMPORARY TABLE temp_daily_sales AS
SELECT
    DATE_TRUNC('DAY', sale_date) AS day,
    region,
    SUM(amount) AS total_sales
FROM
    sales
WHERE
    sale_date > CURRENT_DATE() - 30
GROUP BY
    DATE_TRUNC('DAY', sale_date), region;

-- Dashboard queries (benefit from both result cache and warehouse cache)
-- Query 1: Daily sales by region
SELECT * FROM temp_daily_sales WHERE day > CURRENT_DATE() - 7;

-- Query 2: Top products
CREATE TEMPORARY TABLE temp_top_products AS
SELECT
    product_id,
    product_name,
    SUM(quantity) AS total_quantity,
    SUM(amount) AS total_sales
FROM
    sales s
JOIN
    products p ON s.product_id = p.product_id
WHERE
    sale_date > CURRENT_DATE() - 7
GROUP BY
    product_id, product_name
ORDER BY
    total_sales DESC
LIMIT 10;

SELECT * FROM temp_top_products;

-- Query 3: Customer analytics
CREATE TEMPORARY TABLE temp_customer_analytics AS
SELECT
    region,
    COUNT(DISTINCT customer_id) AS active_customers,
    SUM(amount) AS total_sales,
    AVG(amount) AS avg_order_value
FROM
    sales
WHERE
    sale_date > CURRENT_DATE() - 7
GROUP BY
    region;

SELECT * FROM temp_customer_analytics;

-- Check cache performance
SELECT
    query_id,
    query_text,
    used_cached_result,
    execution_time,
    bytes_scanned
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    warehouse_name = 'DASHBOARD_WH'
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY
    execution_time;
```

#### **Example 3: Multi-Cluster Warehouse for High Concurrency**
```sql
-- Create a multi-cluster warehouse for high concurrency workloads
CREATE WAREHOUSE high_concurrency_wh
  WAREHOUSE_SIZE = 'LARGE'
  MAX_CLUSTER_COUNT = 4
  MIN_CLUSTER_COUNT = 1
  SCALING_POLICY = 'STANDARD'
  AUTO_SUSPEND = 1800
  AUTO_RESUME = TRUE;

-- Each cluster has its own warehouse cache
-- Query 1 (runs on Cluster 1)
SELECT * FROM table1 WHERE date > CURRENT_DATE() - 7;

-- Query 2 (runs on Cluster 2)
SELECT * FROM table2 WHERE region = 'US';

-- Query 3 (runs on Cluster 3)
SELECT * FROM table3 WHERE category = 'Electronics';

-- Query 4 (runs on Cluster 4)
SELECT * FROM table4 WHERE status = 'Active';

-- Subsequent queries to the same tables will benefit from warehouse cache
-- on their respective clusters
SELECT * FROM table1 WHERE date > CURRENT_DATE() - 7;  -- Cluster 1 cache
SELECT * FROM table2 WHERE region = 'US';            -- Cluster 2 cache

-- Check cluster usage
SELECT
    warehouse_name,
    cluster_number,
    start_time,
    end_time,
    running_queries,
    queued_queries,
    credit_usage
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE
    warehouse_name = 'HIGH_CONCURRENCY_WH'
    AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY
    cluster_number, start_time;
```

#### **Example 4: Temporary Table Reuse**
```sql
-- Create a warehouse for ETL workloads
CREATE WAREHOUSE etl_wh
  WAREHOUSE_SIZE = 'X-LARGE'
  AUTO_SUSPEND = 3600
  AUTO_RESUME = TRUE;

-- Create temporary tables for intermediate results
-- These will be cached in the warehouse's local SSD cache
CREATE TEMPORARY TABLE temp_customers AS
SELECT * FROM customers WHERE signup_date > CURRENT_DATE() - 30;

CREATE TEMPORARY TABLE temp_orders AS
SELECT * FROM orders WHERE order_date > CURRENT_DATE() - 30;

-- Query the temporary tables (benefits from warehouse cache)
SELECT
    c.region,
    COUNT(o.order_id) AS order_count,
    SUM(o.amount) AS total_sales
FROM
    temp_customers c
LEFT JOIN
    temp_orders o ON c.customer_id = o.customer_id
GROUP BY
    c.region
ORDER BY
    total_sales DESC;

-- Reuse the temporary tables in subsequent queries
SELECT
    c.customer_id,
    c.name,
    COUNT(o.order_id) AS order_count
FROM
    temp_customers c
LEFT JOIN
    temp_orders o ON c.customer_id = o.customer_id
GROUP BY
    c.customer_id, c.name
ORDER BY
    order_count DESC;

-- Check query performance
SELECT
    query_id,
    query_text,
    execution_time,
    bytes_scanned
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    warehouse_name = 'ETL_WH'
    AND query_text LIKE '%temp_customers%'
    AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY
    execution_time;
```

#### **Example 5: Monitor and Alert on Warehouse Cache Performance**
```sql
-- Create an alert for high execution time on temporary tables
CREATE OR REPLACE ALERT high_temp_table_execution_time_alert
  WAREHOUSE = monitoring_wh
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'  -- Every 5 minutes
AS
  SELECT
    query_id,
    query_text,
    warehouse_name,
    execution_time,
    bytes_scanned,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE
    query_text LIKE '%TEMPORARY TABLE%'
    OR query_text LIKE '%temp_%'
    AND execution_time > 5000  -- >5 seconds
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
  ORDER BY
    execution_time DESC;

-- Create an alert for warehouse restarts (which invalidate cache)
CREATE OR REPLACE ALERT warehouse_restart_alert
  WAREHOUSE = monitoring_wh
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'  -- Every 5 minutes
AS
  SELECT
    warehouse_name,
    event_time,
    event_type,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_EVENTS_HISTORY
  WHERE
    event_type IN ('SUSPEND', 'RESUME', 'ALTER')
    AND warehouse_name IN ('ETL_WH', 'ANALYTICS_WH')
    AND event_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
  ORDER BY
    event_time DESC;

-- Create an alert for high I/O latency (may indicate cache misses)
CREATE OR REPLACE ALERT high_io_latency_alert
  WAREHOUSE = monitoring_wh
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'  -- Every 5 minutes
AS
  SELECT
    warehouse_name,
    query_id,
    query_text,
    execution_time,
    bytes_scanned,
    execution_time / NULLIF(bytes_scanned, 0) AS io_latency_per_gb,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE
    warehouse_name IN ('ETL_WH', 'ANALYTICS_WH')
    AND execution_time / NULLIF(bytes_scanned, 0) > 0.01  -- >10ms per GB
    AND bytes_scanned > 1 * 1024 * 1024 * 1024  -- >1GB
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
  ORDER BY
    io_latency_per_gb DESC;
```

## **4. Combined Architecture and Interaction**

### **Mermaid: Combined Query Result Cache and Warehouse Cache Architecture**
```mermaid
%% Combined Query Result Cache and Warehouse Cache Architecture
flowchart TD
    subgraph Client["Client Layer"]
        A[("Query")] --> B[("Query Parser")]
    end

    subgraph Caching["Caching Layer"]
        B --> C[("Query Result Cache\nCheck")]
        C --> D{Result Cache Hit?}
        D -->|Yes| E[("Return Cached Results\n<10ms")]
        D -->|No| F[("Warehouse Cache\nCheck")]
        F --> G{Warehouse Cache Hit?}
        G -->|Yes| H[("Return Cached Data\n<10ms")]
        G -->|No| I[("Execute Query")]
        I --> J[("Cache Results in\nWarehouse Cache")]
        J --> K[("Cache Results in\nQuery Result Cache")]
        K --> E
        H --> E
    end

    subgraph Execution["Execution Layer"]
        I --> L[("Query Execution Engine")]
    end

    subgraph Storage["Storage Layer"]
        J --> M[("Warehouse Cache\n(SSD)")]
        K --> N[("Query Result Cache\n(SSD)")]
        I --> O[("Cloud Storage\n(S3/Azure/GCS)")]
    end

    subgraph CacheDetails["Cache Details"]
        C --> P[("Result Cache Key:")]
        P --> Q[("Query Text")]
        P --> R[("User Permissions")]
        P --> S[("Session Parameters")]
        P --> T[("Data Version")]
        F --> U[("Warehouse Cache Key:")]
        U --> V[("Micro-Partition ID")]
        U --> W[("Temporary Table Name")]
        U --> X[("CTE Name")]
    end

    subgraph Monitoring["Monitoring Layer"]
        Y[("QUERY_HISTORY")]
        Z[("QUERY_PROFILE")]
        AA[("WAREHOUSE_LOAD_HISTORY")]
    end
    L --> Y
    E --> Y
    H --> Y
    L --> Z
    I --> AA

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef client fill:#4285f4,stroke:#1976d2;
    classDef caching fill:#ff9800,stroke:#f57c00;
    classDef decision fill:#009688,stroke:#00796b;
    classDef cache fill:#29abe2,stroke:#1a8fb8;
    classDef execution fill:#e91e63,stroke:#c2185b;
    classDef storage fill:#9c27b0,stroke:#7b1fa2;
    classDef details fill:#3f51b5,stroke:#303f9f;
    classDef monitoring fill:#795548,stroke:#5d4037;
    class A client;
    class B,C,D,E,F,G,H,I,J,K caching;
    class D,G decision;
    class E,H cache;
    class L execution;
    class M,N,O storage;
    class P,Q,R,S,T,U,V,W,X details;
    class Y,Z,AA monitoring;
```

### **How the Caches Work Together**

1. **Query Submission**:
   - A query is submitted to Snowflake

2. **Query Result Cache Check**:
   - Snowflake first checks the **query result cache**
   - If a **cache hit** occurs (identical query, same permissions, same data version), results are returned immediately from the result cache
   - If a **cache miss** occurs, proceed to warehouse cache check

3. **Warehouse Cache Check**:
   - Snowflake checks the **warehouse cache** for required data (micro-partitions, temporary tables, CTEs, etc.)
   - If a **cache hit** occurs, data is returned from the warehouse cache
   - If a **cache miss** occurs, data is fetched from cloud storage

4. **Query Execution**:
   - If both caches miss, the query is executed normally
   - Data is fetched from cloud storage
   - Intermediate results may be cached in the warehouse cache
   - Final results are cached in the query result cache

5. **Cache Population**:
   - **Query Result Cache**: Final results are cached for future identical queries
   - **Warehouse Cache**: Micro-partitions, temporary tables, CTEs, and intermediate results are cached for future access

### **Cache Interaction Examples**

#### **Example 1: Dashboard Query with Both Caches**
```sql
-- Create a warehouse for dashboards
CREATE WAREHOUSE dashboard_wh
  WAREHOUSE_SIZE = 'LARGE'
  AUTO_SUSPEND = 1800
  AUTO_RESUME = TRUE;

-- Enable result caching
ALTER WAREHOUSE dashboard_wh SET USE_CACHED_RESULTS = TRUE;
ALTER WAREHOUSE dashboard_wh SET RESULT_CACHE_TTL = 3600;  -- 1 hour

-- Create a temporary table for dashboard data
CREATE TEMPORARY TABLE temp_dashboard_data AS
SELECT
    region,
    DATE_TRUNC('DAY', sale_date) AS day,
    SUM(amount) AS total_sales
FROM
    sales
WHERE
    sale_date > CURRENT_DATE() - 30
GROUP BY
    region, DATE_TRUNC('DAY', sale_date);

-- Dashboard query (benefits from both caches)
-- First execution:
-- 1. Query result cache: MISS (first time)
-- 2. Warehouse cache: MISS (first time for temp table)
-- 3. Query executes, populates both caches
SELECT * FROM temp_dashboard_data WHERE day > CURRENT_DATE() - 7;

-- Second execution (within 1 hour):
-- 1. Query result cache: HIT (identical query)
-- 2. Returns results immediately from result cache
SELECT * FROM temp_dashboard_data WHERE day > CURRENT_DATE() - 7;

-- Third execution (after warehouse restart):
-- 1. Query result cache: HIT (still within TTL)
-- 2. Returns results immediately from result cache
SELECT * FROM temp_dashboard_data WHERE day > CURRENT_DATE() - 7;

-- Fourth execution (after TTL expiration):
-- 1. Query result cache: MISS (TTL expired)
-- 2. Warehouse cache: HIT (temp table still cached)
-- 3. Returns results from warehouse cache (faster than full execution)
SELECT * FROM temp_dashboard_data WHERE day > CURRENT_DATE() - 7;
```

#### **Example 2: ETL Pipeline with Both Caches**
```sql
-- Create a warehouse for ETL
CREATE WAREHOUSE etl_wh
  WAREHOUSE_SIZE = 'X-LARGE'
  AUTO_SUSPEND = 3600
  AUTO_RESUME = TRUE;

-- Disable result caching for ETL (queries are unique)
ALTER WAREHOUSE etl_wh SET USE_CACHED_RESULTS = FALSE;

-- ETL pipeline with temporary tables
-- Step 1: Extract (warehouse cache: MISS, result cache: N/A)
CREATE TEMPORARY TABLE temp_customers AS
SELECT * FROM source_customers WHERE last_updated > CURRENT_DATE() - 1;

-- Step 2: Transform (warehouse cache: HIT for temp_customers, MISS for source_orders)
CREATE TEMPORARY TABLE temp_customer_orders AS
SELECT
    c.customer_id,
    c.name,
    o.order_id,
    o.order_date,
    o.amount
FROM
    temp_customers c  -- Cached in warehouse cache
JOIN
    source_orders o ON c.customer_id = o.customer_id;

-- Step 3: Load (warehouse cache: HIT for both temp tables)
INSERT INTO target_sales
SELECT * FROM temp_customer_orders;

-- Subsequent ETL runs (within same session):
-- Step 1: Extract (warehouse cache: MISS for new data, but old data may still be cached)
CREATE OR REPLACE TEMPORARY TABLE temp_customers AS
SELECT * FROM source_customers WHERE last_updated > CURRENT_DATE() - 1;

-- Step 2: Transform (warehouse cache: HIT for temp_customers if data is still in cache)
CREATE OR REPLACE TEMPORARY TABLE temp_customer_orders AS
SELECT
    c.customer_id,
    c.name,
    o.order_id,
    o.order_date,
    o.amount
FROM
    temp_customers c  -- May be cached in warehouse cache
JOIN
    source_orders o ON c.customer_id = o.customer_id;
```

#### **Example 3: Complex Query with CTEs and Both Caches**
```sql
-- Enable result caching
ALTER SESSION SET USE_CACHED_RESULTS = TRUE;
ALTER SESSION SET RESULT_CACHE_TTL = 3600;  -- 1 hour

-- Complex query with CTEs
WITH
-- CTE 1: Filter recent sales (warehouse cache: MISS first time, HIT subsequent)
recent_sales AS (
    SELECT * FROM sales WHERE sale_date > CURRENT_DATE() - 7
),
-- CTE 2: Join with customers (warehouse cache: MISS first time, HIT subsequent)
sales_with_customers AS (
    SELECT
        s.sale_id,
        s.customer_id,
        s.sale_date,
        s.amount,
        c.name AS customer_name,
        c.region
    FROM
        recent_sales s
    JOIN
        customers c ON s.customer_id = c.customer_id
),
-- CTE 3: Aggregate by region (warehouse cache: MISS first time, HIT subsequent)
region_sales AS (
    SELECT
        region,
        SUM(amount) AS total_sales,
        COUNT(*) AS transaction_count
    FROM
        sales_with_customers
    GROUP BY
        region
)
-- Final query (result cache: MISS first time, HIT subsequent)
SELECT * FROM region_sales ORDER BY total_sales DESC;

-- First execution:
-- 1. Result cache: MISS
-- 2. Warehouse cache: MISS for all CTEs
-- 3. Query executes, populates both caches

-- Second execution (within 1 hour):
-- 1. Result cache: HIT
-- 2. Returns results immediately

-- Third execution (after warehouse restart, within TTL):
-- 1. Result cache: HIT
-- 2. Returns results immediately

-- Fourth execution (after TTL expiration):
-- 1. Result cache: MISS
-- 2. Warehouse cache: HIT for CTEs
-- 3. Returns results from warehouse cache (faster than full execution)
```

## **5. Performance Comparison and Benchmarks**

### **Performance Comparison Matrix**

| **Scenario** | **Without Caching** | **With Query Result Cache** | **With Warehouse Cache** | **With Both Caches** |
|--------------|---------------------|-----------------------------|---------------------------|------------------------|
| **Execution Time** | Baseline | 10-1000x faster | 2-10x faster | 10-1000x faster |
| **Bytes Scanned** | Baseline | 0 (for cache hits) | Baseline (but faster I/O) | 0 (for result cache hits) |
| **Credits Used** | Baseline | 0 (for cache hits) | Reduced (1.1-2x) | 0 (for result cache hits) |
| **I/O Latency** | 100-500ms | <10ms | 1-10ms | <10ms |
| **Compilation Time** | Baseline | Baseline | Baseline | Baseline |
| **Network Traffic** | Baseline | Minimal | Baseline | Minimal |
| **Warehouse Load** | Baseline | None (for cache hits) | Reduced | None (for result cache hits) |

### **Benchmark Examples**

#### **Benchmark 1: Dashboard Query Performance**
**Query**:
```sql
SELECT
    region,
    DATE_TRUNC('DAY', sale_date) AS day,
    SUM(amount) AS total_sales,
    COUNT(*) AS transaction_count
FROM
    sales
WHERE
    sale_date > CURRENT_DATE() - 30
GROUP BY
    region, DATE_TRUNC('DAY', sale_date)
ORDER BY
    day, region;
```

| **Configuration** | **First Execution** | **Subsequent Executions** | **After Data Change** | **After TTL Expiration** |
|-------------------|----------------------|----------------------------|-------------------------|--------------------------|
| **No Caching** | 8.5s, 4.25 credits | 8.5s, 4.25 credits | 8.5s, 4.25 credits | 8.5s, 4.25 credits |
| **Query Result Cache Only** | 8.5s, 4.25 credits | 8ms, 0 credits | 8.5s, 4.25 credits | 8.5s, 4.25 credits |
| **Warehouse Cache Only** | 8.5s, 4.25 credits | 1.2s, 0.5 credits | 1.2s, 0.5 credits | 1.2s, 0.5 credits |
| **Both Caches** | 8.5s, 4.25 credits | 8ms, 0 credits | 8.5s, 4.25 credits | 1.2s, 0.5 credits |

**Analysis**:
- **Query Result Cache** provides the **best performance** for subsequent identical queries (8ms vs 8.5s)
- **Warehouse Cache** provides **good performance** even after data changes (1.2s vs 8.5s)
- **Combined** approach provides **optimal performance** in all scenarios


#### **Benchmark 2: ETL Pipeline Performance**
**ETL Pipeline**:
```sql
-- Step 1: Extract
CREATE TEMPORARY TABLE temp_customers AS
SELECT * FROM source_customers WHERE last_updated > CURRENT_DATE() - 1;

-- Step 2: Transform
CREATE TEMPORARY TABLE temp_customer_orders AS
SELECT
    c.customer_id,
    c.name,
    o.order_id,
    o.order_date,
    o.amount
FROM
    temp_customers c
JOIN
    source_orders o ON c.customer_id = o.customer_id;

-- Step 3: Load
INSERT INTO target_sales
SELECT * FROM temp_customer_orders;
```

| **Configuration** | **Total Execution Time** | **Step 1 Time** | **Step 2 Time** | **Step 3 Time** |
|-------------------|----------------------------|----------------|----------------|----------------|
| **No Caching** | 17.3s | 5.2s | 8.7s | 3.4s |
| **Warehouse Cache Only** | 6.8s | 5.2s | 1.1s | 0.5s |
| **Query Result Cache Only** | 17.3s | 5.2s | 8.7s | 3.4s |
| **Both Caches** | 6.8s | 5.2s | 1.1s | 0.5s |

**Analysis**:
- **Warehouse Cache** provides **significant performance improvement** for ETL pipelines (6.8s vs 17.3s)
- **Query Result Cache** provides **no benefit** for ETL pipelines (queries are unique)
- **Warehouse Cache** is the **primary optimization** for ETL workloads

#### **Benchmark 3: Complex Query with CTEs**
**Query**:
```sql
WITH
sales_2023 AS (
    SELECT * FROM sales WHERE YEAR(sale_date) = 2023
),
customer_sales AS (
    SELECT
        customer_id,
        SUM(amount) AS total_spend
    FROM
        sales_2023
    GROUP BY
        customer_id
),
top_customers AS (
    SELECT * FROM customer_sales
    WHERE total_spend > 10000
    ORDER BY
        total_spend DESC
    LIMIT 100
)
SELECT
    c.customer_id,
    c.name,
    c.region,
    tc.total_spend
FROM
    customers c
JOIN
    top_customers tc ON c.customer_id = tc.customer_id;
```

| **Configuration** | **First Execution** | **Subsequent Executions** | **After Warehouse Restart** |
|-------------------|----------------------|----------------------------|-------------------------------|
| **No Caching** | 12.8s | 12.8s | 12.8s |
| **Query Result Cache Only** | 12.8s | 1.25s | 1.25s |
| **Warehouse Cache Only** | 12.8s | 2.5s | 12.8s |
| **Both Caches** | 12.8s | 1.25s | 2.5s |

**Analysis**:
- **Query Result Cache** provides the **best performance** for subsequent identical queries (1.25s vs 12.8s)
- **Warehouse Cache** provides **good performance** for subsequent executions (2.5s vs 12.8s)
- **Combined** approach provides **optimal performance** in all scenarios

## **6. Best Practices for Combined Usage**

### **A. When to Use Both Caches Together**

Use **both Query Result Cache and Warehouse Cache** for:
1. **Dashboard Queries**: Repetitive queries that benefit from both result caching and warehouse caching
2. **BI Tools**: Queries that run frequently with the same or similar parameters
3. **Reporting**: Static reports that are regenerated periodically
4. **ETL Pipelines with Repetitive Steps**: ETL pipelines where some steps are repeated
5. **Complex Queries with CTEs**: Queries with CTEs that are executed multiple times
6. **Hot Data Workloads**: Workloads that access the same data repeatedly

### **B. Configuration Recommendations**

| **Workload Type** | **Query Result Cache** | **Warehouse Cache** | **Warehouse Size** | **Auto-Suspend** | **RESULT_CACHE_TTL** |
|-------------------|------------------------|--------------------|-------------------|-----------------|----------------------|
| **Dashboards** | ✅ Enable | ✅ Automatic | Large - X-Large | 10-30 minutes | 1-4 hours |
| **BI Tools** | ✅ Enable | ✅ Automatic | Medium - Large | 10-30 minutes | 1-2 hours |
| **ETL Pipelines** | ❌ Disable | ✅ Automatic | X-Large - 4X-Large | 30-60 minutes | N/A |
| **Ad-Hoc Analysis** | ⚠️ Selective | ✅ Automatic | Medium | 5-10 minutes | 30-60 minutes |
| **Real-Time Analytics** | ❌ Disable | ✅ Automatic | Large - X-Large | 5 minutes | N/A |
| **Batch Processing** | ❌ Disable | ✅ Automatic | X-Large - 4X-Large | 60+ minutes | N/A |

### **C. Optimization Strategies**

| **Strategy** | **Description** | **Implementation** |
|--------------|-----------------|--------------------|
| **Enable Result Caching for Repetitive Queries** | Cache results for queries that run frequently | `ALTER SESSION SET USE_CACHED_RESULTS = TRUE` |
| **Use Larger Warehouses for Hot Data** | More cache capacity for frequently accessed data | `CREATE WAREHOUSE hot_data_wh WAREHOUSE_SIZE = 'X-LARGE'` |
| **Set Appropriate TTL** | Balance freshness and performance | `ALTER SESSION SET RESULT_CACHE_TTL = 3600` |
| **Use Temporary Tables for Intermediate Results** | Cache intermediate results in warehouse cache | `CREATE TEMPORARY TABLE temp AS SELECT ...` |
| **Reuse Temporary Tables** | Query the same temporary table multiple times | `SELECT * FROM temp_table WHERE ...` |
| **Use CTEs for Complex Queries** | Cache CTE results in warehouse cache | `WITH cte AS (SELECT ...) SELECT * FROM cte` |
| **Use Clustering for Cache Efficiency** | Improve cache hit rates with clustering | `ALTER TABLE my_table CLUSTER BY (date)` |
| **Use Auto-Suspend for Cost Savings** | Suspend warehouses when idle | `ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = 600` |
| **Avoid Frequent Warehouse Restarts** | Retain cache during idle periods | Use auto-suspend/resume instead of stop/start |
| **Monitor Cache Performance** | Track effectiveness of both caches | `SELECT used_cached_result, execution_time FROM QUERY_HISTORY` |
| **Combine with Other Optimizations** | Use with materialized views, result caching, etc. | `CLUSTER BY (date) + USE_CACHED_RESULTS = TRUE + larger warehouse` |
| **Document Cache Strategies** | Document which caches are used and why | Internal wiki or Confluence page |

### **D. Common Anti-Patterns and Solutions**

| **Anti-Pattern** | **Description** | **Impact** | **Solution** |
|------------------|-----------------|------------|--------------|
| **Not Using Any Caching** | Disabling all caching | ❌ Poor performance, ❌ High costs | Enable at least warehouse cache (automatic) |
| **Using Result Cache for Unique Queries** | Caching queries that are never repeated | ❌ Cache overhead, ❌ No benefit | Disable result cache for unique queries |
| **Using Small Warehouses for Hot Data** | Limited cache capacity | ❌ Poor performance, ❌ Cache inefficiency | Use larger warehouses for hot data |
| **Frequent Warehouse Restarts** | Cache lost on restart | ❌ Performance degradation | Use auto-suspend instead of stop/start |
| **Not Reusing Temporary Tables** | Temporary tables not queried multiple times | ❌ No warehouse cache benefit | Reuse temporary tables in subsequent queries |
| **Not Using CTEs for Complex Queries** | Missing out on warehouse cache benefits | ❌ Suboptimal performance | Use CTEs to break down complex queries |
| **Not Monitoring Cache Performance** | No visibility into cache effectiveness | ❌ Issues not detected, ❌ Suboptimal performance | Monitor cache hits, misses, and performance |
| **Using Result Cache for Real-Time Data** | Caching data that needs to be real-time | ❌ Stale results | Disable result cache or use shorter TTL |
| **Not Combining Cache Types** | Using only one cache type | ❌ Suboptimal performance | Combine cache types for maximum benefit |
| **Not Updating Statistics** | Poor query optimization | ❌ Cache inefficiency | Update statistics for large tables |

## **7. Monitoring and Troubleshooting**

### **A. Monitoring Query Result Cache**

#### **1. Check Cache Usage**
```sql
-- Basic cache usage check
SELECT
    query_id,
    query_text,
    used_cached_result,
    execution_time,
    bytes_scanned,
    credits_used,
    start_time,
    warehouse_name,
    user_name
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;

-- Cache hit rate by user
SELECT
    user_name,
    COUNT(*) AS total_queries,
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

-- Cache hit rate by warehouse
SELECT
    warehouse_name,
    COUNT(*) AS total_queries,
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

#### **2. Performance Comparison**
```sql
-- Compare performance with and without caching
SELECT
    query_id,
    query_text,
    used_cached_result,
    execution_time,
    bytes_scanned,
    credits_used
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%your_query_pattern%'
    AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY
    used_cached_result, execution_time;

-- Calculate average performance improvement
SELECT
    AVG(CASE WHEN used_cached_result = TRUE THEN execution_time ELSE NULL END) AS avg_cached_time,
    AVG(CASE WHEN used_cached_result = FALSE THEN execution_time ELSE NULL END) AS avg_uncached_time,
    AVG(CASE WHEN used_cached_result = FALSE THEN execution_time ELSE NULL END) /
        NULLIF(AVG(CASE WHEN used_cached_result = TRUE THEN execution_time ELSE NULL END), 0) AS speedup_factor
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%your_query_pattern%'
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP());
```

#### **3. Cache Size Estimation**
```sql
-- Estimate cache size for a specific query pattern
SELECT
    query_text,
    COUNT(*) AS execution_count,
    SUM(credits_used) AS total_credits_saved,
    SUM(bytes_scanned) / 1024 / 1024 AS estimated_cache_size_mb
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    used_cached_result = TRUE
    AND query_text LIKE '%your_query_pattern%'
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    query_text
ORDER BY
    total_credits_saved DESC;
```

### **B. Monitoring Warehouse Cache**

#### **1. Monitor Temporary Table Performance**
```sql
-- Check query performance for temporary tables
SELECT
    query_id,
    query_text,
    execution_time,
    bytes_scanned,
    start_time,
    warehouse_name
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%TEMPORARY TABLE%'
    OR query_text LIKE '%temp_%'
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    execution_time;

-- Compare performance with and without temporary tables
-- Without temporary table
SELECT * FROM sales WHERE sale_date > CURRENT_DATE() - 7;
SELECT LAST_QUERY_ID() AS query_id_without_temp;

-- With temporary table
CREATE TEMPORARY TABLE temp_sales AS SELECT * FROM sales WHERE sale_date > CURRENT_DATE() - 7;
SELECT * FROM temp_sales;
SELECT LAST_QUERY_ID() AS query_id_with_temp;

-- Compare
SELECT
    query_id,
    query_text,
    execution_time,
    bytes_scanned
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_id IN ('query_id_without_temp', 'query_id_with_temp')
ORDER BY
    query_id;
```

#### **2. Monitor CTE Performance**
```sql
-- Check query profile for CTE performance
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'));

-- Look for CTE-related steps in the profile
SELECT
    step_id,
    operation,
    execution_time,
    bytes_scanned,
    rows_produced
FROM
    TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))
WHERE
    operation LIKE '%CTE%'
ORDER BY
    execution_time;

-- Compare performance with and without CTEs
-- Without CTE
SELECT
    region,
    SUM(amount) AS total_sales
FROM
    (SELECT * FROM sales WHERE sale_date > CURRENT_DATE() - 7) s
GROUP BY
    region;
SELECT LAST_QUERY_ID() AS query_id_without_cte;

-- With CTE
WITH sales_cte AS (
    SELECT * FROM sales WHERE sale_date > CURRENT_DATE() - 7
)
SELECT
    region,
    SUM(amount) AS total_sales
FROM
    sales_cte
GROUP BY
    region;
SELECT LAST_QUERY_ID() AS query_id_with_cte;

-- Compare
SELECT
    query_id,
    query_text,
    execution_time,
    compilation_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_id IN ('query_id_without_cte', 'query_id_with_cte')
ORDER BY
    query_id;
```

#### **3. Monitor Warehouse Utilization**
```sql
-- Check warehouse load history
SELECT
    warehouse_name,
    start_time,
    end_time,
    running_queries,
    queued_queries,
    credit_usage
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE
    warehouse_name = 'MY_WH'
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;

-- Check for queries with low execution time (may indicate cache hits)
SELECT
    query_id,
    query_text,
    warehouse_name,
    execution_time,
    bytes_scanned,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    warehouse_name = 'MY_WH'
    AND execution_time < 100  -- <100ms (likely cache hit)
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    execution_time;
```

### **C. Monitoring Cache Eviction**

#### **1. Check for Warehouse Restarts**
```sql
-- Check for warehouse events that may invalidate cache
SELECT
    warehouse_name,
    event_time,
    event_type,
    old_size,
    new_size
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_EVENTS_HISTORY
WHERE
    warehouse_name = 'MY_WH'
    AND event_type IN ('SUSPEND', 'RESUME', 'ALTER')
    AND event_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    event_time DESC;
```

#### **2. Check for Large Queries That May Evict Cache**
```sql
-- Check for queries with high bytes_scanned that may have filled the cache
SELECT
    query_id,
    query_text,
    warehouse_name,
    bytes_scanned,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    warehouse_name = 'MY_WH'
    AND bytes_scanned > 100 * 1024 * 1024 * 1024  -- >100GB
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    bytes_scanned DESC;
```

### **D. Troubleshooting Common Issues**

#### **Issue 1: Query Result Cache Not Working**
**Symptoms**:
- `used_cached_result` is always `FALSE` in `QUERY_HISTORY`
- No performance improvement for repetitive queries

**Diagnosis**:
1. Check if result caching is enabled:
   ```sql
   SELECT USE_CACHED_RESULTS FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.CURRENT_SESSION());
   ```

2. Check for exact query text matches:
   ```sql
   SELECT query_text FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE query_text LIKE '%your_query_pattern%'
   ORDER BY start_time DESC;
   ```

3. Check for data changes between queries:
   ```sql
   SELECT start_time, query_text FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE query_text LIKE '%INSERT%UPDATE%DELETE%'
     AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
   ORDER BY start_time;
   ```

**Solutions**:
1. Enable result caching:
   ```sql
   ALTER SESSION SET USE_CACHED_RESULTS = TRUE;
   ```

2. Standardize query formatting:
   ```sql
   -- Use consistent formatting
   SELECT col1, col2 FROM my_table WHERE col3 = 1;
   ```

3. Use parameterized queries:
   ```sql
   -- Use bind variables
   SELECT * FROM my_table WHERE id = ?;
   ```

4. Set consistent session parameters:
   ```sql
   ALTER SESSION SET TIMEZONE = 'UTC';
   ```

5. Batch data changes:
   ```sql
   -- Batch updates instead of frequent small updates
   UPDATE my_table SET col1 = 1 WHERE id IN (1, 2, 3, ..., 1000);
   ```


#### **Issue 2: Warehouse Cache Not Improving Performance**
**Symptoms**:
- No performance improvement for queries on the same data
- High execution times despite repeated queries

**Diagnosis**:
1. Check warehouse size:
   ```sql
   SELECT warehouse_name, warehouse_size
   FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSES
   WHERE warehouse
