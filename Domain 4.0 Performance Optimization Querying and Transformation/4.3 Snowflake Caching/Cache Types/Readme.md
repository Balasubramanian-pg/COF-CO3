# **Snowflake Cache Types: Comprehensive Technical Reference**

---

## **1. Overview of Snowflake Cache Types**

Snowflake implements **six distinct caching mechanisms**, each optimized for different workload patterns and data access scenarios. These caches work **together** to minimize computation, reduce I/O operations, and deliver sub-second response times for repetitive operations.

---

### **Mermaid: Snowflake Cache Type Architecture**
```mermaid
%% Snowflake Cache Type Architecture
flowchart TD
    subgraph Client["Client Layer"]
        A[("Query")] --> B[("Query Parser")]
    end

    subgraph CacheTypes["Cache Types"]
        B --> C[("Result Cache\n(Query Results)")]
        B --> D[("Metadata Cache\n(Schema/Stats)")]
        B --> E[("Local Disk Cache\n(Frequent Data)")]
        B --> F[("Query Cache\n(Plans/Compiled)")]
        B --> G[("File Metadata Cache\n(External Tables)")]
        B --> H[("Warehouse Cache\n(Temp Data)")]
    end

    subgraph Execution["Execution Layer"]
        C --> I[("Return Cached\nResults")]
        D --> J[("Optimize with\nMetadata")]
        E --> K[("Return Cached\nData")]
        F --> L[("Reuse Cached\nPlan")]
        G --> M[("Optimize External\nTable Access")]
        H --> N[("Return Cached\nTemp Data")]
        I --> O[("Query Execution\nEngine")]
        J --> O
        K --> O
        L --> O
        M --> O
        N --> O
    end

    subgraph Storage["Storage Layer"]
        C --> P[("SSD Storage")]
        E --> Q[("SSD Storage")]
        H --> R[("SSD Storage")]
        G --> S[("Cloud Storage\nMetadata")]
    end

    subgraph Monitoring["Monitoring Layer"]
        T[("QUERY_HISTORY")]
        U[("QUERY_PROFILE")]
        V[("ACCOUNT_USAGE")]
    end
    O --> T
    O --> U
    C --> V
    D --> V
    E --> V

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef client fill:#4285f4,stroke:#1976d2;
    classDef result fill:#ff9800,stroke:#f57c00;
    classDef metadata fill:#009688,stroke:#00796b;
    classDef local fill:#e91e63,stroke:#c2185b;
    classDef query fill:#9c27b0,stroke:#7b1fa2;
    classDef file fill:#3f51b5,stroke:#303f9f;
    classDef warehouse fill:#795548,stroke:#5d4037;
    classDef execution fill:#29abe2,stroke:#1a8fb8;
    classDef storage fill:#f44336,stroke:#d32f2f;
    classDef monitoring fill:#607d8b,stroke:#455a64;
    class A client;
    class B cacheTypes;
    class C result;
    class D metadata;
    class E local;
    class F query;
    class G file;
    class H warehouse;
    class I,J,K,L,M,N execution;
    class P,Q,R,S storage;
    class T,U,V monitoring;
```

---

### **Cache Types Comparison Matrix**

| **Cache Type** | **Purpose** | **Scope** | **TTL** | **Storage Medium** | **What It Caches** | **Performance Impact** | **Cost** | **Configuration** | **Invalidation Triggers** |
|----------------|-------------|-----------|---------|---------------------|--------------------|-------------------------|----------|------------------|---------------------------|
| **Result Cache** | Cache query results | Per-user, per-query | 24 hours (configurable) | SSD | Complete query result sets | ⭐⭐⭐⭐⭐ (10-1000x faster) | Included | `USE_CACHED_RESULTS`, `RESULT_CACHE_TTL` | Data changes, query text changes, permissions changes, TTL expiry |
| **Metadata Cache** | Cache table metadata | Per-session | Session | Memory | Table schema, statistics, partition info, clustering info | ⭐⭐ (1.1-2x faster compilation) | Included | Automatic | DDL changes, significant data changes, session end |
| **Local Disk Cache** | Cache frequently accessed micro-partitions | Per-warehouse | Session | SSD | Hot micro-partitions from cloud storage | ⭐⭐⭐ (2-10x faster I/O) | Included | Automatic | Warehouse restart, session end, LRU eviction |
| **Query Cache** | Cache query plans and compiled queries | Per-session | Session | Memory | Query execution plans, compiled queries, parameterized query templates | ⭐⭐ (1.1-2x faster compilation) | Included | Automatic | DDL changes, schema changes, session parameters changes, session end |
| **File Metadata Cache** | Cache external table metadata | Per-session | Session | Memory | External stage definitions, file formats, partition metadata, file listings | ⭐⭐ (1.1-2x faster compilation) | Included | Automatic | DDL changes, external file changes, stage changes, session end |
| **Warehouse Cache** | Cache warehouse-specific data | Per-warehouse | Session | SSD | Temporary tables, CTE results, intermediate query results, spill data | ⭐⭐⭐ (2-10x faster I/O) | Included | Automatic | Warehouse restart, session end, LRU eviction |

---

### **Cache Type Selection Decision Tree**

```mermaid
%% Cache Type Selection Decision Tree
flowchart TD
    A[("Performance Issue")] --> B{Query Type?}
    B -->|Repetitive identical queries| C[("Result Cache")]
    B -->|Frequently accessed tables| D[("Local Disk Cache")]
    B -->|Complex queries| E[("Query Cache")]
    B -->|External tables| F[("File Metadata Cache")]
    B -->|Temporary data| G[("Warehouse Cache")]
    B -->|All queries| H[("Metadata Cache")]

    C --> I[("Enable USE_CACHED_RESULTS")]
    D --> J[("Use larger warehouses")]
    E --> K[("Use parameterized queries")]
    F --> L[("Use partitioned external tables")]
    G --> M[("Use TEMPORARY tables")]
    H --> N[("Automatic - no config needed")]

    I --> O[("Set RESULT_CACHE_TTL")]
    J --> P[("Query hot data repeatedly")]
    K --> Q[("Use stored procedures")]
    L --> R[("Refresh metadata when needed")]
    M --> S[("Reuse temp tables")]
    N --> T[("Update statistics for large tables")]

    O --> U[("Use for dashboards/reports")]
    P --> V[("Use for BI tools")]
    Q --> W[("Use for application queries")]
    R --> X[("Use for external table queries")]
    S --> Y[("Use for ETL/intermediate results")]
    T --> Z[("Use for all queries")]

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef start fill:#4285f4,stroke:#1976d2;
    classDef result fill:#ff9800,stroke:#f57c00;
    classDef local fill:#e91e63,stroke:#c2185b;
    classDef query fill:#9c27b0,stroke:#7b1fa2;
    classDef file fill:#3f51b5,stroke:#303f9f;
    classDef warehouse fill:#795548,stroke:#5d4037;
    classDef metadata fill:#009688,stroke:#00796b;
    class A start;
    class B start;
    class C result;
    class D local;
    class E query;
    class F file;
    class G warehouse;
    class H metadata;
    class I,O result;
    class J,P local;
    class K,Q query;
    class L,R file;
    class M,S warehouse;
    class N,T metadata;
```

---

---
## **2. Result Cache**

### **A. Definition and Purpose**

**Result Cache** is Snowflake's **primary query result caching mechanism** that stores the **complete result sets** of executed queries. When an identical query is submitted, Snowflake can **return the cached results immediately** without re-executing the query, providing **sub-second response times** for repetitive operations.

**Primary Use Cases:**
- Dashboard queries that run repeatedly
- BI tool refreshes
- Ad-hoc queries that are re-run
- Reporting queries with static parameters

---

### **B. Architecture and Workflow**

```mermaid
%% Result Cache Workflow
flowchart TD
    A[("Query Submission")] --> B[("Parse Query")]
    B --> C[("Generate Cache Key")]
    C --> D{Cache Hit?}
    D -->|Yes| E[("Return Cached Results\n<10ms")]
    D -->|No| F[("Execute Query")]
    F --> G[("Store Results in Cache\n+ Return Results")]
    E --> H[("Client")]
    G --> H
    C --> I[("Cache Key Components:")]
    I --> J[("Query Text\n(Exact Match)")]
    I --> K[("User Permissions")]
    I --> L[("Session Parameters")]
    I --> M[("Underlying Data\nVersion")]

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef process fill:#4285f4,stroke:#1976d2;
    classDef decision fill:#ff9800,stroke:#f57c00;
    classDef cache fill:#009688,stroke:#00796b;
    classDef client fill:#e91e63,stroke:#c2185b;
    class A,B,C,F,G process;
    class D decision;
    class E cache;
    class H client;
    class I,J,K,L,M default;
```

---

### **C. Technical Deep Dive**

#### **1. Cache Key Generation**
The cache key is a **hash** generated from:
- **Query text** (exact match, including whitespace, case, and comments)
- **User permissions** (roles, privileges)
- **Session parameters** (timezone, date formats, etc.)
- **Underlying data version** (changes to source tables invalidate cache)

**Important**: Even minor differences in query text (extra spaces, different case, comments) will result in **cache misses**.

#### **2. Cache Storage**
- **Storage Medium**: SSD (local to the warehouse)
- **Storage Location**: Managed by Snowflake, not user-visible
- **Storage Cost**: Included in Snowflake's storage pricing
- **Compression**: Results are stored in compressed format

#### **3. Cache Invalidation**
The result cache is **automatically invalidated** when:
1. **Data changes**: INSERT, UPDATE, DELETE, MERGE, or TRUNCATE on source tables
2. **Schema changes**: ALTER TABLE, ADD COLUMN, DROP COLUMN, etc.
3. **Query text changes**: Any modification to the query string
4. **Permission changes**: Changes to user roles or privileges
5. **Session parameter changes**: Changes to timezone, date formats, etc.
6. **TTL expiration**: After the configured TTL (default: 24 hours)
7. **Warehouse restart**: When the warehouse is stopped and restarted

#### **4. Cache Behavior with Different Query Types**
| **Query Type** | **Cacheable?** | **Notes** |
|----------------|----------------|-----------|
| SELECT | ✅ Yes | Primary use case for result cache |
| SELECT with CTEs | ✅ Yes | CTE results may also be cached |
| SELECT with subqueries | ✅ Yes | Subquery results may be cached separately |
| INSERT | ❌ No | DML operations are not cached |
| UPDATE | ❌ No | DML operations are not cached |
| DELETE | ❌ No | DML operations are not cached |
| CREATE TABLE | ❌ No | DDL operations are not cached |
| ALTER TABLE | ❌ No | DDL operations are not cached |
| DROP TABLE | ❌ No | DDL operations are not cached |
| CREATE VIEW | ❌ No | Views themselves are not cached (but their underlying queries may be) |
| CREATE MATERIALIZED VIEW | ❌ No | MVs are not cached (but their queries may use other caches) |
| SHOW commands | ❌ No | Metadata queries are not cached |
| DESCRIBE commands | ❌ No | Metadata queries are not cached |

---

### **D. Configuration**

#### **1. Enable/Disable Result Caching**
```sql
-- Enable result caching for current session (default)
ALTER SESSION SET USE_CACHED_RESULTS = TRUE;

-- Disable result caching for current session
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

---

### **E. Monitoring and Metrics**

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

---
### **F. Performance Characteristics**

#### **1. Performance Impact**
| **Metric** | **Without Cache** | **With Cache (Hit)** | **With Cache (Miss)** | **Improvement (Hit)** |
|------------|-------------------|---------------------|----------------------|------------------------|
| Execution Time | 100ms - 10s+ | 1ms - 10ms | 100ms - 10s+ | **10x - 1000x faster** |
| Bytes Scanned | Full query scan | 0 | Full query scan | **Infinite reduction** |
| Credits Used | Normal query cost | 0 | Normal query cost | **Infinite reduction** |
| Network Traffic | Full result set | Minimal | Full result set | **90-99% reduction** |
| Warehouse Load | Full query execution | None | Full query execution | **100% reduction** |

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

| **Scenario** | **Execution Time** | **Bytes Scanned** | **Credits Used** |
|--------------|--------------------|-------------------|------------------|
| First Execution | 8.5 seconds | 45 GB | 4.25 credits |
| Subsequent (Cache Hit) | 8 ms | 0 | 0 credits |
| After Data Change | 8.5 seconds | 45 GB | 4.25 credits |

**Improvement**: **1000x faster** with cache hit

---

**Example 2: Complex Join Query**
```sql
-- Query: Customer order history
SELECT
    c.customer_id,
    c.name,
    c.region,
    COUNT(o.order_id) AS order_count,
    SUM(o.amount) AS total_spend,
    MAX(o.order_date) AS last_order_date
FROM
    customers c
LEFT JOIN
    orders o ON c.customer_id = o.customer_id
WHERE
    o.order_date > CURRENT_DATE() - 90
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

---
### **G. Best Practices**

