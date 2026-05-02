# **Snowflake Query Acceleration and Search Optimization: Production-Grade Technical Deep Dive**

---

## **1. Overview of Query Acceleration and Search Optimization**

### **Mermaid: Query Acceleration and Search Optimization Architecture**
```mermaid
%% Query Acceleration and Search Optimization Architecture
flowchart TD
    subgraph Client["Client Layer"]
        A[("Query")] --> B[("Query Router")]
    end

    subgraph QueryAcceleration["Query Acceleration"]
        B --> C[("Query Parser")]
        C --> D[("Query Optimizer")]
        D --> E[("Automatic Query Rewriting")]
        E --> F[("Vectorized Execution")]
        D --> G[("Result Caching")]
        D --> H[("Metadata Caching")]
        D --> I[("Warehouse Optimization")]
    end

    subgraph SearchOptimization["Search Optimization"]
        B --> J[("Search Index Manager")]
        J --> K[("Inverted Index")]
        J --> L[("N-Gram Index")]
        K --> M[("Index Lookup")]
        L --> M
    end

    subgraph Execution["Execution Layer"]
        F --> N[("Query Execution Engine")]
        M --> N
        N --> O[("Virtual Warehouse")]
        O --> P[("Storage Service")]
    end

    subgraph Storage["Storage Layer"]
        P --> Q[("Cloud Storage")]
        K --> R[("Search Index Storage")]
        L --> R
    end

    subgraph Monitoring["Monitoring Layer"]
        S[("QUERY_HISTORY")]
        T[("QUERY_PROFILE")]
        U[("SEARCH_OPTIMIZATION_HISTORY")]
        V[("WAREHOUSE_LOAD_HISTORY")]
    end
    N --> S
    N --> T
    J --> U
    O --> V

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef client fill:#4285f4,stroke:#1976d2;
    classDef queryaccel fill:#ff9800,stroke:#f57c00;
    classDef searchopt fill:#009688,stroke:#00796b;
    classDef execution fill:#29abe2,stroke:#1a8fb8;
    classDef storage fill:#e91e63,stroke:#c2185b;
    classDef monitoring fill:#9c27b0,stroke:#7b1fa2;
    class A client;
    class B,C,D,E,F,G,H,I queryaccel;
    class J,K,L,M searchopt;
    class N,O execution;
    class P,Q,R storage;
    class S,T,U,V monitoring;
```

---

### **Key Concepts**

#### **1. Query Acceleration in Snowflake**
**Query Acceleration** refers to Snowflake's **built-in optimizations** that **automatically improve query performance** without requiring manual intervention. These optimizations leverage Snowflake's **unique architecture** (separation of compute and storage, micro-partitioning, columnar storage) to **accelerate queries at runtime**.

**Core Query Acceleration Techniques**:
- **Automatic Query Rewriting**: Predicate pushdown, column pruning, join reordering, etc.
- **Vectorized Execution**: Process data in batches using SIMD instructions
- **Result Caching**: Cache query results for 24 hours
- **Metadata Caching**: Cache table metadata (statistics, schema)
- **Warehouse Optimization**: Auto-suspend, auto-resume, multi-cluster scaling
- **Partition Pruning**: Skip irrelevant micro-partitions
- **Column Pruning**: Only read required columns

#### **2. Search Optimization Service (SOS)**
**Search Optimization Service** is a **paid add-on service** that **accelerates full-text search and pattern matching queries** by creating and maintaining **search indexes** on your tables. SOS is designed to **dramatically improve the performance** of queries that use:
- **LIKE** operators (e.g., `WHERE column LIKE '%pattern%'`)
- **REGEXP** functions (e.g., `WHERE REGEXP_LIKE(column, 'pattern')`)
- **MATCH** functions (e.g., `WHERE MATCH(column, 'search term')`)
- **CONTAINS** functions (e.g., `WHERE CONTAINS(column, 'term')`)

**Core Search Optimization Techniques**:
- **Inverted Index**: Maps terms to row IDs for fast full-text search
- **N-Gram Index**: Maps n-grams to row IDs for prefix/suffix/wildcard searches
- **Index Lookup**: Quickly find matching rows using search indexes

### **Query Acceleration vs. Search Optimization**

| **Feature** | **Query Acceleration** | **Search Optimization** |
|-------------|-------------------------|--------------------------|
| **Purpose** | Improve performance of all queries | Accelerate full-text search and pattern matching |
| **Scope** | All queries | Queries with LIKE, REGEXP, MATCH, CONTAINS |
| **How It Works** | Automatic query rewriting, vectorized execution, caching | Search indexes (inverted, n-gram) |
| **Configuration** | Automatic (no configuration needed) | Enable per table/column |
| **Cost** | Included in Snowflake pricing | Pay-per-use (storage + compute) |
| **Performance Impact** | 1.5-10x faster | 10-100x faster |
| **Best For** | All queries, especially complex joins and aggregations | Full-text search, pattern matching |
| **Monitoring** | QUERY_HISTORY, QUERY_PROFILE | SEARCH_OPTIMIZATION_HISTORY |
| **Limitations** | Limited by query complexity | Limited to full-text search queries |

### **When to Use Query Acceleration vs. Search Optimization**

| **Use Case** | **Recommended Service** | **Example** |
|-------------|-------------------------|-------------|
| **Complex Joins** | Query Acceleration | `SELECT * FROM t1 JOIN t2 ON t1.id = t2.id JOIN t3 ON t2.id = t3.id` |
| **Aggregations** | Query Acceleration | `SELECT region, SUM(sales) FROM sales GROUP BY region` |
| **Window Functions** | Query Acceleration | `SELECT *, SUM(sales) OVER (PARTITION BY region) FROM sales` |
| **Repetitive Queries** | Query Acceleration (Result Caching) | Dashboard queries run every 5 minutes |
| **Full-Text Search** | Search Optimization | `SELECT * FROM articles WHERE MATCH(content, 'snowflake')` |
| **Pattern Matching** | Search Optimization | `SELECT * FROM logs WHERE message LIKE '%error%'` |
| **Prefix/Suffix Search** | Search Optimization | `SELECT * FROM customers WHERE name LIKE 'John%'` |
| **Exact Match** | Query Acceleration (Clustering) | `SELECT * FROM users WHERE id = 123` |
| **Range Queries** | Query Acceleration (Clustering) | `SELECT * FROM sales WHERE date BETWEEN '2023-01-01' AND '2023-01-31'` |

## **2. Query Acceleration Deep Dive**

### **A. Automatic Query Rewriting**

#### **1. Definition and Architecture**
**Automatic Query Rewriting** is Snowflake's **built-in optimization feature** that **automatically transforms queries** into **more efficient forms** during query compilation. This happens **transparently** and requires **no manual intervention**.

```mermaid
%% Automatic Query Rewriting Architecture
flowchart TD
    subgraph Input["Input Layer"]
        A[("Original Query")] --> B[("Query Parser")]
    end

    subgraph Rewriting["Rewriting Layer"]
        B --> C[("Query Rewriter")]
        C --> D[("Predicate Pushdown")]
        C --> E[("Column Pruning")]
        C --> F[("Partition Pruning")]
        C --> G[("Join Reordering")]
        C --> H[("Common Subexpression Elimination")]
        C --> I[("Constant Folding")]
        C --> J[("Query Folding")]
    end

    subgraph Output["Output Layer"]
        C --> K[("Optimized Query")]
    end

    subgraph Execution["Execution Layer"]
        K --> L[("Query Execution Engine")]
    end

    subgraph Monitoring["Monitoring Layer"]
        M[("QUERY_PROFILE")]
    end
    L --> M

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef input fill:#4285f4,stroke:#1976d2;
    classDef rewriting fill:#ff9800,stroke:#f57c00;
    classDef output fill:#009688,stroke:#00796b;
    classDef execution fill:#29abe2,stroke:#1a8fb8;
    classDef monitoring fill:#9c27b0,stroke:#7b1fa2;
    class A input;
    class B,C,D,E,F,G,H,I,J rewriting;
    class K output;
    class L execution;
    class M monitoring;
```

#### **2. Query Rewriting Techniques**