#### **1. When to Use Result Cache**
✅ **Repetitive queries** (dashboards, reports, BI tools)
✅ **Identical query text** (same SQL, same parameters)
✅ **Read-only workloads** (no DML operations)
✅ **Expensive queries** (high credit usage)
✅ **Static data** (data that doesn't change frequently)
✅ **Parameterized queries** (with consistent parameters)

#### **2. When NOT to Use Result Cache**
❌ **Unique queries** (each query is different)
❌ **Frequently changing data** (cache will be constantly invalidated)
❌ **Write-heavy workloads** (DML operations invalidate cache)
❌ **Real-time data** (cache TTL may be too long)
❌ **Ad-hoc analysis** (queries are typically unique)
❌ **DML/DDL operations** (not cacheable)

#### **3. Optimization Techniques**
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
| **Combine with Other Optimizations** | Use with clustering, materialized views | `CLUSTER BY (date) + USE_CACHED_RESULTS = TRUE` |

#### **4. Common Pitfalls and Solutions**
| **Pitfall** | **Symptom** | **Root Cause** | **Solution** |
|-------------|-------------|----------------|--------------|
| **Cache Misses Due to Query Text Differences** | Low cache hit rate | Whitespace, case, or comment differences | Standardize query formatting |
| **Cache Invalidation from Frequent Updates** | Poor performance despite caching | Data changes invalidating cache | Batch updates, use shorter TTL |
| **Cache Not Used for Parameterized Queries** | No cache hits for similar queries | Different parameter values | Use bind variables consistently |
| **High Memory Usage from Large Result Sets** | Warehouse memory pressure | Large cached results | Use filtering to reduce result size |
| **Cache Not Working for Views** | Views not cached | Views are not directly cached | Query the underlying tables directly |
| **Cache Not Working Across Sessions** | Cache misses in new sessions | Session-scoped cache | Use connection pooling |
| **Cache TTL Too Long for Volatile Data** | Stale results | TTL longer than data freshness | Reduce TTL or disable caching |
| **Cache TTL Too Short for Static Data** | Unnecessary re-executions | TTL shorter than data freshness | Increase TTL |

---
### **H. Limitations**

| **Limitation** | **Description** | **Workaround** |
|----------------|-----------------|---------------|
| **Exact Query Text Match Required** | Cache key includes exact query text | Standardize query formatting, use parameterized queries |
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

---
### **I. Practical Examples**

#### **Example 1: Dashboard Optimization**
```sql
-- Enable result caching for dashboard users
ALTER ROLE dashboard_role SET USE_CACHED_RESULTS = TRUE;

-- Set shorter TTL for frequently updated dashboards
ALTER ROLE dashboard_role SET RESULT_CACHE_TTL = 1800;  -- 30 minutes

-- Dashboard query (will be cached)
SELECT
    region,
    DATE_TRUNC('HOUR', sale_date) AS hour,
    SUM(amount) AS hourly_sales,
    COUNT(*) AS transaction_count
FROM
    sales
WHERE
    sale_date > CURRENT_DATE() - 1
GROUP BY
    region, DATE_TRUNC('HOUR', sale_date)
ORDER BY
    hour, region;

-- Check cache usage
SELECT
    query_id,
    query_text,
    used_cached_result,
    execution_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%hourly_sales%'
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
```

#### **Example 3: Parameterized Query Caching**
```sql
-- Application code using parameterized queries (Java example)
String sql = "SELECT * FROM customers WHERE region = ? AND signup_date > ?";
PreparedStatement stmt = connection.prepareStatement(sql);
stmt.setString(1, "US");
stmt.setDate(2, Date.valueOf("2023-01-01"));
ResultSet rs = stmt.executeQuery();

-- Same query with different parameters (will NOT use cache)
stmt.setString(1, "EU");
stmt.setDate(2, Date.valueOf("2023-01-01"));
rs = stmt.executeQuery();

-- To enable caching for parameterized queries, use consistent parameters
-- or implement application-level caching
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
    AND warehouse_name = 'dashboard_wh'
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
    AND warehouse_name = 'dashboard_wh'
    AND query_text LIKE '%daily_sales%'  -- Specific query pattern
  GROUP BY
    query_text
  HAVING
    COUNT(*) > 5  -- Query executed more than 5 times
    AND SUM(CASE WHEN used_cached_result = TRUE THEN 1 ELSE 0 END) = 0;  -- No cache hits
```

---
---
## **3. Metadata Cache**

### **A. Definition and Purpose**

**Metadata Cache** stores **table and column metadata** to **accelerate query compilation** and **enable optimization features** like partition pruning and predicate pushdown. Unlike result cache which stores query outputs, metadata cache stores **structural information** about your data.

**Primary Use Cases:**
- Accelerating query compilation for large tables
- Enabling partition pruning
- Supporting predicate pushdown
- Improving join optimization

---

### **B. Architecture and Workflow**

```mermaid
%% Metadata Cache Workflow
flowchart TD
    A[("Query Submission")] --> B[("Parse Query")]
    B --> C[("Extract Metadata Requirements")]
    C --> D[("Metadata Cache Lookup")]
    D --> E{Cache Hit?}
    E -->|Yes| F[("Use Cached Metadata")]
    E -->|No| G[("Fetch from Metadata Service")]
    G --> H[("Cache Metadata")]
    H --> F
    F --> I[("Optimize Query Plan")]
    I --> J[("Execute Query")]
    D --> K[("Cache Key Components:")]
    K --> L[("Table Name")]
    K --> M[("Column Names")]
    K --> N[("Table Statistics")]
    K --> O[("Partition Information")]
    K --> P[("Clustering Information")]

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef process fill:#4285f4,stroke:#1976d2;
    classDef decision fill:#ff9800,stroke:#f57c00;
    classDef cache fill:#009688,stroke:#00796b;
    classDef execution fill:#e91e63,stroke:#c2185b;
    class A,B,C,G,H,I,J process;
    class D decision;
    class E,F cache;
    class K,L,M,N,O,P default;
```

---

### **C. Technical Deep Dive**

#### **1. Metadata Types Cached**
Snowflake caches several types of metadata:

| **Metadata Type** | **Description** | **Usage** | **Update Frequency** |
|-------------------|-----------------|-----------|----------------------|
| **Table Schema** | Column names, data types, constraints | Query validation, optimization | On DDL changes |
| **Table Statistics** | Row count, byte count, partition count | Query optimization, cardinality estimation | On DDL changes or manual update |
| **Column Statistics** | Min/max values, histograms, distinct counts | Predicate pushdown, join optimization | On DDL changes or manual update |
| **Partition Information** | Micro-partition metadata (min/max values for each column) | Partition pruning | On data changes or manual update |
| **Clustering Information** | Clustering keys, clustering depth | Partition pruning | On reclustering or manual update |
| **Storage Information** | Storage usage, file formats | Query optimization | On data changes |
| **Index Information** | Search optimization indexes | Query optimization | On index changes |
| **Materialized View Information** | MV definitions, refresh status | Query rewriting | On MV changes |

#### **2. Cache Storage**
- **Storage Medium**: Memory (in-memory cache)
- **Scope**: Per-session
- **Storage Cost**: Included in Snowflake's compute pricing
- **Size**: Typically a few MB per session (scales with number of tables accessed)

#### **3. Cache Invalidation**
Metadata cache is **automatically invalidated** when:
1. **DDL Changes**: CREATE, ALTER, DROP TABLE/COLUMN
2. **Significant Data Changes**: Large bulk loads, significant updates
3. **Manual Statistics Updates**: `ALTER TABLE ... UPDATE STATISTICS`
4. **Session End**: When the session terminates
5. **Warehouse Restart**: When the warehouse is stopped and restarted

#### **4. Automatic Statistics Maintenance**
Snowflake **automatically maintains statistics** for:
- **Table row counts**
- **Column min/max values**
- **Partition statistics**
- **Clustering information**

However, for **large tables** or **complex workloads**, you may need to **manually update statistics** to ensure optimal performance.

---
### **D. Configuration**

#### **1. Update Statistics Manually**
```sql
-- Update statistics for a specific table
ALTER TABLE my_table UPDATE STATISTICS;

-- Update statistics for specific columns
ALTER TABLE my_table UPDATE STATISTICS (col1, col2, col3);

-- Update statistics for all tables in a schema
FOR table IN (
    SELECT table_name
    FROM INFORMATION_SCHEMA.TABLES
    WHERE table_schema = 'MY_SCHEMA'
) DO
    EXECUTE IMMEDIATE 'ALTER TABLE ' || table || ' UPDATE STATISTICS';
END FOR;

-- Update statistics for all tables in a database
FOR table IN (
    SELECT table_name
    FROM INFORMATION_SCHEMA.TABLES
    WHERE table_catalog = 'MY_DB'
) DO
    EXECUTE IMMEDIATE 'ALTER TABLE ' || table || ' UPDATE STATISTICS';
END FOR;
```

#### **2. Check Current Statistics**
```sql
-- Check table statistics
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.TABLE_STATISTICS('MY_TABLE'));

-- Check column statistics for a specific column
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.COLUMN_STATISTICS('MY_TABLE', 'COL1'));

-- Check all column statistics for a table
SELECT
    column_name,
    min_value,
    max_value,
    distinct_count,
    null_count,
    avg_value_length
FROM
    TABLE(SNOWFLAKE.INFORMATION_SCHEMA.COLUMN_STATISTICS('MY_TABLE'))
ORDER BY
    column_name;

-- Check when statistics were last updated
SELECT
    table_name,
    last_updated
FROM
    SNOWFLAKE.ACCOUNT_USAGE.TABLE_STATISTICS
WHERE
    table_name = 'MY_TABLE';
```

#### **3. Configure Automatic Statistics**
```sql
-- Snowflake automatically maintains basic statistics
-- For more control, you can enable automatic statistics updates
ALTER TABLE my_table SET AUTO_UPDATE_STATISTICS = TRUE;

-- Check if automatic statistics are enabled
SELECT
    table_name,
    auto_update_statistics
FROM
    INFORMATION_SCHEMA.TABLES
WHERE
    table_name = 'MY_TABLE';
```

---
### **E. Monitoring and Metrics**

#### **1. Monitor Compilation Time**
```sql
-- Check compilation time for recent queries
SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time,
    start_time,
    warehouse_name
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    compilation_time > 0
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    compilation_time DESC;

-- Check average compilation time by table
SELECT
    REGEXP_SUBSTR(query_text, 'FROM\\s+([^\\s,;]+)', 1, 1, '', 1) AS table_name,
    AVG(compilation_time) AS avg_compilation_time,
    COUNT(*) AS query_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    compilation_time > 0
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    table_name
ORDER BY
    avg_compilation_time DESC;
```

#### **2. Monitor Partition Pruning Effectiveness**
```sql
-- Check if partition pruning is working (indirect indicator of metadata cache)
SELECT
    query_id,
    query_text,
    partitions_scanned,
    bytes_scanned,
    execution_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%MY_TABLE%'
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    partitions_scanned;

-- Compare partitions scanned before and after statistics update
-- Before statistics update
SELECT partitions_scanned
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_id = 'before_stats_query_id';

-- After statistics update
ALTER TABLE my_table UPDATE STATISTICS;
SELECT partitions_scanned
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_id = 'after_stats_query_id';
```

#### **3. Monitor Statistics Freshness**
```sql
-- Check when statistics were last updated for all tables
SELECT
    table_name,
    schema_name,
    database_name,
    last_updated,
    DATEDIFF('day', last_updated, CURRENT_TIMESTAMP()) AS days_since_update
FROM
    SNOWFLAKE.ACCOUNT_USAGE.TABLE_STATISTICS
WHERE
    last_updated IS NOT NULL
ORDER BY
    days_since_update DESC;

-- Check for tables with stale statistics
SELECT
    table_name,
    schema_name,
    database_name,
    last_updated,
    DATEDIFF('day', last_updated, CURRENT_TIMESTAMP()) AS days_since_update
FROM
    SNOWFLAKE.ACCOUNT_USAGE.TABLE_STATISTICS
WHERE
    DATEDIFF('day', last_updated, CURRENT_TIMESTAMP()) > 7  -- Older than 7 days
ORDER BY
    days_since_update DESC;
```

---
### **F. Performance Characteristics**

#### **1. Performance Impact**
| **Metric** | **Without Metadata Cache** | **With Metadata Cache** | **Improvement** | **Notes** |
|------------|-------------------------------|----------------------------|-----------------|-----------|
| Compilation Time | 100-2000ms | 10-200ms | **5-10x faster** | Depends on table size and complexity |
| Query Optimization | Limited | Improved | Better execution plans | Enables partition pruning, predicate pushdown |
| First Query Performance | Slow | Fast | Faster subsequent queries | Metadata is cached after first query |
| Partition Pruning | Disabled | Enabled | **10-100x less data scanned** | Requires clustering and metadata |
| Join Optimization | Limited | Improved | Better join algorithms | Requires column statistics |
| Cardinality Estimation | Inaccurate | Accurate | Better query plans | Requires table and column statistics |

#### **2. Real-World Performance Examples**
**Example 1: Large Table Query with Partition Pruning**
```sql
-- Table with 1TB of data, clustered by date
ALTER TABLE large_sales CLUSTER BY (sale_date);

-- Query with date filter (benefits from partition pruning)
SELECT * FROM large_sales WHERE sale_date = '2023-01-15';
```

| **Scenario** | **Compilation Time** | **Execution Time** | **Partitions Scanned** | **Bytes Scanned** |
|--------------|-----------------------|--------------------|------------------------|-------------------|
| Without Metadata Cache | 1500ms | 8.5s | 2000 | 100 GB |
| With Metadata Cache | 150ms | 0.5s | 10 | 5 GB |

**Improvement**:
- **Compilation Time**: 10x faster
- **Execution Time**: 17x faster
- **Partitions Scanned**: 200x reduction
- **Bytes Scanned**: 20x reduction

---

**Example 2: Complex Join Query**
```sql
-- Tables with statistics
ALTER TABLE customers UPDATE STATISTICS;
ALTER TABLE orders UPDATE STATISTICS;

-- Complex join query
SELECT
    c.customer_id,
    c.name,
    COUNT(o.order_id) AS order_count,
    SUM(o.amount) AS total_spend
FROM
    customers c
LEFT JOIN
    orders o ON c.customer_id = o.customer_id
WHERE
    c.region = 'US'
    AND o.order_date > CURRENT_DATE() - 90
GROUP BY
    c.customer_id, c.name
ORDER BY
    total_spend DESC;
```

| **Scenario** | **Compilation Time** | **Execution Time** | **Join Efficiency** |
|--------------|-----------------------|--------------------|---------------------|
| Without Metadata Cache | 800ms | 12.3s | Basic hash join |
| With Metadata Cache | 80ms | 2.1s | Optimized hash join with better cardinality estimates |

**Improvement**:
- **Compilation Time**: 10x faster
- **Execution Time**: 6x faster
- **Join Efficiency**: Better join algorithm selection

---
### **G. Best Practices**

#### **1. When Metadata Cache is Most Effective**
✅ **Large tables** (>1TB)
✅ **Complex queries** (joins, aggregations, subqueries)
✅ **Repetitive queries** (dashboards, reports)
✅ **Clustered tables** (partition pruning)
✅ **Tables with filters** (predicate pushdown)

#### **2. When to Update Statistics Manually**
✅ **After bulk data loads**
✅ **After significant data changes** (>10% of table size)
✅ **For large tables** (>1TB)
✅ **Before performance-critical queries**
✅ **When query performance degrades unexpectedly**

#### **3. Optimization Techniques**
| **Technique** | **Description** | **Example** |
|---------------|-----------------|-------------|
| **Update Statistics for Large Tables** | Ensure statistics are current for large tables | `ALTER TABLE large_table UPDATE STATISTICS` |
| **Update Statistics After Bulk Loads** | Refresh statistics after loading large datasets | `COPY INTO my_table FROM @my_stage; ALTER TABLE my_table UPDATE STATISTICS;` |
| **Use Clustering for Partition Pruning** | Cluster tables to enable partition pruning | `ALTER TABLE my_table CLUSTER BY (date)` |
| **Avoid Frequent DDL Changes** | Minimize schema changes that invalidate cache | Batch DDL changes |
| **Monitor Compilation Time** | Track metadata cache effectiveness | `SELECT compilation_time FROM QUERY_HISTORY` |
| **Use Automatic Statistics** | Let Snowflake maintain basic statistics | `ALTER TABLE my_table SET AUTO_UPDATE_STATISTICS = TRUE` |
| **Update Column Statistics for Filtered Columns** | Improve predicate pushdown | `ALTER TABLE my_table UPDATE STATISTICS (date, region)` |
| **Combine with Other Optimizations** | Use with clustering, result cache, etc. | `CLUSTER BY (date) + UPDATE STATISTICS` |

#### **4. Common Pitfalls and Solutions**
| **Pitfall** | **Symptom** | **Root Cause** | **Solution** |
|-------------|-------------|----------------|--------------|
| **High Compilation Time** | Slow query compilation | Missing or stale statistics | Update statistics manually |
| **No Partition Pruning** | Full table scans despite filters | Missing clustering or statistics | Cluster table and update statistics |
| **Poor Join Performance** | Slow joins | Missing column statistics | Update column statistics |
| **Inaccurate Cardinality Estimates** | Suboptimal query plans | Stale statistics | Update statistics |
| **Frequent Statistics Updates** | High overhead | Too many manual updates | Use automatic statistics, update selectively |
| **Statistics Not Updated After Bulk Load** | Poor performance after load | Statistics not refreshed | Update statistics after bulk load |
| **Statistics Not Updated for Large Tables** | Poor performance on large tables | Automatic statistics not sufficient | Update statistics manually for large tables |

---
### **H. Limitations**

| **Limitation** | **Description** | **Workaround** |
|----------------|-----------------|---------------|
| **Session-Scoped Cache** | Metadata cache is session-scoped | Use connection pooling to reuse sessions |
| **Invalidated by DDL** | DDL changes invalidate metadata cache | Batch DDL changes |
| **Invalidated by Significant Data Changes** | Large data changes may invalidate cache | Update statistics manually after bulk operations |
| **No Direct Monitoring** | Metadata cache usage is not directly visible | Monitor compilation time and query performance |
| **No Partial Metadata Caching** | Cannot cache partial metadata | Use filtered tables or views |
| **No Custom Metadata** | Cannot customize metadata | Use Snowflake's built-in metadata |
| **No Metadata for External Tables** | Limited metadata for external tables | Use internal tables or materialized views |
| **Storage Overhead** | Metadata consumes memory | Monitor memory usage |
| **Automatic Statistics Limitations** | Automatic statistics may not be sufficient for complex queries | Update statistics manually for performance-critical tables |
| **Statistics Update Overhead** | Updating statistics consumes credits | Update statistics selectively and during off-peak hours |

---
### **I. Practical Examples**

#### **Example 1: Large Table Optimization**
```sql
-- Create a large table
CREATE TABLE large_sales (
    sale_id BIGINT,
    customer_id BIGINT,
    product_id BIGINT,
    sale_date DATE,
    amount FLOAT,
    region STRING,
    PRIMARY KEY (sale_id)
);

-- Load data into the table
COPY INTO large_sales FROM @my_stage;

-- Update statistics for the table
ALTER TABLE large_sales UPDATE STATISTICS;

-- Cluster the table for partition pruning
ALTER TABLE large_sales CLUSTER BY (sale_date, region);

-- Query the table with filters (benefits from metadata cache and clustering)
SELECT
    region,
    DATE_TRUNC('MONTH', sale_date) AS month,
    SUM(amount) AS monthly_sales
FROM
    large_sales
WHERE
    sale_date BETWEEN '2023-01-01' AND '2023-12-31'
    AND region IN ('US', 'EU', 'APAC')
GROUP BY
    region, DATE_TRUNC('MONTH', sale_date)
ORDER BY
    month, region;

-- Check if partition pruning is working
SELECT
    query_id,
    query_text,
    partitions_scanned,
    bytes_scanned
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%large_sales%'
ORDER BY
    partitions_scanned;
```

#### **Example 2: Complex Query Optimization**
```sql
-- Create tables with statistics
CREATE TABLE customers (
    customer_id BIGINT,
    name STRING,
    region STRING,
    signup_date DATE,
    PRIMARY KEY (customer_id)
);

CREATE TABLE orders (
    order_id BIGINT,
    customer_id BIGINT,
    order_date DATE,
    amount FLOAT,
    PRIMARY KEY (order_id)
);

-- Load data
COPY INTO customers FROM @customers_stage;
COPY INTO orders FROM @orders_stage;

-- Update statistics
ALTER TABLE customers UPDATE STATISTICS;
ALTER TABLE orders UPDATE STATISTICS (customer_id, order_date, amount);

-- Complex query with joins and aggregations
SELECT
    c.region,
    DATE_TRUNC('MONTH', o.order_date) AS month,
    COUNT(DISTINCT c.customer_id) AS active_customers,
    COUNT(o.order_id) AS order_count,
    SUM(o.amount) AS total_sales,
    AVG(o.amount) AS avg_order_value
FROM
    customers c
LEFT JOIN
    orders o ON c.customer_id = o.customer_id
WHERE
    o.order_date BETWEEN '2023-01-01' AND '2023-12-31'
    AND c.region IN ('US', 'EU')
GROUP BY
    c.region, DATE_TRUNC('MONTH', o.order_date)
ORDER BY
    month, region;

-- Check query performance
SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time,
    partitions_scanned,
    bytes_scanned
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%customers%orders%'
ORDER BY
    execution_time;
```

#### **Example 3: Statistics Maintenance Automation**
```sql
-- Create a stored procedure to update statistics for all tables in a schema
CREATE OR REPLACE PROCEDURE update_schema_statistics(schema_name STRING)
RETURNS STRING
LANGUAGE JAVASCRIPT
AS
$$
  const updateStatsSQL = `
    FOR table IN (
        SELECT table_name
        FROM INFORMATION_SCHEMA.TABLES
        WHERE table_schema = '${SCHEMA_NAME}'
    ) DO
        EXECUTE IMMEDIATE 'ALTER TABLE ${SCHEMA_NAME}.' || table || ' UPDATE STATISTICS';
    END FOR;
  `;

  const result = snowflake.execute({sqlText: updateStatsSQL.replace('${SCHEMA_NAME}', SCHEMA_NAME)});
  return 'Updated statistics for all tables in schema: ' + SCHEMA_NAME;
$$;

-- Call the procedure to update statistics for a schema
CALL update_schema_statistics('MY_SCHEMA');

-- Create a task to update statistics daily
CREATE TASK daily_statistics_update
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON 0 2 * * * America/Los_Angeles'  -- 2 AM daily
AS
  CALL update_schema_statistics('MY_SCHEMA');
```

#### **Example 4: Monitor and Alert on Stale Statistics**
```sql
-- Create an alert for tables with stale statistics
CREATE OR REPLACE ALERT stale_statistics_alert
  WAREHOUSE = monitoring_wh
  SCHEDULE = 'USING CRON 0 8 * * * America/Los_Angeles'  -- 8 AM daily
AS
  SELECT
    table_name,
    schema_name,
    database_name,
    last_updated,
    DATEDIFF('day', last_updated, CURRENT_TIMESTAMP()) AS days_since_update,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.TABLE_STATISTICS
  WHERE
    DATEDIFF('day', last_updated, CURRENT_TIMESTAMP()) > 7  -- Older than 7 days
    AND table_name NOT LIKE '%TEMP%'  -- Exclude temporary tables
  ORDER BY
    days_since_update DESC;

-- Create an alert for high compilation times
CREATE OR REPLACE ALERT high_compilation_time_alert
  WAREHOUSE = monitoring_wh
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
    compilation_time > 1000  -- >1 second
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
  ORDER BY
    compilation_time DESC;
```

#### **Example 5: Performance Comparison Before/After Statistics Update**
```sql
-- Run a query and capture performance before statistics update
SELECT
    region,
    COUNT(*) AS customer_count,
    SUM(total_spend) AS total_spend
FROM
    customers
WHERE
    signup_date > '2023-01-01'
GROUP BY
    region;
SELECT LAST_QUERY_ID() AS query_id_before;

-- Update statistics
ALTER TABLE customers UPDATE STATISTICS;

-- Run the same query and capture performance after statistics update
SELECT
    region,
    COUNT(*) AS customer_count,
    SUM(total_spend) AS total_spend
FROM
    customers
WHERE
    signup_date > '2023-01-01'
GROUP BY
    region;
SELECT LAST_QUERY_ID() AS query_id_after;

-- Compare performance
SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time,
    partitions_scanned,
    bytes_scanned
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_id IN ('query_id_before', 'query_id_after')
ORDER BY
    query_id;
```

---
---
## **4. Local Disk Cache**

### **A. Definition and Purpose**

**Local Disk Cache** is a **warehouse-level cache** that stores **frequently accessed micro-partitions** on **local SSD storage** to **reduce I/O latency** and **improve query performance**. This cache is **automatically managed** by Snowflake and is **transparent to users**.

**Primary Use Cases:**
- Repeatedly accessed hot data
- Large tables with frequent queries
- BI tools and dashboards
- Analytical workloads with data locality

---

### **B. Architecture and Workflow**

```mermaid
%% Local Disk Cache Workflow
flowchart TD
    A[("Query Submission")] --> B[("Query Parser")]
    B --> C[("Identify Required Micro-Partitions")]
    C --> D[("Local Disk Cache Lookup")]
    D --> E{Cache Hit?}
    E -->|Yes| F[("Return Cached Micro-Partitions\n<10ms")]
    E -->|No| G[("Fetch from Cloud Storage\n100-500ms")]
    G --> H[("Cache Micro-Partitions\nin Local SSD")]
    H --> F
    F --> I[("Query Execution Engine")]
    G --> I

    subgraph CacheDetails["Cache Details"]
        D --> J[("Cache Key: Micro-Partition ID")]
        H --> K[("Storage: SSD")]
        H --> L[("Eviction: LRU")]
        H --> M[("Scope: Per-Warehouse")]
    end

    subgraph CloudStorage["Cloud Storage"]
        G --> N[("S3 / Azure Blob / GCS")]
    end

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef process fill:#4285f4,stroke:#1976d2;
    classDef decision fill:#ff9800,stroke:#f57c00;
    classDef cache fill:#e91e63,stroke:#c2185b;
    classDef execution fill:#009688,stroke:#00796b;
    classDef storage fill:#29abe2,stroke:#1a8fb8;
    class A,B,C,G,H,I process;
    class D decision;
    class E,F cache;
    class J,K,L,M default;
    class N storage;
```

---

### **C. Technical Deep Dive**

#### **1. How Local Disk Cache Works**
1. **Data Locality**:
   - When a query accesses data, Snowflake **fetches micro-partitions** from **cloud storage** (S3, Azure Blob, GCS).
   - Frequently accessed micro-partitions are **cached in local SSD** for the warehouse.

2. **Cache Lookup**:
   - For subsequent queries, Snowflake **checks the local disk cache** first.
   - If the micro-partition is **cached**, it is **returned from SSD** (typical latency: **1-10ms**).
   - If the micro-partition is **not cached**, it is **fetched from cloud storage** (typical latency: **100-500ms**).

3. **Cache Management**:
   - **Cache Size**: Limited by the **warehouse size** (larger warehouses have more cache capacity).
     - X-Small: ~16GB cache
     - Small: ~32GB cache
     - Medium: ~64GB cache
     - Large: ~128GB cache
     - X-Large: ~256GB cache
     - 2X-Large: ~512GB cache
     - 3X-Large: ~1TB cache
     - 4X-Large: ~2TB cache
   - **Eviction Policy**: **Least Recently Used (LRU)** - When the cache is full, the least recently accessed micro-partitions are evicted.
   - **Scope**: **Per-warehouse** - Each warehouse has its own local disk cache.

4. **Cache Population**:
   - The cache is **populated on-demand** as queries access data.
   - **Hot data** (frequently accessed micro-partitions) stays in cache.
   - **Cold data** (infrequently accessed micro-partitions) may be evicted.

#### **2. Cache Storage Details**
- **Storage Medium**: SSD (NVMe in most cases)
- **Data Format**: Compressed columnar format (same as cloud storage)
- **Compression**: Micro-partitions are stored in compressed format
- **Encryption**: Data is encrypted at rest

#### **3. Cache Invalidation**
Local disk cache is **automatically invalidated** when:
1. **Warehouse Restart**: When the warehouse is **stopped and restarted** (auto-suspend does not invalidate cache)
2. **Session End**: When the **session terminates** (session-scoped data may be evicted)
3. **LRU Eviction**: When the **cache is full**, least recently used micro-partitions are evicted
4. **Data Changes**: **Not automatically invalidated** for DML operations (Snowflake uses MVCC, so old versions may still be in cache)

**Note**: Unlike result cache, local disk cache is **not invalidated by DML operations** because Snowflake uses **Multi-Version Concurrency Control (MVCC)**. However, queries will see the **latest version** of the data.

#### **4. Cache Behavior with Different Data Types**
| **Data Type** | **Cacheable?** | **Notes** |
|---------------|----------------|-----------|
| Internal Tables | ✅ Yes | Primary use case for local disk cache |
| External Tables | ✅ Yes | Micro-partitions from external stages are cached |
| Temporary Tables | ✅ Yes | Cached in warehouse cache |
| Views | ❌ No | Views themselves are not cached (but underlying data may be) |
| Materialized Views | ✅ Yes | Cached like regular tables |
| Stage Files | ❌ No | Files in stages are not cached (but external table data is) |

---
### **D. Configuration**

**Note**: Local disk cache is **automatic** and **requires no configuration**. However, you can **influence** the cache by:

#### **1. Warehouse Sizing for Cache Capacity**
```sql
-- Create a warehouse with appropriate size for your cache needs
CREATE WAREHOUSE small_wh WAREHOUSE_SIZE = 'SMALL';      -- ~32GB cache
CREATE WAREHOUSE medium_wh WAREHOUSE_SIZE = 'MEDIUM';    -- ~64GB cache
CREATE WAREHOUSE large_wh WAREHOUSE_SIZE = 'LARGE';      -- ~128GB cache
CREATE WAREHOUSE xlarge_wh WAREHOUSE_SIZE = 'X-LARGE';    -- ~256GB cache
CREATE WAREHOUSE xxlarge_wh WAREHOUSE_SIZE = 'XX-LARGE';  -- ~512GB cache

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

**Note**: Auto-suspend **does not invalidate** the local disk cache. The cache is **retained** while the warehouse is suspended and **available immediately** when the warehouse resumes.

#### **3. Multi-Cluster Warehouse Configuration**
```sql
-- Create a multi-cluster warehouse for high concurrency
CREATE WAREHOUSE mc_wh
  WAREHOUSE_SIZE = 'LARGE'
  MAX_CLUSTER_COUNT = 4
  MIN_CLUSTER_COUNT = 1
  SCALING_POLICY = 'STANDARD';

-- Each cluster has its own local disk cache
-- Data accessed by one cluster is not automatically cached in other clusters
```

---
### **E. Monitoring and Metrics**

#### **1. Monitor Cache Usage via Query Profile**
```sql
-- Get query profile for a specific query
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'));

-- Look for cached data in the profile
SELECT
    step_id,
    operation,
    rows_produced,
    bytes_scanned,
    execution_time,
    spill_to_disk,
    spill_to_remote
FROM
    TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))
WHERE
    execution_time < 100  -- Fast steps may indicate cache hits
ORDER BY
    execution_time;

-- Check for steps with low bytes_scanned (may indicate cache hits)
SELECT
    step_id,
    operation,
    bytes_scanned,
    rows_produced
FROM
    TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))
WHERE
    bytes_scanned = 0  -- No bytes scanned from cloud storage
ORDER BY
    step_id;
```

#### **2. Compare Performance with and without Cache**
```sql
-- First query (cache miss)
SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7;
SELECT LAST_QUERY_ID() AS query_id_1;

-- Subsequent queries (cache hit)
SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7;
SELECT LAST_QUERY_ID() AS query_id_2;

SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7;
SELECT LAST_QUERY_ID() AS query_id_3;

-- Compare execution times
SELECT
    query_id,
    query_text,
    execution_time,
    bytes_scanned,
    partitions_scanned
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_id IN ('query_id_1', 'query_id_2', 'query_id_3')
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

---
### **F. Performance Characteristics**

#### **1. Performance Impact**
| **Metric** | **Without Local Disk Cache** | **With Local Disk Cache (Hit)** | **With Local Disk Cache (Miss)** | **Improvement (Hit)** |
|------------|--------------------------------|----------------------------------|------------------------------------|------------------------|
| I/O Latency | 100-500ms (cloud storage) | 1-10ms (SSD) | 100-500ms (cloud storage) | **10-100x faster** |
| Throughput | Limited by cloud storage | Limited by SSD | Limited by cloud storage | **2-10x higher** |
| Execution Time | Higher (waiting for I/O) | Lower (fast I/O) | Higher (waiting for I/O) | **2-10x faster** |
| Bytes Scanned | Full scan from cloud storage | Cached data from SSD | Full scan from cloud storage | **Same** (but faster) |
| Credit Usage | Higher (more time spent waiting for I/O) | Lower (less time waiting for I/O) | Higher (more time spent waiting for I/O) | **1.1-2x reduction** |
| Concurrency | Limited by I/O | Improved | Limited by I/O | **Better resource utilization** |

#### **2. Real-World Performance Examples**
**Example 1: Hot Table Access**
```sql
-- Table with 10TB of data
CREATE TABLE large_table (
    id BIGINT,
    data VARCHAR(1000),
    category STRING,
    PRIMARY KEY (id)
);

-- Query a subset of the table (hot data)
SELECT * FROM large_table WHERE category = 'A' LIMIT 1000;
```

| **Scenario** | **Execution Time** | **I/O Latency** | **Bytes Scanned** | **Credit Usage** |
|--------------|--------------------|-----------------|-------------------|------------------|
| First Execution | 12.5 seconds | 300ms | 50 GB | 6.25 credits |
| Subsequent (Cache Hit) | 1.2 seconds | 5ms | 50 GB | 0.625 credits |
| After Warehouse Restart | 12.5 seconds | 300ms | 50 GB | 6.25 credits |

**Improvement**:
- **Execution Time**: 10x faster with cache
- **I/O Latency**: 60x faster with cache
- **Credit Usage**: 10x reduction with cache

---

**Example 2: Dashboard with Multiple Queries**
```sql
-- Dashboard with 10 queries, each scanning 10GB
-- First run (all cache misses)
SELECT * FROM sales WHERE date = CURRENT_DATE() - 1;  -- 5s
SELECT * FROM customers WHERE region = 'US';         -- 3s
SELECT * FROM products WHERE category = 'Electronics'; -- 2s
-- Total: 10s

-- Subsequent runs (all cache hits)
SELECT * FROM sales WHERE date = CURRENT_DATE() - 1;  -- 500ms
SELECT * FROM customers WHERE region = 'US';         -- 300ms
SELECT * FROM products WHERE category = 'Electronics'; -- 200ms
-- Total: 1s
```

| **Scenario** | **Total Execution Time** | **Total Credit Usage** |
|--------------|----------------------------|-------------------------|
| First Run | 10 seconds | 10 credits |
| Subsequent Runs | 1 second | 1 credit |

**Improvement**: **10x faster** with local disk cache

---
### **G. Best Practices**

#### **1. When to Use Local Disk Cache**
✅ **Frequently accessed data** (hot data)
✅ **Large tables** with repeated queries
✅ **BI tools and dashboards** with repetitive queries
✅ **Analytical workloads** with data locality
✅ **Multi-cluster warehouses** for high concurrency
✅ **Queries with filters** on clustered columns

#### **2. When Local Disk Cache is Less Effective**
⚠️ **Ad-hoc queries** (data not likely to be re-accessed)
⚠️ **Write-heavy workloads** (cache may be frequently evicted)
⚠️ **Small warehouses** (limited cache capacity)
⚠️ **Cold data** (infrequently accessed data)
⚠️ **Very large scans** (may evict other cached data)

#### **3. Optimization Techniques**
| **Technique** | **Description** | **Example** |
|---------------|-----------------|-------------|
| **Use Larger Warehouses for Hot Data** | Larger warehouses have more cache capacity | `CREATE WAREHOUSE hot_data_wh WAREHOUSE_SIZE = 'X-LARGE'` |
| **Query the Same Data Repeatedly** | Populate the cache with frequently accessed data | Dashboards, reports, BI tools |
| **Use Clustering for Cache Efficiency** | Cluster tables to improve cache hit rates | `ALTER TABLE my_table CLUSTER BY (date)` |
| **Use Auto-Suspend for Cost Savings** | Suspend warehouses when idle to save costs | `ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = 600` |
| **Avoid Frequent Warehouse Restarts** | Cache is lost when warehouse is restarted | Use auto-suspend/resume instead of stop/start |
| **Use Multi-Cluster Warehouses for High Concurrency** | Distribute cache across multiple clusters | `CREATE WAREHOUSE mc_wh MAX_CLUSTER_COUNT = 4` |
| **Combine with Result Caching** | Use both local disk cache and result caching | `ALTER SESSION SET USE_CACHED_RESULTS = TRUE` |
| **Monitor Cache Performance** | Check QUERY_PROFILE for cache hits | `SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))` |
| **Use Temporary Tables for Intermediate Results** | Temporary tables are cached in warehouse cache | `CREATE TEMPORARY TABLE temp AS SELECT ...` |
| **Document Cacheable Data** | Document which tables benefit from caching | Internal wiki or Confluence page |

#### **4. Common Pitfalls and Solutions**
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

---
### **H. Limitations**

| **Limitation** | **Description** | **Workaround** |
|----------------|-----------------|---------------|
| **Per-Warehouse Cache** | Each warehouse has its own cache | Use larger warehouses for hot data, multi-cluster for high concurrency |
| **Limited by Warehouse Size** | Cache size is limited by warehouse size | Use larger warehouses for more cache |
| **LRU Eviction** | Least recently used data is evicted when cache is full | Query important data frequently to keep it in cache |
| **No Direct Control** | Cannot manually manage cache contents | Use query patterns and warehouse sizing to influence cache |
| **Cache Lost on Warehouse Restart** | Cache is lost when warehouse is stopped and restarted | Use auto-suspend/resume instead of stop/start |
| **No Monitoring for Cache Hits** | Cache hits are not directly visible | Monitor query performance and I/O latency |
| **No Caching for All Data Types** | Some data types may not be cached effectively | Use internal tables for hot data |
| **Storage Overhead** | Cache consumes SSD storage | Monitor warehouse storage usage |
| **Not Invalidated by DML** | Cache may contain stale data after DML | Snowflake uses MVCC, so queries see latest data (cache is eventually consistent) |
| **No Cache for Views** | Views themselves are not cached | Query the underlying tables directly |

---
### **I. Practical Examples**

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

-- Subsequent queries will benefit from local disk cache
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

#### **Example 2: Dashboard Optimization with Local Disk Cache**
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

-- Dashboard queries (will benefit from both result cache and local disk cache)
-- Query 1: Daily sales
SELECT
    DATE_TRUNC('DAY', sale_date) AS day,
    region,
    SUM(amount) AS daily_sales
FROM
    sales
WHERE
    sale_date > CURRENT_DATE() - 7
GROUP BY
    day, region
ORDER BY
    day, region;

-- Query 2: Top products
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

-- Query 3: Customer analytics
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
    region
ORDER BY
    total_sales DESC;

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

-- Each cluster has its own local disk cache
-- Query 1 (runs on Cluster 1)
SELECT * FROM table1 WHERE date > CURRENT_DATE() - 7;

-- Query 2 (runs on Cluster 2)
SELECT * FROM table2 WHERE region = 'US';

-- Query 3 (runs on Cluster 3)
SELECT * FROM table3 WHERE category = 'Electronics';

-- Query 4 (runs on Cluster 4)
SELECT * FROM table4 WHERE status = 'Active';

-- Subsequent queries to the same tables will benefit from local disk cache
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

#### **Example 4: Temporary Tables with Local Disk Cache**
```sql
-- Create a warehouse for ETL workloads
CREATE WAREHOUSE etl_wh
  WAREHOUSE_SIZE = 'X-LARGE'
  AUTO_SUSPEND = 3600
  AUTO_RESUME = TRUE;

-- Create temporary tables for intermediate results
-- These will be cached in the warehouse's local disk cache
CREATE TEMPORARY TABLE temp_customers AS
SELECT * FROM customers WHERE signup_date > CURRENT_DATE() - 30;

CREATE TEMPORARY TABLE temp_orders AS
SELECT * FROM orders WHERE order_date > CURRENT_DATE() - 30;

-- Query the temporary tables (benefits from local disk cache)
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

#### **Example 5: Cache Monitoring and Alerting**
```sql
-- Create an alert for high execution time (may indicate cache misses)
CREATE OR REPLACE ALERT high_execution_time_alert
  WAREHOUSE = monitoring_wh
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'  -- Every 5 minutes
AS
  SELECT
    query_id,
    query_text,
    warehouse_name,
    execution_time,
    bytes_scanned,
    partitions_scanned,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE
    execution_time > 5000  -- >5 seconds
    AND bytes_scanned > 10 * 1024 * 1024 * 1024  -- >10GB
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
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE
    execution_time / NULLIF(bytes_scanned, 0) > 0.01  -- >10ms per GB (high I/O latency)
    AND bytes_scanned > 1 * 1024 * 1024 * 1024  -- >1GB
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
  ORDER BY
    execution_time / bytes_scanned DESC;
```

---
---
## **5. Query Cache**

### **A. Definition and Purpose**

**Query Cache** stores **compiled query plans** and **optimized execution paths** to **accelerate query compilation** for **repetitive or similar queries**. This cache is **automatic** and **transparent to users**, reducing the overhead of query parsing and optimization.

**Primary Use Cases:**
- Repetitive queries with different parameters
- Complex queries that are expensive to compile
- Stored procedures and parameterized queries
- Applications with consistent query patterns

---

### **B. Architecture and Workflow**

```mermaid
%% Query Cache Workflow
flowchart TD
    A[("Query Submission")] --> B[("Query Parser")]
    B --> C[("Generate Query Signature")]
    C --> D[("Query Plan Cache Lookup")]
    D --> E{Cache Hit?}
    E -->|Yes| F[("Reuse Cached Query Plan")]
    E -->|No| G[("Generate Query Plan")]
    G --> H[("Cache Query Plan")]
    H --> F
    F --> I[("Query Execution Engine")]

    subgraph CacheDetails["Cache Details"]
        C --> J[("Query Signature Components:")]
        J --> K[("Query Structure")]
        J --> L[("Table Schema")]
        J --> M[("Join Conditions")]
        J --> N[("Filter Conditions")]
        J --> O[("Session Parameters")]
    end

    subgraph CacheTypes["Cache Types"]
        D --> P[("Query Plan Cache")]
        D --> Q[("Compiled Query Cache")]
        D --> R[("Parameterized Query Cache")]
    end

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef process fill:#4285f4,stroke:#1976d2;
    classDef decision fill:#ff9800,stroke:#f57c00;
    classDef cache fill:#9c27b0,stroke:#7b1fa2;
    classDef execution fill:#009688,stroke:#00796b;
    class A,B,C,G,H,I process;
    class D decision;
    class E,F cache;
    class J,K,L,M,N,O default;
    class P,Q,R cache;
```

---

### **C. Technical Deep Dive**

#### **1. How Query Cache Works**
1. **Query Parsing**:
   - When a query is submitted, Snowflake **parses the query** to understand its structure.

2. **Query Signature Generation**:
   - Snowflake generates a **signature** for the query based on:
     - **Query structure** (SELECT, JOIN, GROUP BY, etc.)
     - **Table schema** (columns, data types)
     - **Join conditions**
     - **Filter conditions**
     - **Session parameters** (timezone, date formats, etc.)

3. **Cache Lookup**:
   - Snowflake **checks the query cache** for a matching signature.
   - A **cache hit** occurs if a query with the **same signature** has been executed before.

4. **Cache Hit**:
   - If a cache hit occurs, Snowflake **reuses the cached query plan** (skipping parsing and optimization).
   - The **compilation_time** in `QUERY_HISTORY` will be **very low** (typically <10ms).

5. **Cache Miss**:
   - If no cache hit occurs, Snowflake **generates a new query plan** (parsing, optimization, compilation).
   - After compilation, Snowflake **caches the query plan** for future use.

6. **Cache Types**:
   - **Query Plan Cache**: Stores the **logical query plan** (operators, joins, filters).
   - **Compiled Query Cache**: Stores the **compiled query** (ready to execute).
   - **Parameterized Query Cache**: Stores **templates** for parameterized queries (e.g., `SELECT * FROM table WHERE id = ?`).

#### **2. Cache Storage**
- **Storage Medium**: Memory (in-memory cache)
- **Scope**: Per-session
- **Storage Cost**: Included in Snowflake's compute pricing
- **Size**: Typically a few MB per session (scales with query complexity)

#### **3. Cache Invalidation**
Query cache is **automatically invalidated** when:
1. **DDL Changes**: CREATE, ALTER, DROP TABLE/COLUMN
2. **Schema Changes**: Changes to table or column definitions
3. **Session Parameter Changes**: Changes to timezone, date formats, etc.
4. **Session End**: When the session terminates
5. **Warehouse Restart**: When the warehouse is stopped and restarted

#### **4. Query Signature Generation**
The query signature is generated based on:
- **Query structure** (not exact text, but logical structure)
- **Table and column names**
- **Join conditions**
- **Filter conditions**
- **Group by expressions**
- **Session parameters**

**Example**:
- These queries may have the **same signature**:
  ```sql
  SELECT col1, col2 FROM table1 WHERE col3 = 1;
  SELECT col2, col1 FROM table1 WHERE col3 = 1;
  ```
- These queries will have **different signatures**:
  ```sql
  SELECT col1, col2 FROM table1 WHERE col3 = 1;
  SELECT col1, col2 FROM table1 WHERE col3 = 2;
  ```

#### **5. Parameterized Query Caching**
Snowflake can cache **parameterized query templates** to improve performance for queries with different parameter values.

**Example**:
```sql
-- Parameterized query (can be cached as a template)
SELECT * FROM customers WHERE region = ? AND signup_date > ?;