| **Technique** | **Description** | **Example Before** | **Example After** | **Performance Impact** | **When It Works Best** |
|---------------|-----------------|--------------------|-------------------|-------------------------|------------------------|
| **Predicate Pushdown** | Moves filters closer to the data source | `SELECT * FROM (SELECT * FROM my_table) WHERE date > '2023-01-01'` | `SELECT * FROM my_table WHERE date > '2023-01-01'` | ⬆️ 2-10x faster | Queries with filters on large tables |
| **Column Pruning** | Removes unreferenced columns from scans | `SELECT col1, col2 FROM my_table` | Only reads col1, col2 | ⬆️ 1.5-5x faster | Queries with many columns but only a few referenced |
| **Partition Pruning** | Skips irrelevant micro-partitions | `SELECT * FROM my_table WHERE date = '2023-01-01'` | Only scans partitions with date = '2023-01-01' | ⬆️ 10-100x faster | Queries with filters on clustered columns |
| **Join Reordering** | Reorders joins to minimize intermediate results | `SELECT * FROM large_table JOIN small_table ON ...` | `SELECT * FROM small_table JOIN large_table ON ...` | ⬆️ 2-10x faster | Queries with multiple joins |
| **Common Subexpression Elimination (CSE)** | Reuses subqueries or expressions | `SELECT col1 + col2, col1 + col2 * 2 FROM my_table` | `SELECT t1, t1 * 2 FROM (SELECT col1 + col2 AS t1 FROM my_table)` | ⬆️ 1.1-2x faster | Queries with repeated expressions |
| **Constant Folding** | Evaluates constant expressions at compile time | `SELECT * FROM my_table WHERE col1 > 1 + 2` | `SELECT * FROM my_table WHERE col1 > 3` | ⬆️ 1.01-1.1x faster | Queries with constant expressions |
| **Query Folding** | Combines nested views or subqueries into a single query | `SELECT * FROM (SELECT * FROM my_view) WHERE col1 > 1` | `SELECT * FROM my_table WHERE col1 > 1` (if my_view is a simple view) | ⬆️ 2-10x faster | Queries with nested views or subqueries |
| **Subquery Unnesting** | Converts subqueries into joins | `SELECT * FROM t1 WHERE id IN (SELECT id FROM t2)` | `SELECT * FROM t1 JOIN t2 ON t1.id = t2.id` | ⬆️ 2-10x faster | Queries with IN or EXISTS subqueries |
| **Join Elimination** | Removes unnecessary joins | `SELECT * FROM t1 JOIN t2 ON t1.id = t2.id WHERE t2.col IS NULL` | `SELECT * FROM t1 WHERE NOT EXISTS (SELECT 1 FROM t2 WHERE t2.id = t1.id)` | ⬆️ 2-5x faster | Queries with joins that can be eliminated |

#### **3. How Automatic Query Rewriting Works**
1. **Query Parsing**:
   - The **query parser** validates the SQL syntax and semantics.

2. **Query Rewriting**:
   - The **query rewriter** applies a series of **optimization rules** to transform the query:
     - **Predicate Pushdown**: Moves `WHERE` clauses closer to the data source.
     - **Column Pruning**: Removes unreferenced columns from scans.
     - **Partition Pruning**: Skips irrelevant micro-partitions based on clustering keys.
     - **Join Reordering**: Reorders joins to minimize intermediate results.
     - **Common Subexpression Elimination (CSE)**: Reuses subqueries or expressions.
     - **Constant Folding**: Evaluates constant expressions at compile time.
     - **Query Folding**: Combines nested views or subqueries into a single query.
     - **Subquery Unnesting**: Converts subqueries into joins.
     - **Join Elimination**: Removes unnecessary joins.

3. **Query Compilation**:
   - The **optimized query** is compiled into an **executable plan**.

4. **Query Execution**:
   - The **query execution engine** executes the optimized plan.

#### **4. Automatic Query Rewriting Configuration**
**Note**: Automatic query rewriting is **enabled by default** and **requires no configuration**. However, you can **influence** the rewriting process by:
- **Using clustering** to enable partition pruning.
- **Using proper data types** to enable column pruning.
- **Writing efficient queries** to enable other optimizations.

**Check if Query Rewriting is Working**:
```sql
-- Check the query plan for rewriting
EXPLAIN SELECT * FROM my_table WHERE date > '2023-01-01';

-- Check for partition pruning in the plan
-- Look for "filters" in TableScan operators
```

#### **5. Automatic Query Rewriting Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use Clustering for Partition Pruning** | Cluster tables on frequently filtered columns | `ALTER TABLE my_table CLUSTER BY (date)` |
| **Use Proper Data Types** | Use appropriate data types for column pruning | `INTEGER` instead of `VARCHAR` for IDs |
| **Avoid Unnecessary Subqueries** | Use joins or CTEs instead of nested subqueries | `WITH cte AS (SELECT * FROM t1) SELECT * FROM cte` |
| **Use Simple Queries** | Complex queries are harder to optimize | `SELECT col1, col2 FROM my_table WHERE col3 > 1` |
| **Avoid Functions on Filtered Columns** | Functions prevent predicate pushdown | `SELECT * FROM my_table WHERE col1 = 'value'` (not `WHERE UPPER(col1) = 'VALUE'`) |
| **Use Parameterized Queries** | Parameterized queries enable plan caching | `SELECT * FROM my_table WHERE id = ?` |
| **Monitor Query Plans** | Check EXPLAIN for rewriting | `EXPLAIN SELECT * FROM my_table WHERE ...` |
| **Test Query Performance** | Compare performance before and after changes | Run queries and compare execution times |
| **Document Query Patterns** | Document repetitive query patterns for optimization | Internal wiki or Confluence page |
| **Use Views for Complex Queries** | Views can be folded into the main query | `CREATE VIEW my_view AS SELECT * FROM my_table WHERE ...` |

### **B. Vectorized Execution**

#### **1. Definition and Architecture**
**Vectorized Execution** is a **query execution technique** that processes data in **batches** (vectors) rather than row-by-row. This enables **high throughput** and **low CPU usage** by leveraging **SIMD (Single Instruction, Multiple Data)** instructions.

```mermaid
%% Vectorized Execution Architecture
flowchart TD
    subgraph Input["Input Layer"]
        A[("Micro-Partition")] --> B[("Vector Creation")]
    end

    subgraph Execution["Execution Layer"]
        B --> C[("Vector Processing")]
        C --> D[("SIMD Instructions")]
        D --> E[("Result Materialization")]
    end

    subgraph Output["Output Layer"]
        E --> F[("Result Set")]
    end

    subgraph Details["Details"]
        C --> G[("Batch Size: 1000-10000 rows")]
        D --> H[("CPU: SIMD Instructions")]
        D --> I[("Memory: Vector Registers")]
    end

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef input fill:#4285f4,stroke:#1976d2;
    classDef execution fill:#ff9800,stroke:#f57c00;
    classDef output fill:#009688,stroke:#00796b;
    classDef details fill:#29abe2,stroke:#1a8fb8;
    class A input;
    class B,C,D,E execution;
    class F output;
    class G,H,I details;
```

#### **2. How Vectorized Execution Works**
1. **Vector Creation**:
   - Snowflake **reads data in batches** (vectors) from micro-partitions.
   - **Batch size**: Typically **1000-10000 rows** (configurable).

2. **Vector Processing**:
   - **SIMD instructions** process the entire vector in a **single CPU instruction**.
   - Example: Adding 1000 values in a single instruction.

3. **Result Materialization**:
   - Results are **materialized** and returned to the client.

#### **3. Vectorized Execution Performance**

| **Metric** | **Row-by-Row Execution** | **Vectorized Execution** | **Improvement** | **Notes** |
|------------|--------------------------|---------------------------|-----------------|-----------|
| **Throughput** | 100-1000 rows/sec | 10,000-1,000,000 rows/sec | 10-100x faster | Depends on batch size and CPU |
| **CPU Usage** | High | Low | 2-10x reduction | Fewer instructions per row |
| **Memory Usage** | Low | Medium | 1.1-2x increase | Vector registers consume memory |
| **Latency** | Low | Low | No significant change | Batch processing adds minimal latency |

#### **4. Vectorized Execution Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use Simple Expressions** | Avoid complex expressions in vectors | `SELECT col1 + col2 FROM my_table` (instead of `SELECT complex_function(col1) FROM my_table`) |
| **Use Built-in Functions** | Built-in functions are optimized for vectorization | `SELECT SUM(col1) FROM my_table` |
| **Avoid UDFs** | UDFs disable vectorization | Use built-in functions instead of JavaScript/Python UDFs |
| **Use Approximate Functions** | Approximate functions are vectorized | `SELECT APPROX_COUNT_DISTINCT(col1) FROM my_table` |
| **Monitor Vectorization** | Check QUERY_PROFILE for vectorized operators | `SELECT operation, rows_produced FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('...'))` |

### **C. Result Caching**

#### **1. Definition and Architecture**
**Result Caching** in Snowflake **automatically caches query results** for **24 hours** (configurable) if:
- The **query text** is identical.
- The **underlying data** has not changed.
- The **user's permissions** are the same.