-- Different parameter values can reuse the cached template
-- Execution 1: region = 'US', signup_date = '2023-01-01'
-- Execution 2: region = 'EU', signup_date = '2023-02-01'
-- Both can reuse the same cached query plan
```

---
### **D. Configuration**

**Note**: Query caching is **automatic** and **requires no configuration**. However, you can **influence** the query cache by:

#### **1. Use Parameterized Queries**
```sql
-- In your application code, use prepared statements
// Java example:
PreparedStatement stmt = connection.prepareStatement(
    "SELECT * FROM customers WHERE region = ? AND signup_date > ?"
);
stmt.setString(1, "US");
stmt.setDate(2, Date.valueOf("2023-01-01"));
ResultSet rs = stmt.executeQuery();

// Python example (using Snowflake Connector):
cursor = conn.cursor()
cursor.execute(
    "SELECT * FROM customers WHERE region = %s AND signup_date > %s",
    ("US", "2023-01-01")
)
results = cursor.fetchall()
```

#### **2. Use Stored Procedures**
```sql
-- Create a stored procedure
CREATE OR REPLACE PROCEDURE get_customer_orders(
    customer_id INT,
    start_date DATE
)
RETURNS TABLE ()
AS
$$
  SELECT * FROM orders
  WHERE customer_id = customer_id
    AND order_date >= start_date;
$$;

-- Call the stored procedure (reuses cached plan)
CALL get_customer_orders(123, '2023-01-01');
CALL get_customer_orders(456, '2023-02-01');
```

#### **3. Use Consistent Query Structure**
```sql
-- These queries may reuse the same cached plan
SELECT col1, col2 FROM table1 WHERE col3 = 1;
SELECT col2, col1 FROM table1 WHERE col3 = 1;

-- These queries will have different plans
SELECT col1, col2 FROM table1 WHERE col3 = 1;
SELECT col1, col2 FROM table1 WHERE col3 = 2;
```

---
### **E. Monitoring and Metrics**

#### **1. Monitor Compilation Time**
```sql
-- Check compilation time for recent queries
SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time,
    start_time,
    warehouse_name,
    user_name
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

#### **2. Identify Queries with High Compilation Time**
```sql
-- Find queries with high compilation time (potential cache misses)
SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time,
    warehouse_name,
    user_name,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    compilation_time > 500  -- >500ms
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    compilation_time DESC;
```

#### **3. Compare Compilation Time with and without Cache**
```sql
-- First query (cache miss)
SELECT * FROM my_table WHERE col1 = 1;

-- Subsequent queries (cache hit)
SELECT * FROM my_table WHERE col1 = 2;
SELECT * FROM my_table WHERE col1 = 3;

-- Compare compilation times
SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%my_table%col1%'
    AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY
    compilation_time;
```

---
### **F. Performance Characteristics**

#### **1. Performance Impact**
| **Metric** | **Without Query Cache** | **With Query Cache (Hit)** | **With Query Cache (Miss)** | **Improvement (Hit)** |
|------------|---------------------------|-----------------------------|-------------------------------|-------------------------|
| Compilation Time | 100-2000ms | 1-10ms | 100-2000ms | **10-100x faster** |
| Execution Time | Unchanged | Unchanged | Unchanged | No change |
| Credit Usage | Unchanged | Unchanged | Unchanged | No change |
| Concurrency | Limited by compilation | Improved | Limited by compilation | **Better resource utilization** |
| First Query | Full compilation | Full compilation | Full compilation | No change |
| Subsequent Queries | Full compilation | Reuse cached plan | Full compilation | **10-100x faster compilation** |

#### **2. Real-World Performance Examples**
**Example 1: Complex Query with Multiple Joins**
```sql
-- Complex query with multiple joins and aggregations
SELECT
    c.region,
    p.category,
    DATE_TRUNC('MONTH', o.order_date) AS month,
    COUNT(o.order_id) AS order_count,
    SUM(o.amount) AS total_sales,
    AVG(o.amount) AS avg_order_value
FROM
    customers c
JOIN
    orders o ON c.customer_id = o.customer_id
JOIN
    products p ON o.product_id = p.product_id
WHERE
    o.order_date BETWEEN '2023-01-01' AND '2023-12-31'
    AND c.region IN ('US', 'EU', 'APAC')
GROUP BY
    c.region, p.category, DATE_TRUNC('MONTH', o.order_date)
ORDER BY
    month, region, category;
```

| **Scenario** | **Compilation Time** | **Execution Time** | **Total Time** |
|--------------|-----------------------|--------------------|---------------|
| First Execution | 1200ms | 8500ms | 9700ms |
| Subsequent Executions (Cache Hit) | 50ms | 8500ms | 8550ms |
| After Schema Change | 1200ms | 8500ms | 9700ms |

**Improvement**: **24x faster compilation** with cache hit

---

**Example 2: Parameterized Query in Application**
```sql
-- Application code using parameterized queries
// First execution (cache miss)
PreparedStatement stmt = connection.prepareStatement(
    "SELECT * FROM products WHERE category = ? AND price > ?"
);
stmt.setString(1, "Electronics");
stmt.setDouble(2, 100.0);
ResultSet rs = stmt.executeQuery();  // Compilation time: 300ms

// Second execution (cache hit)
stmt.setString(1, "Clothing");
stmt.setDouble(2, 50.0);
rs = stmt.executeQuery();  // Compilation time: 5ms

// Third execution (cache hit)
stmt.setString(1, "Furniture");
stmt.setDouble(2, 200.0);
rs = stmt.executeQuery();  // Compilation time: 5ms
```

| **Execution** | **Compilation Time** | **Execution Time** | **Total Time** |
|---------------|-----------------------|--------------------|---------------|
| First | 300ms | 150ms | 450ms |
| Second | 5ms | 120ms | 125ms |
| Third | 5ms | 110ms | 115ms |

**Improvement**: **60x faster compilation** with cache hit

---
### **G. Best Practices**

#### **1. When to Use Query Cache**
✅ **Repetitive queries** with the same structure
✅ **Parameterized queries** (different parameters, same structure)
✅ **Complex queries** that are expensive to compile
✅ **Stored procedures** with consistent query patterns
✅ **Applications with consistent query patterns**
✅ **Queries with high compilation time** (>100ms)

#### **2. When Query Cache is Less Effective**
⚠️ **Unique queries** (each query is different)
⚠️ **Ad-hoc queries** with varying structures
⚠️ **Queries with dynamic SQL** (cannot be cached)
⚠️ **Queries on frequently changing schemas** (cache frequently invalidated)

#### **3. Optimization Techniques**
| **Technique** | **Description** | **Example** |
|---------------|-----------------|-------------|
| **Use Parameterized Queries** | Enables parameterized query caching | `SELECT * FROM table WHERE id = ?` |
| **Use Stored Procedures** | Stored procedures can reuse cached plans | `CREATE PROCEDURE my_proc() AS SELECT ...` |
| **Use Consistent Query Structure** | Similar queries can reuse cached plans | `SELECT col1, col2 FROM table WHERE ...` |
| **Avoid Dynamic SQL** | Dynamic SQL prevents query caching | Use parameterized queries instead |
| **Use Connection Pooling** | Reuse connections to reuse cached query plans | Configure connection pooling in your application |
| **Monitor Compilation Time** | Track query cache effectiveness | `SELECT compilation_time FROM QUERY_HISTORY` |
| **Avoid Frequent DDL Changes** | DDL changes invalidate query cache | Batch DDL changes |
| **Use Consistent Session Parameters** | Ensure same timezone, date formats, etc. | `ALTER SESSION SET TIMEZONE = 'UTC'` |
| **Combine with Other Optimizations** | Use with result cache, local disk cache, etc. | `USE_CACHED_RESULTS = TRUE + parameterized queries` |

#### **4. Common Pitfalls and Solutions**
| **Pitfall** | **Symptom** | **Root Cause** | **Solution** |
|-------------|-------------|----------------|--------------|
| **High Compilation Time for Similar Queries** | Slow compilation despite similar queries | Different query signatures | Use parameterized queries, consistent query structure |
| **Query Cache Not Reusing Plans** | No improvement in compilation time | Different session parameters or schema | Use consistent session parameters, avoid DDL changes |
| **Query Cache Invalidated by DDL** | Performance degrades after schema changes | DDL changes invalidate cache | Batch DDL changes, use views to abstract schema changes |
| **Dynamic SQL Not Cached** | High compilation time for dynamic SQL | Dynamic SQL cannot be cached | Use parameterized queries instead of dynamic SQL |
| **Stored Procedures Not Cached** | High compilation time for stored procedures | Stored procedure definitions may change | Use consistent stored procedure definitions |
| **Different Query Structures** | No cache reuse for logically equivalent queries | Different query signatures | Standardize query structure |
| **Session Parameter Changes** | Cache misses due to parameter changes | Different session parameters | Set session parameters explicitly |
| **Frequent Schema Changes** | Cache frequently invalidated | Schema changes invalidate cache | Batch schema changes |

---
### **H. Limitations**

| **Limitation** | **Description** | **Workaround** |
|----------------|-----------------|---------------|
| **Session-Scoped Cache** | Query cache is session-scoped | Use connection pooling to reuse sessions |
| **Invalidated by DDL** | DDL changes invalidate query cache | Batch DDL changes |
| **Invalidated by Schema Changes** | Schema changes invalidate query cache | Avoid frequent schema changes |
| **No Direct Monitoring** | Query cache usage is not directly visible | Monitor compilation time |
| **No Caching for All Queries** | Some complex queries cannot be cached | Use materialized views or result caching |
| **No Caching for Dynamic SQL** | Dynamic SQL cannot be cached | Use parameterized queries |
| **No Custom Cache Size** | Cannot configure query cache size | Use parameterized queries for better reuse |
| **Memory Overhead** | Query cache consumes memory | Monitor warehouse memory usage |
| **No Cache for Views** | Views themselves are not cached (but underlying queries may be) | Query the underlying tables directly |
| **No Cache for External Tables** | Limited caching for external tables | Use internal tables or materialized views |

---
### **I. Practical Examples**

#### **Example 1: Parameterized Query Optimization**
```sql
-- Application code using parameterized queries (Java)
String sql = "SELECT customer_id, name, email FROM customers WHERE region = ? AND signup_date > ?";
PreparedStatement stmt = connection.prepareStatement(sql);

// Execution 1 (cache miss)
stmt.setString(1, "US");
stmt.setDate(2, Date.valueOf("2023-01-01"));
ResultSet rs = stmt.executeQuery();  // Compilation time: 200ms

// Execution 2 (cache hit)
stmt.setString(1, "EU");
stmt.setDate(2, Date.valueOf("2023-02-01"));
rs = stmt.executeQuery();  // Compilation time: 5ms

// Execution 3 (cache hit)
stmt.setString(1, "APAC");
stmt.setDate(2, Date.valueOf("2023-03-01"));
rs = stmt.executeQuery();  // Compilation time: 5ms

-- Check compilation times
SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%customers%region%signup_date%'
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY
    compilation_time;
```

#### **Example 2: Stored Procedure Optimization**
```sql
-- Create a stored procedure for customer analytics
CREATE OR REPLACE PROCEDURE get_customer_analytics(
    region_filter STRING,
    start_date DATE,
    end_date DATE
)
RETURNS TABLE (
    customer_id BIGINT,
    name STRING,
    total_spend FLOAT,
    order_count INT
)
AS
$$
  SELECT
      c.customer_id,
      c.name,
      SUM(o.amount) AS total_spend,
      COUNT(o.order_id) AS order_count
  FROM
      customers c
  LEFT JOIN
      orders o ON c.customer_id = o.customer_id
  WHERE
      (region_filter IS NULL OR c.region = region_filter)
      AND o.order_date BETWEEN start_date AND end_date
  GROUP BY
      c.customer_id, c.name
  ORDER BY
      total_spend DESC;
$$;

-- Call the stored procedure (first execution - cache miss)
CALL get_customer_analytics('US', '2023-01-01', '2023-12-31');

-- Call the stored procedure (second execution - cache hit)
CALL get_customer_analytics('EU', '2023-01-01', '2023-12-31');

-- Call the stored procedure (third execution - cache hit)
CALL get_customer_analytics(NULL, '2023-06-01', '2023-06-30');

-- Check compilation times
SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%get_customer_analytics%'
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY
    compilation_time;
```

#### **Example 3: Complex Query with Consistent Structure**
```sql
-- Query 1 (cache miss)
SELECT
    region,
    product_category,
    SUM(sales) AS total_sales
FROM
    sales
WHERE
    sale_date BETWEEN '2023-01-01' AND '2023-01-31'
GROUP BY
    region, product_category
ORDER BY
    total_sales DESC;

-- Query 2 (cache hit - same structure, different filters)
SELECT
    region,
    product_category,
    SUM(sales) AS total_sales
FROM
    sales
WHERE
    sale_date BETWEEN '2023-02-01' AND '2023-02-28'
GROUP BY
    region, product_category
ORDER BY
    total_sales DESC;

-- Query 3 (cache hit - same structure, different filters)
SELECT
    region,
    product_category,
    SUM(sales) AS total_sales
FROM
    sales
WHERE
    sale_date BETWEEN '2023-03-01' AND '2023-03-31'
GROUP BY
    region, product_category
ORDER BY
    total_sales DESC;

-- Check compilation times
SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%region%product_category%SUM(sales)%'
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY
    compilation_time;
```

#### **Example 4: Monitor and Alert on High Compilation Time**
```sql
-- Create an alert for high compilation time
CREATE OR REPLACE ALERT high_compilation_time_alert
  WAREHOUSE = monitoring_wh
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
    compilation_time > 1000  -- >1 second
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
  ORDER BY
    compilation_time DESC;

-- Create an alert for queries with high compilation time relative to execution time
CREATE OR REPLACE ALERT high_compilation_ratio_alert
  WAREHOUSE = monitoring_wh
  SCHEDULE = 'USING CRON 0 * * * * America/Los_Angeles'  -- Every hour
AS
  SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time,
    compilation_time * 100.0 / NULLIF(execution_time, 0) AS compilation_ratio_percent,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE
    compilation_time * 100.0 / NULLIF(execution_time, 0) > 10  -- Compilation > 10% of execution time
    AND execution_time > 1000  -- Execution time > 1 second
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
  ORDER BY
    compilation_ratio_percent DESC;
```

#### **Example 5: Connection Pooling for Query Cache Reuse**
```sql
// Application code using connection pooling (Java example)
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:snowflake://myaccount.snowflakecomputing.com");
config.setUsername("myuser");
config.setPassword("mypassword");
config.setMaximumPoolSize(10);  // Reuse connections
config.setConnectionTimeout(30000);
config.setIdleTimeout(600000);  // 10 minutes

HikariDataSource dataSource = new HikariDataSource(config);

// First query (cache miss)
try (Connection connection = dataSource.getConnection();
     PreparedStatement stmt = connection.prepareStatement(
         "SELECT * FROM products WHERE category = ?"
     )) {
    stmt.setString(1, "Electronics");
    ResultSet rs = stmt.executeQuery();
    // Compilation time: 200ms
}

// Second query (reuses connection and cache)
try (Connection connection = dataSource.getConnection();
     PreparedStatement stmt = connection.prepareStatement(
         "SELECT * FROM products WHERE category = ?"
     )) {
    stmt.setString(1, "Clothing");
    ResultSet rs = stmt.executeQuery();
    // Compilation time: 5ms (cache hit)
}

// Third query (reuses connection and cache)
try (Connection connection = dataSource.getConnection();
     PreparedStatement stmt = connection.prepareStatement(
         "SELECT * FROM products WHERE category = ?"
     )) {
    stmt.setString(1, "Furniture");
    ResultSet rs = stmt.executeQuery();
    // Compilation time: 5ms (cache hit)
}
```

---
---
## **6. File Metadata Cache**

### **A. Definition and Purpose**

**File Metadata Cache** stores **metadata for external tables** and **cloud storage files** to **accelerate query compilation** and **enable optimization** for external data sources. This cache is particularly important for **external tables** (S3, Azure Blob, GCS) where metadata lookups can be expensive.

**Primary Use Cases:**
- External tables on cloud storage
- Partitioned external tables
- Queries with predicate pushdown on external tables
- Frequent access to external data

---

### **B. Architecture and Workflow**

```mermaid
%% File Metadata Cache Workflow
flowchart TD
    A[("Query on External Table")] --> B[("Parse Query")]
    B --> C[("Extract File Metadata Requirements")]
    C --> D[("File Metadata Cache Lookup")]
    D --> E{Cache Hit?}
    E -->|Yes| F[("Use Cached Metadata")]
    E -->|No| G[("Fetch from Cloud Storage")]
    G --> H[("Cache Metadata")]
    H --> F
    F --> I[("Optimize Query Plan")]
    I --> J[("Execute Query")]

    subgraph CacheDetails["Cache Details"]
        D --> K[("Cache Key Components:")]
        K --> L[("Stage Name")]
        K --> M[("File Format")]
        K --> N[("External Table Definition")]
        K --> O[("Partition Information")]
        K --> P[("File Listings")]
    end

    subgraph CloudStorage["Cloud Storage"]
        G --> Q[("S3 / Azure Blob / GCS")]
    end

    subgraph Monitoring["Monitoring"]
        R[("QUERY_HISTORY.compilation_time")]
    end
    J --> R

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef process fill:#4285f4,stroke:#1976d2;
    classDef decision fill:#ff9800,stroke:#f57c00;
    classDef cache fill:#3f51b5,stroke:#303f9f;
    classDef execution fill:#009688,stroke:#00796b;
    classDef storage fill:#29abe2,stroke:#1a8fb8;
    classDef monitoring fill:#9c27b0,stroke:#7b1fa2;
    class A,B,C,G,H,I,J process;
    class D decision;
    class E,F cache;
    class K,L,M,N,O,P default;
    class Q storage;
    class R monitoring;
```

---

### **C. Technical Deep Dive**

#### **1. How File Metadata Cache Works**
1. **Query Parsing**:
   - When a query on an **external table** is submitted, Snowflake **parses the query** to understand the required data.

2. **Metadata Requirements Extraction**:
   - Snowflake **identifies the metadata** needed for the query:
     - **Stage definitions** (URL, credentials, file formats)
     - **File formats** (type, compression, delimiters, etc.)
     - **External table definitions** (column mappings, partitioning)
     - **Partition information** (partition columns, values)
     - **File listings** (list of files in the stage)

3. **Cache Lookup**:
   - Snowflake **checks the file metadata cache** for the required metadata.
   - A **cache hit** occurs if the metadata is already cached.

4. **Cache Hit**:
   - If a cache hit occurs, Snowflake **uses the cached metadata** for query optimization.
   - The **compilation_time** in `QUERY_HISTORY` will be **reduced**.

5. **Cache Miss**:
   - If no cache hit occurs, Snowflake **fetches the metadata** from **cloud storage** (S3, Azure Blob, GCS).
   - After fetching, Snowflake **caches the metadata** for future use.

6. **Cache Invalidation**:
   - The file metadata cache is **automatically invalidated** when:
     - The **external table definition changes** (ALTER EXTERNAL TABLE)
     - The **stage definition changes** (ALTER STAGE)
     - The **file format changes** (ALTER FILE FORMAT)
     - The **external files change** (new files added, files deleted)
     - The **session ends**
     - The **warehouse restarts**

#### **2. Metadata Types Cached**
| **Metadata Type** | **Description** | **Usage** | **Update Frequency** |
|-------------------|-----------------|-----------|----------------------|
| **Stage Metadata** | Stage URL, credentials, file formats | Query optimization, file access | On DDL changes |
| **File Format Metadata** | File format type (PARQUET, CSV, JSON, etc.), options (compression, field delimiter, etc.) | Query optimization, file parsing | On DDL changes |
| **External Table Metadata** | External table definition, column mappings, partitioning | Query optimization, file access | On DDL changes |
| **File Metadata** | File sizes, last modified times, partitions | Partition pruning, file access | On external file changes |
| **Partition Metadata** | Partition columns, partition values | Partition pruning | On external file changes |
| **File Listings** | List of files in the stage | File access | On external file changes |

#### **3. Cache Storage**
- **Storage Medium**: Memory (in-memory cache)
- **Scope**: Per-session
- **Storage Cost**: Included in Snowflake's compute pricing
- **Size**: Typically a few MB per session (scales with number of external tables and files)

#### **4. Cache Behavior with External Tables**
- **Partitioned External Tables**: File metadata cache enables **partition pruning** for partitioned external tables.
- **Non-Partitioned External Tables**: File metadata cache still improves performance by caching file listings and metadata.
- **External Stages**: File metadata cache stores information about **stages** (URL, credentials, file formats).
- **File Formats**: File metadata cache stores **file format definitions** (type, compression, delimiters, etc.).

---
### **D. Configuration**

**Note**: File metadata cache is **automatic** and **requires no configuration**. However, you can **influence** the cache by:

#### **1. Create Partitioned External Tables**
```sql
-- Create a stage
CREATE STAGE my_s3_stage
  URL = 's3://my-bucket/sales/'
  CREDENTIALS = (AWS_KEY_ID = '...' AWS_SECRET_KEY = '...');

-- Create a file format
CREATE FILE FORMAT my_parquet_format
  TYPE = 'PARQUET'
  COMPRESSION = 'SNAPPY';

-- Create a partitioned external table
CREATE EXTERNAL TABLE my_partitioned_external_table (
    sale_id BIGINT,
    customer_id BIGINT,
    sale_date DATE,
    amount FLOAT,
    region STRING
)
WITH LOCATION = @my_s3_stage
FILE_FORMAT = (TYPE = 'PARQUET')
PARTITION BY (YEAR(sale_date), MONTH(sale_date), DAY(sale_date));
```

#### **2. Refresh External Table Metadata**
```sql
-- Refresh metadata for an external table (if external files have changed)
ALTER EXTERNAL TABLE my_external_table REFRESH;

-- Refresh metadata for all external tables in a schema
FOR table IN (
    SELECT table_name
    FROM INFORMATION_SCHEMA.EXTERNAL_TABLES
    WHERE table_schema = 'MY_SCHEMA'
) DO
    EXECUTE IMMEDIATE 'ALTER EXTERNAL TABLE ' || table || ' REFRESH';
END FOR;
```

#### **3. Use Consistent Stage and File Format Definitions**
```sql
-- Use consistent stage definitions
CREATE STAGE my_stage URL = 's3://my-bucket/';

-- Use consistent file format definitions
CREATE FILE FORMAT my_format TYPE = 'PARQUET';

-- Use these consistently for external tables
CREATE EXTERNAL TABLE my_table
WITH LOCATION = @my_stage
FILE_FORMAT = (TYPE = 'PARQUET');
```

---
### **E. Monitoring and Metrics**

#### **1. Monitor Compilation Time for External Tables**
```sql
-- Check compilation time for external table queries
SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time,
    start_time,
    warehouse_name
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%my_external_table%'
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    compilation_time DESC;

-- Check average compilation time for external tables
SELECT
    REGEXP_SUBSTR(query_text, 'FROM\\s+([^\\s,;]+)', 1, 1, '', 1) AS table_name,
    AVG(compilation_time) AS avg_compilation_time,
    COUNT(*) AS query_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%FROM%'
    AND compilation_time > 0
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    table_name
HAVING
    table_name IN (
        SELECT table_name
        FROM INFORMATION_SCHEMA.EXTERNAL_TABLES
    )
ORDER BY
    avg_compilation_time DESC;
```

#### **2. Monitor Partition Pruning for External Tables**
```sql
-- Check if partition pruning is working for external tables
SELECT
    query_id,
    query_text,
    partitions_scanned,
    bytes_scanned,
    execution_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%my_partitioned_external_table%'
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    partitions_scanned;

-- Compare partitions scanned before and after metadata refresh
-- Before refresh
SELECT partitions_scanned
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_id = 'before_refresh_query_id';

-- After refresh
ALTER EXTERNAL TABLE my_partitioned_external_table REFRESH;
SELECT partitions_scanned
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_id = 'after_refresh_query_id';
```

#### **3. Monitor External Table Metadata**
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

-- Check when external table metadata was last refreshed
SELECT
    table_name,
    last_refreshed
FROM
    SNOWFLAKE.ACCOUNT_USAGE.EXTERNAL_TABLE_REFRESH_HISTORY
WHERE
    table_name = 'MY_EXTERNAL_TABLE'
ORDER BY
    last_refreshed DESC;
```

---
### **F. Performance Characteristics**

#### **1. Performance Impact**
| **Metric** | **Without File Metadata Cache** | **With File Metadata Cache** | **Improvement** | **Notes** |
|------------|-------------------------------------|----------------------------------|-----------------|-----------|
| Compilation Time | 500-5000ms | 50-500ms | **5-10x faster** | Depends on number of files and partitions |
| Query Optimization | Limited | Improved | Better execution plans | Enables partition pruning, predicate pushdown |
| First Query Performance | Slow | Fast | Faster subsequent queries | Metadata is cached after first query |
| Partition Pruning | Disabled | Enabled | **10-100x less data scanned** | Requires partitioned external tables |
| File Access | Slow | Fast | Faster file listings | Cached file listings reduce cloud storage lookups |

#### **2. Real-World Performance Examples**
**Example 1: Partitioned External Table Query**
```sql
-- Create a partitioned external table
CREATE EXTERNAL TABLE sales_external (
    sale_id BIGINT,
    customer_id BIGINT,
    sale_date DATE,
    amount FLOAT,
    region STRING
)
WITH LOCATION = @my_s3_stage
FILE_FORMAT = (TYPE = 'PARQUET')
PARTITION BY (YEAR(sale_date), MONTH(sale_date), DAY(sale_date));

-- Query with partition pruning
SELECT * FROM sales_external
WHERE YEAR(sale_date) = 2023 AND MONTH(sale_date) = 1;
```

| **Scenario** | **Compilation Time** | **Execution Time** | **Partitions Scanned** | **Bytes Scanned** |
|--------------|-----------------------|--------------------|------------------------|-------------------|
| First Execution | 2000ms | 15.5s | 365 (all partitions) | 100 GB |
| With File Metadata Cache | 200ms | 0.8s | 31 (January partitions) | 8 GB |

**Improvement**:
- **Compilation Time**: 10x faster
- **Execution Time**: 19x faster
- **Partitions Scanned**: 12x reduction
- **Bytes Scanned**: 12.5x reduction

---

**Example 2: Non-Partitioned External Table Query**
```sql
-- Create a non-partitioned external table
CREATE EXTERNAL TABLE logs_external (
    log_id BIGINT,
    timestamp TIMESTAMP_NTZ,
    log_level STRING,
    message STRING
)
WITH LOCATION = @my_logs_stage
FILE_FORMAT = (TYPE = 'JSON');

-- Query with filter
SELECT * FROM logs_external
WHERE timestamp > CURRENT_DATE() - 7 AND log_level = 'ERROR';
```

| **Scenario** | **Compilation Time** | **Execution Time** | **Files Scanned** |
|--------------|-----------------------|--------------------|-------------------|
| First Execution | 1500ms | 8.2s | 1000 |
| With File Metadata Cache | 150ms | 2.1s | 1000 |

**Improvement**:
- **Compilation Time**: 10x faster
- **Execution Time**: 4x faster (due to better optimization)

**Note**: For non-partitioned external tables, the **files scanned** remains the same, but **compilation time** and **optimization** improve.

---
### **G. Best Practices**

#### **1. When to Use File Metadata Cache**
✅ **External tables** on cloud storage (S3, Azure Blob, GCS)
✅ **Partitioned external tables** (enables partition pruning)
✅ **Frequently accessed external data**
✅ **Queries with filters** on external tables
✅ **Large external tables** with many files

#### **2. When File Metadata Cache is Less Effective**
⚠️ **Ad-hoc queries** on external tables (metadata not reused)
⚠️ **Small external tables** (metadata overhead may outweigh benefits)
⚠️ **Frequently changing external files** (cache frequently invalidated)
⚠️ **Non-partitioned external tables** (limited optimization benefits)

#### **3. Optimization Techniques**
| **Technique** | **Description** | **Example** |
|---------------|-----------------|-------------|
| **Use Partitioned External Tables** | Enables partition pruning | `CREATE EXTERNAL TABLE ... PARTITION BY (date)` |
| **Refresh Metadata When Needed** | Update metadata when external files change | `ALTER EXTERNAL TABLE my_table REFRESH` |
| **Use Consistent Stage Definitions** | Avoid changing stage definitions frequently | `CREATE STAGE my_stage URL = 's3://my-bucket/'` |
| **Use Consistent File Formats** | Avoid changing file format definitions frequently | `CREATE FILE FORMAT my_format TYPE = 'PARQUET'` |
| **Monitor Compilation Time** | Track file metadata cache effectiveness | `SELECT compilation_time FROM QUERY_HISTORY` |
| **Avoid Frequent External Table Changes** | Changes to external tables invalidate cache | Use consistent external table definitions |
| **Use External Table Clustering** | Cluster external tables for better performance | `ALTER EXTERNAL TABLE my_table CLUSTER BY (date)` |
| **Combine with Other Optimizations** | Use with result cache, local disk cache, etc. | `PARTITION BY (date) + USE_CACHED_RESULTS = TRUE` |
| **Document External Table Configurations** | Document stage, file format, and external table configurations | Internal wiki or Confluence page |

#### **4. Common Pitfalls and Solutions**
| **Pitfall** | **Symptom** | **Root Cause** | **Solution** |
|-------------|-------------|----------------|--------------|
| **High Compilation Time for External Tables** | Slow query compilation | Missing or stale file metadata | Refresh external table metadata |
| **No Partition Pruning for External Tables** | Full table scans despite filters | Missing partitioning or metadata | Use partitioned external tables, refresh metadata |
| **File Metadata Cache Not Improving Performance** | No improvement despite caching | External files not accessed repeatedly | Query external data more frequently |
| **Cache Frequently Invalidated** | Performance degrades after external file changes | External file changes invalidate cache | Use internal tables for hot data, refresh metadata selectively |
| **Slow First Query on External Tables** | First query is slow | Metadata not cached | Expected behavior (metadata is cached after first query) |
| **No Cache for Dynamic External Table References** | Queries with dynamic stage references not cached | Different query signatures | Use consistent stage references |
| **External File Changes Not Reflected** | Queries return stale data | File metadata cache not refreshed | Use `ALTER EXTERNAL TABLE ... REFRESH` |
| **High Cloud Storage Costs** | Expensive metadata lookups | Frequent cache misses for external tables | Use internal tables for hot data, refresh metadata selectively |

---
### **H. Limitations**

| **Limitation** | **Description** | **Workaround** |
|----------------|-----------------|---------------|
| **Session-Scoped Cache** | File metadata cache is session-scoped | Use connection pooling to reuse sessions |
| **Invalidated by DDL** | DDL changes invalidate file metadata cache | Batch DDL changes |
| **Invalidated by External Changes** | Changes to external files may not be reflected | Use `ALTER EXTERNAL TABLE ... REFRESH` |
| **No Direct Monitoring** | File metadata cache usage is not directly visible | Monitor compilation time and query performance |
| **No Partial Metadata Caching** | Cannot cache partial metadata | Use filtered external tables or views |
| **No Custom Metadata** | Cannot customize metadata | Use Snowflake's built-in metadata |
| **Cloud Storage Latency** | File metadata cache does not eliminate cloud storage latency | Use local stages or internal tables for hot data |
| **Storage Overhead** | File metadata consumes memory | Monitor memory usage |
| **No Cache for All External Table Configurations** | Some configurations may not be cached effectively | Use standard configurations |
| **Partitioning Required for Best Performance** | Partition pruning requires partitioned external tables | Use `PARTITION BY` for external tables |

---
### **I. Practical Examples**

#### **Example 1: Partitioned External Table with File Metadata Cache**
```sql
-- Create a stage
CREATE STAGE sales_stage
  URL = 's3://my-bucket/sales/'
  CREDENTIALS = (AWS_KEY_ID = '...' AWS_SECRET_KEY = '...');

-- Create a file format
CREATE FILE FORMAT parquet_format
  TYPE = 'PARQUET'
  COMPRESSION = 'SNAPPY';

-- Create a partitioned external table
CREATE EXTERNAL TABLE sales_external (
    sale_id BIGINT,
    customer_id BIGINT,
    sale_date DATE,
    amount FLOAT,
    region STRING,
    product_id BIGINT
)
WITH LOCATION = @sales_stage
FILE_FORMAT = (TYPE = 'PARQUET')
PARTITION BY (YEAR(sale_date), MONTH(sale_date), DAY(sale_date));

-- Query with partition pruning (benefits from file metadata cache)
SELECT
    region,
    SUM(amount) AS daily_sales,
    COUNT(*) AS transaction_count
FROM
    sales_external
WHERE
    YEAR(sale_date) = 2023
    AND MONTH(sale_date) = 1
    AND DAY(sale_date) = 15
GROUP BY
    region
ORDER BY
    daily_sales DESC;

-- Check partition pruning effectiveness
SELECT
    query_id,
    query_text,
    partitions_scanned,
    bytes_scanned,
    execution_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%sales_external%'
    AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY
    partitions_scanned;
```

#### **Example 2: Refresh External Table Metadata**
```sql
-- New files added to the stage
-- Upload new sales data to s3://my-bucket/sales/2023/06/15/

-- Refresh external table metadata to include new files
ALTER EXTERNAL TABLE sales_external REFRESH;

-- Query the external table (now includes new files)
SELECT * FROM sales_external
WHERE sale_date = '2023-06-15';

-- Check when metadata was last refreshed
SELECT
    table_name,
    last_refreshed
FROM
    SNOWFLAKE.ACCOUNT_USAGE.EXTERNAL_TABLE_REFRESH_HISTORY
WHERE
    table_name = 'SALES_EXTERNAL'
ORDER BY
    last_refreshed DESC;
```

#### **Example 3: Monitor File Metadata Cache Performance**
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
    query_text LIKE '%sales_external%'
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    compilation_time DESC;

-- Check average compilation time by external table
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

#### **Example 4: Automate Metadata Refresh**
```sql
-- Create a stored procedure to refresh external table metadata
CREATE OR REPLACE PROCEDURE refresh_external_tables_metadata(schema_name STRING)
RETURNS STRING
LANGUAGE JAVASCRIPT
AS
$$
  const refreshSQL = `
    FOR table IN (
        SELECT table_name
        FROM INFORMATION_SCHEMA.EXTERNAL_TABLES
        WHERE table_schema = '${SCHEMA_NAME}'
    ) DO
        EXECUTE IMMEDIATE 'ALTER EXTERNAL TABLE ${SCHEMA_NAME}.' || table || ' REFRESH';
    END FOR;
  `;

  const result = snowflake.execute({sqlText: refreshSQL.replace('${SCHEMA_NAME}', SCHEMA_NAME)});
  return 'Refreshed metadata for all external tables in schema: ' + SCHEMA_NAME;
$$;

-- Call the procedure to refresh metadata for a schema
CALL refresh_external_tables_metadata('MY_SCHEMA');

-- Create a task to refresh metadata daily
CREATE TASK daily_external_table_refresh
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON 0 3 * * * America/Los_Angeles'  -- 3 AM daily
AS
  CALL refresh_external_tables_metadata('MY_SCHEMA');
```

#### **Example 5: Compare Performance Before/After Partitioning**
```sql
-- Create a non-partitioned external table
CREATE EXTERNAL TABLE sales_non_partitioned (
    sale_id BIGINT,
    customer_id BIGINT,
    sale_date DATE,
    amount FLOAT,
    region STRING
)
WITH LOCATION = @sales_stage
FILE_FORMAT = (TYPE = 'PARQUET');

-- Query the non-partitioned table
SELECT * FROM sales_non_partitioned
WHERE sale_date = '2023-01-15';
SELECT LAST_QUERY_ID() AS query_id_non_partitioned;

-- Create a partitioned external table
CREATE EXTERNAL TABLE sales_partitioned (
    sale_id BIGINT,
    customer_id BIGINT,
    sale_date DATE,
    amount FLOAT,
    region STRING
)
WITH LOCATION = @sales_stage
FILE_FORMAT = (TYPE = 'PARQUET')
PARTITION BY (YEAR(sale_date), MONTH(sale_date), DAY(sale_date));

-- Query the partitioned table
SELECT * FROM sales_partitioned
WHERE sale_date = '2023-01-15';
SELECT LAST_QUERY_ID() AS query_id_partitioned;

-- Compare performance
SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time,
    partitions_scanned,
    bytes_scanned
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_id IN ('query_id_non_partitioned', 'query_id_partitioned')
ORDER BY
    query_id;
```

---
---
## **7. Warehouse Cache**

### **A. Definition and Purpose**

**Warehouse Cache** stores **warehouse-specific data** to **improve query performance** for operations within a warehouse. This includes **temporary tables**, **CTEs (Common Table Expressions)**, **intermediate query results**, and **spill data**. The warehouse cache is **automatic** and **transparent to users**.

**Primary Use Cases:**
- Temporary tables in ETL pipelines
- Intermediate results in complex queries
- Spill data from large operations
- Session-specific data

---

### **B. Architecture and Workflow**

```mermaid
%% Warehouse Cache Workflow
flowchart TD
    A[("Query Submission")] --> B[("Query Parser")]
    B --> C[("Identify Warehouse-Specific Data")]
    C --> D[("Warehouse Cache Lookup")]
    D --> E{Cache Hit?}
    E -->|Yes| F[("Return Cached Data\n<10ms")]
    E -->|No| G[("Execute Query")]
    G --> H[("Cache Results in Warehouse Cache")]
    H --> F
    F --> I[("Query Execution Engine")]

    subgraph CacheDetails["Cache Details"]
        D --> J[("Cache Key Components:")]
        J --> K[("Temporary Table Name")]
        J --> L[("CTE Name")]
        J --> M[("Query ID")]
        J --> N[("Session ID")]
    end

    subgraph CachedData["Cached Data Types"]
        H --> O[("Temporary Tables")]
        H --> P[("CTE Results")]
        H --> Q[("Intermediate Results")]
        H --> R[("Spill Data")]
    end

    subgraph Storage["Storage Layer"]
        H --> S[("SSD Storage")]
    end

    subgraph Monitoring["Monitoring"]
        T[("QUERY_PROFILE")]
    end
    I --> T

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef process fill:#4285f4,stroke:#1976d2;
    classDef decision fill:#ff9800,stroke:#f57c00;
    classDef cache fill:#795548,stroke:#5d4037;
    classDef execution fill:#009688,stroke:#00796b;
    classDef storage fill:#29abe2,stroke:#1a8fb8;
    classDef monitoring fill:#9c27b0,stroke:#7b1fa2;
    class A,B,C,G,H,I process;
    class D decision;
    class E,F cache;
    class J,K,L,M,N default;
    class O,P,Q,R cachedData;
    class S storage;
    class T monitoring;