```mermaid
%% Result Caching Architecture
flowchart TD
    subgraph Query["Query Layer"]
        A[("Query")] --> B[("Cache Lookup")]
    end

    subgraph Cache["Cache Layer"]
        B --> C{Cache Hit?}
        C -->|Yes| D[("Return Cached Results")]
        C -->|No| E[("Execute Query")]
        E --> F[("Cache Results")]
    end

    subgraph Execution["Execution Layer"]
        E --> G[("Query Execution Engine")]
    end

    subgraph Storage["Storage Layer"]
        F --> H[("Result Cache\n(SSD)")]
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

#### **2. How Result Caching Works**
1. **Cache Lookup**:
   - When a query is submitted, Snowflake **checks the result cache** for a matching entry.
   - A **cache hit** occurs if:
     - The **query text** is identical (including whitespace and case).
     - The **underlying data** has not changed since the result was cached.
     - The **user's permissions** are the same.
     - The **session parameters** (e.g., time zone, role) are the same.

2. **Cache Hit**:
   - If a cache hit occurs, Snowflake **returns the cached results** immediately.
   - The **execution_time** in `QUERY_HISTORY` will be **very low** (typically <10ms).

3. **Cache Miss**:
   - If no cache hit occurs, Snowflake **executes the query** normally.
   - After execution, Snowflake **caches the results** for future use.

4. **Cache Invalidation**:
   - The cache is **automatically invalidated** if:
     - The **underlying data changes** (e.g., INSERT, UPDATE, DELETE on source tables).
     - The **query text changes**.
     - The **user's permissions change**.
     - The **cache TTL expires** (default: 24 hours).

#### **3. Result Cache Types**

| **Cache Type** | **Description** | **TTL** | **Scope** | **Cost** |
|----------------|-----------------|---------|-----------|----------|
| **Result Cache** | Caches query results | 24 hours (configurable) | Per-user, per-query | Included in compute |
| **Metadata Cache** | Caches table metadata (statistics, schema) | Session | Per-session | Included in compute |
| **Local Disk Cache** | Caches frequently accessed data in SSD | Session | Per-warehouse | Included in compute |

#### **4. Result Caching Configuration**

| **Parameter** | **Description** | **Default** | **Valid Values** | **Example** |
|---------------|-----------------|-------------|------------------|-------------|
| **USE_CACHED_RESULTS** | Enable/disable result caching | `TRUE` | `TRUE`, `FALSE` | `ALTER SESSION SET USE_CACHED_RESULTS = TRUE` |
| **RESULT_CACHE_TTL** | Result cache TTL (in seconds) | `86400` (24 hours) | 0-86400 | `ALTER SESSION SET RESULT_CACHE_TTL = 3600` (1 hour) |

**Configuration Examples**:
```sql
-- Enable result caching (default)
ALTER SESSION SET USE_CACHED_RESULTS = TRUE;

-- Disable result caching
ALTER SESSION SET USE_CACHED_RESULTS = FALSE;

-- Set cache TTL to 1 hour
ALTER SESSION SET RESULT_CACHE_TTL = 3600;
```

#### **5. Result Caching Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use for Repetitive Queries** | Cache results for queries run frequently | Dashboards, reports |
| **Set Appropriate TTL** | Adjust TTL based on data freshness requirements | `ALTER SESSION SET RESULT_CACHE_TTL = 3600` (1 hour) |
| **Monitor Cache Usage** | Check used_cached_result in QUERY_HISTORY | `SELECT used_cached_result FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` |
| **Avoid Cache Invalidation** | Minimize changes to underlying data | Batch updates instead of frequent small updates |
| **Use for Read-Only Workloads** | Cache works best for read-only workloads | BI tools, reporting |
| **Disable for Unique Queries** | Disable caching for unique queries | `ALTER SESSION SET USE_CACHED_RESULTS = FALSE` |
| **Use Consistent Query Text** | Ensure query text is identical for cache hits | Avoid dynamic SQL, use parameterized queries |
| **Use Consistent Session Parameters** | Ensure session parameters (e.g., time zone) are consistent | Set session parameters explicitly |

### **D. Metadata Caching**

#### **1. Definition and Architecture**
**Metadata Caching** in Snowflake **caches table metadata** (e.g., statistics, schema, partition information) to **improve query performance**. Metadata caching is **automatic** and **transparent to users**.

```mermaid
%% Metadata Caching Architecture
flowchart TD
    subgraph Query["Query Layer"]
        A[("Query")] --> B[("Metadata Lookup")]
    end

    subgraph Cache["Cache Layer"]
        B --> C{Cache Hit?}
        C -->|Yes| D[("Return Cached Metadata")]
        C -->|No| E[("Fetch Metadata from Storage")]
        E --> F[("Cache Metadata")]
    end

    subgraph Storage["Storage Layer"]
        E --> G[("Metadata Service")]
        G --> H[("Cloud Storage")]
    end

    subgraph Execution["Execution Layer"]
        D --> I[("Query Execution Engine")]
        F --> I
    end

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef query fill:#4285f4,stroke:#1976d2;
    classDef cache fill:#ff9800,stroke:#f57c00;
    classDef storage fill:#29abe2,stroke:#1a8fb8;
    classDef execution fill:#009688,stroke:#00796b;
    class A query;
    class B,C,D,E,F cache;
    class G,H storage;
    class I execution;
```

#### **2. How Metadata Caching Works**
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
     - The **cache TTL expires** (session duration).

#### **3. Metadata Cached by Snowflake**

| **Metadata Type** | **Description** | **Usage** | **Cache Duration** |
|-------------------|-----------------|-----------|-------------------|
| **Table Statistics** | Row count, byte count, partition count | Query optimization, partition pruning | Session |
| **Column Statistics** | Min/max values, histograms, distinct counts | Query optimization, predicate pushdown | Session |
| **Schema Information** | Table schema, column data types | Query validation, optimization | Session |
| **Partition Information** | Micro-partition metadata (min/max values) | Partition pruning | Session |
| **Clustering Information** | Clustering keys, clustering depth | Partition pruning | Session |
| **Storage Information** | Storage usage, file formats | Query optimization | Session |

#### **4. Metadata Caching Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Update Statistics for Large Tables** | Ensure statistics are up-to-date for large tables | `ALTER TABLE my_table UPDATE STATISTICS` |
| **Update Statistics After Data Changes** | Update statistics after significant data changes | `ALTER TABLE my_table UPDATE STATISTICS` after bulk loads |
| **Avoid Frequent DDL Changes** | DDL changes invalidate metadata cache | Batch DDL changes |
| **Use Clustering for Better Metadata** | Clustering improves partition pruning | `ALTER TABLE my_table CLUSTER BY (date)` |
| **Monitor Metadata Cache Hits** | Check metadata cache usage (not directly visible) | Improved query performance |

### **E. Warehouse Optimization**

#### **1. Auto-Suspend and Auto-Resume**
- **Auto-Suspend**: Warehouse **automatically suspends** after a period of **inactivity** (default: **10 minutes**).
- **Auto-Resume**: Warehouse **automatically resumes** when a new query is submitted.

**Best Practices**:
| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Set Appropriate Auto-Suspend Times** | Configure based on workload patterns | `ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = 300` (5 minutes) |
| **Use Shorter Times for Development** | Development warehouses can be suspended more aggressively | `ALTER WAREHOUSE dev_wh SET AUTO_SUSPEND = 300` |
| **Use Longer Times for Production** | Production warehouses should have longer suspend times | `ALTER WAREHOUSE prod_wh SET AUTO_SUSPEND = 1800` (30 minutes) |
| **Disable for Always-On Warehouses** | Disable auto-suspend for 24/7 workloads | `ALTER WAREHOUSE always_on_wh SET AUTO_SUSPEND = NULL` |
| **Avoid Frequent Suspend/Resume Cycles** | Frequent cycles can negate cost savings | Monitor warehouse usage patterns |

**Configuration**:
```sql
-- Set auto-suspend time (in seconds)
ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = 600;  -- 10 minutes

-- Disable auto-suspend
ALTER WAREHOUSE always_on_wh SET AUTO_SUSPEND = NULL;

-- Enable auto-resume
ALTER WAREHOUSE my_wh SET AUTO_RESUME = TRUE;

-- Disable auto-resume
ALTER WAREHOUSE batch_wh SET AUTO_RESUME = FALSE;
```

#### **2. Multi-Cluster Warehouses**
- **Definition**: Multi-cluster warehouses can **scale out** to handle **high concurrency** by adding **additional clusters**.

**Best Practices**:
| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Start with MAX_CLUSTER_COUNT = 2** | Test with 2 clusters before scaling up | `CREATE WAREHOUSE my_wh MAX_CLUSTER_COUNT = 2` |
| **Use STANDARD Scaling for Critical Workloads** | Ensure performance SLAs are met | `ALTER WAREHOUSE my_wh SET SCALING_POLICY = 'STANDARD'` |
| **Use ECONOMY Scaling for Cost-Sensitive Workloads** | Save costs for non-critical workloads | `ALTER WAREHOUSE my_wh SET SCALING_POLICY = 'ECONOMY'` |
| **Set MIN_CLUSTER_COUNT = 1** | Avoid idle clusters to save costs | `ALTER WAREHOUSE my_wh SET MIN_CLUSTER_COUNT = 1` |
| **Monitor Cluster Usage** | Check WAREHOUSE_LOAD_HISTORY for cluster usage | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY` |

**Configuration**:
```sql
-- Create a multi-cluster warehouse with STANDARD scaling
CREATE WAREHOUSE my_mc_wh
  WAREHOUSE_SIZE = 'LARGE'
  MAX_CLUSTER_COUNT = 4
  MIN_CLUSTER_COUNT = 1
  SCALING_POLICY = 'STANDARD'
  AUTO_SUSPEND = 1800
  AUTO_RESUME = TRUE;
```

#### **3. Query Prioritization**
- **Definition**: Assign **priority levels** (HIGH, MEDIUM, LOW) to queries to **manage resource contention**.