```

---

### **C. Technical Deep Dive**

#### **1. How Warehouse Cache Works**
1. **Data Caching**:
   - When a query is executed, Snowflake **caches warehouse-specific data** in **local SSD**, including:
     - **Temporary tables**: Tables created with `CREATE TEMPORARY TABLE`
     - **CTE results**: Results from Common Table Expressions (WITH clauses)
     - **Intermediate query results**: Results from subqueries or join operations
     - **Spill data**: Data spilled to disk during query execution

2. **Cache Lookup**:
   - For subsequent queries, Snowflake **checks the warehouse cache** first.
   - If the data is **cached**, it is **returned from SSD** (typical latency: **1-10ms**).
   - If the data is **not cached**, it is **recomputed or fetched from cloud storage**.

3. **Cache Management**:
   - **Cache Size**: Limited by the **warehouse size** (larger warehouses have more cache capacity).
   - **Eviction Policy**: **Least Recently Used (LRU)** - When the cache is full, the least recently accessed data is evicted.
   - **Scope**: **Per-warehouse** - Each warehouse has its own cache.
   - **Session Scope**: Some cached data (e.g., temporary tables) is **session-scoped**.

4. **Cache Invalidation**:
   Warehouse cache is **automatically invalidated** when:
   - The **warehouse is restarted** (stopped and started)
   - The **session ends** (for session-scoped data)
   - The **cache is full** and data is evicted (LRU)
   - The **underlying data changes** (for some cached data types)

#### **2. Cached Data Types**
| **Data Type** | **Description** | **Scope** | **Invalidation** |
|---------------|-----------------|-----------|-----------------|
| **Temporary Tables** | Tables created with `CREATE TEMPORARY TABLE` | Session | Session end, warehouse restart |
| **CTE Results** | Results from Common Table Expressions (WITH clauses) | Query | Warehouse restart, LRU eviction |
| **Intermediate Results** | Results from subqueries or join operations | Query | Warehouse restart, LRU eviction |
| **Spill Data** | Data spilled to disk during query execution | Query | Warehouse restart, LRU eviction |

#### **3. Cache Storage Details**
- **Storage Medium**: SSD (NVMe in most cases)
- **Data Format**: Compressed columnar format (same as cloud storage)
- **Compression**: Data is stored in compressed format
- **Encryption**: Data is encrypted at rest

#### **4. Cache Behavior with Different Operations**
| **Operation** | **Cacheable?** | **Notes** |
|---------------|----------------|-----------|
| CREATE TEMPORARY TABLE | ✅ Yes | Primary use case for warehouse cache |
| WITH clause (CTE) | ✅ Yes | CTE results may be cached |
| Subqueries | ✅ Yes | Intermediate results may be cached |
| JOIN operations | ✅ Yes | Intermediate join results may be cached |
| Aggregations | ✅ Yes | Intermediate aggregation results may be cached |
| Spill to Disk | ✅ Yes | Spill data is cached in warehouse cache |
| DML on Temporary Tables | ✅ Yes | Changes to temporary tables are cached |
| DDL on Temporary Tables | ❌ No | DDL operations are not cached |
| Session End | ❌ No | Session-scoped data is invalidated |

---
### **D. Configuration**

**Note**: Warehouse cache is **automatic** and **requires no configuration**. However, you can **influence** the cache by:

#### **1. Use Temporary Tables**
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

#### **2. Use CTEs for Intermediate Results**
```sql
-- Use CTEs for intermediate results (may be cached in warehouse cache)
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

-- Reuse the CTE in subsequent queries
WITH sales_cte AS (
    SELECT * FROM sales WHERE sale_date > CURRENT_DATE() - 7
)
SELECT
    product_category,
    SUM(amount) AS total_sales
FROM
    sales_cte
GROUP BY
    product_category;
```

#### **3. Warehouse Sizing for Cache Capacity**
```sql
-- Create a warehouse with appropriate size for your cache needs
CREATE WAREHOUSE etl_wh WAREHOUSE_SIZE = 'X-LARGE';  -- ~256GB cache
CREATE WAREHOUSE analytics_wh WAREHOUSE_SIZE = '2X-LARGE';  -- ~512GB cache

-- Resize an existing warehouse
ALTER WAREHOUSE my_wh SET WAREHOUSE_SIZE = 'X-LARGE';
```

#### **4. Auto-Suspend Configuration**
```sql
-- Set auto-suspend to balance cost and cache retention
ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = 1800;  -- 30 minutes (cache retained during suspend)

-- Disable auto-suspend for always-on warehouses
ALTER WAREHOUSE always_on_wh SET AUTO_SUSPEND = NULL;
```

**Note**: Auto-suspend **does not invalidate** the warehouse cache. The cache is **retained** while the warehouse is suspended and **available immediately** when the warehouse resumes.

---
### **E. Monitoring and Metrics**

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
    query_text LIKE '%temp_%'
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

---
### **F. Performance Characteristics**

#### **1. Performance Impact**
| **Metric** | **Without Warehouse Cache** | **With Warehouse Cache (Hit)** | **With Warehouse Cache (Miss)** | **Improvement (Hit)** |
|------------|--------------------------------|----------------------------------|------------------------------------|------------------------|
| I/O Latency | 100-500ms (cloud storage) | 1-10ms (SSD) | 100-500ms (cloud storage) | **10-100x faster** |
| Throughput | Limited by cloud storage | Limited by SSD | Limited by cloud storage | **2-10x higher** |
| Execution Time | Higher (waiting for I/O) | Lower (fast I/O) | Higher (waiting for I/O) | **2-10x faster** |
| Bytes Scanned | Full scan from cloud storage | Cached data from SSD | Full scan from cloud storage | **Same** (but faster) |
| Credit Usage | Higher (more time spent waiting for I/O) | Lower (less time waiting for I/O) | Higher (more time spent waiting for I/O) | **1.1-2x reduction** |
| Temporary Table Performance | Slow (recomputed each time) | Fast (cached) | Slow (recomputed) | **10-100x faster** |

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
    source_orders o ON c.customer_id = o.customer_id
WHERE
    o.order_date > CURRENT_DATE() - 1;

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

---

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

---
### **G. Best Practices**

#### **1. When to Use Warehouse Cache**
✅ **Temporary tables** in ETL pipelines
✅ **CTEs (Common Table Expressions)** in complex queries
✅ **Intermediate results** in multi-step queries
✅ **Frequently accessed data** within a session
✅ **Large queries** that spill to disk
✅ **Multi-step transformations** where intermediate results are reused

#### **2. When Warehouse Cache is Less Effective**
⚠️ **Ad-hoc queries** (data not likely to be re-accessed)
⚠️ **Small warehouses** (limited cache capacity)
⚠️ **Short-lived sessions** (cache not populated)
⚠️ **Unique queries** (no data reuse)
⚠️ **Write-heavy workloads** (cache may be frequently evicted)

#### **3. Optimization Techniques**
| **Technique** | **Description** | **Example** |
|---------------|-----------------|-------------|
| **Use Temporary Tables for Intermediate Results** | Store intermediate results in temporary tables | `CREATE TEMPORARY TABLE temp AS SELECT ...` |
| **Reuse Temporary Tables** | Query the same temporary table multiple times | `SELECT * FROM temp_table WHERE ...` |
| **Use CTEs for Complex Queries** | Use CTEs to break down complex queries | `WITH cte AS (SELECT ...) SELECT * FROM cte` |
| **Use Larger Warehouses for Cache-Intensive Workloads** | Larger warehouses have more cache capacity | `CREATE WAREHOUSE my_wh WAREHOUSE_SIZE = 'X-LARGE'` |
| **Use Auto-Suspend for Cost Savings** | Suspend warehouses when idle to save costs | `ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = 600` |
| **Avoid Frequent Warehouse Restarts** | Cache is lost when warehouse is restarted | Use auto-suspend/resume instead of stop/start |
| **Monitor Cache Performance** | Check QUERY_PROFILE for cache hits | `SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))` |
| **Combine with Other Optimizations** | Use with result cache, local disk cache, etc. | `CREATE TEMPORARY TABLE temp AS SELECT ... + USE_CACHED_RESULTS = TRUE` |
| **Document Cacheable Data** | Document which data benefits from caching | Internal wiki or Confluence page |

#### **4. Common Pitfalls and Solutions**
| **Pitfall** | **Symptom** | **Root Cause** | **Solution** |
|-------------|-------------|----------------|--------------|
| **Cache Not Populated** | No performance improvement | Data not accessed repeatedly | Query data multiple times to populate cache |
| **Cache Eviction Due to Large Data Volume** | Performance degrades after large queries | Large queries evict hot data from cache | Use smaller warehouses for large queries, larger warehouses for hot data |
| **Cache Lost on Warehouse Restart** | Performance degrades after warehouse restart | Cache is invalidated on restart | Use auto-suspend instead of stop/start |
| **Low Cache Hit Rate** | Poor performance despite caching | Data access patterns not cache-friendly | Use temporary tables, CTEs, query hot data more frequently |
| **Cache Not Effective for Ad-Hoc Queries** | No improvement for ad-hoc queries | Ad-hoc queries access different data each time | Use result caching or materialized views for repetitive ad-hoc queries |
| **Cache Not Shared Across Sessions** | Performance varies across sessions | Cache is session-scoped for some data | Use global temporary tables or persistent tables |
| **High Memory Usage** | Warehouse memory pressure | Large cached data | Use filtering to reduce data volume, use smaller warehouses |
| **Temporary Tables Not Reused** | Temporary tables not providing benefit | Temporary tables not queried multiple times | Reuse temporary tables in subsequent queries |

---
### **H. Limitations**

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

---
### **I. Practical Examples**

#### **Example 1: ETL Pipeline with Temporary Tables**
```sql
-- Create a warehouse for ETL workloads
CREATE WAREHOUSE etl_wh
  WAREHOUSE_SIZE = 'X-LARGE'
  AUTO_SUSPEND = 3600
  AUTO_RESUME = TRUE;

-- Step 1: Extract recent data
CREATE TEMPORARY TABLE temp_customers AS
SELECT * FROM source_customers
WHERE last_updated > CURRENT_DATE() - 1;

CREATE TEMPORARY TABLE temp_orders AS
SELECT * FROM source_orders
WHERE order_date > CURRENT_DATE() - 1;

-- Step 2: Transform data
CREATE TEMPORARY TABLE temp_customer_orders AS
SELECT
    c.customer_id,
    c.name,
    c.region,
    o.order_id,
    o.order_date,
    o.amount,
    o.product_id
FROM
    temp_customers c
JOIN
    temp_orders o ON c.customer_id = o.customer_id;

-- Step 3: Enrich with product data
CREATE TEMPORARY TABLE temp_enriched_orders AS
SELECT
    t.customer_id,
    t.name,
    t.region,
    t.order_id,
    t.order_date,
    t.amount,
    p.product_name,
    p.category
FROM
    temp_customer_orders t
JOIN
    products p ON t.product_id = p.product_id;

-- Step 4: Aggregate data
CREATE TEMPORARY TABLE temp_aggregated AS
SELECT
    region,
    category,
    DATE_TRUNC('DAY', order_date) AS day,
    COUNT(DISTINCT customer_id) AS active_customers,
    COUNT(order_id) AS order_count,
    SUM(amount) AS total_sales
FROM
    temp_enriched_orders
GROUP BY
    region, category, DATE_TRUNC('DAY', order_date);

-- Step 5: Load to target
INSERT INTO target_sales
SELECT * FROM temp_aggregated;

-- Check performance of each step
SELECT
    query_id,
    query_text,
    execution_time,
    bytes_scanned,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    warehouse_name = 'ETL_WH'
    AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY
    start_time;
```

#### **Example 2: Complex Query with CTEs**
```sql
-- Complex query with multiple CTEs
WITH
-- CTE 1: Filter recent sales
recent_sales AS (
    SELECT * FROM sales
    WHERE sale_date > CURRENT_DATE() - 30
),

-- CTE 2: Join with customers
sales_with_customers AS (
    SELECT
        s.sale_id,
        s.customer_id,
        s.sale_date,
        s.amount,
        s.product_id,
        c.name AS customer_name,
        c.region
    FROM
        recent_sales s
    JOIN
        customers c ON s.customer_id = c.customer_id
),

-- CTE 3: Join with products
sales_with_products AS (
    SELECT
        swc.*,
        p.product_name,
        p.category
    FROM
        sales_with_customers swc
    JOIN
        products p ON swc.product_id = p.product_id
),

-- CTE 4: Aggregate by region and category
region_category_sales AS (
    SELECT
        region,
        category,
        DATE_TRUNC('DAY', sale_date) AS day,
        SUM(amount) AS total_sales,
        COUNT(*) AS transaction_count
    FROM
        sales_with_products
    GROUP BY
        region, category, DATE_TRUNC('DAY', sale_date)
)

-- Final query
SELECT
    region,
    category,
    day,
    total_sales,
    transaction_count,
    RANK() OVER (PARTITION BY region ORDER BY total_sales DESC) AS region_rank,
    RANK() OVER (PARTITION BY category ORDER BY total_sales DESC) AS category_rank
FROM
    region_category_sales
ORDER BY
    day, total_sales DESC;

-- Check CTE performance
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('last_query_id'));
```

#### **Example 3: Temporary Table Reuse**
```sql
-- Create a temporary table for hot data
CREATE TEMPORARY TABLE temp_hot_products AS
SELECT * FROM products
WHERE category IN ('Electronics', 'Clothing', 'Furniture')
AND price > 100;

-- Query 1: Top products by sales
SELECT
    p.product_id,
    p.product_name,
    p.category,
    SUM(s.amount) AS total_sales
FROM
    temp_hot_products p
JOIN
    sales s ON p.product_id = s.product_id
WHERE
    s.sale_date > CURRENT_DATE() - 7
GROUP BY
    p.product_id, p.product_name, p.category
ORDER BY
    total_sales DESC
LIMIT 10;

-- Query 2: Product inventory
SELECT
    p.product_id,
    p.product_name,
    p.category,
    p.price,
    i.quantity AS inventory_quantity
FROM
    temp_hot_products p
JOIN
    inventory i ON p.product_id = i.product_id
WHERE
    i.quantity < 100
ORDER BY
    i.quantity;

-- Query 3: Product performance by region
SELECT
    p.category,
    s.region,
    SUM(s.amount) AS region_sales,
    COUNT(*) AS region_transactions
FROM
    temp_hot_products p
JOIN
    sales s ON p.product_id = s.product_id
WHERE
    s.sale_date > CURRENT_DATE() - 7
GROUP BY
    p.category, s.region
ORDER BY
    region_sales DESC;

-- Check performance of queries on temporary table
SELECT
    query_id,
    query_text,
    execution_time,
    bytes_scanned,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%temp_hot_products%'
    AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY
    execution_time;
```

#### **Example 4: Monitor and Alert on Warehouse Cache Performance**
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

#### **Example 5: Compare Performance with and without Temporary Tables**
```sql
-- Query without temporary table
SELECT
    c.region,
    COUNT(o.order_id) AS order_count,
    SUM(o.amount) AS total_sales
FROM
    customers c
JOIN
    orders o ON c.customer_id = o.customer_id
WHERE
    o.order_date > CURRENT_DATE() - 7
GROUP BY
    c.region
ORDER BY
    total_sales DESC;
SELECT LAST_QUERY_ID() AS query_id_without_temp;

-- Query with temporary table
CREATE TEMPORARY TABLE temp_recent_orders AS
SELECT * FROM orders WHERE order_date > CURRENT_DATE() - 7;

SELECT
    c.region,
    COUNT(o.order_id) AS order_count,
    SUM(o.amount) AS total_sales
FROM
    customers c
JOIN
    temp_recent_orders o ON c.customer_id = o.customer_id
GROUP BY
    c.region
ORDER BY
    total_sales DESC;
SELECT LAST_QUERY_ID() AS query_id_with_temp;

-- Compare performance
SELECT
    query_id,
    query_text,
    execution_time,
    compilation_time,
    bytes_scanned
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_id IN ('query_id_without_temp', 'query_id_with_temp')
ORDER BY
    query_id;