**Best Practices**:
| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use HIGH Priority for Critical Queries** | Prioritize SLA-bound queries | `ALTER WAREHOUSE prod_wh SET QUERY_PRIORITY = 'HIGH'` |
| **Use MEDIUM Priority for General Queries** | Default priority for most queries | Default |
| **Use LOW Priority for Ad-Hoc Queries** | Deprioritize development and ad-hoc queries | `ALTER SESSION SET QUERY_PRIORITY = 'LOW'` |
| **Set Query Timeouts** | Prevent long-running queries from blocking others | `ALTER WAREHOUSE my_wh SET STATEMENT_TIMEOUT_IN_SECONDS = 300` |
| **Set Queue Timeouts** | Prevent queries from waiting too long in the queue | `ALTER WAREHOUSE my_wh SET STATEMENT_QUEUE_TIMEOUT_IN_SECONDS = 60` |

**Configuration**:
```sql
-- Set default priority for a warehouse
ALTER WAREHOUSE my_wh SET QUERY_PRIORITY = 'HIGH';

-- Set priority for the current session
ALTER SESSION SET QUERY_PRIORITY = 'HIGH';

-- Set priority for a specific query (using hint)
SELECT * FROM my_table /*+ PRIORITY(HIGH) */;
```

## **3. Search Optimization Service (SOS) Deep Dive**

### **A. Definition and Architecture**

The **Search Optimization Service (SOS)** is a **paid add-on service** that **accelerates full-text search and pattern matching queries** by creating and maintaining **search indexes** on your tables. SOS is designed to **dramatically improve the performance** of queries that use:
- **LIKE** operators (e.g., `WHERE column LIKE '%pattern%'`)
- **REGEXP** functions (e.g., `WHERE REGEXP_LIKE(column, 'pattern')`)
- **MATCH** functions (e.g., `WHERE MATCH(column, 'search term')`)
- **CONTAINS** functions (e.g., `WHERE CONTAINS(column, 'term')`)

```mermaid
%% Search Optimization Service Architecture
flowchart TD
    subgraph Query["Query Layer"]
        A[("Full-Text Search Query")] --> B[("Query Parser")]
    end

    subgraph SOS["Search Optimization Service"]
        B --> C[("Search Index Manager")]
        C --> D[("Search Index\n(Inverted Index)")]
        C --> E[("Search Index\n(N-Gram Index)")]
        D --> F[("Index Lookup")]
        E --> F
    end

    subgraph Execution["Execution Layer"]
        F --> G[("Query Execution Engine")]
        G --> H[("Table Scan")]
    end

    subgraph Storage["Storage Layer"]
        H --> I[("Cloud Storage")]
        D --> J[("Search Index Storage")]
        E --> J
    end

    subgraph Monitoring["Monitoring Layer"]
        K[("SEARCH_OPTIMIZATION_HISTORY")]
    end
    C --> K
    D --> K
    E --> K

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef query fill:#4285f4,stroke:#1976d2;
    classDef sos fill:#ff9800,stroke:#f57c00;
    classDef execution fill:#009688,stroke:#00796b;
    classDef storage fill:#29abe2,stroke:#1a8fb8;
    classDef monitoring fill:#9c27b0,stroke:#7b1fa2;
    class A query;
    class B,C,D,E,F sos;
    class G,H execution;
    class I,J storage;
    class K monitoring;
```

#### **1. How Search Optimization Service Works**
1. **Index Creation**:
   - When you enable **Search Optimization** on a table, Snowflake creates a **search index** (inverted index or n-gram index) on the specified columns.
   - The index is **stored separately** from the table data in **cloud storage**.

2. **Index Maintenance**:
   - The index is **automatically updated** when data in the table changes (INSERT, UPDATE, DELETE).
   - Index maintenance is **handled by Snowflake** (no manual intervention required).

3. **Query Execution**:
   - When a **full-text search query** is executed, Snowflake:
     - **Checks if a search index** exists for the table and columns in the query.
     - **Uses the index** to quickly find matching rows (instead of scanning the entire table).
     - **Returns the matching rows** to the query execution engine.
   - If no index exists, Snowflake **falls back to a full table scan**.

4. **Index Types**:
   - **Inverted Index**: Maps **terms** (words, phrases) to **row IDs** (micro-partition IDs). Ideal for **full-text search**.
   - **N-Gram Index**: Maps **n-grams** (substrings of length n) to **row IDs**. Ideal for **prefix/suffix/wildcard searches**.

#### **2. When to Use Search Optimization Service**

**Good Candidates for SOS**:
✅ **Full-text search queries** (e.g., `WHERE column LIKE '%search term%'`).
✅ **Pattern matching queries** (e.g., `WHERE REGEXP_LIKE(column, 'pattern')`).
✅ **Large tables** (>1TB) with frequent full-text search queries.
✅ **Queries with high bytes_scanned** due to LIKE/REGEXP operators.
✅ **Queries with poor performance** despite clustering and filtering.
✅ **Applications requiring sub-second search response times**.

**Poor Candidates for SOS**:
❌ **Simple equality queries** (e.g., `WHERE column = 'value'`; use clustering instead).
❌ **Exact match queries** (e.g., `WHERE column = 'exact_value'`; use clustering instead).
❌ **Small tables** (<1GB; the overhead of index maintenance may outweigh benefits).
❌ **Ad-hoc queries** (no repetitive patterns to optimize for).
❌ **Write-heavy tables** (frequent updates may impact index maintenance performance).
❌ **Cost-sensitive environments** (SOS has additional storage and compute costs).

### **B. Search Optimization Service Configuration**

#### **1. Enable Search Optimization on a Table**
```sql
-- Enable Search Optimization on a table (default: INVERTED index)
ALTER TABLE my_table SET SEARCH_OPTIMIZATION = TRUE;

-- Enable Search Optimization on specific columns
ALTER TABLE my_table SET SEARCH_OPTIMIZATION = TRUE
  WITH (COLUMNS = (col1, col2, col3));

-- Enable Search Optimization with NGRAM index for prefix/suffix searches
ALTER TABLE my_table SET SEARCH_OPTIMIZATION = TRUE
  WITH (COLUMNS = (col1, col2), INDEX_TYPE = 'NGRAM');
```

#### **2. Disable Search Optimization on a Table**
```sql
-- Disable Search Optimization on a table
ALTER TABLE my_table SET SEARCH_OPTIMIZATION = FALSE;
```

#### **3. Check Search Optimization Status**
```sql
-- Check Search Optimization status for a table
SELECT
    table_name,
    search_optimization,
    search_optimization_columns,
    search_optimization_index_type,
    search_optimization_last_indexed,
    search_optimization_storage_bytes
FROM
    INFORMATION_SCHEMA.TABLES
WHERE
    table_name = 'MY_TABLE';

-- Check Search Optimization status for all tables
SELECT
    table_name,
    schema_name,
    database_name,
    search_optimization,
    search_optimization_columns,
    search_optimization_index_type,
    search_optimization_storage_bytes / 1024 / 1024 AS storage_mb
FROM
    INFORMATION_SCHEMA.TABLES
WHERE
    search_optimization = TRUE
ORDER BY
    storage_mb DESC;
```

#### **4. Check Search Optimization History**
```sql
-- Check Search Optimization history for a table
SELECT
    table_name,
    action,
    status,
    start_time,
    end_time,
    rows_processed,
    bytes_processed,
    credits_used,
    error_message
FROM
    SNOWFLAKE.ACCOUNT_USAGE.SEARCH_OPTIMIZATION_HISTORY
WHERE
    table_name = 'MY_TABLE'
ORDER BY
    start_time DESC;

-- Check Search Optimization history for all tables
SELECT
    table_name,
    action,
    status,
    start_time,
    end_time,
    rows_processed,
    bytes_processed,
    credits_used
FROM
    SNOWFLAKE.ACCOUNT_USAGE.SEARCH_OPTIMIZATION_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;
```

### **C. Search Optimization Service Performance**

#### **1. Performance Impact**
| **Query Type** | **Without SOS** | **With SOS** | **Improvement** | **Notes** |
|----------------|-----------------|--------------|-----------------|-----------|
| **LIKE '%term%'** | Full table scan | Index lookup | 10-100x faster | Depends on selectivity |
| **LIKE 'term%'** | Partial scan | Index lookup | 10-50x faster | Prefix search |
| **LIKE '%term'** | Full table scan | Index lookup | 2-10x faster | Suffix search (NGRAM index) |
| **REGEXP_LIKE** | Full table scan | Index lookup | 10-100x faster | Depends on regex complexity |
| **MATCH** | Full table scan | Index lookup | 10-100x faster | Full-text search |
| **CONTAINS** | Full table scan | Index lookup | 10-100x faster | Full-text search |
| **Exact Match** | Full table scan | Full table scan | No improvement | Use clustering instead |

#### **2. Performance Example**
**Query**:
```sql
-- Full-text search query
SELECT * FROM my_table
WHERE description LIKE '%snowflake%';
```

**Performance Comparison**:
| **Metric** | **Without SOS** | **With SOS (INVERTED)** | **With SOS (NGRAM)** | **Improvement** |
|------------|-----------------|-------------------------|----------------------|-----------------|
| **Execution Time** | 30 seconds | 300 ms | 500 ms | 60-100x faster |
| **Bytes Scanned** | 100 GB | 100 MB | 200 MB | 500-1000x reduction |
| **Credit Usage** | 50 credits | 0.5 credits | 0.7 credits | 70-100x reduction |
| **Partitions Scanned** | 1000 | 10 | 20 | 50-100x reduction |

### **D. Search Optimization Service Cost**

#### **1. Cost Model**
The **Search Optimization Service** has the following costs:
1. **Storage Cost**:
   - Search indexes are stored in **cloud storage** (S3, Azure Blob, GCS).
   - **Cost**: Same as your cloud storage provider's rates (e.g., $0.023/GB/month for S3 Standard).
   - **Size**: Typically **10-30% of the table size** (depends on data cardinality).

2. **Compute Cost**:
   - Index **creation** and **maintenance** consume **Snowflake credits**.
   - **Cost**: Based on the **warehouse size** used for index operations.
   - **Typical Cost**: **0.1-1% of query compute costs** (varies based on data changes).

3. **Query Cost**:
   - Queries using search indexes **consume credits** like regular queries.
   - **Cost**: Typically **lower** than full table scans due to reduced bytes_scanned.

#### **2. Cost Monitoring**
```sql
-- Check Search Optimization storage usage
SELECT
    table_name,
    schema_name,
    database_name,
    search_optimization_storage_bytes / 1024 / 1024 / 1024 AS storage_gb,
    search_optimization_storage_bytes * 0.023 / 1024 / 1024 / 1024 AS estimated_monthly_cost_usd
FROM
    INFORMATION_SCHEMA.TABLES
WHERE
    search_optimization = TRUE
ORDER BY
    storage_gb DESC;

-- Check Search Optimization compute usage
SELECT
    table_name,
    action,
    SUM(credits_used) AS total_credits_used,
    SUM(credits_used) * 0.028 AS estimated_cost_usd
FROM
    SNOWFLAKE.ACCOUNT_USAGE.SEARCH_OPTIMIZATION_HISTORY
WHERE
    start_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY
    table_name, action
ORDER BY
    total_credits_used DESC;
```

### **E. Search Optimization Service Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Enable on Large Tables** | SOS is most effective for large tables (>1TB) | `ALTER TABLE large_table SET SEARCH_OPTIMIZATION = TRUE` |
| **Enable on Frequently Searched Columns** | Only enable SOS on columns used in LIKE/REGEXP queries | `ALTER TABLE my_table SET SEARCH_OPTIMIZATION = TRUE WITH (COLUMNS = (description, notes))` |
| **Use INVERTED Index for Full-Text Search** | INVERTED index is ideal for full-text search | `ALTER TABLE my_table SET SEARCH_OPTIMIZATION = TRUE WITH (INDEX_TYPE = 'INVERTED')` |
| **Use NGRAM Index for Prefix/Suffix Search** | NGRAM index is ideal for prefix/suffix/wildcard searches | `ALTER TABLE my_table SET SEARCH_OPTIMIZATION = TRUE WITH (INDEX_TYPE = 'NGRAM')` |
| **Combine with Clustering** | Combine SOS with clustering for better performance | `ALTER TABLE my_table CLUSTER BY (date) SET SEARCH_OPTIMIZATION = TRUE` |
| **Monitor Index Size** | Check index size to ensure it's not excessive | `SELECT search_optimization_storage_bytes FROM INFORMATION_SCHEMA.TABLES` |
| **Monitor Index Maintenance** | Check SEARCH_OPTIMIZATION_HISTORY for maintenance costs | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.SEARCH_OPTIMIZATION_HISTORY` |
| **Use for Full-Text Search** | SOS is ideal for full-text search queries | `SELECT * FROM my_table WHERE MATCH(description, 'snowflake')` |
| **Use for Pattern Matching** | SOS accelerates REGEXP and LIKE queries | `SELECT * FROM my_table WHERE description LIKE '%snowflake%'` |
| **Avoid on Small Tables** | SOS may not be worth it for small tables (<1GB) | Avoid enabling SOS on small tables |
| **Avoid on Write-Heavy Tables** | Frequent updates may impact index maintenance performance | Avoid enabling SOS on tables with >1000 updates/sec |
| **Test Before Enabling** | Test SOS on a subset of data before enabling on production | Create a test table and enable SOS |
| **Disable When Not Needed** | Disable SOS when no longer needed to save costs | `ALTER TABLE my_table SET SEARCH_OPTIMIZATION = FALSE` |
| **Use INDEX_TYPE for Specific Needs** | Use INVERTED for full-text, NGRAM for prefix/suffix | `ALTER TABLE my_table SET SEARCH_OPTIMIZATION = TRUE WITH (INDEX_TYPE = 'INVERTED')` |

### **F. Search Optimization Service Limitations**

| **Limitation** | **Description** | **Workaround** |
|----------------|-----------------|---------------|
| **Paid Service** | SOS requires additional licensing | Contact Snowflake sales to enable |
| **Storage Costs** | Search indexes consume additional storage | Monitor index size, disable when not needed |
| **Compute Costs** | Index maintenance consumes credits | Monitor maintenance costs, optimize update frequency |
| **Index Creation Time** | Initial index creation can take time for large tables | Create indexes during off-peak hours |
| **Index Maintenance Overhead** | Frequent updates can impact performance | Avoid SOS on write-heavy tables |
| **No Partial Indexes** | Cannot create indexes on a subset of rows | Use filtered tables or views |
| **No Custom Analyzers** | Cannot customize text analysis (e.g., stemming, stop words) | Pre-process text before inserting into Snowflake |
| **No Synonym Support** | Does not support synonyms in search | Use OR conditions in queries |
| **No Fuzzy Search** | Does not support fuzzy search (e.g., typos) | Use REGEXP or application-level fuzzy search |
| **No Phrase Search** | Limited phrase search support | Use exact match or REGEXP |
| **No Wildcard at Start** | LIKE '%term' is slower than LIKE 'term%' | Use NGRAM index type for prefix/suffix searches |
| **No Support for All Data Types** | Some data types (e.g., VARIANT, GEOGRAPHY) are not supported | Use supported data types (STRING, VARCHAR, etc.) |

### **G. Search Optimization Service Examples**

#### **Example 1: Full-Text Search on Product Descriptions**
```sql
-- Enable Search Optimization on the products table with INVERTED index
ALTER TABLE products SET SEARCH_OPTIMIZATION = TRUE
  WITH (COLUMNS = (description, name), INDEX_TYPE = 'INVERTED');

-- Query with full-text search using MATCH
SELECT
    product_id,
    name,
    description,
    price
FROM
    products
WHERE
    MATCH(description, 'snowflake data warehouse')
    OR MATCH(name, 'snowflake')
ORDER BY
    price DESC
LIMIT 100;

-- Query with full-text search using LIKE
SELECT
    product_id,
    name,
    description
FROM
    products
WHERE
    description LIKE '%snowflake%'
    OR name LIKE '%snowflake%'
LIMIT 100;
```

#### **Example 2: Pattern Matching on Log Data**
```sql
-- Enable Search Optimization on the logs table with INVERTED index
ALTER TABLE logs SET SEARCH_OPTIMIZATION = TRUE
  WITH (COLUMNS = (message, details), INDEX_TYPE = 'INVERTED');

-- Query with pattern matching using REGEXP_LIKE
SELECT
    log_id,
    timestamp,
    message,
    details
FROM
    logs
WHERE
    REGEXP_LIKE(message, 'error|fail|exception|timeout')
    AND timestamp > CURRENT_DATE() - 7
ORDER BY
    timestamp DESC
LIMIT 1000;
```

#### **Example 3: Prefix Search on Customer Names**
```sql
-- Enable Search Optimization on the customers table with NGRAM index
ALTER TABLE customers SET SEARCH_OPTIMIZATION = TRUE
  WITH (COLUMNS = (name), INDEX_TYPE = 'NGRAM');

-- Query with prefix search using LIKE
SELECT
    customer_id,
    name,
    email,
    phone
FROM
    customers
WHERE
    name LIKE 'John%'
ORDER BY
    name
LIMIT 50;

-- Query with suffix search using LIKE
SELECT
    customer_id,
    name,
    email
FROM
    customers
WHERE
    name LIKE '%Smith'
ORDER BY
    name
LIMIT 50;
```

#### **Example 4: Combined with Clustering**
```sql
-- Enable Search Optimization and clustering on the articles table
ALTER TABLE articles CLUSTER BY (publish_date)
  SET SEARCH_OPTIMIZATION = TRUE
  WITH (COLUMNS = (title, content), INDEX_TYPE = 'INVERTED');

-- Query with full-text search and date filter
SELECT
    article_id,
    title,
    publish_date,
    author
FROM
    articles
WHERE
    MATCH(content, 'machine learning')
    AND publish_date > CURRENT_DATE() - 30
ORDER BY
    publish_date DESC