```

---
---
## **8. Cache Type Comparison and Selection Guide**

---

### **A. Comprehensive Cache Type Comparison**

| **Feature** | **Result Cache** | **Metadata Cache** | **Local Disk Cache** | **Query Cache** | **File Metadata Cache** | **Warehouse Cache** |
|-------------|------------------|--------------------|----------------------|-----------------|--------------------------|---------------------|
| **Primary Purpose** | Cache query results | Cache table metadata | Cache hot micro-partitions | Cache query plans | Cache external table metadata | Cache warehouse-specific data |
| **Scope** | Per-user, per-query | Per-session | Per-warehouse | Per-session | Per-session | Per-warehouse |
| **TTL** | 24 hours (configurable) | Session | Session | Session | Session | Session |
| **Storage Medium** | SSD | Memory | SSD | Memory | Memory | SSD |
| **What It Caches** | Complete query result sets | Table schema, statistics, partition info | Frequently accessed micro-partitions | Query execution plans, compiled queries | External stage definitions, file formats, partition metadata | Temporary tables, CTE results, intermediate results, spill data |
| **Performance Impact** | ⭐⭐⭐⭐⭐ (10-1000x faster) | ⭐⭐ (1.1-2x faster compilation) | ⭐⭐⭐ (2-10x faster I/O) | ⭐⭐ (1.1-2x faster compilation) | ⭐⭐ (1.1-2x faster compilation) | ⭐⭐⭐ (2-10x faster I/O) |
| **Cost** | Included | Included | Included | Included | Included | Included |
| **Configuration** | `USE_CACHED_RESULTS`, `RESULT_CACHE_TTL` | Automatic | Automatic | Automatic | Automatic | Automatic |
| **Best For** | Repetitive identical queries | All queries | Hot tables, BI tools | Parameterized queries, stored procedures | External tables | Temporary tables, CTEs, intermediate results |
| **Invalidation Triggers** | Data changes, query text changes, permissions changes, TTL expiry | DDL changes, significant data changes, session end | Warehouse restart, session end, LRU eviction | DDL changes, schema changes, session parameters changes, session end | DDL changes, external file changes, stage changes, session end | Warehouse restart, session end, LRU eviction |
| **Monitoring** | `QUERY_HISTORY.used_cached_result` | `QUERY_HISTORY.compilation_time` | `QUERY_PROFILE` (indirect) | `QUERY_HISTORY.compilation_time` | `QUERY_HISTORY.compilation_time` | `QUERY_PROFILE` (indirect) |
| **Storage Overhead** | Medium (result sets) | Low (metadata) | High (micro-partitions) | Low (query plans) | Low (metadata) | Medium (temporary data) |
| **Memory Overhead** | Low | Medium | Low | Medium | Medium | Low |
| **Automatic** | Yes (with configuration) | Yes | Yes | Yes | Yes | Yes |
| **User Control** | Limited (TTL, enable/disable) | No | No | No | No | No |
| **Works with Internal Tables** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No | ✅ Yes |
| **Works with External Tables** | ⚠️ Limited | ❌ No | ⚠️ Limited | ❌ No | ✅ Yes | ❌ No |
| **Works with Temporary Tables** | ⚠️ Limited | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No | ✅ Yes |

---

### **B. Cache Type Selection Matrix**

| **Use Case** | **Result Cache** | **Metadata Cache** | **Local Disk Cache** | **Query Cache** | **File Metadata Cache** | **Warehouse Cache** | **Recommended Approach** |
|-------------|------------------|--------------------|----------------------|-----------------|--------------------------|---------------------|--------------------------|
| **Dashboard Queries** | ✅✅✅ | ✅✅ | ✅✅✅ | ✅✅ | ❌ | ✅✅ | Result Cache + Local Disk Cache + Warehouse Cache |
| **Repetitive Aggregations** | ✅✅✅ | ✅✅ | ✅✅ | ✅✅ | ❌ | ✅✅ | Result Cache + Metadata Cache |
| **Hot Tables** | ✅✅ | ✅✅✅ | ✅✅✅ | ✅ | ❌ | ✅ | Local Disk Cache + Result Cache |
| **Complex Joins** | ✅✅ | ✅✅✅ | ✅✅ | ✅✅✅ | ❌ | ✅✅ | Metadata Cache + Query Cache |
| **Parameterized Queries** | ⚠️ | ✅✅ | ✅✅ | ✅✅✅ | ❌ | ✅ | Query Cache + Local Disk Cache |
| **External Table Queries** | ⚠️ | ❌ | ✅ | ❌ | ✅✅✅ | ❌ | File Metadata Cache + Local Disk Cache |
| **Temporary Tables** | ⚠️ | ✅✅ | ✅✅ | ✅✅ | ❌ | ✅✅✅ | Warehouse Cache + Local Disk Cache |
| **ETL Pipelines** | ⚠️ | ✅✅ | ✅✅ | ✅ | ⚠️ | ✅✅✅ | Warehouse Cache + Local Disk Cache + Metadata Cache |
| **Ad-Hoc Analysis** | ⚠️ | ✅✅ | ✅✅ | ✅✅ | ⚠️ | ⚠️ | Metadata Cache + Query Cache + Local Disk Cache |
| **Real-Time Analytics** | ❌ | ✅✅ | ✅✅ | ✅✅ | ⚠️ | ✅✅ | Local Disk Cache + Query Cache (disable Result Cache) |
| **Large Scans** | ❌ | ✅✅ | ✅✅✅ | ✅ | ⚠️ | ✅✅ | Local Disk Cache + Metadata Cache |
| **Small Tables** | ✅✅ | ✅✅ | ⚠️ | ✅✅ | ❌ | ⚠️ | Result Cache + Metadata Cache + Query Cache |
| **Frequently Changing Data** | ❌ | ✅✅ | ✅✅ | ✅✅ | ⚠️ | ✅✅ | Metadata Cache + Query Cache + Local Disk Cache |
| **Static Data** | ✅✅✅ | ✅✅ | ✅✅✅ | ✅✅ | ⚠️ | ✅✅✅ | Result Cache + Local Disk Cache + Warehouse Cache |

**Key**:
- ✅✅✅ = Highly recommended
- ✅✅ = Recommended
- ✅ = Some benefit
- ⚠️ = Limited benefit
- ❌ = Not applicable or not recommended

---
### **C. Cache Type Decision Tree**

```mermaid
%% Cache Type Decision Tree
flowchart TD
    A[("Query Performance\nIssue")] --> B{Query Type?}
    B -->|Repetitive identical queries| C[("Result Cache")]
    B -->|Frequently accessed tables| D[("Local Disk Cache")]
    B -->|Complex queries| E[("Query Cache + Metadata Cache")]
    B -->|External tables| F[("File Metadata Cache + Local Disk Cache")]
    B -->|Temporary data| G[("Warehouse Cache + Local Disk Cache")]
    B -->|All queries| H[("Metadata Cache\n(Automatic)")]

    C --> I[("Enable USE_CACHED_RESULTS")]
    D --> J[("Use larger warehouses")]
    E --> K[("Use parameterized queries\n+ Update statistics")]
    F --> L[("Use partitioned external tables\n+ Refresh metadata")]
    G --> M[("Use TEMPORARY tables\n+ Reuse data")]
    H --> N[("No configuration needed")]

    I --> O[("Set RESULT_CACHE_TTL")]
    J --> P[("Query hot data repeatedly")]
    K --> Q[("Use stored procedures\n+ Connection pooling")]
    L --> R[("Monitor compilation time")]
    M --> S[("Monitor QUERY_PROFILE")]
    N --> T[("Monitor compilation time")]

    O --> U[("Use for dashboards/reports")]
    P --> V[("Use for BI tools")]
    Q --> W[("Use for application queries")]
    R --> X[("Use for external table queries")]
    S --> Y[("Use for ETL/intermediate results")]
    T --> Z[("Use for all queries")]

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef start fill:#4285f4,stroke:#1976d2;
    classDef result fill:#ff9800,stroke:#f57c00;
    classDef local fill:#e91e63,stroke:#c2185b;
    classDef query fill:#9c27b0,stroke:#7b1fa2;
    classDef file fill:#3f51b5,stroke:#303f9f;
    classDef warehouse fill:#795548,stroke:#5d4037;
    classDef metadata fill:#009688,stroke:#00796b;
    class A start;
    class B start;
    class C result;
    class D local;
    class E query;
    class F file;
    class G warehouse;
    class H metadata;
    class I,O result;
    class J,P local;
    class K,Q query;
    class L,R file;
    class M,S warehouse;
    class N,T metadata;
    class U,V,W,X,Y,Z default;
```

---
### **D. Cache Type Implementation Patterns**

#### **Pattern 1: Dashboard Optimization (Result Cache + Local Disk Cache + Warehouse Cache)**
```sql
-- Create a dedicated warehouse for dashboards
CREATE WAREHOUSE dashboard_wh
  WAREHOUSE_SIZE = 'LARGE'
  MAX_CLUSTER_COUNT = 2
  AUTO_SUSPEND = 1800
  AUTO_RESUME = TRUE;

-- Enable result caching for dashboard users
ALTER ROLE dashboard_role SET USE_CACHED_RESULTS = TRUE;
ALTER ROLE dashboard_role SET RESULT_CACHE_TTL = 3600;  -- 1 hour

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
    day, region;

-- Dashboard queries (benefit from all cache types)
-- Query 1: Daily sales by region
SELECT * FROM temp_daily_sales WHERE day > CURRENT_DATE() - 7;

-- Query 2: Top regions by sales
SELECT
    region,
    SUM(total_sales) AS monthly_sales
FROM
    temp_daily_sales
WHERE
    day > CURRENT_DATE() - 30
GROUP BY
    region
ORDER BY
    monthly_sales DESC;

-- Query 3: Sales trend
SELECT
    day,
    SUM(total_sales) AS daily_sales
FROM
    temp_daily_sales
WHERE
    day > CURRENT_DATE() - 30
GROUP BY
    day
ORDER BY
    day;

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

#### **Pattern 2: ETL Pipeline Optimization (Warehouse Cache + Local Disk Cache + Metadata Cache)**
```sql
-- Create a warehouse for ETL workloads
CREATE WAREHOUSE etl_wh
  WAREHOUSE_SIZE = 'X-LARGE'
  AUTO_SUSPEND = 3600
  AUTO_RESUME = TRUE;

-- Update statistics for source tables
ALTER TABLE source_customers UPDATE STATISTICS;
ALTER TABLE source_orders UPDATE STATISTICS;
ALTER TABLE source_products UPDATE STATISTICS;

-- ETL pipeline with temporary tables
-- Step 1: Extract
CREATE TEMPORARY TABLE temp_customers AS
SELECT * FROM source_customers WHERE last_updated > CURRENT_DATE() - 1;

CREATE TEMPORARY TABLE temp_orders AS
SELECT * FROM source_orders WHERE order_date > CURRENT_DATE() - 1;

-- Step 2: Transform
CREATE TEMPORARY TABLE temp_customer_orders AS
SELECT
    c.customer_id,
    c.name,
    c.region,
    o.order_id,
    o.order_date,
    o.amount,
    o.product_id
FROM
    temp_customers c
JOIN
    temp_orders o ON c.customer_id = o.customer_id;

-- Step 3: Enrich
CREATE TEMPORARY TABLE temp_enriched_orders AS
SELECT
    t.*,
    p.product_name,
    p.category
FROM
    temp_customer_orders t
JOIN
    source_products p ON t.product_id = p.product_id;

-- Step 4: Aggregate
CREATE TEMPORARY TABLE temp_aggregated AS
SELECT
    region,
    category,
    DATE_TRUNC('DAY', order_date) AS day,
    COUNT(DISTINCT customer_id) AS active_customers,
    COUNT(order_id) AS order_count,
    SUM(amount) AS total_sales
FROM
    temp_enriched_orders
GROUP BY
    region, category, day;

-- Step 5: Load
INSERT INTO target_sales
SELECT * FROM temp_aggregated;

-- Check ETL performance
SELECT
    query_id,
    query_text,
    execution_time,
    bytes_scanned,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    warehouse_name = 'ETL_WH'
    AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY
    start_time;
```

#### **Pattern 3: External Table Optimization (File Metadata Cache + Local Disk Cache)**
```sql
-- Create a stage
CREATE STAGE sales_stage
  URL = 's3://my-bucket/sales/'
  CREDENTIALS = (AWS_KEY_ID = '...' AWS_SECRET_KEY = '...');

-- Create a file format
CREATE FILE FORMAT parquet_format
  TYPE = 'PARQUET'
  COMPRESSION = 'SNAPPY';

-- Create a partitioned external table
CREATE EXTERNAL TABLE sales_external (
    sale_id BIGINT,
    customer_id BIGINT,
    sale_date DATE,
    amount FLOAT,
    region STRING,
    product_id BIGINT
)
WITH LOCATION = @sales_stage
FILE_FORMAT = (TYPE = 'PARQUET')
PARTITION BY (YEAR(sale_date), MONTH(sale_date), DAY(sale_date));

-- Refresh metadata (if external files have changed)
ALTER EXTERNAL TABLE sales_external REFRESH;

-- Query with partition pruning (benefits from file metadata cache and local disk cache)
SELECT
    region,
    product_id,
    SUM(amount) AS total_sales
FROM
    sales_external
WHERE
    YEAR(sale_date) = 2023
    AND MONTH(sale_date) = 1
GROUP BY
    region, product_id
ORDER BY
    total_sales DESC;

-- Check external table performance
SELECT
    query_id,
    query_text,
    compilation_time,
    execution_time,
    partitions_scanned,
    bytes_scanned,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%sales_external%'
    AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY
    execution_time;
```

#### **Pattern 4: Application Query Optimization (Query Cache + Result Cache + Local Disk Cache)**
```sql
-- Create a warehouse for application queries
CREATE WAREHOUSE app_wh
  WAREHOUSE_SIZE = 'MEDIUM'
  AUTO_SUSPEND = 600
  AUTO_RESUME = TRUE;

-- Enable result caching for application users
ALTER ROLE app_role SET USE_CACHED_RESULTS = TRUE;
ALTER ROLE app_role SET RESULT_CACHE_TTL = 1800;  -- 30 minutes

-- Application code using parameterized queries and connection pooling
// Java example with HikariCP
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:snowflake://myaccount.snowflakecomputing.com");
config.setUsername("app_user");
config.setPassword("app_password");
config.setMaximumPoolSize(20);  // Connection pool
config.setConnectionTimeout(30000);
config.setIdleTimeout(600000);  // 10 minutes

HikariDataSource dataSource = new HikariDataSource(config);

// Parameterized query (benefits from query cache and result cache)
String sql = "SELECT customer_id, name, email FROM customers WHERE region = ? AND signup_date > ?";
try (Connection connection = dataSource.getConnection();
     PreparedStatement stmt = connection.prepareStatement(sql)) {

    // Execution 1 (cache miss)
    stmt.setString(1, "US");
    stmt.setDate(2, Date.valueOf("2023-01-01"));
    ResultSet rs = stmt.executeQuery();  // Compilation: 200ms, Execution: 150ms

    // Execution 2 (query cache hit, result cache miss)
    stmt.setString(1, "EU");
    stmt.setDate(2, Date.valueOf("2023-02-01"));
    rs = stmt.executeQuery();  // Compilation: 5ms, Execution: 120ms

    // Execution 3 (query cache hit, result cache hit)
    stmt.setString(1, "US");
    stmt.setDate(2, Date.valueOf("2023-01-01"));
    rs = stmt.executeQuery();  // Compilation: 5ms, Execution: 5ms
}

// Check application query performance
SELECT
    query_id,
    query_text,
    used_cached_result,
    compilation_time,
    execution_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    warehouse_name = 'APP_WH'
    AND user_name = 'APP_USER'
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY
    compilation_time;
```

#### **Pattern 5: Mixed Workload Optimization (All Cache Types)**
```sql
-- Create warehouses for different workloads
CREATE WAREHOUSE dashboard_wh WAREHOUSE_SIZE = 'LARGE' AUTO_SUSPEND = 1800;
CREATE WAREHOUSE etl_wh WAREHOUSE_SIZE = 'X-LARGE' AUTO_SUSPEND = 3600;
CREATE WAREHOUSE app_wh WAREHOUSE_SIZE = 'MEDIUM' AUTO_SUSPEND = 600;
CREATE WAREHOUSE reporting_wh WAREHOUSE_SIZE = '2X-LARGE' MAX_CLUSTER_COUNT = 4;

-- Configure caching for each workload
-- Dashboards: Result Cache + Local Disk Cache + Warehouse Cache
ALTER ROLE dashboard_role SET USE_CACHED_RESULTS = TRUE;
ALTER ROLE dashboard_role SET RESULT_CACHE_TTL = 3600;

-- ETL: Warehouse Cache + Local Disk Cache + Metadata Cache
ALTER TABLE source_customers UPDATE STATISTICS;
ALTER TABLE source_orders UPDATE STATISTICS;

-- Application: Query Cache + Result Cache + Local Disk Cache
ALTER ROLE app_role SET USE_CACHED_RESULTS = TRUE;
ALTER ROLE app_role SET RESULT_CACHE_TTL = 1800;

-- Reporting: All cache types
ALTER ROLE reporting_role SET USE_CACHED_RESULTS = TRUE;
ALTER ROLE reporting_role SET RESULT_CACHE_TTL = 7200;
ALTER TABLE reporting_sales UPDATE STATISTICS;
ALTER TABLE reporting_customers UPDATE STATISTICS;

-- Create temporary tables for reporting
CREATE TEMPORARY TABLE temp_reporting_data AS
SELECT * FROM reporting_sales WHERE sale_date > CURRENT_DATE() - 90;

-- Monitor performance across all workloads
SELECT
    warehouse_name,
    query_id,
    query_text,
    used_cached_result,
    compilation_time,
    execution_time,
    bytes_scanned,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY
    warehouse_name, execution_time;
```

---
### **E. Cache Type Anti-Patterns**

| **Anti-Pattern** | **Description** | **Impact** | **Solution** |
|------------------|-----------------|------------|--------------|
| **Not Using Any Caching** | Disabling all caching | ❌ Poor performance, ❌ High costs | Enable at least metadata cache and local disk cache |
| **Using Result Cache for Unique Queries** | Caching queries that are never repeated | ❌ Cache overhead, ❌ No benefit | Disable result cache for unique queries |
| **Frequent Cache Invalidation** | Data or schema changes invalidating cache | ❌ Cache inefficiency, ❌ High costs | Batch changes, use appropriate TTL |
| **Inconsistent Query Text** | Different query text for the same logical query | ❌ Cache misses, ❌ Poor performance | Standardize query formatting, use parameterized queries |
| **Not Using Parameterized Queries** | Using dynamic SQL instead of parameterized queries | ❌ No query cache reuse, ❌ Poor performance | Use parameterized queries |
| **Using Small Warehouses for Hot Data** | Using small warehouses for frequently accessed data | ❌ Limited cache capacity, ❌ Poor performance | Use larger warehouses for hot data |
| **Not Updating Statistics** | Not updating statistics for large tables | ❌ Poor query optimization, ❌ Cache inefficiency | Update statistics manually or automatically |
| **Frequent DDL Changes** | Making frequent schema changes | ❌ Cache invalidation, ❌ Poor performance | Batch DDL changes |
| **Not Using Temporary Tables** | Not using temporary tables for intermediate results | ❌ No warehouse cache benefits | Use temporary tables for intermediate results |
| **Not Monitoring Cache Performance** | Not tracking cache effectiveness | ❌ No visibility into performance, ❌ Issues not detected | Monitor cache hits, misses, and performance |
| **Using Result Cache for Real-Time Data** | Caching data that needs to be real-time | ❌ Stale results | Disable result cache or use shorter TTL |
| **Not Combining Cache Types** | Using only one cache type | ❌ Suboptimal performance | Combine cache types for maximum benefit |

---
---
## **9. Production Checklists**

---
### **A. Result Cache Checklist**

#### **1. Configuration**
- [ ] Enable result caching for appropriate users/roles:
  ```sql
  ALTER SESSION SET USE_CACHED_RESULTS = TRUE;
  ALTER USER my_user SET USE_CACHED_RESULTS = TRUE;
  ALTER ROLE my_role SET USE_CACHED_RESULTS = TRUE;
  ```
- [ ] Set appropriate TTL for each workload:
  ```sql
  -- Dashboards (1 hour)
  ALTER SESSION SET RESULT_CACHE_TTL = 3600;

  -- Reports (2 hours)
  ALTER SESSION SET RESULT_CACHE_TTL = 7200;

  -- Static data (24 hours - max)
  ALTER SESSION SET RESULT_CACHE_TTL = 86400;
  ```
- [ ] Disable result caching for unique queries:
  ```sql
  ALTER SESSION SET USE_CACHED_RESULTS = FALSE;
  ```

#### **2. Monitoring**
- [ ] Monitor cache usage:
  ```sql
  SELECT used_cached_result FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY;
  ```
- [ ] Monitor cache hit rate:
  ```sql
  SELECT used_cached_result, COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY GROUP BY used_cached_result;
  ```
- [ ] Monitor cache performance:
  ```sql
  SELECT query_id, query_text, execution_time, credits_used
  FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE used_cached_result = TRUE;
  ```
- [ ] Set up alerts for low cache hit rate:
  ```sql
  CREATE ALERT low_cache_hit_rate_alert ...;
  ```

#### **3. Optimization**
- [ ] Use consistent query text for cache hits
- [ ] Use parameterized queries for better cache reuse
- [ ] Set consistent session parameters (timezone, date formats, etc.)
- [ ] Use for repetitive queries (dashboards, reports, BI tools)
- [ ] Use for expensive queries (high credit usage)
- [ ] Avoid for unique queries (ad-hoc analysis)
- [ ] Batch data changes to minimize cache invalidation
- [ ] Document cacheable queries
- [ ] Combine with other optimizations (clustering, materialized views)