LIMIT 20;
```

#### **Example 5: Monitor Search Optimization Usage**
```sql
-- Check Search Optimization status for all tables
SELECT
    table_name,
    schema_name,
    database_name,
    search_optimization,
    search_optimization_columns,
    search_optimization_index_type,
    search_optimization_storage_bytes / 1024 / 1024 AS storage_mb
FROM
    INFORMATION_SCHEMA.TABLES
WHERE
    search_optimization = TRUE
ORDER BY
    storage_mb DESC;

-- Check Search Optimization history for a specific table
SELECT
    table_name,
    action,
    status,
    start_time,
    end_time,
    rows_processed,
    bytes_processed,
    credits_used,
    error_message
FROM
    SNOWFLAKE.ACCOUNT_USAGE.SEARCH_OPTIMIZATION_HISTORY
WHERE
    table_name = 'PRODUCTS'
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;

-- Check if queries are using Search Optimization
SELECT
    query_id,
    query_text,
    used_search_optimization,
    execution_time,
    bytes_scanned,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%LIKE%'
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    used_search_optimization DESC, execution_time;
```

## **4. Query Acceleration vs. Search Optimization: Decision Matrix**

### **Mermaid: Query Acceleration vs. Search Optimization Decision Tree**
```mermaid
%% Query Acceleration vs. Search Optimization Decision Tree
flowchart TD
    A[("Query Performance\nIssue")] --> B{Query Type?}
    B -->|Full-Text Search| C[("Use Search Optimization\n(SOS)")]
    B -->|Pattern Matching| C
    B -->|LIKE/REGEXP| C
    B -->|MATCH/CONTAINS| C
    B -->|Complex Joins| D[("Use Query Acceleration\n(Join Reordering)")]
    B -->|Aggregations| D
    B -->|Window Functions| D
    B -->|Repetitive Queries| E[("Use Query Acceleration\n(Result Caching)")]
    B -->|All Queries| F[("Use Query Acceleration\n(Automatic Rewriting)")]
    B -->|High Concurrency| G[("Use Query Acceleration\n(Multi-Cluster)")]

    C --> H[("Enable SOS on Table\n(ALTER TABLE ... SET SEARCH_OPTIMIZATION = TRUE)")]
    D --> I[("Optimize Query Design\n(Add Filters, Use Clustering)")]
    E --> J[("Enable Result Caching\n(ALTER SESSION SET USE_CACHED_RESULTS = TRUE)")]
    F --> K[("No Configuration Needed\n(Automatic)")]
    G --> L[("Use Multi-Cluster Warehouse\n(CREATE WAREHOUSE ... MAX_CLUSTER_COUNT)")]

    H --> M[("Use for LIKE/REGEXP/MATCH")]
    I --> N[("Use for All Queries")]
    J --> O[("Use for Repetitive Queries")]
    K --> P[("Use for All Queries")]
    L --> Q[("Use for High Concurrency Workloads")]

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef query fill:#4285f4,stroke:#1976d2;
    classDef sos fill:#ff9800,stroke:#f57c00;
    classDef qa fill:#009688,stroke:#00796b;
    class A query;
    class B query;
    class C sos;
    class D,E,F,G qa;
    class H,M sos;
    class I,J,K,P qa;
    class L,Q qa;
```

### **Comparison Matrix: Query Acceleration vs. Search Optimization**

| **Feature** | **Query Acceleration** | **Search Optimization** | **Notes** |
|-------------|-------------------------|--------------------------|-----------|
| **Purpose** | Improve performance of all queries | Accelerate full-text search and pattern matching | Query Acceleration is broader; SOS is specialized |
| **Scope** | All queries | Queries with LIKE, REGEXP, MATCH, CONTAINS | SOS only applies to full-text search queries |
| **How It Works** | Automatic query rewriting, vectorized execution, caching | Search indexes (inverted, n-gram) | Different underlying mechanisms |
| **Configuration** | Automatic (no configuration needed) | Enable per table/column | SOS requires explicit configuration |
| **Cost** | Included in Snowflake pricing | Pay-per-use (storage + compute) | SOS has additional costs |
| **Performance Impact** | 1.5-10x faster | 10-100x faster | SOS has higher performance impact for applicable queries |
| **Best For** | All queries, especially complex joins and aggregations | Full-text search, pattern matching | Use SOS for search queries, Query Acceleration for everything else |
| **Monitoring** | QUERY_HISTORY, QUERY_PROFILE | SEARCH_OPTIMIZATION_HISTORY | Different monitoring views |
| **Limitations** | Limited by query complexity | Limited to full-text search queries | Both have limitations but for different use cases |
| **Automatic** | Yes (all automatic) | No (requires enabling) | Query Acceleration is always on; SOS must be enabled |
| **Storage Overhead** | No | Yes (search indexes) | SOS consumes additional storage |
| **Compute Overhead** | Minimal | Yes (index maintenance) | SOS has index maintenance overhead |
| **Real-Time** | Yes | Eventual consistency | SOS indexes are updated asynchronously |
| **Supported Data Types** | All | STRING, VARCHAR, TEXT | SOS does not support all data types |
| **Wildcard Support** | Limited | Yes (with NGRAM index) | SOS handles wildcards better |
| **Fuzzy Search** | No | No | Neither supports fuzzy search natively |
| **Synonym Support** | No | No | Neither supports synonyms natively |

### **When to Use Query Acceleration**
Use **Query Acceleration** for:
1. **All queries** (it's automatic and included).
2. **Complex joins** (join reordering, hash joins).
3. **Aggregations** (group by, count, sum, avg).
4. **Window functions** (over, partition by).
5. **Repetitive queries** (result caching).
6. **Large tables** (partition pruning, column pruning).
7. **High concurrency workloads** (multi-cluster warehouses).
8. **Mixed workloads** (query prioritization).

**Example Use Cases**:
- ETL pipelines with complex transformations
- Reporting dashboards with aggregations
- Ad-hoc analysis queries
- Batch processing jobs

### **When to Use Search Optimization**
Use **Search Optimization** for:
1. **Full-text search queries** (e.g., `WHERE MATCH(column, 'search term')`).
2. **Pattern matching queries** (e.g., `WHERE REGEXP_LIKE(column, 'pattern')`).
3. **LIKE queries with wildcards** (e.g., `WHERE column LIKE '%pattern%'`).
4. **Large tables with frequent search queries** (>1TB).
5. **Applications requiring sub-second search response times**.

**Example Use Cases**:
- Product search in e-commerce applications
- Log analysis with pattern matching
- Document search in content management systems
- Customer support ticket search

### **When to Use Both**
Use **both Query Acceleration and Search Optimization** for:
1. **Applications with mixed workloads** (e.g., dashboards with both aggregations and search).
2. **Large tables with both search and analytical queries**.
3. **High-performance applications** requiring both fast search and fast analytics.

**Example Use Case**:
```sql
-- Enable Search Optimization for full-text search
ALTER TABLE products SET SEARCH_OPTIMIZATION = TRUE
  WITH (COLUMNS = (description, name));

-- Use clustering for analytical queries
ALTER TABLE products CLUSTER BY (category, price);

-- Use result caching for repetitive queries
ALTER SESSION SET USE_CACHED_RESULTS = TRUE;

-- Query with both search and analytics
SELECT
    category,
    COUNT(*) AS product_count,
    AVG(price) AS avg_price
FROM
    products
WHERE
    MATCH(description, 'snowflake')
    AND price > 100
GROUP BY
    category;
```

## **5. Implementation Patterns**

### **A. Query Acceleration Implementation Patterns**

#### **Pattern 1: Automatic Query Rewriting for Complex Joins**
```sql
-- Create tables with clustering for join optimization
CREATE TABLE orders (
    order_id INTEGER,
    customer_id INTEGER,
    order_date DATE,
    amount FLOAT
)
CLUSTER BY (customer_id, order_date);

CREATE TABLE customers (
    customer_id INTEGER,
    name STRING,
    region STRING,
    join_date DATE
)
CLUSTER BY (customer_id);

-- Query with complex joins (automatically optimized)
SELECT
    c.region,
    COUNT(o.order_id) AS order_count,
    SUM(o.amount) AS total_amount
FROM
    customers c
JOIN
    orders o ON c.customer_id = o.customer_id
WHERE
    o.order_date > CURRENT_DATE() - 30
GROUP BY
    c.region
ORDER BY
    total_amount DESC;
```

**Performance Impact**:
- **Join Reordering**: Snowflake may reorder the join to minimize intermediate results.
- **Partition Pruning**: Only scans relevant partitions in the `orders` table.
- **Column Pruning**: Only reads the columns needed for the query.

#### **Pattern 2: Result Caching for Dashboard Queries**
```sql
-- Enable result caching for dashboard queries
ALTER SESSION SET USE_CACHED_RESULTS = TRUE;
ALTER SESSION SET RESULT_CACHE_TTL = 3600;  -- 1 hour

-- Dashboard query 1 (cached)
SELECT
    region,
    SUM(sales) AS total_sales
FROM
    sales
WHERE
    date > CURRENT_DATE() - 7
GROUP BY
    region;

-- Dashboard query 2 (cached)
SELECT
    product_category,
    COUNT(*) AS product_count
FROM
    products
WHERE
    last_updated > CURRENT_DATE() - 1
GROUP BY
    product_category;

-- Dashboard query 3 (cached)
SELECT
    customer_segment,
    AVG(purchase_amount) AS avg_purchase
FROM
    customers
WHERE
    signup_date > CURRENT_DATE() - 30
GROUP BY
    customer_segment;
```

**Performance Impact**:
- **First Execution**: Full query execution.
- **Subsequent Executions**: Results returned from cache in <10ms.

#### **Pattern 3: Multi-Cluster Warehouse for High Concurrency**
```sql
-- Create a multi-cluster warehouse for high concurrency workloads
CREATE WAREHOUSE reporting_wh
  WAREHOUSE_SIZE = 'LARGE'
  MAX_CLUSTER_COUNT = 4
  MIN_CLUSTER_COUNT = 1
  SCALING_POLICY = 'STANDARD'
  AUTO_SUSPEND = 1800
  AUTO_RESUME = TRUE;

-- Grant access to reporting users
GRANT USAGE ON WAREHOUSE reporting_wh TO ROLE reporting_role;

-- Run concurrent queries (automatically distributed across clusters)
-- Query 1
SELECT * FROM sales WHERE date > CURRENT_DATE() - 7;

-- Query 2
SELECT * FROM customers WHERE region = 'US';

-- Query 3
SELECT * FROM products WHERE category = 'Electronics';
```

**Performance Impact**:
- **Concurrency**: Handles 10-100+ concurrent queries.
- **Scalability**: Automatically scales out to meet demand.
- **Performance**: Each query runs on a dedicated cluster.

#### **Pattern 4: Warehouse Optimization for Cost Savings**
```sql
-- Create warehouses with auto-suspend for different workloads
CREATE WAREHOUSE etl_wh
  WAREHOUSE_SIZE = 'X-LARGE'
  AUTO_SUSPEND = 1800  -- 30 minutes
  AUTO_RESUME = FALSE;  -- Disable auto-resume for batch workloads

CREATE WAREHOUSE reporting_wh
  WAREHOUSE_SIZE = 'LARGE'
  AUTO_SUSPEND = 600  -- 10 minutes
  AUTO_RESUME = TRUE;  -- Enable auto-resume for interactive workloads

CREATE WAREHOUSE adhoc_wh
  WAREHOUSE_SIZE = 'MEDIUM'
  AUTO_SUSPEND = 300  -- 5 minutes
  AUTO_RESUME = TRUE;

-- Set query timeouts
ALTER WAREHOUSE etl_wh SET STATEMENT_TIMEOUT_IN_SECONDS = 3600;  -- 1 hour
ALTER WAREHOUSE reporting_wh SET STATEMENT_TIMEOUT_IN_SECONDS = 300;  -- 5 minutes
ALTER WAREHOUSE adhoc_wh SET STATEMENT_TIMEOUT_IN_SECONDS = 600;  -- 10 minutes
```

**Cost Savings Impact**:
- **ETL Warehouse**: Suspends after 30 minutes of inactivity, no auto-resume.
- **Reporting Warehouse**: Suspends after 10 minutes of inactivity, auto-resumes for interactive queries.
- **Ad-Hoc Warehouse**: Suspends after 5 minutes of inactivity, auto-resumes for interactive queries.

### **B. Search Optimization Implementation Patterns**

#### **Pattern 1: Full-Text Search on Product Catalog**
```sql
-- Enable Search Optimization on the products table
ALTER TABLE products SET SEARCH_OPTIMIZATION = TRUE
  WITH (COLUMNS = (name, description, keywords), INDEX_TYPE = 'INVERTED');

-- Create a materialized view for common aggregations
CREATE MATERIALIZED VIEW product_stats_mv AS
SELECT
    category,
    COUNT(*) AS product_count,
    AVG(price) AS avg_price,
    MAX(last_updated) AS last_updated
FROM
    products
GROUP BY
    category;

-- Query with full-text search and aggregations
SELECT
    p.category,
    p.name,
    p.description,
    p.price,
    ps.product_count AS category_product_count,
    ps.avg_price AS category_avg_price
FROM
    products p
JOIN
    product_stats_mv ps ON p.category = ps.category
WHERE
    MATCH(p.name, 'laptop')
    OR MATCH(p.description, '16GB RAM')
    OR MATCH(p.keywords, 'gaming')
ORDER BY
    p.price DESC
LIMIT 50;
```

**Performance Impact**:
- **Search Optimization**: Accelerates the `MATCH` conditions.
- **Materialized View**: Provides pre-computed aggregations.
- **Combined**: Sub-second response times for complex queries.

#### **Pattern 2: Log Analysis with Pattern Matching**
```sql
-- Enable Search Optimization on the logs table with INVERTED index
ALTER TABLE logs SET SEARCH_OPTIMIZATION = TRUE
  WITH (COLUMNS = (message, details), INDEX_TYPE = 'INVERTED');

-- Cluster the logs table by timestamp for time-based queries
ALTER TABLE logs CLUSTER BY (timestamp);

-- Query with pattern matching and time filter
SELECT
    log_id,
    timestamp,
    severity,
    message,
    details
FROM
    logs
WHERE
    REGEXP_LIKE(message, 'error|fail|exception|timeout')
    AND timestamp > CURRENT_DATE() - 7
    AND severity IN ('ERROR', 'CRITICAL')
ORDER BY
    timestamp DESC
LIMIT 1000;
```

**Performance Impact**:
- **Search Optimization**: Accelerates the `REGEXP_LIKE` condition.
- **Clustering**: Reduces the number of partitions scanned for the time filter.
- **Combined**: Fast pattern matching on large log tables.

#### **Pattern 3: Customer Search with Prefix Matching**
```sql
-- Enable Search Optimization on the customers table with NGRAM index
ALTER TABLE customers SET SEARCH_OPTIMIZATION = TRUE
  WITH (COLUMNS = (first_name, last_name, email), INDEX_TYPE = 'NGRAM');

-- Cluster the customers table by region for regional queries
ALTER TABLE customers CLUSTER BY (region);

-- Query with prefix matching and region filter
SELECT
    customer_id,
    first_name,
    last_name,
    email,
    phone,
    region
FROM
    customers
WHERE
    first_name LIKE 'John%'
    OR last_name LIKE 'Smith%'
    OR email LIKE '%@gmail.com'
    AND region = 'US'
ORDER BY
    last_name, first_name
LIMIT 100;
```

**Performance Impact**:
- **Search Optimization**: Accelerates the `LIKE` conditions with wildcards.
- **Clustering**: Reduces the number of partitions scanned for the region filter.
- **Combined**: Fast prefix matching on customer data.

#### **Pattern 4: Document Search with Full-Text and Metadata**
```sql
-- Enable Search Optimization on the documents table
ALTER TABLE documents SET SEARCH_OPTIMIZATION = TRUE
  WITH (COLUMNS = (title, content, author, tags), INDEX_TYPE = 'INVERTED');

-- Cluster the documents table by creation date
ALTER TABLE documents CLUSTER BY (creation_date);

-- Create a materialized view for document metadata
CREATE MATERIALIZED VIEW document_metadata_mv AS
SELECT
    document_id,
    title,
    author,
    creation_date,
    word_count,
    last_updated
FROM
    documents;

-- Query with full-text search and metadata filters
SELECT
    d.document_id,
    d.title,
    d.author,
    d.creation_date,
    d.content,
    dm.word_count,
    dm.last_updated
FROM
    documents d
JOIN
    document_metadata_mv dm ON d.document_id = dm.document_id
WHERE
    MATCH(d.title, 'annual report')
    OR MATCH(d.content, 'financial results')
    OR MATCH(d.tags, 'finance')
    AND d.creation_date > CURRENT_DATE() - 365
    AND dm.word_count > 1000
ORDER BY
    d.creation_date DESC
LIMIT 20;
```

**Performance Impact**:
- **Search Optimization**: Accelerates the `MATCH` conditions.
- **Clustering**: Reduces the number of partitions scanned for the date filter.
- **Materialized View**: Provides fast access to document metadata.
- **Combined**: Fast and flexible document search.

## **6. Monitoring and Alerting**

### **A. Query Acceleration Monitoring**

#### **1. Monitor Automatic Query Rewriting**
```sql
-- Check query plans for rewriting
EXPLAIN SELECT * FROM my_table WHERE date > '2023-01-01';

-- Check for partition pruning in query plans
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'));

-- Check for column pruning in query plans
-- Look for Project operators with only needed columns
```

#### **2. Monitor Vectorized Execution**
```sql
-- Check for vectorized operators in query profiles
SELECT
    step_id,
    operation,
    rows_produced,
    execution_time
FROM
    TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))
WHERE
    operation LIKE '%Vectorized%'
    OR operation LIKE '%Batch%'
ORDER BY
    execution_time DESC;
```

#### **3. Monitor Result Caching**
```sql
-- Check if queries are using cached results
SELECT
    query_id,
    query_text,
    used_cached_result,
    execution_time,
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
    AVG(execution_time) AS avg_execution_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    used_cached_result;