---
### **B. Metadata Cache Checklist**

#### **1. Configuration**
- [ ] Update statistics for large tables:
  ```sql
  ALTER TABLE my_table UPDATE STATISTICS;
  ```
- [ ] Update statistics after bulk data loads:
  ```sql
  COPY INTO my_table FROM @my_stage;
  ALTER TABLE my_table UPDATE STATISTICS;
  ```
- [ ] Enable automatic statistics for important tables:
  ```sql
  ALTER TABLE my_table SET AUTO_UPDATE_STATISTICS = TRUE;
  ```

#### **2. Monitoring**
- [ ] Monitor compilation time:
  ```sql
  SELECT compilation_time FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY;
  ```
- [ ] Monitor query performance:
  ```sql
  SELECT query_id, query_text, compilation_time, execution_time
  FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY;
  ```
- [ ] Monitor statistics freshness:
  ```sql
  SELECT table_name, last_updated
  FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STATISTICS;
  ```
- [ ] Set up alerts for high compilation time:
  ```sql
  CREATE ALERT high_compilation_time_alert ...;
  ```

#### **3. Optimization**
- [ ] Update statistics for large tables (>1TB)
- [ ] Update statistics after significant data changes (>10% of table size)
- [ ] Use clustering for better metadata (partition pruning)
- [ ] Avoid frequent DDL changes
- [ ] Monitor partition pruning effectiveness
- [ ] Use consistent table schemas
- [ ] Document metadata management strategies

---
### **C. Local Disk Cache Checklist**

#### **1. Configuration**
- [ ] Use larger warehouses for hot data:
  ```sql
  CREATE WAREHOUSE hot_data_wh WAREHOUSE_SIZE = 'X-LARGE';
  ```
- [ ] Use multi-cluster warehouses for high concurrency:
  ```sql
  CREATE WAREHOUSE mc_wh MAX_CLUSTER_COUNT = 4;
  ```
- [ ] Set appropriate auto-suspend times:
  ```sql
  ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = 1800;
  ```

#### **2. Monitoring**
- [ ] Monitor query profile for cache usage:
  ```sql
  SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'));
  ```
- [ ] Monitor warehouse utilization:
  ```sql
  SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY;
  ```
- [ ] Monitor warehouse events:
  ```sql
  SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_EVENTS_HISTORY;
  ```
- [ ] Set up alerts for warehouse restarts:
  ```sql
  CREATE ALERT warehouse_restart_alert ...;
  ```

#### **3. Optimization**
- [ ] Query the same data repeatedly to populate cache
- [ ] Use clustering for cache efficiency
- [ ] Avoid frequent warehouse restarts
- [ ] Use auto-suspend for idle warehouses
- [ ] Use multi-cluster warehouses for high concurrency
- [ ] Combine with result caching
- [ ] Monitor cache performance
- [ ] Document cacheable data

---
### **D. Query Cache Checklist**

#### **1. Configuration**
- [ ] Use parameterized queries in application code
- [ ] Use stored procedures for repetitive queries:
  ```sql
  CREATE PROCEDURE my_proc() AS SELECT * FROM my_table;
  ```
- [ ] Use connection pooling to reuse connections

#### **2. Monitoring**
- [ ] Monitor compilation time:
  ```sql
  SELECT compilation_time FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY;
  ```
- [ ] Monitor query performance:
  ```sql
  SELECT query_id, query_text, compilation_time, execution_time
  FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY;
  ```
- [ ] Set up alerts for high compilation time:
  ```sql
  CREATE ALERT high_compilation_time_alert ...;
  ```

#### **3. Optimization**
- [ ] Use consistent query structure for better cache reuse
- [ ] Avoid dynamic SQL
- [ ] Use stored procedures
- [ ] Monitor compilation time
- [ ] Avoid frequent DDL changes
- [ ] Use connection pooling
- [ ] Use consistent session parameters
- [ ] Document query patterns

---
### **E. File Metadata Cache Checklist**

#### **1. Configuration**
- [ ] Use partitioned external tables:
  ```sql
  CREATE EXTERNAL TABLE my_table PARTITION BY (date);
  ```
- [ ] Refresh external table metadata when needed:
  ```sql
  ALTER EXTERNAL TABLE my_table REFRESH;
  ```
- [ ] Use consistent stage definitions:
  ```sql
  CREATE STAGE my_stage URL = 's3://my-bucket/';
  ```
- [ ] Use consistent file format definitions:
  ```sql
  CREATE FILE FORMAT my_format TYPE = 'PARQUET';
  ```

#### **2. Monitoring**
- [ ] Monitor compilation time for external tables:
  ```sql
  SELECT compilation_time FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE query_text LIKE '%my_external_table%';
  ```
- [ ] Monitor external table performance:
  ```sql
  SELECT query_id, query_text, execution_time
  FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE query_text LIKE '%my_external_table%';
  ```
- [ ] Set up alerts for high compilation time on external tables:
  ```sql
  CREATE ALERT high_external_compilation_time_alert ...;
  ```

#### **3. Optimization**
- [ ] Use partitioned external tables for partition pruning
- [ ] Refresh external table metadata when external files change
- [ ] Use consistent stage and file format definitions
- [ ] Monitor partition pruning effectiveness
- [ ] Avoid frequent external table changes
- [ ] Use external table clustering
- [ ] Combine with other optimizations
- [ ] Document external table configurations

---
### **F. Warehouse Cache Checklist**

#### **1. Configuration**
- [ ] Use temporary tables for intermediate results:
  ```sql
  CREATE TEMPORARY TABLE temp AS SELECT * FROM my_table;
  ```
- [ ] Use CTEs for complex queries:
  ```sql
  WITH cte AS (SELECT * FROM my_table) SELECT * FROM cte;
  ```
- [ ] Use larger warehouses for cache-intensive workloads:
  ```sql
  CREATE WAREHOUSE my_wh WAREHOUSE_SIZE = 'X-LARGE';
  ```

#### **2. Monitoring**
- [ ] Monitor query profile for cache usage:
  ```sql
  SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'));
  ```
- [ ] Monitor temporary table performance:
  ```sql
  SELECT query_id, query_text, execution_time
  FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE query_text LIKE '%TEMPORARY TABLE%';
  ```
- [ ] Set up alerts for warehouse restarts:
  ```sql
  CREATE ALERT warehouse_restart_alert ...;
  ```

#### **3. Optimization**
- [ ] Reuse temporary tables to benefit from caching
- [ ] Use CTEs for intermediate results
- [ ] Avoid frequent warehouse restarts
- [ ] Use multi-cluster warehouses for high concurrency
- [ ] Monitor cache performance
- [ ] Combine with other optimizations
- [ ] Document cacheable data

---
---
## **10. Troubleshooting Guide**

---
### **A. Result Cache Troubleshooting**

#### **Symptom 1: Result Cache Not Working**
**Diagnosis**:
1. Check if result caching is enabled:
   ```sql
   SELECT USE_CACHED_RESULTS FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.CURRENT_SESSION());
   ```

2. Check if query used cached results:
   ```sql
   SELECT used_cached_result FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE query_id = 'your_query_id';
   ```

3. Check for data changes between queries:
   ```sql
   SELECT start_time, query_text FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE query_text LIKE '%your_table%' AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
   ORDER BY start_time;
   ```

**Solutions**:
1. Enable result caching:
   ```sql
   ALTER SESSION SET USE_CACHED_RESULTS = TRUE;
   ```

2. Ensure query text matches exactly (whitespace, case, comments):
   ```sql
   -- These will NOT match:
   SELECT * FROM my_table;
   select * from my_table;

   -- These WILL match:
   SELECT * FROM my_table;
   SELECT * FROM my_table;
   ```

3. Ensure session parameters are consistent:
   ```sql
   ALTER SESSION SET TIMEZONE = 'UTC';
   ALTER SESSION SET DATE_FORMAT = 'YYYY-MM-DD';
   ```

4. Avoid data changes between queries (batch updates):
   ```sql
   -- Bad: Frequent small updates
   UPDATE my_table SET col1 = 1 WHERE id = 1;
   UPDATE my_table SET col1 = 1 WHERE id = 2;

   -- Good: Batch updates
   UPDATE my_table SET col1 = 1 WHERE id IN (1, 2, 3, ...);
   ```

5. Set appropriate TTL:
   ```sql
   ALTER SESSION SET RESULT_CACHE_TTL = 3600;  -- 1 hour
   ```

---

#### **Symptom 2: Low Cache Hit Rate**
**Diagnosis**:
1. Check cache hit rate:
   ```sql
   SELECT
       used_cached_result,
       COUNT(*) AS query_count
   FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   GROUP BY used_cached_result;
   ```

2. Check for unique queries:
   ```sql
   SELECT
       query_text,
       COUNT(*) AS query_count
   FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   GROUP BY query_text
   HAVING COUNT(*) = 1
   ORDER BY query_count DESC;
   ```

**Solutions**:
1. Standardize query formatting:
   ```sql
   -- Bad: Inconsistent formatting
   SELECT * FROM my_table;
   select * from my_table;

   -- Good: Consistent formatting
   SELECT * FROM my_table;
   SELECT * FROM my_table;
   ```

2. Use parameterized queries:
   ```sql
   -- Bad: Dynamic SQL
   String sql = "SELECT * FROM my_table WHERE id = " + userId;

   -- Good: Parameterized query
   String sql = "SELECT * FROM my_table WHERE id = ?";
   PreparedStatement stmt = connection.prepareStatement(sql);
   stmt.setInt(1, userId);
   ```

3. Use materialized views for non-cacheable queries:
   ```sql
   CREATE MATERIALIZED VIEW my_mv AS SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7;
   ```

4. Increase TTL for time-sensitive data:
   ```sql
   ALTER SESSION SET RESULT_CACHE_TTL = 7200;  -- 2 hours
   ```

---

#### **Symptom 3: Cache Invalidation Due to Data Changes**
**Diagnosis**:
1. Check for data changes between queries:
   ```sql
   SELECT start_time, query_text
   FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE query_text LIKE '%INSERT%UPDATE%DELETE%'
     AND query_text LIKE '%my_table%'
     AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
   ORDER BY start_time;
   ```

2. Check for DML on source tables:
   ```sql
   SELECT query_id, query_text, start_time
   FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE (query_text LIKE '%INSERT%my_table%' OR
          query_text LIKE '%UPDATE%my_table%' OR
          query_text LIKE '%DELETE%my_table%')
     AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
   ORDER BY start_time;
   ```

**Solutions**:
1. Batch data changes:
   ```sql
   -- Bad: Frequent small updates
   UPDATE my_table SET col1 = 1 WHERE id = 1;
   UPDATE my_table SET col1 = 1 WHERE id = 2;

   -- Good: Batch updates
   UPDATE my_table SET col1 = 1 WHERE id IN (1, 2, 3, ..., 1000);
   ```

2. Use materialized views for volatile data:
   ```sql
   CREATE MATERIALIZED VIEW my_mv AS SELECT * FROM my_table;
   ```

3. Disable caching for frequently updated tables:
   ```sql
   ALTER SESSION SET USE_CACHED_RESULTS = FALSE;
   ```

4. Use shorter TTL for volatile data:
   ```sql
   ALTER SESSION SET RESULT_CACHE_TTL = 300;  -- 5 minutes
   ```

---
### **B. Metadata Cache Troubleshooting**

#### **Symptom 1: High Compilation Time**
**Diagnosis**:
1. Check compilation time for queries:
   ```sql
   SELECT query_id, query_text, compilation_time
   FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE compilation_time > 500 AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY compilation_time DESC;
   ```

2. Check for DDL changes:
   ```sql
   SELECT query_id, query_text, start_time
   FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE query_text LIKE '%ALTER TABLE%' OR query_text LIKE '%CREATE TABLE%'
     AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY start_time DESC;
   ```

**Solutions**:
1. Update statistics for large tables:
   ```sql
   ALTER TABLE my_table UPDATE STATISTICS;
   ```

2. Update statistics after bulk data loads:
   ```sql
   COPY INTO my_table FROM @my_stage;
   ALTER TABLE my_table UPDATE STATISTICS;
   ```

3. Use clustering for better metadata:
   ```sql
   ALTER TABLE my_table CLUSTER BY (date);
   ```

4. Avoid frequent DDL changes:
   ```sql
   -- Bad: Frequent DDL changes
   ALTER TABLE my_table ADD COLUMN col1 INT;
   ALTER TABLE my_table ADD COLUMN col2 INT;

   -- Good: Batch DDL changes
   ALTER TABLE my_table
   ADD COLUMN col1 INT,
   ADD COLUMN col2 INT;
   ```

---
#### **Symptom 2: No Partition Pruning**
**Diagnosis**:
1. Check if partition pruning is working:
   ```sql
   SELECT query_id, query_text, partitions_scanned
   FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE query_text LIKE '%my_table%' AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY partitions_scanned;
   ```

2. Check if table is clustered:
   ```sql
   SELECT table_name, clustering_information:'CLUSTER_BY'
   FROM INFORMATION_SCHEMA.TABLES
   WHERE table_name = 'MY_TABLE';
   ```

**Solutions**:
1. Cluster the table on filtered columns:
   ```sql
   ALTER TABLE my_table CLUSTER BY (date, region);
   ```

2. Update statistics:
   ```sql
   ALTER TABLE my_table UPDATE STATISTICS;
   ```

3. Use filters on clustered columns:
   ```sql
   -- Bad: No filter on clustered column
   SELECT * FROM my_table;

   -- Good: Filter on clustered column
   SELECT * FROM my_table WHERE date > '2023-01-01';
   ```

---
### **C. Local Disk Cache Troubleshooting**

#### **Symptom 1: Local Disk Cache Not Improving Performance**
**Diagnosis**:
1. Check warehouse size:
   ```sql
   SELECT warehouse_name, warehouse_size
   FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSES
   WHERE warehouse_name = 'MY_WH';
   ```

2. Check query profile for cache hits:
   ```sql
   SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'));
   -- Look for steps with low execution_time and bytes_scanned
   ```

**Solutions**:
1. Use larger warehouses for hot data:
   ```sql
   ALTER WAREHOUSE my_wh SET WAREHOUSE_SIZE = 'X-LARGE';
   ```

2. Query the same data repeatedly to populate cache:
   ```sql
   -- Run the same query multiple times
   SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7;
   SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7;
   SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7;
   ```

3. Use clustering for cache efficiency:
   ```sql
   ALTER TABLE my_table CLUSTER BY (date);
   ```

4. Avoid frequent warehouse restarts:
   ```sql
   -- Bad: Stop and start warehouse
   ALTER WAREHOUSE my_wh SUSPEND;
   ALTER WAREHOUSE my_wh RESUME;

   -- Good: Use auto-suspend
   ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = 1800;
   ```

---
#### **Symptom 2: High I/O Latency**
**Diagnosis**:
1. Check query profile for I/O latency:
   ```sql
   SELECT step_id, operation, execution_time, bytes_scanned
   FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))
   ORDER BY execution_time DESC;
   ```

2. Check warehouse load:
   ```sql
   SELECT warehouse_name, running_queries, queued_queries
   FROM SNOWFLAKE.INFORMATION_SCHEMA.WAREHOUSE_MONITOR
   WHERE warehouse_name = 'MY_WH';
   ```

**Solutions**:
1. Use larger warehouses:
   ```sql
   ALTER WAREHOUSE my_wh SET WAREHOUSE_SIZE = 'X-LARGE';
   ```

2. Use multi-cluster warehouses for high concurrency:
   ```sql
   ALTER WAREHOUSE my_wh SET MAX_CLUSTER_COUNT = 4;
   ```

3. Use clustering for better I/O efficiency:
   ```sql
   ALTER TABLE my_table CLUSTER BY (date);
   ```

4. Use local disk cache for hot data:
   ```sql
   -- Query the same data repeatedly
   SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7;
   ```

---
### **D. Query Cache Troubleshooting**

#### **Symptom 1: High Compilation Time for Parameterized Queries**
**Diagnosis**:
1. Check compilation time for parameterized queries:
   ```sql
   SELECT query_id, query_text, compilation_time
   FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE query_text LIKE '%?%' AND compilation_time > 100
     AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY compilation_time DESC;
   ```

2. Check for dynamic SQL:
   ```sql
   SELECT query_id, query_text, compilation_time
   FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE query_text LIKE '%EXECUTE IMMEDIATE%'
     AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY compilation_time DESC;
   ```

**Solutions**:
1. Use parameterized queries:
   ```sql
   -- Bad: Dynamic SQL
   String sql = "SELECT * FROM my_table WHERE id = " + userId;

   -- Good: Parameterized query
   String sql = "SELECT * FROM my_table WHERE id = ?";
   PreparedStatement stmt = connection.prepareStatement(sql);
   stmt.setInt(1, userId);
   ```

2. Use stored procedures:
   ```sql
   CREATE PROCEDURE get_customer(customer_id INT)
   RETURNS TABLE ()
   AS
   $$
     SELECT * FROM customers WHERE customer_id = customer_id;
   $$;
   ```

3. Use connection pooling:
   ```java
   // Use connection pooling (e.g., HikariCP)
   HikariDataSource dataSource = new HikariDataSource(config);
   ```

4. Avoid frequent DDL changes:
   ```sql
   -- Batch DDL changes
   ALTER TABLE my_table
   ADD COLUMN col1 INT,
   ADD COLUMN col2 INT;
   ```

---
#### **Symptom 2: Query Cache Not Reusing Plans**
**Diagnosis**:
1. Check for schema changes:
   ```sql
   SELECT query_id, query_text, start_time
   FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE query_text LIKE '%ALTER TABLE%'
     AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY start_time DESC;
   ```

2. Check for different session parameters:
   ```sql
   SELECT query_id, query_text, session_parameters
   FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE query_text LIKE '%my_query%'
     AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY start_time DESC;
   ```

**Solutions**:
1. Use consistent session parameters:
   ```sql
   ALTER SESSION SET TIMEZONE = 'UTC';
   ALTER SESSION SET DATE_FORMAT = 'YYYY-MM-DD';
   ```

2. Avoid frequent schema changes:
   ```sql
   -- Batch schema changes
   ALTER TABLE my_table
   ADD COLUMN col1 INT,
   ADD COLUMN col2 INT;
   ```

3. Use stored procedures:
   ```sql
   CREATE PROCEDURE my_proc() AS SELECT * FROM my_table;
   ```

---
### **E. File Metadata Cache Troubleshooting**

#### **Symptom 1: High Compilation Time for External Tables**
**Diagnosis**:
1. Check compilation time for external table queries:
   ```sql
   SELECT query_id, query_text, compilation_time
   FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE query_text LIKE '%my_external_table%' AND compilation_time > 500
     AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY compilation_time DESC;
   ```

2. Check for external table changes:
   ```sql
   SELECT query_id, query_text, start_time
   FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE query_text LIKE '%ALTER EXTERNAL TABLE%'
     AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY start_time DESC;
   ```

**Solutions**:
1. Use partitioned external tables:
   ```sql
   CREATE EXTERNAL TABLE my_table PARTITION BY (date);
   ```

2. Refresh external table metadata:
   ```sql
   ALTER EXTERNAL TABLE my_table REFRESH;
   ```

3. Use consistent stage definitions:
   ```sql
   CREATE STAGE my_stage URL = 's3://my-bucket/';
   ```

4. Use consistent file format definitions:
   ```sql
  