```

#### **4. Monitor Metadata Caching**
```sql
-- Check for metadata cache hits (indirectly via query performance)
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
    compilation_time;
```

#### **5. Monitor Warehouse Optimization**
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
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;

-- Check warehouse events history
SELECT
    warehouse_name,
    event_time,
    event_type,
    old_size,
    new_size
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_EVENTS_HISTORY
WHERE
    event_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    event_time DESC;
```

### **B. Search Optimization Monitoring**

#### **1. Monitor Search Optimization Status**
```sql
-- Check Search Optimization status for all tables
SELECT
    table_name,
    schema_name,
    database_name,
    search_optimization,
    search_optimization_columns,
    search_optimization_index_type,
    search_optimization_last_indexed,
    search_optimization_storage_bytes / 1024 / 1024 AS storage_mb
FROM
    INFORMATION_SCHEMA.TABLES
WHERE
    search_optimization = TRUE
ORDER BY
    storage_mb DESC;
```

#### **2. Monitor Search Optimization History**
```sql
-- Check Search Optimization history for all tables
SELECT
    table_name,
    action,
    status,
    start_time,
    end_time,
    rows_processed,
    bytes_processed,
    credits_used,
    error_message
FROM
    SNOWFLAKE.ACCOUNT_USAGE.SEARCH_OPTIMIZATION_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;

-- Check Search Optimization errors
SELECT
    table_name,
    action,
    status,
    error_message,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.SEARCH_OPTIMIZATION_HISTORY
WHERE
    status = 'FAILED'
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;
```

#### **3. Monitor Search Optimization Usage**
```sql
-- Check if queries are using Search Optimization
SELECT
    query_id,
    query_text,
    used_search_optimization,
    execution_time,
    bytes_scanned,
    partitions_scanned,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%LIKE%' OR query_text LIKE '%REGEXP%' OR query_text LIKE '%MATCH%'
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    used_search_optimization DESC, execution_time;

-- Check Search Optimization performance improvement
SELECT
    used_search_optimization,
    AVG(execution_time) AS avg_execution_time,
    AVG(bytes_scanned) AS avg_bytes_scanned,
    COUNT(*) AS query_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_text LIKE '%LIKE%' OR query_text LIKE '%REGEXP%' OR query_text LIKE '%MATCH%'
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    used_search_optimization;
```

#### **4. Monitor Search Optimization Costs**
```sql
-- Check Search Optimization storage costs
SELECT
    table_name,
    schema_name,
    database_name,
    search_optimization_storage_bytes / 1024 / 1024 / 1024 AS storage_gb,
    search_optimization_storage_bytes * 0.023 / 1024 / 1024 / 1024 AS estimated_monthly_cost_usd
FROM
    INFORMATION_SCHEMA.TABLES
WHERE
    search_optimization = TRUE
ORDER BY
    estimated_monthly_cost_usd DESC;

-- Check Search Optimization compute costs
SELECT
    table_name,
    action,
    SUM(credits_used) AS total_credits_used,
    SUM(credits_used) * 0.028 AS estimated_cost_usd
FROM
    SNOWFLAKE.ACCOUNT_USAGE.SEARCH_OPTIMIZATION_HISTORY
WHERE
    start_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY
    table_name, action
ORDER BY
    estimated_cost_usd DESC;
```

### **C. Proactive Alerts**

#### **1. Query Acceleration Alerts**

**Slow Query Alert**:
```sql
CREATE OR REPLACE ALERT slow_query_alert
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'
AS
  SELECT
    query_id,
    query_text,
    warehouse_name,
    execution_time,
    bytes_scanned,
    start_time,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE
    execution_time > 10000  -- >10 seconds
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
    AND warehouse_name IN ('PROD_WH', 'REPORTING_WH')
  ORDER BY
    execution_time DESC;
```

**High Bytes Scanned Alert**:
```sql
CREATE OR REPLACE ALERT high_bytes_scanned_alert
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'
AS
  SELECT
    query_id,
    query_text,
    warehouse_name,
    bytes_scanned / 1024 / 1024 AS bytes_scanned_mb,
    start_time,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE
    bytes_scanned > 1024 * 1024 * 1024  -- >1GB
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
    AND warehouse_name IN ('PROD_WH', 'REPORTING_WH')
  ORDER BY
    bytes_scanned DESC;
```

**Warehouse Overloaded Alert**:
```sql
CREATE OR REPLACE ALERT warehouse_overloaded_alert
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'
AS
  SELECT
    warehouse_name,
    running_queries,
    queued_queries,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.INFORMATION_SCHEMA.WAREHOUSE_MONITOR
  WHERE
    queued_queries > 0
    OR running_queries = total_clusters * 10  -- All clusters are busy
  AND
    warehouse_name IN ('PROD_WH', 'REPORTING_WH');
```

#### **2. Search Optimization Alerts**

**Search Optimization Errors Alert**:
```sql
CREATE OR REPLACE ALERT search_optimization_error_alert
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'
AS
  SELECT
    table_name,
    action,
    status,
    error_message,
    start_time,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.SEARCH_OPTIMIZATION_HISTORY
  WHERE
    status = 'FAILED'
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
  ORDER BY
    start_time DESC;
```

**Search Optimization Not Used Alert**:
```sql
CREATE OR REPLACE ALERT search_optimization_not_used_alert
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'
AS
  SELECT
    query_id,
    query_text,
    warehouse_name,
    start_time,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE
    query_text LIKE '%LIKE%' OR query_text LIKE '%REGEXP%' OR query_text LIKE '%MATCH%'
    AND used_search_optimization = FALSE
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
  ORDER BY
    start_time DESC;
```

**High Search Optimization Storage Alert**:
```sql
CREATE OR REPLACE ALERT high_search_optimization_storage_alert
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON 0 9 * * * America/Los_Angeles'  -- 9 AM daily
AS
  SELECT
    table_name,
    schema_name,
    database_name,
    search_optimization_storage_bytes / 1024 / 1024 / 1024 AS storage_gb,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    INFORMATION_SCHEMA.TABLES
  WHERE
    search_optimization = TRUE
    AND search_optimization_storage_bytes / 1024 / 1024 / 1024 > 100  -- >100 GB
  ORDER BY
    storage_gb DESC;
```

## **7. Performance Tuning Workflow**

### **Mermaid: Performance Tuning Workflow for Query Acceleration and Search Optimization**
```mermaid
%% Performance Tuning Workflow
flowchart TD
    A[("Identify Performance\nIssue")] --> B{Query Type?}
    B -->|Full-Text Search| C[("Check SOS Status")]
    B -->|Pattern Matching| C
    B -->|LIKE/REGEXP| C
    B -->|Other| D[("Check Query Acceleration")]
    C --> E{Search Optimization\nEnabled?}
    E -->|No| F[("Enable SOS\n(ALTER TABLE ...)")]
    E -->|Yes| G[("Check Query Profile\n(TABLE(QUERY_PROFILE))")]
    D --> H[("Check Query Profile\n(TABLE(QUERY_PROFILE))")]
    G --> I{High Bytes Scanned?}
    H --> I
    I -->|Yes| J[("Optimize Scanning\n(Add Filters, Clustering)")]
    I -->|No| K{High Execution Time?}
    K -->|Yes| L[("Optimize Query Design\n(Rewrite, Vectorized)")]
    K -->|No| M{Spill to Disk/Remote?}
    M -->|Yes| N[("Increase Warehouse Size\n(ALTER WAREHOUSE ...)")]
    M -->|No| O{High Queue Time?}
    O -->|Yes| P[("Increase Warehouse Size\n(Use Multi-Cluster)")]
    O -->|No| Q[("Check Other Issues")]

    F --> R[("Enable SOS on Table/Columns")]
    J --> S[("Add WHERE Clauses\nUse Clustering\nUse Partition Pruning")]
    L --> T[("Use Proper Join Types\nUse Approximate Functions\nReduce Data Volume")]
    N --> U[("ALTER WAREHOUSE\nSET WAREHOUSE_SIZE")]
    P --> V[("CREATE WAREHOUSE\nMAX_CLUSTER_COUNT")]
    Q --> W[("Check Network\nCheck External Systems")]

    R --> X[("Verify SOS Usage\n(SELECT used_search_optimization FROM QUERY_HISTORY)")]
    S --> Y[("Monitor Partitions Scanned\n(SELECT partitions_scanned FROM QUERY_HISTORY)")]
    T --> Z[("Monitor Execution Time\n(SELECT execution_time FROM QUERY_HISTORY)")]
    U --> AA[("Monitor Spill Metrics\n(SELECT spill_to_disk FROM QUERY_PROFILE)")]
    V --> AB[("Monitor Cluster Usage\n(SELECT * FROM WAREHOUSE_LOAD_HISTORY)")]
    W --> AC[("Monitor External Factors\n(Check cloud storage, network)")]

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80,81,82,83,84,85,86,87,88,89,90,91,92,93,94,95,96,97,98,99,100,101,102,103,104,105,106,107
