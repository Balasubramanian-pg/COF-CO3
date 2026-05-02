# **Optimize Query Performance: Production-Grade Technical Deep Dive**


## **1. Query Optimization Fundamentals**

### **Mermaid: Snowflake Query Optimization Framework**
```mermaid
%% Snowflake Query Optimization Framework
flowchart TD
    subgraph Input["Input Layer"]
        A[("SQL Query")] --> B[("Query Parser")]
    end

    subgraph Optimization["Optimization Layer"]
        B --> C[("Query Rewriter")]
        C --> D[("Query Optimizer")]
        D --> E[("Query Compiler")]
    end

    subgraph Execution["Execution Layer"]
        E --> F[("Query Execution Engine")]
        F --> G[("Virtual Warehouse")]
        G --> H[("Storage Service")]
        G --> I[("Metadata Service")]
    end

    subgraph Output["Output Layer"]
        F --> J[("Result Set")]
    end

    subgraph OptimizationTechniques["Optimization Techniques"]
        K[("Clustering")]
        L[("Partition Pruning")]
        M[("Column Pruning")]
        N[("Predicate Pushdown")]
        O[("Join Optimization")]
        P[("Aggregate Optimization")]
        Q[("Caching")]
        R[("Approximate Functions")]
        S[("Materialized Views")]
        T[("Query Rewriting")]
    end

    D --> K
    D --> L
    D --> M
    D --> N
    D --> O
    D --> P
    D --> Q
    D --> R
    D --> S
    D --> T

    subgraph Monitoring["Monitoring Layer"]
        U[("QUERY_HISTORY")]
        V[("QUERY_PROFILE")]
        W[("WAREHOUSE_LOAD_HISTORY")]
    end
    F --> U
    F --> V
    G --> W

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef input fill:#4285f4,stroke:#1976d2;
    classDef optimization fill:#ff9800,stroke:#f57c00;
    classDef execution fill:#009688,stroke:#00796b;
    classDef output fill:#e91e63,stroke:#c2185b;
    classDef techniques fill:#9c27b0,stroke:#7b1fa2;
    classDef monitoring fill:#3f51b5,stroke:#303f9f;
    class A input;
    class B,C,D,E optimization;
    class F,G,H,I execution;
    class J output;
    class K,L,M,N,O,P,Q,R,S,T techniques;
    class U,V,W monitoring;
```


### **Snowflake's Unique Architecture for Query Optimization**

Snowflake's **multi-cluster, shared-data architecture** provides several inherent advantages for query optimization:

1. **Separation of Compute and Storage**:
   - **Compute**: Virtual warehouses handle query execution
   - **Storage**: Cloud storage (S3, Azure Blob, GCS) stores data
   - **Metadata**: Metadata service manages all metadata

2. **Micro-Partitioning**:
   - Data is automatically divided into **micro-partitions** (50MB-500MB)
   - Enables **partition pruning** (skip irrelevant partitions)
   - Enables **parallel processing** (each partition processed independently)

3. **Columnar Storage**:
   - Data stored in **columnar format** (Parquet)
   - Enables **column pruning** (only read required columns)
   - Enables **vectorized execution** (process columns as vectors)

4. **Caching Layers**:
   - **Result Cache**: Caches query results for 24 hours
   - **Metadata Cache**: Caches table metadata (statistics, schema)
   - **Local Disk Cache**: Caches frequently accessed data in SSD

5. **Query Optimization Features**:
   - **Automatic Clustering**: Improves partition pruning
   - **Query Rewriting**: Optimizes queries automatically
   - **Vectorized Execution**: Processes data in batches using SIMD
   - **Multi-Cluster Warehouses**: Scales out for high concurrency

### **Key Query Performance Metrics**

| **Metric** | **Definition** | **Target Value** | **Measurement Method** | **Optimization Impact** |
|------------|---------------|------------------|--------------------------|-------------------------|
| **Execution Time** | Time to execute the query (end-to-end) | <1s (OLTP), <10s (OLAP) | QUERY_HISTORY.EXECUTION_TIME | Reduce via query design, warehouse sizing |
| **Compilation Time** | Time to parse, optimize, and compile | <100ms | QUERY_HISTORY.COMPILATION_TIME | Simplify queries, use cached plans |
| **Queue Time** | Time query spends waiting in queue | <100ms | QUERY_HISTORY.QUEUE_TIME | Use larger warehouse, multi-cluster, prioritization |
| **Scan Time** | Time spent scanning data | Minimize | QUERY_PROFILE.SCAN_TIME | Use clustering, partition pruning |
| **Join Time** | Time spent joining tables | Minimize | QUERY_PROFILE.JOIN_TIME | Optimize join types, reduce join size |
| **Aggregate Time** | Time spent aggregating data | Minimize | QUERY_PROFILE.AGGREGATE_TIME | Use approximate functions, reduce groups |
| **Sort Time** | Time spent sorting data | Minimize | QUERY_PROFILE.SORT_TIME | Avoid ORDER BY, use LIMIT |
| **Bytes Scanned** | Amount of data read from storage | Minimize | QUERY_HISTORY.BYTES_SCANNED | Use filters, clustering, column pruning |
| **Partitions Scanned** | Number of micro-partitions scanned | Minimize | QUERY_HISTORY.PARTITIONS_SCANNED | Use clustering, partition pruning |
| **Rows Produced** | Number of rows returned | As needed | QUERY_HISTORY.ROWS_PRODUCED | Use LIMIT, pagination |
| **Credit Usage** | Snowflake credits consumed | Minimize | QUERY_HISTORY.CREDITS_USED | Optimize queries, right-size warehouse |
| **Spill to Disk** | Data spilled to disk due to memory limits | 0 | QUERY_PROFILE.SPILL_TO_DISK | Increase warehouse size, reduce data volume |
| **Spill to Remote** | Data spilled to remote storage | 0 | QUERY_PROFILE.SPILL_TO_REMOTE | Increase warehouse size, reduce data volume |

### **Query Optimization Principles**

1. **Minimize Data Scanned**:
   - Use **filters** to reduce the amount of data scanned
   - Use **clustering** to co-locate related data
   - Use **partition pruning** to skip irrelevant partitions
   - Use **column pruning** to only read required columns

2. **Optimize Joins**:
   - Use **proper join types** (INNER JOIN, LEFT JOIN, etc.)
   - **Filter before joining** to reduce join size
   - Use **broadcast joins** for small tables
   - Avoid **Cartesian products**

3. **Optimize Aggregations**:
   - Use **approximate functions** (APPROX_COUNT_DISTINCT, APPROX_QUANTILE)
   - **Filter before aggregating** to reduce groups
   - Use **GROUP BY on clustered columns**
   - Avoid **unnecessary aggregations**

4. **Leverage Caching**:
   - Use **result caching** for repetitive queries
   - Use **metadata caching** for table statistics
   - Use **local disk caching** for frequently accessed data

5. **Right-Size Warehouses**:
   - Use the **smallest warehouse** that meets performance requirements
   - Use **multi-cluster warehouses** for high concurrency
   - Use **auto-suspend** to save costs during idle periods

6. **Use Efficient File Formats**:
   - Use **Parquet** or **ORC** for external tables
   - Use **compression** to reduce file sizes
   - Use **partitioned data** for better pruning

7. **Monitor and Tune**:
   - **Monitor query performance** using QUERY_HISTORY and QUERY_PROFILE
   - **Analyze query plans** using EXPLAIN
   - **Tune queries** based on performance data
   - **Set up alerts** for performance issues

### **Common Query Performance Bottlenecks**

| **Bottleneck** | **Symptoms** | **Root Causes** | **Diagnosis** | **Solutions** |
|----------------|--------------|-----------------|---------------|---------------|
| **I/O Bottleneck** | High bytes_scanned, high execution_time, low CPU usage | Full table scans, missing filters, poor clustering | QUERY_PROFILE.bytes_scanned, QUERY_HISTORY.bytes_scanned | Add filters, use clustering, use partition pruning, use column pruning |
| **CPU Bottleneck** | High execution_time, high CPU usage, low bytes_scanned | Complex joins, aggregations, window functions | QUERY_PROFILE.execution_time, WAREHOUSE_LOAD_HISTORY.running_queries | Optimize joins, use approximate functions, reduce data volume |
| **Memory Bottleneck** | Spill to disk/remote, high execution_time | Large result sets, large joins, large aggregations | QUERY_PROFILE.spill_to_disk, QUERY_PROFILE.spill_to_remote | Increase warehouse size, reduce data volume, use LIMIT |
| **Concurrency Bottleneck** | High queue_time, low running_queries | Warehouse overloaded, too many concurrent queries | QUERY_HISTORY.queue_time, WAREHOUSE_LOAD_HISTORY.queued_queries | Use larger warehouse, use multi-cluster warehouse, use query prioritization |
| **Network Bottleneck** | High latency, low throughput | Client-server distance, small result sets | QUERY_HISTORY.execution_time, client-side metrics | Use larger result sets, reduce query frequency, use caching, use PrivateLink |
| **Compilation Bottleneck** | High compilation_time, low execution_time | Complex queries, many subqueries, dynamic SQL | QUERY_HISTORY.compilation_time | Simplify queries, use cached plans, avoid dynamic SQL |
| **Storage Bottleneck** | High bytes_scanned for external tables | Slow cloud storage, large external tables | QUERY_HISTORY.bytes_scanned, TABLE_STORAGE_METRICS.storage_bytes | Use internal tables, use materialized views, use caching, use PrivateLink |

## **2. Query Design Optimization**

### **A. SELECT Statement Optimization**

#### **1. Basic SELECT Optimization**

**Best Practices**:
- **Select only needed columns** (avoid `SELECT *`)
- **Use column aliases** for clarity
- **Use table aliases** for readability
- **Use explicit JOIN syntax** (not comma-separated tables)
- **Use WHERE before GROUP BY/HAVING**

**Examples**:

```sql
-- Bad: SELECT *
SELECT * FROM my_table WHERE date > CURRENT_DATE();

-- Good: Select only needed columns
SELECT id, name, value FROM my_table WHERE date > CURRENT_DATE();

-- Bad: Implicit column selection
SELECT id, name, value, date, region FROM my_table WHERE date > CURRENT_DATE();

-- Good: Explicit column selection with aliases
SELECT
    t.id AS user_id,
    t.name AS user_name,
    t.value AS user_value,
    t.date AS created_date
FROM
    my_table t
WHERE
    t.date > CURRENT_DATE();
```

#### **2. WHERE Clause Optimization**

**Best Practices**:
- **Push filters early** in the query
- **Use sargable conditions** (avoid functions on filtered columns)
- **Use parameterized queries** for better plan caching
- **Use IN for small lists**, EXISTS for large subqueries
- **Use BETWEEN for range queries**

**Examples**:

```sql
-- Bad: Function on filtered column (not sargable)
SELECT * FROM my_table WHERE UPPER(name) = 'ALICE';

-- Good: Sargable condition
SELECT * FROM my_table WHERE name = 'Alice';

-- Bad: OR conditions (hard to optimize)
SELECT * FROM my_table WHERE status = 'A' OR status = 'B' OR status = 'C';

-- Good: IN for small lists
SELECT * FROM my_table WHERE status IN ('A', 'B', 'C');

-- Bad: NOT IN (handles NULLs poorly)
SELECT * FROM my_table WHERE id NOT IN (SELECT id FROM other_table);

-- Good: NOT EXISTS (handles NULLs better)
SELECT * FROM my_table WHERE NOT EXISTS (SELECT 1 FROM other_table WHERE other_table.id = my_table.id);

-- Bad: Multiple OR conditions
SELECT * FROM my_table WHERE (col1 = 'A' AND col2 = 'B') OR (col1 = 'C' AND col2 = 'D');

-- Good: UNION ALL for complex OR conditions
SELECT * FROM my_table WHERE col1 = 'A' AND col2 = 'B'
UNION ALL
SELECT * FROM my_table WHERE col1 = 'C' AND col2 = 'D';

-- Good: BETWEEN for range queries
SELECT * FROM my_table WHERE date BETWEEN '2023-01-01' AND '2023-01-31';
```

#### **3. Filter Order Optimization**

**Best Practices**:
- **Place most selective filters first** to reduce data early
- **Use AND before OR** to maximize filter effectiveness
- **Filter before JOIN** to reduce join size

**Examples**:

```sql
-- Bad: Less selective filter first
SELECT * FROM my_table
WHERE region = 'US' AND date > CURRENT_DATE() - 30;

-- Good: Most selective filter first
SELECT * FROM my_table
WHERE date > CURRENT_DATE() - 30 AND region = 'US';

-- Bad: Filter after join
SELECT * FROM table1 JOIN table2 ON table1.id = table2.id
WHERE table1.date > CURRENT_DATE();

-- Good: Filter before join
SELECT * FROM
    (SELECT * FROM table1 WHERE date > CURRENT_DATE()) t1
JOIN
    table2 t2 ON t1.id = t2.id;
```

### **B. JOIN Optimization**

#### **1. Join Types and When to Use**

| **Join Type** | **Description** | **Syntax** | **Performance** | **When to Use** | **When NOT to Use** |
|---------------|-----------------|------------|-----------------|-----------------|-------------------|
| **INNER JOIN** | Returns rows with matches in both tables | `SELECT * FROM t1 INNER JOIN t2 ON t1.id = t2.id` | ⭐⭐⭐⭐⭐ | Default join type, most efficient | When you need unmatched rows |
| **LEFT JOIN** | Returns all rows from left table + matches from right | `SELECT * FROM t1 LEFT JOIN t2 ON t1.id = t2.id` | ⭐⭐⭐⭐ | When you need all rows from left table | When you only need matching rows |
| **RIGHT JOIN** | Returns all rows from right table + matches from left | `SELECT * FROM t1 RIGHT JOIN t2 ON t1.id = t2.id` | ⭐⭐⭐ | Rarely used | Use LEFT JOIN with swapped tables instead |
| **FULL OUTER JOIN** | Returns all rows from both tables | `SELECT * FROM t1 FULL OUTER JOIN t2 ON t1.id = t2.id` | ⭐⭐ | When you need all rows from both tables | When performance is critical |
| **CROSS JOIN** | Returns Cartesian product of both tables | `SELECT * FROM t1 CROSS JOIN t2` | ⭐ | Only for small tables | For large tables (creates row explosion) |
| **SEMI JOIN** | Returns rows from left table with matches in right | `SELECT * FROM t1 WHERE id IN (SELECT id FROM t2)` | ⭐⭐⭐⭐⭐ | Filtering (use EXISTS or IN) | When you need columns from right table |
| **ANTI JOIN** | Returns rows from left table without matches in right | `SELECT * FROM t1 WHERE id NOT IN (SELECT id FROM t2)` | ⭐⭐⭐⭐ | Exclusion (use NOT EXISTS or NOT IN) | When you need columns from right table |

#### **2. Join Algorithms in Snowflake**

| **Algorithm** | **Description** | **When Used** | **Performance** | **Memory Usage** | **Best For** |
|---------------|-----------------|---------------|-----------------|------------------|--------------|
| **Hash Join** | Builds a hash table on one table and probes with the other | Default for most joins | ⭐⭐⭐⭐ | Medium | General-purpose joins |
| **Sort-Merge Join** | Sorts both tables and merges them | Used for large sorted tables | ⭐⭐⭐ | Low | Large tables with sort keys |
| **Nested Loop Join** | Nested loop over rows (rare in Snowflake) | Small tables, specific cases | ⭐ | High | Small dimension tables |
| **Broadcast Join** | Broadcasts the smaller table to all nodes | Small dimension tables | ⭐⭐⭐⭐⭐ | Low | Small tables (<10MB) |

**Note**: Snowflake automatically chooses the best join algorithm based on table sizes, statistics, and query conditions.

#### **3. Join Optimization Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Filter Before Joining** | Apply WHERE clauses before joining | `SELECT * FROM (SELECT * FROM t1 WHERE date > '2023-01-01') a JOIN t2 b ON a.id = b.id` |
| **Join on Indexed Columns** | Join on clustered or partitioned columns | `SELECT * FROM t1 CLUSTER BY (id) JOIN t2 ON t1.id = t2.id` |
| **Use Broadcast Join for Small Tables** | Snowflake automatically uses broadcast join for small tables | `SELECT * FROM large_table JOIN small_table ON ...` |
| **Avoid Cartesian Products** | Ensure join conditions are specified | `SELECT * FROM t1 JOIN t2 ON t1.id = t2.id` (not `SELECT * FROM t1, t2`) |
| **Use Semi-Joins for Filtering** | Use EXISTS or IN for filtering | `SELECT * FROM t1 WHERE EXISTS (SELECT 1 FROM t2 WHERE t2.id = t1.id)` |
| **Use Anti-Joins for Exclusion** | Use NOT EXISTS or NOT IN for exclusion | `SELECT * FROM t1 WHERE NOT EXISTS (SELECT 1 FROM t2 WHERE t2.id = t1.id)` |
| **Join Small Tables First** | Join small tables before large tables | `SELECT * FROM small_table JOIN medium_table ON ... JOIN large_table ON ...` |
| **Avoid Redundant Joins** | Remove unnecessary joins | Remove joins to tables not used in SELECT or WHERE |
| **Use CTEs for Complex Joins** | Use CTEs to break down complex joins | `WITH cte AS (SELECT * FROM t1 JOIN t2 ON ...) SELECT * FROM cte JOIN t3 ON ...` |
| **Use JOIN Hints (if needed)** | Use hints to force a specific join algorithm | `SELECT * FROM t1 /*+ HASH_JOIN(t2) */ JOIN t2 ON t1.id = t2.id` |

**Examples**:

```sql
-- Bad: No filter before join
SELECT * FROM large_table JOIN small_table ON large_table.id = small_table.id
WHERE large_table.date > CURRENT_DATE();

-- Good: Filter before join
SELECT * FROM
    (SELECT * FROM large_table WHERE date > CURRENT_DATE()) l
JOIN
    small_table s ON l.id = s.id;

-- Bad: Cartesian product
SELECT * FROM table1, table2;

-- Good: Explicit join condition
SELECT * FROM table1 JOIN table2 ON table1.id = table2.id;

-- Bad: NOT IN (handles NULLs poorly)
SELECT * FROM table1 WHERE id NOT IN (SELECT id FROM table2);

-- Good: NOT EXISTS (handles NULLs better)
SELECT * FROM table1 WHERE NOT EXISTS (SELECT 1 FROM table2 WHERE table2.id = table1.id);

-- Good: Semi-join for filtering
SELECT * FROM table1 WHERE id IN (SELECT id FROM table2);

-- Good: Broadcast join (automatic for small tables)
SELECT * FROM large_table JOIN small_table ON large_table.id = small_table.id;
```

### **C. GROUP BY Optimization**

#### **1. GROUP BY Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Filter Before GROUP BY** | Apply WHERE before GROUP BY | `SELECT col1, COUNT(*) FROM (SELECT col1 FROM my_table WHERE date > '2023-01-01') GROUP BY col1` |
| **Use HAVING for Filtering Groups** | Use HAVING to filter groups | `SELECT col1, COUNT(*) FROM my_table GROUP BY col1 HAVING COUNT(*) > 100` |
| **Use GROUP BY on Clustered Columns** | Group by clustered columns for better performance | `SELECT region, COUNT(*) FROM my_table CLUSTER BY (region) GROUP BY region` |
| **Use Approximate Functions** | Use APPROX_COUNT_DISTINCT, APPROX_QUANTILE | `SELECT APPROX_COUNT_DISTINCT(user_id) FROM my_table` |
| **Avoid Unnecessary Aggregations** | Remove aggregations that are not needed | `SELECT col1, col2 FROM my_table` (instead of `SELECT col1, COUNT(*) FROM my_table GROUP BY col1, col2`) |
| **Use ROLLUP for Hierarchical Aggregations** | Use ROLLUP for multi-level aggregations | `SELECT region, product, SUM(sales) FROM my_table GROUP BY ROLLUP(region, product)` |
| **Use CUBE for Multi-Dimensional Aggregations** | Use CUBE for all possible groupings | `SELECT region, product, SUM(sales) FROM my_table GROUP BY CUBE(region, product)` |
| **Use GROUPING SETS for Multiple Aggregations** | Use GROUPING SETS for multiple aggregations in one query | `SELECT region, SUM(sales) FROM my_table GROUP BY GROUPING SETS ((region), ())` |

**Examples**:

```sql
-- Bad: GROUP BY before filtering
SELECT col1, COUNT(*) FROM my_table GROUP BY col1
WHERE date > CURRENT_DATE();

-- Good: Filter before GROUP BY
SELECT col1, COUNT(*) FROM (SELECT col1 FROM my_table WHERE date > CURRENT_DATE()) GROUP BY col1;

-- Bad: COUNT(DISTINCT) on large table
SELECT COUNT(DISTINCT user_id) FROM my_table;

-- Good: APPROX_COUNT_DISTINCT for large table
SELECT APPROX_COUNT_DISTINCT(user_id) FROM my_table;

-- Good: ROLLUP for hierarchical aggregations
SELECT region, product_category, SUM(revenue) AS total_revenue
FROM sales
GROUP BY ROLLUP(region, product_category)
ORDER BY region, product_category;

-- Good: CUBE for multi-dimensional aggregations
SELECT region, product_category, SUM(revenue) AS total_revenue
FROM sales
GROUP BY CUBE(region, product_category)
ORDER BY region, product_category;

-- Good: GROUPING SETS for multiple aggregations
SELECT
    region,
    SUM(revenue) AS region_revenue,
    SUM(SUM(revenue)) OVER () AS total_revenue
FROM sales
GROUP BY GROUPING SETS ((region), ());
```

### **D. ORDER BY Optimization**

#### **1. ORDER BY Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Avoid ORDER BY on Large Result Sets** | Sorting large result sets is expensive | `SELECT * FROM my_table LIMIT 1000` (instead of `ORDER BY`) |
| **Use LIMIT with ORDER BY** | Limit the number of rows sorted | `SELECT * FROM my_table ORDER BY sales DESC LIMIT 100` |
| **Sort by Clustered Columns** | Sort by clustered columns for better performance | `SELECT * FROM my_table CLUSTER BY (date) ORDER BY date` |
| **Use Keyset Pagination** | Use keyset pagination for large datasets | `SELECT * FROM my_table WHERE id > last_id ORDER BY id LIMIT 1000` |
| **Avoid Unnecessary Sorting** | Remove ORDER BY if not needed | `SELECT * FROM my_table` (instead of `ORDER BY id`) |
| **Use Materialized Views for Sorted Data** | Pre-compute sorted results | `CREATE MATERIALIZED VIEW my_mv AS SELECT * FROM my_table ORDER BY sales DESC` |

**Examples**:

```sql
-- Bad: ORDER BY on large result set
SELECT * FROM my_table ORDER BY sales DESC;

-- Good: LIMIT with ORDER BY
SELECT * FROM my_table ORDER BY sales DESC LIMIT 100;

-- Bad: OFFSET for pagination (slow for large offsets)
SELECT * FROM my_table ORDER BY id LIMIT 100 OFFSET 1000000;

-- Good: Keyset pagination
SELECT * FROM my_table WHERE id > 1000000 ORDER BY id LIMIT 100;

-- Good: Sort by clustered column
SELECT * FROM my_table CLUSTER BY (date) ORDER BY date;
```

### **E. Subquery Optimization**

#### **1. Subquery Types and Optimization**

| **Subquery Type** | **Description** | **Performance** | **Optimization** | **Example** |
|-------------------|-----------------|-----------------|------------------|-------------|
| **Scalar Subquery** | Returns a single value | ⭐⭐⭐ | Use joins or CTEs | `SELECT * FROM t1 WHERE col1 > (SELECT AVG(col1) FROM t2)` |
| **Row Subquery** | Returns a single row | ⭐⭐⭐⭐ | Use joins or CTEs | `SELECT * FROM t1 WHERE (col1, col2) = (SELECT col1, col2 FROM t2)` |
| **Table Subquery** | Returns multiple rows | ⭐⭐⭐ | Use joins or CTEs | `SELECT * FROM t1 WHERE col1 IN (SELECT col1 FROM t2)` |
| **Correlated Subquery** | References columns from outer query | ⭐ | Use joins or EXISTS | `SELECT * FROM t1 WHERE col1 IN (SELECT col1 FROM t2 WHERE t2.col2 = t1.col2)` |
| **Uncorrelated Subquery** | Does not reference outer query | ⭐⭐⭐⭐ | Use CTEs or joins | `SELECT * FROM t1 WHERE col1 IN (SELECT col1 FROM t2)` |

#### **2. Subquery Optimization Best Practices**

| **Best Practice** | **Description** | **Before** | **After** |
|-------------------|-----------------|------------|-----------|
| **Replace Correlated Subqueries with Joins** | Correlated subqueries are slow | `SELECT * FROM t1 WHERE id IN (SELECT id FROM t2 WHERE t2.col = t1.col)` | `SELECT * FROM t1 JOIN t2 ON t1.id = t2.id AND t2.col = t1.col` |
| **Replace EXISTS with IN for Small Subqueries** | IN is faster for small subqueries | `SELECT * FROM t1 WHERE EXISTS (SELECT 1 FROM t2 WHERE t2.id = t1.id)` | `SELECT * FROM t1 WHERE id IN (SELECT id FROM t2)` |
| **Replace NOT IN with NOT EXISTS** | NOT EXISTS handles NULLs better | `SELECT * FROM t1 WHERE id NOT IN (SELECT id FROM t2)` | `SELECT * FROM t1 WHERE NOT EXISTS (SELECT 1 FROM t2 WHERE t2.id = t1.id)` |
| **Use CTEs for Readability and Performance** | CTEs are optimized and easier to read | `SELECT * FROM (SELECT * FROM t1 JOIN t2 ON ...) WHERE ...` | `WITH cte AS (SELECT * FROM t1 JOIN t2 ON ...) SELECT * FROM cte WHERE ...` |
| **Avoid Nested Subqueries** | Nested subqueries are hard to optimize | `SELECT * FROM (SELECT * FROM (SELECT * FROM t1))` | `WITH cte1 AS (SELECT * FROM t1) SELECT * FROM cte1` |
| **Use Materialized Subqueries** | Use MATERIALIZE hint for subqueries | `SELECT * FROM (SELECT * FROM t1) WHERE ...` | `SELECT * FROM MATERIALIZE((SELECT * FROM t1)) WHERE ...` |
| **Limit Subquery Results** | Limit the number of rows returned by subqueries | `SELECT * FROM t1 WHERE id IN (SELECT id FROM t2)` | `SELECT * FROM t1 WHERE id IN (SELECT id FROM t2 LIMIT 1000)` |

**Examples**:

```sql
-- Bad: Correlated subquery
SELECT * FROM employees e
WHERE salary > (SELECT AVG(salary) FROM employees WHERE department = e.department);

-- Good: Join instead of correlated subquery
SELECT e.*
FROM employees e
JOIN (
    SELECT department, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
) d ON e.department = d.department
WHERE e.salary > d.avg_salary;

-- Bad: NOT IN (handles NULLs poorly)
SELECT * FROM table1 WHERE id NOT IN (SELECT id FROM table2);

-- Good: NOT EXISTS (handles NULLs better)
SELECT * FROM table1 WHERE NOT EXISTS (SELECT 1 FROM table2 WHERE table2.id = table1.id);

-- Bad: Nested subqueries
SELECT * FROM (SELECT * FROM (SELECT * FROM my_table)) WHERE date > CURRENT_DATE();

-- Good: CTEs
WITH cte AS (SELECT * FROM my_table)
SELECT * FROM cte WHERE date > CURRENT_DATE();

-- Good: Materialized subquery
SELECT * FROM MATERIALIZE((SELECT * FROM my_table WHERE date > CURRENT_DATE())) a
JOIN other_table b ON a.id = b.id;
```

### **F. CTE (Common Table Expression) Optimization**

#### **1. CTE Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use CTEs for Readability** | CTEs make complex queries more readable | `WITH cte AS (SELECT * FROM t1 JOIN t2 ON ...) SELECT * FROM cte` |
| **Use CTEs for Performance** | CTEs can be optimized by the query optimizer | `WITH cte AS (SELECT * FROM t1 WHERE ...) SELECT * FROM cte JOIN t2 ON ...` |
| **Use Materialized CTEs** | Use MATERIALIZE hint for CTEs that are referenced multiple times | `WITH cte AS MATERIALIZE (SELECT * FROM t1) SELECT * FROM cte JOIN cte ON ...` |
| **Avoid Recursive CTEs** | Recursive CTEs can be slow in Snowflake | Use iterative approaches instead |
| **Use CTEs for Complex Joins** | Break down complex joins into CTEs | `WITH cte1 AS (SELECT * FROM t1 JOIN t2 ON ...), cte2 AS (SELECT * FROM t3 JOIN t4 ON ...) SELECT * FROM cte1 JOIN cte2 ON ...` |
| **Use CTEs for Subquery Reuse** | Reuse CTEs to avoid repeating subqueries | `WITH cte AS (SELECT * FROM t1) SELECT COUNT(*) FROM cte, SELECT SUM(col1) FROM cte` |

**Examples**:

```sql
-- Bad: Repeated subquery
SELECT
    (SELECT COUNT(*) FROM my_table WHERE date > CURRENT_DATE()) AS today_count,
    (SELECT COUNT(*) FROM my_table WHERE date > CURRENT_DATE() - 7) AS week_count;

-- Good: CTE for subquery reuse
WITH today_data AS (SELECT * FROM my_table WHERE date > CURRENT_DATE()),
     week_data AS (SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7)
SELECT
    (SELECT COUNT(*) FROM today_data) AS today_count,
    (SELECT COUNT(*) FROM week_data) AS week_count;

-- Good: Materialized CTE for performance
WITH cte AS MATERIALIZE (SELECT * FROM my_table WHERE date > CURRENT_DATE())
SELECT
    COUNT(*) AS count,
    SUM(value) AS total_value
FROM
    cte;

-- Good: CTEs for complex joins
WITH customer_orders AS (
    SELECT
        c.customer_id,
        c.name,
        o.order_id,
        o.order_date,
        o.amount
    FROM
        customers c
    JOIN
        orders o ON c.customer_id = o.customer_id
),
order_items AS (
    SELECT
        o.order_id,
        i.item_id,
        i.quantity,
        i.price
    FROM
        orders o
    JOIN
        order_items i ON o.order_id = i.order_id
)
SELECT
    co.customer_id,
    co.name,
    co.order_date,
    oi.item_id,
    oi.quantity,
    oi.price,
    co.amount AS order_amount
FROM
    customer_orders co
JOIN
    order_items oi ON co.order_id = oi.order_id;
```

### **G. Window Function Optimization**

#### **1. Window Function Types**

| **Type** | **Description** | **Performance** | **Example** |
|----------|-----------------|-----------------|-------------|
| **Ranking** | ROW_NUMBER(), RANK(), DENSE_RANK() | ⭐⭐⭐⭐ | ROW_NUMBER() OVER (ORDER BY sales) |
| **Aggregate** | SUM(), AVG(), COUNT(), MIN(), MAX() | ⭐⭐⭐ | SUM(sales) OVER (PARTITION BY region) |
| **Value** | FIRST_VALUE(), LAST_VALUE(), LAG(), LEAD() | ⭐⭐⭐⭐ | LAG(sales) OVER (ORDER BY date) |
| **Analytic** | PERCENT_RANK(), CUME_DIST(), NTILE() | ⭐⭐⭐ | PERCENT_RANK() OVER (ORDER BY sales) |

#### **2. Window Function Optimization Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Partition by Clustered Columns** | Partition by clustered columns for better performance | `SELECT *, SUM(sales) OVER (PARTITION BY region ORDER BY date) FROM my_table CLUSTER BY (region)` |
| **Limit Window Frame** | Use RANGE or ROWS to limit the window frame | `SUM(sales) OVER (PARTITION BY region ORDER BY date ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING)` |
| **Avoid Unbounded Windows** | Unbounded windows can be expensive | `SUM(sales) OVER (PARTITION BY region ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` |
| **Use Materialized Views for Window Functions** | Pre-compute window functions | `CREATE MATERIALIZED VIEW my_mv AS SELECT *, SUM(sales) OVER (PARTITION BY region ORDER BY date) FROM my_table` |
| **Filter Before Window Functions** | Apply WHERE before window functions | `SELECT *, SUM(sales) OVER (PARTITION BY region) FROM (SELECT * FROM my_table WHERE date > '2023-01-01')` |
| **Use QUALIFY for Filtering Window Results** | Use QUALIFY to filter window function results | `SELECT *, ROW_NUMBER() OVER (PARTITION BY region ORDER BY sales DESC) AS rn FROM my_table QUALIFY rn <= 10` |
| **Avoid Redundant Window Functions** | Remove window functions that are not needed | `SELECT *, ROW_NUMBER() OVER (ORDER BY id) AS rn FROM my_table` (if rn is not used) |
| **Use INDEX OFF for Large Windows** | Disable index usage for large windows | `SELECT *, SUM(sales) OVER (PARTITION BY region ORDER BY date) FROM my_table /*+ INDEX(OFF) */` |

**Examples**:

```sql
-- Bad: Unbounded window
SELECT
    date,
    sales,
    SUM(sales) OVER (PARTITION BY region ORDER BY date) AS running_total
FROM
    sales;

-- Good: Limited window frame
SELECT
    date,
    sales,
    SUM(sales) OVER (PARTITION BY region ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM
    sales;

-- Good: Partition by clustered column
SELECT
    date,
    region,
    sales,
    SUM(sales) OVER (PARTITION BY region ORDER BY date) AS region_running_total
FROM
    sales CLUSTER BY (region);

-- Good: Filter before window function
SELECT
    date,
    region,
    sales,
    SUM(sales) OVER (PARTITION BY region ORDER BY date) AS region_running_total
FROM
    (SELECT * FROM sales WHERE date > '2023-01-01');

-- Good: QUALIFY for filtering window results
SELECT
    region,
    product,
    sales,
    ROW_NUMBER() OVER (PARTITION BY region ORDER BY sales DESC) AS rank
FROM
    sales
QUALIFY
    rank <= 5;

-- Good: Multiple window functions
SELECT
    date,
    region,
    sales,
    SUM(sales) OVER (PARTITION BY region ORDER BY date) AS region_running_total,
    AVG(sales) OVER (PARTITION BY region ORDER BY date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS region_3day_avg,
    ROW_NUMBER() OVER (PARTITION BY region ORDER BY sales DESC) AS region_rank
FROM
    sales;
```

## **3. Table Design Optimization**

### **A. Clustering**

#### **1. Clustering Overview**

**Definition**:
Clustering in Snowflake is a **data organization technique** that **co-locates related data** within the same **micro-partitions** to **minimize I/O** and **improve query performance**. Unlike traditional databases, Snowflake's clustering is **not a physical index** but rather a **logical organization** of data that guides the **query optimizer**.

**How It Works**:
1. **Clustering Keys**: Define 1-4 columns as clustering keys.
2. **Data Reorganization**: Snowflake reorganizes data within micro-partitions to group rows with similar clustering key values.
3. **Partition Pruning**: The query optimizer uses clustering metadata to skip irrelevant micro-partitions.
4. **Automatic Reclustering**: Snowflake automatically reclusters data in the background as new data is loaded.

**Example**:
- Clustering on `date` groups all rows with the same date in the same micro-partitions.
- A query with `WHERE date = '2023-01-01'` will only scan micro-partitions containing data for that date.

#### **2. Clustering Types**

| **Type** | **Description** | **Use Case** | **Example** |
|----------|-----------------|--------------|-------------|
| **Single-Column** | Cluster on one column | Simple filtering | `ALTER TABLE my_table CLUSTER BY (date)` |
| **Multi-Column** | Cluster on 2-4 columns | Complex filtering | `ALTER TABLE my_table CLUSTER BY (region, date)` |
| **Automatic** | Snowflake manages clustering | Hands-off optimization | `ALTER TABLE my_table CLUSTER BY AUTO` |
| **None** | No clustering | Default | `ALTER TABLE my_table CLUSTER BY NONE` |

#### **3. Clustering Performance Impact**

| **Clustering Depth** | **Pruning Effectiveness** | **Reclustering Overhead** | **Best For** |
|----------------------|---------------------------|----------------------------|--------------|
| **Depth 0 (None)** | None | None | Small tables, ad-hoc queries |
| **Depth 1** | Low | Low | Single-column filtering |
| **Depth 2** | Medium | Medium | Two-column filtering |
| **Depth 3** | High | High | Three-column filtering |
| **Depth 4** | Very High | Very High | Four-column filtering |

**Note**: Higher depth = better pruning but higher maintenance overhead.

#### **4. When to Use Clustering**

**Good Candidates for Clustering**:
- **Large tables** (>1TB) with repetitive queries
- **Frequently filtered columns** (e.g., `date`, `region`, `customer_id`)
- **Range queries** (e.g., `WHERE date BETWEEN '2023-01-01' AND '2023-01-31'`)
- **Join columns** (clustering on join keys can improve join performance)
- **High-cardinality columns** (many distinct values) for point queries
- **Time-series data** (cluster on `date` or `timestamp`)

**Poor Candidates for Clustering**:
- **Small tables** (<1GB; clustering overhead outweighs benefits)
- **Ad-hoc queries** (no repetitive patterns to optimize for)
- **Low-cardinality columns** (few distinct values; e.g., `gender`, `status`)
- **Columns not used in filters** (clustering has no effect)
- **Frequently updated tables** (reclustering overhead may impact performance)

#### **5. Clustering Configuration**

**Create a Table with Clustering**:
```sql
-- Create a table with single-column clustering
CREATE TABLE my_table (
    id INTEGER,
    name STRING,
    value FLOAT,
    date DATE,
    region STRING
)
CLUSTER BY (date);

-- Create a table with multi-column clustering
CREATE TABLE my_table (
    id INTEGER,
    name STRING,
    value FLOAT,
    date DATE,
    region STRING
)
CLUSTER BY (region, date);
```

**Alter a Table to Add/Change Clustering**:
```sql
-- Add clustering to an existing table
ALTER TABLE my_table CLUSTER BY (date);

-- Change clustering keys
ALTER TABLE my_table CLUSTER BY (region, date);

-- Remove clustering
ALTER TABLE my_table CLUSTER BY NONE;

-- Enable automatic clustering
ALTER TABLE my_table CLUSTER BY AUTO;
```

**Manually Recluster a Table**:
```sql
-- Force a reclustering of the table
ALTER TABLE my_table RECLUSTER;
```

#### **6. Clustering Monitoring**

**Check Clustering Information**:
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
```

**Check Reclustering Status**:
```sql
-- Check reclustering history for a table
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

#### **7. Clustering Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Cluster on Frequently Filtered Columns** | Choose columns used in WHERE clauses | `CLUSTER BY (date, region)` |
| **Prioritize High-Cardinality Columns** | Cluster on columns with many distinct values | `CLUSTER BY (customer_id, date)` |
| **Avoid Low-Cardinality Columns** | Avoid clustering on columns with few distinct values | Avoid `CLUSTER BY (gender)` |
| **Use Multi-Column Clustering for Complex Queries** | Cluster on multiple columns for complex filters | `CLUSTER BY (region, date, product_id)` |
| **Monitor Clustering Effectiveness** | Check CLUSTERING_DEPTH and PARTITIONS_SCANNED | `SELECT CLUSTERING_INFORMATION FROM INFORMATION_SCHEMA.TABLES` |
| **Recluster Manually if Needed** | Force reclustering for critical queries | `ALTER TABLE my_table RECLUSTER` |
| **Use Automatic Clustering for Hands-Off Optimization** | Let Snowflake manage clustering | `ALTER TABLE my_table CLUSTER BY AUTO` |
| **Combine with Partitioning** | Use clustering with partitioning for large tables | `CREATE TABLE my_table (id INT, date DATE) PARTITION BY (date) CLUSTER BY (region)` |
| **Cluster External Tables** | Cluster external tables for better performance | `ALTER EXTERNAL TABLE my_external_table CLUSTER BY (date)` |
| **Consider Clustering for Joins** | Cluster on join columns to improve join performance | `ALTER TABLE my_table CLUSTER BY (user_id)` |

**Examples**:

```sql
-- Good: Cluster on date for time-series data
ALTER TABLE sales CLUSTER BY (date);

-- Good: Multi-column clustering for complex queries
ALTER TABLE sales CLUSTER BY (region, date, product_category);

-- Good: Automatic clustering for hands-off optimization
ALTER TABLE sales CLUSTER BY AUTO;

-- Good: Monitor clustering effectiveness
SELECT
    table_name,
    clustering_information:'CLUSTERING_DEPTH' AS depth,
    clustering_information:'CLUSTER_BY' AS cluster_by,
    clustering_information:'TOTAL_PARTITION_COUNT' AS partition_count,
    clustering_information:'RECLUSTERING_REASON' AS recluster_reason
FROM
    INFORMATION_SCHEMA.TABLES
WHERE
    table_name = 'SALES';
```

### **B. Partitioning (Snowflake-Specific)**

**Note**: Snowflake does not support traditional partitioning like other databases (e.g., Oracle, PostgreSQL). However, you can achieve similar benefits through:

1. **Clustering**: As described above, clustering provides many of the benefits of partitioning.
2. **External Tables with Partitioned Data**: When using external tables (S3, Azure Blob, GCS), you can leverage the native partitioning of the underlying cloud storage.
3. **Iceberg/Delta Lake Tables**: Snowflake supports querying Iceberg and Delta Lake tables, which have built-in partitioning.

#### **1. External Table Partitioning**

When creating external tables that point to partitioned data in cloud storage (e.g., S3, Azure Blob, GCS), Snowflake can leverage the partitioning for **partition pruning**.

**Example: Partitioned External Table (S3)**:
```sql
-- Create an external stage
CREATE STAGE my_s3_stage
  URL = 's3://my-bucket/sales/'
  CREDENTIALS = (AWS_KEY_ID = '...' AWS_SECRET_KEY = '...')
  FILE_FORMAT = (TYPE = 'PARQUET');

-- Create a partitioned external table
CREATE EXTERNAL TABLE my_partitioned_table (
    id INTEGER,
    name STRING,
    value FLOAT,
    date DATE
)
WITH LOCATION = @my_s3_stage
FILE_FORMAT = (TYPE = 'PARQUET')
PARTITION BY (YEAR(date), MONTH(date), DAY(date));

-- Query with partition pruning
SELECT * FROM my_partitioned_table
WHERE date = '2023-01-15';  -- Only scans the partition for 2023/01/15
```

**Example: Partitioned External Table (Iceberg)**:
```sql
-- Create an external stage for Iceberg
CREATE STAGE my_iceberg_stage
  URL = 's3://my-bucket/iceberg/sales/'
  CREDENTIALS = (AWS_KEY_ID = '...' AWS_SECRET_KEY = '...');

-- Create an Iceberg external table
CREATE EXTERNAL TABLE my_iceberg_table
WITH LOCATION = @my_iceberg_stage
FILE_FORMAT = (TYPE = 'PARQUET');

-- Query with partition pruning (Iceberg handles partitioning internally)
SELECT * FROM my_iceberg_table
WHERE date = '2023-01-15';
```

#### **2. Partitioning Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use Partitioning for Large External Tables** | Partition external tables by date or key | `PARTITION BY (YEAR(date), MONTH(date))` |
| **Use Partition Pruning** | Filter on partitioned columns | `SELECT * FROM my_table WHERE date = '2023-01-15'` |
| **Avoid Over-Partitioning** | Too many partitions can hurt performance | Limit to 10-100 partitions per query |
| **Use Consistent Partitioning** | Use the same partitioning scheme across related tables | `PARTITION BY (date)` for all time-series tables |
| **Monitor Partition Pruning** | Check PARTITIONS_SCANNED in QUERY_HISTORY | `SELECT PARTITIONS_SCANNED FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` |

### **C. Table Compression**

#### **1. Compression in Snowflake**

Snowflake automatically applies **compression** to all data stored in its internal stages. For external stages, you can configure compression when creating file formats.

**Compression Options**:
| **Compression Type** | **Description** | **Best For** | **Compression Ratio** | **Performance** |
|----------------------|-----------------|--------------|----------------------|-----------------|
| **AUTO** | Snowflake automatically chooses the best compression | General use | Varies | ⭐⭐⭐⭐ |
| **GZIP** | GNU Zip compression | General-purpose | High | ⭐⭐⭐ |
| **BZ2** | Bzip2 compression | High compression | Very High | ⭐⭐ |
| **BROTLI** | Brotli compression | Web data | High | ⭐⭐⭐⭐ |
| **ZSTD** | Zstandard compression | High performance | High | ⭐⭐⭐⭐⭐ |
| **DEFLATE** | Deflate compression | General-purpose | Medium | ⭐⭐⭐ |
| **RAW_DEFLATE** | Raw Deflate compression | Fast compression | Low | ⭐⭐⭐⭐ |
| **NONE** | No compression | Uncompressed data | None | ⭐⭐⭐⭐⭐ |

#### **2. Compression Configuration**

**Create a File Format with Compression**:
```sql
-- Create a file format with Gzip compression
CREATE FILE FORMAT my_gzip_format
  TYPE = 'PARQUET'
  COMPRESSION = 'GZIP';

-- Create a file format with Zstd compression
CREATE FILE FORMAT my_zstd_format
  TYPE = 'PARQUET'
  COMPRESSION = 'ZSTD';

-- Create a file format with automatic compression
CREATE FILE FORMAT my_auto_format
  TYPE = 'PARQUET'
  COMPRESSION = 'AUTO';
```

**Use Compression with External Stages**:
```sql
-- Create an external stage with compression
CREATE STAGE my_compressed_stage
  URL = 's3://my-bucket/data/'
  CREDENTIALS = (AWS_KEY_ID = '...' AWS_SECRET_KEY = '...')
  FILE_FORMAT = (TYPE = 'PARQUET' COMPRESSION = 'SNAPPY');
```

#### **3. Compression Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use AUTO Compression** | Let Snowflake choose the best compression | `COMPRESSION = 'AUTO'` |
| **Use ZSTD for Performance** | ZSTD provides a good balance of compression ratio and performance | `COMPRESSION = 'ZSTD'` |
| **Use SNAPPY for Parquet** | Snappy is optimized for Parquet files | `COMPRESSION = 'SNAPPY'` |
| **Avoid NONE Compression** | Compression reduces storage costs and improves performance | Avoid `COMPRESSION = 'NONE'` |
| **Test Compression Ratios** | Test different compression types for your data | Compare file sizes with different compression types |
| **Consider Decompression Overhead** | Compression adds CPU overhead for decompression | Balance compression ratio with performance |

### **D. Column Selection and Data Types**

#### **1. Column Selection Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Avoid SELECT *** | Only select the columns you need | `SELECT col1, col2 FROM my_table` |
| **Use Column Aliases** | Use aliases for clarity | `SELECT id AS user_id, name AS user_name FROM my_table` |
| **Select in the Right Order** | Order columns by frequency of use | `SELECT frequently_used_col, occasionally_used_col FROM my_table` |
| **Use Column Pruning** | Only select columns that are used | `SELECT col1, col2 FROM my_table` (not `SELECT *`) |

#### **2. Data Type Optimization**

**Snowflake Data Types and Storage Sizes**:

| **Data Type** | **Storage Size** | **Description** | **Best For** |
|---------------|------------------|-----------------|--------------|
| **TINYINT** | 1 byte | 8-bit signed integer (-128 to 127) | Small integers |
| **SMALLINT** | 2 bytes | 16-bit signed integer (-32,768 to 32,767) | Medium integers |
| **INTEGER** | 4 bytes | 32-bit signed integer (-2,147,483,648 to 2,147,483,647) | General-purpose integers |
| **BIGINT** | 8 bytes | 64-bit signed integer | Large integers |
| **DECIMAL(p,s)** | 4-16 bytes | Fixed-point decimal | Financial data |
| **REAL** | 4 bytes | 32-bit floating-point | Approximate numbers |
| **DOUBLE** | 8 bytes | 64-bit floating-point | High-precision approximate numbers |
| **FLOAT** | 8 bytes | 64-bit floating-point | Synonym for DOUBLE |
| **BOOLEAN** | 1 byte | True/False | Boolean values |
| **CHAR(n)** | n bytes | Fixed-length character string | Fixed-length strings |
| **VARCHAR(n)** | Variable | Variable-length character string | General-purpose strings |
| **STRING** | Variable | Variable-length character string | Large strings |
| **BINARY** | Variable | Binary data | Binary data |
| **VARBINARY** | Variable | Variable-length binary data | Variable-length binary data |
| **DATE** | 3 bytes | Date (no time) | Dates |
| **TIME** | 4-8 bytes | Time (no date) | Times |
| **TIMESTAMP_NTZ** | 8 bytes | Timestamp without timezone | Timestamps without timezone |
| **TIMESTAMP_LTZ** | 8 bytes | Timestamp with local timezone | Timestamps with local timezone |
| **TIMESTAMP_TZ** | 8 bytes | Timestamp with timezone | Timestamps with timezone |
| **VARIANT** | Variable | Semi-structured data (JSON) | JSON data |
| **OBJECT** | Variable | Semi-structured data (key-value pairs) | Key-value pairs |
| **ARRAY** | Variable | Semi-structured data (arrays) | Arrays |
| **GEOGRAPHY** | Variable | Geographic data | Geographic coordinates |
| **GEOMETRY** | Variable | Geometric data | Geometric shapes |

#### **3. Data Type Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use the Smallest Data Type** | Use the smallest data type that fits your data | `SMALLINT` instead of `INTEGER` for small numbers |
| **Use DECIMAL for Financial Data** | DECIMAL provides exact precision for financial calculations | `DECIMAL(18,2)` for monetary values |
| **Use VARCHAR for Variable-Length Strings** | VARCHAR is more storage-efficient than CHAR for variable-length strings | `VARCHAR(255)` instead of `CHAR(255)` |
| **Use STRING for Large Text** | STRING is optimized for large text | `STRING` for descriptions, comments |
| **Use TIMESTAMP_NTZ for Timestamps** | TIMESTAMP_NTZ is more storage-efficient than TIMESTAMP_TZ | `TIMESTAMP_NTZ` for most timestamp use cases |
| **Use VARIANT for JSON Data** | VARIANT is optimized for semi-structured data | `VARIANT` for JSON columns |
| **Avoid Unnecessary Precision** | Use the minimum precision needed for DECIMAL | `DECIMAL(10,2)` instead of `DECIMAL(38,2)` |
| **Use BOOLEAN for True/False** | BOOLEAN is more storage-efficient than INTEGER for true/false | `BOOLEAN` for flags, indicators |

**Examples**:

```sql
-- Bad: Using INTEGER for small numbers
CREATE TABLE my_table (
    id INTEGER,  -- Uses 4 bytes
    flag INTEGER  -- Uses 4 bytes for a flag
);

-- Good: Using appropriate data types
CREATE TABLE my_table (
    id SMALLINT,  -- Uses 2 bytes
    flag BOOLEAN   -- Uses 1 byte
);

-- Bad: Using CHAR for variable-length strings
CREATE TABLE my_table (
    name CHAR(100)  -- Always uses 100 bytes
);

-- Good: Using VARCHAR for variable-length strings
CREATE TABLE my_table (
    name VARCHAR(100)  -- Uses only the space needed
);

-- Bad: Using FLOAT for financial data
CREATE TABLE my_table (
    amount FLOAT  -- Approximate, may have rounding errors
);

-- Good: Using DECIMAL for financial data
CREATE TABLE my_table (
    amount DECIMAL(18,2)  -- Exact precision
);
```

## **4. Query Execution Optimization**

### **A. EXPLAIN Plan Analysis**

#### **1. What is EXPLAIN?**

The **EXPLAIN** command in Snowflake generates a **query plan** that shows how Snowflake will execute your query. The query plan is a **tree of operators** that describes the steps Snowflake will take to execute the query.

**EXPLAIN Syntax**:
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

-- EXPLAIN with JSON output
EXPLAIN SELECT * FROM my_table WHERE id = 1
  WITH OUTPUT = 'JSON';
```

#### **2. Query Plan Anatomy**

**Query Plan Structure**:
- **Operators**: Each node in the tree is an operator (e.g., `TableScan`, `Filter`, `Join`, `Aggregate`).
- **Input/Output**: Each operator has input (data flowing in) and output (data flowing out).
- **Properties**: Each operator has properties (e.g., `filters`, `join type`, `group by`).
- **Statistics**: Estimated rows, bytes, and cost for each operator.

**Common Query Plan Operators**:

| **Operator** | **Type** | **Description** | **Performance Impact** | **Optimization Tips** |
|--------------|----------|-----------------|------------------------|-----------------------|
| **TableScan** | Scan | Reads data from a table | High I/O, high memory | Use clustering, partition pruning, column pruning |
| **ExternalScan** | Scan | Reads data from external tables | High I/O, depends on cloud storage | Use predicate pushdown, partition pruning |
| **Filter** | Filter | Filters rows based on a condition | Low CPU, reduces data volume | Push filters early in the plan |
| **Project** | Project | Selects specific columns | Low CPU, reduces data volume | Use column pruning |
| **Join** | Join | Joins two datasets | High CPU, high memory | Use proper join type, reduce join size |
| **Aggregate** | Aggregate | Groups and aggregates data | High CPU, high memory | Use approximate functions, reduce groups |
| **Sort** | Sort | Sorts data | High CPU, high memory | Avoid ORDER BY, use LIMIT |
| **Union** | SetOp | Combines results from multiple queries | High CPU, high memory | Use UNION ALL if duplicates are acceptable |
| **WindowFunction** | Window | Applies window functions | High CPU, high memory | Use partitioning to reduce window size |
| **Limit** | Limit | Limits the number of rows returned | Low CPU | Push LIMIT early in the plan |
| **Offset** | Offset | Skips a number of rows | High CPU (if large offset) | Avoid large offsets (use keyset pagination) |

#### **3. EXPLAIN Output Example**

**Query**:
```sql
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

**EXPLAIN Output**:
```text
GlobalLimit [limit=100]
  LocalLimit [limit=100]
    Sort [order_by=[total_amount DESC]]
      Aggregate [group_by=[customer_id, name]]
        HashJoin [type=INNER]
          TableScan [table=customers, filters=[region='US']]
          Filter [filter=order_date > '2023-01-01']
            TableScan [table=orders]
```

**Interpretation**:
1. **TableScan [table=customers, filters=[region='US']]**: Scans the `customers` table with a filter on `region='US'`.
2. **TableScan [table=orders]**: Scans the `orders` table.
3. **Filter [filter=order_date > '2023-01-01']**: Filters the `orders` table to only include orders after '2023-01-01'.
4. **HashJoin [type=INNER]**: Joins the filtered `customers` and `orders` tables using a hash join.
5. **Aggregate [group_by=[customer_id, name]]**: Groups the joined data by `customer_id` and `name`, and aggregates the `order_count` and `total_amount`.
6. **Sort [order_by=[total_amount DESC]]**: Sorts the aggregated data by `total_amount` in descending order.
7. **LocalLimit [limit=100]**: Limits the sorted data to 100 rows per partition.
8. **GlobalLimit [limit=100]**: Limits the final result to 100 rows.

#### **4. EXPLAIN Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use EXPLAIN Before Running Queries** | Analyze the query plan before executing expensive queries | `EXPLAIN SELECT * FROM my_table WHERE ...` |
| **Check for Full Table Scans** | Look for `TableScan` without filters | `EXPLAIN SELECT * FROM my_table` (no WHERE clause) |
| **Check for Expensive Operators** | Look for `Join`, `Aggregate`, `Sort` with high estimated rows | `EXPLAIN SELECT ... JOIN ... GROUP BY ...` |
| **Check for Partition Pruning** | Verify that `TableScan` includes `filters` or `partitions` | `EXPLAIN SELECT * FROM my_table WHERE date = '2023-01-01'` |
| **Check for Column Pruning** | Verify that `Project` only includes needed columns | `EXPLAIN SELECT col1, col2 FROM my_table` |
| **Use WITH COST ESTIMATION** | Check the estimated cost of the query | `EXPLAIN SELECT * FROM my_table WITH COST ESTIMATION = TRUE` |
| **Use WITH STATISTICS** | Check the estimated rows and bytes for each operator | `EXPLAIN SELECT * FROM my_table WITH STATISTICS = TRUE` |
| **Compare Query Plans** | Compare plans before and after optimization | `EXPLAIN SELECT * FROM my_table WHERE ...` (before and after) |
| **Check for Spill** | Look for `SpillToDisk` or `SpillToRemote` in the plan | `EXPLAIN SELECT * FROM my_table ...` (check for spill operators) |

**Examples**:

```sql
-- Check for full table scans
EXPLAIN SELECT * FROM my_table;

-- Check for expensive operators
EXPLAIN SELECT * FROM my_table JOIN other_table ON my_table.id = other_table.id
GROUP BY my_table.region
ORDER BY SUM(other_table.value) DESC;

-- Check for partition pruning
EXPLAIN SELECT * FROM my_table WHERE date = '2023-01-01';

-- Check for column pruning
EXPLAIN SELECT col1, col2 FROM my_table;

-- Check with cost estimation
EXPLAIN SELECT * FROM my_table WHERE id = 1
WITH COST ESTIMATION = TRUE;

-- Check with statistics
EXPLAIN SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7
WITH STATISTICS = TRUE;
```

### **B. Query Profile Analysis**

#### **1. What is Query Profile?**

The **Query Profile** provides **detailed execution metrics** for a specific query, including:
- **Step-by-step execution** (operators, rows, bytes, time)
- **Spill metrics** (spill to disk, spill to remote)
- **Parallelism** (number of threads, partitions processed)
- **Resource usage** (CPU, memory)

**Query Profile Syntax**:
```sql
-- Get query profile for a specific query ID
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'));

-- Get query profile for the last query
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE(LAST_QUERY_ID()));

-- Get query profile with specific columns
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
ORDER BY
    step_id;
```

#### **2. Query Profile Output Example**

**Query**:
```sql
SELECT * FROM my_table WHERE date > CURRENT_DATE() - 7;
```

**Query Profile Output**:

| **Column** | **Value** | **Description** |
|------------|-----------|-----------------|
| query_id | 01a2b3c4-d5e6-78f9 | Unique identifier for the query |
| step_id | 1 | Unique identifier for each step in the query plan |
| parent_step_id | NULL | Parent step ID (NULL for root steps) |
| operation | TableScan | Type of operation |
| options | table_name=my_table, filters=[date > '2023-05-24'] | Operation-specific options |
| rows_produced | 1000000 | Number of rows produced by the step |
| bytes_scanned | 1073741824 | Bytes scanned by the step (1GB) |
| execution_time | 5000 | Time spent in the step (5 seconds) |
| spill_to_disk | 0 | Bytes spilled to disk |
| spill_to_remote | 0 | Bytes spilled to remote storage |
| threads | 8 | Number of threads used |
| partitions_processed | 100 | Number of partitions processed |
| bytes_processed | 2147483648 | Bytes processed by the step (2GB) |

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
   - Look for steps with **high execution_time** (bottlenecks)
   - Look for steps with **high bytes_scanned** (inefficient scanning)
   - Look for **spill to disk/remote** (memory pressure)
   - Look for **low parallelism** (threads)

4. **Optimize Query**:
   - **High bytes_scanned**: Add filters, use clustering, use partition pruning
   - **High execution_time for joins**: Use proper join type, reduce join size
   - **Spill to disk/remote**: Increase warehouse size, reduce data volume
   - **Low parallelism**: Check warehouse concurrency, query complexity

#### **4. Query Profile Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Check for High Execution Time Steps** | Identify bottlenecks in the query plan | Look for steps with execution_time > 1000ms |
| **Check for High Bytes Scanned** | Identify inefficient scanning | Look for steps with bytes_scanned > 1GB |
| **Check for Spill** | Identify memory pressure | Look for steps with spill_to_disk > 0 or spill_to_remote > 0 |
| **Check for Low Parallelism** | Identify underutilized resources | Look for steps with threads < warehouse size |
| **Check for High Rows Produced** | Identify large intermediate results | Look for steps with rows_produced > 1M |
| **Compare Profiles** | Compare profiles before and after optimization | Compare QUERY_PROFILE for the same query |
| **Monitor Profile Trends** | Track profile metrics over time | Store and analyze QUERY_PROFILE data |
| **Set Up Profile Alerts** | Alert on profile anomalies | Set up alerts for high execution_time or spill |

**Examples**:

```sql
-- Get query profile for a slow query
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('01a2b3c4-d5e6-78f9'));

-- Get steps with high execution time
SELECT
    step_id,
    operation,
    execution_time,
    rows_produced,
    bytes_scanned
FROM
    TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('01a2b3c4-d5e6-78f9'))
WHERE
    execution_time > 1000  -- >1 second
ORDER BY
    execution_time DESC;

-- Get steps with high bytes scanned
SELECT
    step_id,
    operation,
    bytes_scanned,
    rows_produced
FROM
    TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('01a2b3c4-d5e6-78f9'))
WHERE
    bytes_scanned > 100000000  -- >100MB
ORDER BY
    bytes_scanned DESC;

-- Get steps with spill
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

### **C. Warehouse Optimization for Query Performance**

#### **1. Warehouse Sizing**

**Warehouse Size Selection**:

| **Size** | **Compute (Credits/Hour)** | **Memory (GB)** | **Max Threads** | **Best For** | **Credit Cost/Hour** |
|----------|----------------------------|-----------------|-----------------|--------------|----------------------|
| **X-Small** | 1 | 16 | 8 | Development, testing, small queries | 0.28 |
| **Small** | 2 | 32 | 16 | Small workloads, medium queries | 0.56 |
| **Medium** | 4 | 64 | 32 | Medium workloads, large queries | 1.12 |
| **Large** | 8 | 128 | 64 | Large workloads, complex queries | 2.24 |
| **X-Large** | 16 | 256 | 128 | Very large workloads, high concurrency | 4.48 |
| **2X-Large** | 32 | 512 | 256 | Extremely large workloads | 8.96 |
| **3X-Large** | 64 | 1024 | 512 | Massive workloads | 17.92 |
| **4X-Large** | 128 | 2048 | 1024 | Largest workloads | 35.84 |

**Warehouse Sizing Best Practices**:

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Start Small and Scale Up** | Begin with a smaller warehouse and monitor performance | `CREATE WAREHOUSE my_wh WAREHOUSE_SIZE = 'SMALL'` |
| **Right-Size for Workload** | Use the smallest warehouse that meets performance requirements | `ALTER WAREHOUSE my_wh SET WAREHOUSE_SIZE = 'MEDIUM'` |
| **Use Multi-Cluster for High Concurrency** | Use multi-cluster warehouses for high concurrency workloads | `CREATE WAREHOUSE my_wh MAX_CLUSTER_COUNT = 4` |
| **Set Auto-Suspend for Idle Warehouses** | Suspend warehouses when idle to save costs | `ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = 600` |
| **Set Auto-Resume for Interactive Workloads** | Resume warehouses automatically for interactive queries | `ALTER WAREHOUSE my_wh SET AUTO_RESUME = TRUE` |
| **Monitor Warehouse Utilization** | Check WAREHOUSE_LOAD_HISTORY for utilization | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY` |
| **Adjust Warehouse Size Based on Usage** | Increase or decrease size based on actual usage | `ALTER WAREHOUSE my_wh SET WAREHOUSE_SIZE = 'LARGE'` |

#### **2. Multi-Cluster Warehouses**

**Multi-Cluster Warehouse Configuration**:

| **Parameter** | **Description** | **Default** | **Valid Values** | **Best Practice** |
|---------------|-----------------|-------------|------------------|-------------------|
| **WAREHOUSE_SIZE** | Size of each cluster | `MEDIUM` | `XSMALL`, `SMALL`, `MEDIUM`, `LARGE`, `XLARGE`, `XXLARGE`, `XXXLARGE`, `XXXXLARGE` | Use the smallest size that meets performance requirements |
| **MAX_CLUSTER_COUNT** | Maximum number of clusters | `1` | 1-10 | Start with 2, increase as needed |
| **MIN_CLUSTER_COUNT** | Minimum number of clusters | `1` | 1-10 | Use 1 for most workloads, increase for consistent high load |
| **SCALING_POLICY** | Scaling policy | `STANDARD` | `STANDARD`, `ECONOMY` | Use `STANDARD` for performance-critical workloads, `ECONOMY` for cost-sensitive workloads |

**Multi-Cluster Warehouse Best Practices**:

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Start with MAX_CLUSTER_COUNT = 2** | Test with 2 clusters before scaling up | `CREATE WAREHOUSE my_wh MAX_CLUSTER_COUNT = 2` |
| **Use STANDARD Scaling for Critical Workloads** | Ensure performance SLAs are met | `ALTER WAREHOUSE my_wh SET SCALING_POLICY = 'STANDARD'` |
| **Use ECONOMY Scaling for Cost-Sensitive Workloads** | Save costs for non-critical workloads | `ALTER WAREHOUSE my_wh SET SCALING_POLICY = 'ECONOMY'` |
| **Set MIN_CLUSTER_COUNT = 1** | Avoid idle clusters to save costs | `ALTER WAREHOUSE my_wh SET MIN_CLUSTER_COUNT = 1` |
| **Monitor Cluster Usage** | Check WAREHOUSE_LOAD_HISTORY for cluster usage | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY` |
| **Set AUTO_SUSPEND for Idle Warehouses** | Suspend warehouses when idle to save costs | `ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = 600` |

**Examples**:

```sql
-- Create a multi-cluster warehouse with STANDARD scaling
CREATE WAREHOUSE my_mc_wh
  WAREHOUSE_SIZE = 'LARGE'
  MAX_CLUSTER_COUNT = 4
  MIN_CLUSTER_COUNT = 1
  SCALING_POLICY = 'STANDARD'
  AUTO_SUSPEND = 1800
  AUTO_RESUME = TRUE;

-- Create a multi-cluster warehouse with ECONOMY scaling
CREATE WAREHOUSE my_cost_effective_wh
  WAREHOUSE_SIZE = 'MEDIUM'
  MAX_CLUSTER_COUNT = 8
  MIN_CLUSTER_COUNT = 2
  SCALING_POLICY = 'ECONOMY'
  AUTO_SUSPEND = 1800
  AUTO_RESUME = TRUE;

-- Monitor multi-cluster warehouse usage
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
    warehouse_name = 'my_mc_wh'
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;
```

#### **3. Query Prioritization**

**Priority Levels**:

| **Priority** | **Description** | **When to Use** | **Example** |
|--------------|-----------------|-----------------|-------------|
| **HIGH** | Highest priority; runs before MEDIUM and LOW | Critical production queries, SLAs | `ALTER WAREHOUSE my_wh SET QUERY_PRIORITY = 'HIGH'` |
| **MEDIUM** | Default priority; runs after HIGH, before LOW | General queries, reporting | Default |
| **LOW** | Lowest priority; runs after HIGH and MEDIUM | Ad-hoc queries, development | `ALTER SESSION SET QUERY_PRIORITY = 'LOW'` |

**Query Prioritization Best Practices**:

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use HIGH Priority for Critical Queries** | Prioritize SLA-bound queries | `ALTER WAREHOUSE prod_wh SET QUERY_PRIORITY = 'HIGH'` |
| **Use MEDIUM Priority for General Queries** | Default priority for most queries | Default |
| **Use LOW Priority for Ad-Hoc Queries** | Deprioritize development and ad-hoc queries | `ALTER SESSION SET QUERY_PRIORITY = 'LOW'` |
| **Set Query Timeouts** | Prevent long-running queries from blocking others | `ALTER WAREHOUSE my_wh SET STATEMENT_TIMEOUT_IN_SECONDS = 300` |
| **Set Queue Timeouts** | Prevent queries from waiting too long in the queue | `ALTER WAREHOUSE my_wh SET STATEMENT_QUEUE_TIMEOUT_IN_SECONDS = 60` |
| **Monitor Queue Status** | Check WAREHOUSE_MONITOR for queued queries | `SELECT * FROM SNOWFLAKE.INFORMATION_SCHEMA.WAREHOUSE_MONITOR` |
| **Use Separate Warehouses for Different Priorities** | Avoid priority conflicts | `CREATE WAREHOUSE high_priority_wh`, `CREATE WAREHOUSE low_priority_wh` |

**Examples**:

```sql
-- Set default priority for a warehouse
ALTER WAREHOUSE my_wh SET QUERY_PRIORITY = 'HIGH';

-- Set priority for the current session
ALTER SESSION SET QUERY_PRIORITY = 'HIGH';

-- Set priority for a specific query (using hint)
SELECT * FROM my_table /*+ PRIORITY(HIGH) */;

-- Set priority for a role
ALTER ROLE high_priority_role SET QUERY_PRIORITY = 'HIGH';
GRANT ROLE high_priority_role TO USER my_user;
```

### **D. Caching Strategies**

#### **1. Result Caching**

**Definition**:
Snowflake **automatically caches query results** for **24 hours** (configurable) if:
- The **query text** is identical
- The **underlying data** has not changed
- The **user's permissions** are the same

**Result Cache Types**:

| **Cache Type** | **Description** | **TTL** | **Scope** |
|----------------|-----------------|---------|-----------|
| **Result Cache** | Caches query results | 24 hours (configurable) | Per-user, per-query |
| **Metadata Cache** | Caches table metadata (e.g., statistics, schema) | Session | Per-session |
| **Local Disk Cache** | Caches frequently accessed data in SSD | Session | Per-warehouse |

**Result Cache Configuration**:

| **Parameter** | **Description** | **Default** | **Valid Values** | **Example** |
|---------------|-----------------|-------------|------------------|-------------|
| **USE_CACHED_RESULTS** | Enable/disable result caching | `TRUE` | `TRUE`, `FALSE` | `ALTER SESSION SET USE_CACHED_RESULTS = TRUE` |
| **RESULT_CACHE_TTL** | Result cache TTL (in seconds) | `86400` (24 hours) | 0-86400 | `ALTER SESSION SET RESULT_CACHE_TTL = 3600` (1 hour) |

**Result Cache Best Practices**:

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use for Repetitive Queries** | Cache results for queries run frequently | Dashboards, reports |
| **Set Appropriate TTL** | Adjust TTL based on data freshness requirements | `ALTER SESSION SET RESULT_CACHE_TTL = 3600` |
| **Monitor Cache Usage** | Check used_cached_result in QUERY_HISTORY | `SELECT used_cached_result FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` |
| **Avoid Cache Invalidation** | Minimize changes to underlying data | Batch updates instead of frequent small updates |
| **Use for Read-Only Workloads** | Cache works best for read-only workloads | BI tools, reporting |
| **Disable for Unique Queries** | Disable caching for unique queries | `ALTER SESSION SET USE_CACHED_RESULTS = FALSE` |

**Examples**:

```sql
-- Enable result caching
ALTER SESSION SET USE_CACHED_RESULTS = TRUE;

-- Set cache TTL to 1 hour
ALTER SESSION SET RESULT_CACHE_TTL = 3600;

-- Check if a query used cached results
SELECT
    query_id,
    used_cached_result,
    execution_time,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;

-- Disable result caching for a session
ALTER SESSION SET USE_CACHED_RESULTS = FALSE;
```

#### **2. Metadata Caching**

**Definition**:
Snowflake caches **table metadata** (e.g., statistics, schema) to improve query performance.

**Metadata Cache Best Practices**:

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use Metadata Caching** | Metadata is automatically cached | No configuration needed |
| **Monitor Metadata Cache Hits** | Check metadata cache usage | Not directly visible, but improves performance |
| **Avoid Frequent DDL Changes** | DDL changes invalidate metadata cache | Batch DDL changes |
| **Use STATISTICS** | Ensure statistics are up-to-date | `ALTER TABLE my_table UPDATE STATISTICS` |

#### **3. Local Disk Caching**

**Definition**:
Snowflake caches **frequently accessed data** in **local SSD** for each warehouse.

**Local Disk Cache Best Practices**:

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use Local Disk Cache** | Cache is automatically managed | No configuration needed |
| **Monitor Cache Hits** | Check cache usage in QUERY_PROFILE | Look for cached data in QUERY_PROFILE |
| **Use Larger Warehouses for More Cache** | Larger warehouses have more local disk | `ALTER WAREHOUSE my_wh SET WAREHOUSE_SIZE = 'LARGE'` |
| **Avoid Frequent Cache Invalidation** | Cache is invalidated when data changes | Batch data changes |

## **5. Advanced Query Optimization Techniques**

### **A. Query Rewriting**

#### **1. Automatic Query Rewriting**

Snowflake automatically applies the following rewrites:
- **Predicate Pushdown**: Moves filters closer to the data source
- **Column Pruning**: Removes unreferenced columns from scans
- **Partition Pruning**: Skips irrelevant micro-partitions
- **Join Reordering**: Reorders joins to minimize intermediate results
- **Common Subexpression Elimination (CSE)**: Reuses subqueries or expressions
- **Constant Folding**: Evaluates constant expressions at compile time
- **Query Folding**: Combines nested views or subqueries into a single query

#### **2. Manual Query Rewriting Techniques**

| **Technique** | **Description** | **Before** | **After** | **Performance Impact** |
|---------------|-----------------|------------|-----------|-------------------------|
| **Predicate Pushdown** | Move filters early in the query | `SELECT * FROM (SELECT * FROM my_table) WHERE date > '2023-01-01'` | `SELECT * FROM my_table WHERE date > '2023-01-01'` | ⬆️ Reduces data scanned |
| **Column Pruning** | Select only needed columns | `SELECT * FROM my_table` | `SELECT col1, col2 FROM my_table` | ⬆️ Reduces I/O and memory |
| **Join Reordering** | Reorder joins to minimize intermediate results | `SELECT * FROM large_table JOIN small_table ON ...` | `SELECT * FROM small_table JOIN large_table ON ...` | ⬆️ Reduces join size |
| **Subquery to Join** | Replace correlated subqueries with joins | `SELECT * FROM table1 WHERE id IN (SELECT id FROM table2)` | `SELECT * FROM table1 JOIN table2 ON table1.id = table2.id` | ⬆️ Improves performance |
| **NOT IN to NOT EXISTS** | Replace NOT IN with NOT EXISTS | `SELECT * FROM table1 WHERE id NOT IN (SELECT id FROM table2)` | `SELECT * FROM table1 WHERE NOT EXISTS (SELECT 1 FROM table2 WHERE table2.id = table1.id)` | ⬆️ Avoids NULL handling issues |
| **EXISTS to IN** | Replace EXISTS with IN for small subqueries | `SELECT * FROM table1 WHERE EXISTS (SELECT 1 FROM table2 WHERE table2.id = table1.id)` | `SELECT * FROM table1 WHERE id IN (SELECT id FROM table2)` | ⬆️ Simpler execution plan |
| **Avoid SELECT *** | Select only needed columns | `SELECT * FROM my_table` | `SELECT col1, col2 FROM my_table` | ⬆️ Reduces I/O and memory |
| **Use WHERE Instead of HAVING** | Filter early with WHERE | `SELECT col1, COUNT(*) FROM my_table GROUP BY col1 HAVING COUNT(*) > 100` | `SELECT col1, COUNT(*) FROM (SELECT col1 FROM my_table WHERE ...) GROUP BY col1` | ⬆️ Reduces data scanned |
| **Use Semi-Join** | Use EXISTS or IN for filtering | `SELECT DISTINCT t1.* FROM table1 t1 JOIN table2 t2 ON t1.id = t2.id` | `SELECT * FROM table1 WHERE id IN (SELECT id FROM table2)` | ⬆️ Reduces duplicate rows |
| **Use Anti-Join** | Use NOT EXISTS or NOT IN for exclusion | `SELECT t1.* FROM table1 t1 LEFT JOIN table2 t2 ON t1.id = t2.id WHERE t2.id IS NULL` | `SELECT * FROM table1 WHERE id NOT IN (SELECT id FROM table2)` | ⬆️ Simpler execution plan |

#### **3. Query Rewriting Examples**

```sql
-- Example 1: Predicate Pushdown
-- Before
SELECT * FROM (SELECT * FROM my_table) WHERE date > '2023-01-01';

-- After
SELECT * FROM my_table WHERE date > '2023-01-01';

-- Example 2: Column Pruning
-- Before
SELECT * FROM my_table;

-- After
SELECT id, name, value FROM my_table;

-- Example 3: Join Reordering
-- Before
SELECT * FROM large_table JOIN small_table ON large_table.id = small_table.id;

-- After
SELECT * FROM small_table JOIN large_table ON small_table.id = large_table.id;

-- Example 4: Subquery to Join
-- Before
SELECT * FROM table1 WHERE id IN (SELECT id FROM table2);

-- After
SELECT * FROM table1 JOIN table2 ON table1.id = table2.id;

-- Example 5: NOT IN to NOT EXISTS
-- Before
SELECT * FROM table1 WHERE id NOT IN (SELECT id FROM table2);

-- After
SELECT * FROM table1 WHERE NOT EXISTS (SELECT 1 FROM table2 WHERE table2.id = table1.id);

-- Example 6: EXISTS to IN
-- Before
SELECT * FROM table1 WHERE EXISTS (SELECT 1 FROM table2 WHERE table2.id = table1.id);

-- After
SELECT * FROM table1 WHERE id IN (SELECT id FROM table2);

-- Example 7: WHERE Instead of HAVING
-- Before
SELECT col1, COUNT(*) FROM my_table GROUP BY col1 HAVING COUNT(*) > 100;

-- After
SELECT col1, COUNT(*) FROM (SELECT col1 FROM my_table WHERE date > '2023-01-01') GROUP BY col1;

-- Example 8: Semi-Join
-- Before
SELECT DISTINCT t1.* FROM table1 t1 JOIN table2 t2 ON t1.id = t2.id;

-- After
SELECT * FROM table1 WHERE id IN (SELECT id FROM table2);
```

### **B. Materialized Views**

#### **1. Materialized View Overview**

**Definition**:
**Materialized Views (MVs)** in Snowflake are **pre-computed query results** that are **stored as tables** and **automatically refreshed** when the underlying data changes. MVs are ideal for **repetitive, expensive queries** that can benefit from **pre-computation**.

**How Materialized Views Work**:
1. **Creation**: Define a materialized view with a query (e.g., `SELECT * FROM table1 JOIN table2 ON ...`).
2. **Initial Population**: Snowflake executes the query and stores the results as a table.
3. **Refresh**: Snowflake automatically refreshes the MV when the underlying data changes.
4. **Querying**: Query the MV like a regular table (e.g., `SELECT * FROM my_mv`).
5. **Optimization**: Snowflake rewrites queries to use the MV when possible.

**Materialized View Types**:

| **Type** | **Description** | **Refresh Strategy** | **Use Case** |
|----------|-----------------|----------------------|--------------|
| **Standard MV** | Pre-computed query results | Automatic, Manual, Scheduled | Repetitive queries |
| **Secure MV** | MV with row-level security (RLS) and data masking | Automatic, Manual, Scheduled | Secure data access |
| **Non-Secure MV** | MV without RLS or data masking | Automatic, Manual, Scheduled | General use |

#### **2. Materialized View Configuration**

**Create a Materialized View**:
```sql
-- Create a simple materialized view
CREATE MATERIALIZED VIEW my_mv AS
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
    o.order_date > CURRENT_DATE() - 30
GROUP BY
    c.customer_id, c.name;

-- Create a secure materialized view
CREATE SECURE MATERIALIZED VIEW my_secure_mv AS
SELECT * FROM my_table WHERE sensitive_data IS NULL;

-- Create a materialized view with clustering
CREATE MATERIALIZED VIEW my_clustered_mv
CLUSTER BY (date, region) AS
SELECT * FROM my_table WHERE date > CURRENT_DATE() - 30;
```

**Refresh a Materialized View**:
```sql
-- Manual refresh
ALTER MATERIALIZED VIEW my_mv REFRESH;

-- Scheduled refresh (using Tasks)
CREATE TASK refresh_my_mv
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON 0 * * * * America/Los_Angeles'  -- Every hour
AS
  ALTER MATERIALIZED VIEW my_mv REFRESH;
```

**Drop a Materialized View**:
```sql
DROP MATERIALIZED VIEW IF EXISTS my_mv;
```

#### **3. Materialized View Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use for Repetitive Queries** | Create MVs for queries run frequently | `CREATE MATERIALIZED VIEW my_mv AS SELECT * FROM my_table WHERE date > CURRENT_DATE()` |
| **Use Automatic Refresh** | Let Snowflake refresh the MV automatically | `CREATE MATERIALIZED VIEW my_mv AS SELECT ...` (default) |
| **Use Manual Refresh for Large MVs** | Refresh large MVs on a schedule | `ALTER MATERIALIZED VIEW my_mv REFRESH` |
| **Use Scheduled Refresh for Batch Workloads** | Refresh MVs during off-peak hours | `CREATE TASK refresh_my_mv AS ALTER MATERIALIZED VIEW my_mv REFRESH` |
| **Monitor Refresh Performance** | Check MATERIALIZED_VIEW_REFRESH_HISTORY for errors | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_REFRESH_HISTORY` |
| **Use Secure MVs for Sensitive Data** | Apply RLS and data masking to MVs | `CREATE SECURE MATERIALIZED VIEW my_mv AS SELECT ...` |
| **Avoid Overlapping MVs** | Avoid creating MVs with redundant data | Use a single MV for multiple queries |
| **Drop Unused MVs** | Remove MVs that are no longer needed | `DROP MATERIALIZED VIEW my_mv` |
| **Use MV for Star Schema Fact Tables** | Pre-compute fact tables for BI tools | `CREATE MATERIALIZED VIEW fact_sales AS SELECT * FROM sales JOIN dim_product ON ...` |
| **Combine with Clustering** | Cluster MVs for better query performance | `CREATE MATERIALIZED VIEW my_mv CLUSTER BY (date) AS SELECT ...` |

**Examples**:

```sql
-- Example 1: Materialized view for daily sales
CREATE MATERIALIZED VIEW daily_sales_mv AS
SELECT
    DATE_TRUNC('DAY', order_date) AS day,
    region,
    product_category,
    SUM(amount) AS total_sales,
    COUNT(*) AS order_count
FROM
    orders
WHERE
    order_date > CURRENT_DATE() - 30
GROUP BY
    DATE_TRUNC('DAY', order_date), region, product_category;

-- Example 2: Materialized view for customer analytics
CREATE MATERIALIZED VIEW customer_analytics_mv AS
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
GROUP BY
    c.customer_id, c.name, c.region;

-- Example 3: Materialized view with clustering
CREATE MATERIALIZED VIEW sales_by_date_mv
CLUSTER BY (day, region) AS
SELECT
    DATE_TRUNC('DAY', order_date) AS day,
    region,
    product_id,
    SUM(amount) AS total_sales
FROM
    orders
WHERE
    order_date > CURRENT_DATE() - 90
GROUP BY
    DATE_TRUNC('DAY', order_date), region, product_id;

-- Example 4: Secure materialized view
CREATE SECURE MATERIALIZED VIEW secure_sales_mv AS
SELECT
    region,
    SUM(amount) AS total_sales
FROM
    orders
WHERE
    sensitive_flag = FALSE
GROUP BY
    region;

-- Example 5: Scheduled refresh
CREATE TASK refresh_daily_sales_mv
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON 0 2 * * * America/Los_Angeles'  -- 2 AM daily
AS
  ALTER MATERIALIZED VIEW daily_sales_mv REFRESH;
```

#### **4. Materialized View Monitoring**

**Check Materialized View Refresh History**:
```sql
-- Check refresh history for a materialized view
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

-- Check refresh history for all materialized views
SELECT
    view_name,
    refresh_time,
    status,
    rows_refreshed,
    bytes_refreshed
FROM
    SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_REFRESH_HISTORY
WHERE
    refresh_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    refresh_time DESC;
```

**Check Materialized View Storage Usage**:
```sql
-- Check storage usage for a materialized view
SELECT
    view_name,
    storage_bytes / 1024 / 1024 / 1024 AS storage_gb
FROM
    SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_STORAGE
WHERE
    view_name = 'MY_MV';

-- Check storage usage for all materialized views
SELECT
    view_name,
    storage_bytes / 1024 / 1024 / 1024 AS storage_gb
FROM
    SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_STORAGE
ORDER BY
    storage_gb DESC;
```

### **C. Approximate Functions**

#### **1. Approximate Function Overview**

Snowflake provides **approximate functions** that trade **precision for performance**. These functions are ideal for **large datasets** where exact precision is not required.

**Approximate Functions**:

| **Function** | **Description** | **Performance** | **Accuracy** | **Use Case** |
|--------------|-----------------|-----------------|--------------|--------------|
| **APPROX_COUNT_DISTINCT(col)** | Approximate count of distinct values | ⭐⭐⭐⭐⭐ | ~95-99% | Large datasets, distinct counts |
| **APPROX_QUANTILE(col, p)** | Approximate quantile (e.g., median) | ⭐⭐⭐⭐ | ~95-99% | Large datasets, percentiles |
| **APPROX_TOP_K(col, k)** | Approximate top K values | ⭐⭐⭐⭐ | ~95-99% | Large datasets, top K queries |
| **APPROX_TOP_SUM(col, k)** | Approximate top K values by sum | ⭐⭐⭐⭐ | ~95-99% | Large datasets, top K by sum |
| **APPROX_PERCENTILE(col, p)** | Approximate percentile | ⭐⭐⭐⭐ | ~95-99% | Large datasets, percentiles |
| **APPROX_MEDIAN(col)** | Approximate median | ⭐⭐⭐⭐ | ~95-99% | Large datasets, median |
| **HLL(col)** | HyperLogLog for distinct counts | ⭐⭐⭐⭐⭐ | ~95-99% | Very large datasets, distinct counts |
| **HLL_ADD(aggregate, hll)** | Add HLL sketches | ⭐⭐⭐⭐⭐ | ~95-99% | Combining HLL sketches |
| **HLL_MERGE(aggregate, hll)** | Merge HLL sketches | ⭐⭐⭐⭐⭐ | ~95-99% | Combining HLL sketches |

#### **2. Approximate Function Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use for Large Datasets** | Approximate functions are faster for large datasets | `SELECT APPROX_COUNT_DISTINCT(user_id) FROM large_table` |
| **Use for Aggregations** | Use approximate functions in GROUP BY clauses | `SELECT region, APPROX_COUNT_DISTINCT(user_id) FROM large_table GROUP BY region` |
| **Use for Percentiles** | Use APPROX_QUANTILE for percentiles | `SELECT APPROX_QUANTILE(value, 0.5) FROM large_table` |
| **Use for Top K Queries** | Use APPROX_TOP_K for top K queries | `SELECT APPROX_TOP_K(product_id, 10) FROM large_table` |
| **Combine with Exact Functions** | Use approximate functions where possible, exact where needed | `SELECT region, APPROX_COUNT_DISTINCT(user_id), SUM(revenue) FROM large_table GROUP BY region` |
| **Monitor Accuracy** | Check the accuracy of approximate functions for your data | Compare with exact functions |
| **Use HLL for Very Large Datasets** | HLL is optimized for very large datasets | `SELECT HLL(user_id) FROM very_large_table` |
| **Use APPROX_COUNT_DISTINCT for Distinct Counts** | APPROX_COUNT_DISTINCT is faster than COUNT(DISTINCT) | `SELECT APPROX_COUNT_DISTINCT(user_id) FROM large_table` |

**Examples**:

```sql
-- Example 1: APPROX_COUNT_DISTINCT
-- Exact count (slow for large tables)
SELECT COUNT(DISTINCT user_id) FROM large_table;

-- Approximate count (fast for large tables)
SELECT APPROX_COUNT_DISTINCT(user_id) FROM large_table;

-- Example 2: APPROX_QUANTILE
-- Exact median (slow for large tables)
SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY value) FROM large_table;

-- Approximate median (fast for large tables)
SELECT APPROX_QUANTILE(value, 0.5) FROM large_table;

-- Example 3: APPROX_TOP_K
-- Exact top 10 (slow for large tables)
SELECT product_id, SUM(revenue) AS total_revenue
FROM large_table
GROUP BY product_id
ORDER BY total_revenue DESC
LIMIT 10;

-- Approximate top 10 (fast for large tables)
SELECT APPROX_TOP_SUM(product_id, 10, revenue) FROM large_table;

-- Example 4: HLL for distinct counts
-- Create an HLL sketch
SELECT HLL(user_id) AS user_sketch FROM large_table;

-- Combine HLL sketches
SELECT HLL_MERGE(user_sketch) AS combined_sketch
FROM (
    SELECT HLL(user_id) AS user_sketch FROM table1
    UNION ALL
    SELECT HLL(user_id) AS user_sketch FROM table2
);

-- Get approximate count from HLL sketch
SELECT HLL_CARDINALITY(combined_sketch) AS approx_distinct_count
FROM (
    SELECT HLL_MERGE(user_sketch) AS combined_sketch
    FROM (
        SELECT HLL(user_id) AS user_sketch FROM table1
        UNION ALL
        SELECT HLL(user_id) AS user_sketch FROM table2
    )
);
```

### **D. Query Hints**

#### **1. Query Hint Overview**

Snowflake supports **query hints** to provide **guidance to the query optimizer**. Hints are specified as **comments** in the SQL query and are **not guaranteed** to be followed (the optimizer may still choose a different plan if it determines it's better).

**Query Hint Syntax**:
```sql
-- Single-line hint
SELECT * FROM my_table /*+ HINT */;

-- Multi-line hint
SELECT * FROM my_table
/*+
  HINT1
  HINT2
*/
;

-- Hint for a specific operator
SELECT * FROM my_table /*+ LEADING(t1 t2) */ JOIN other_table t2 ON my_table.id = t2.id;
```

#### **2. Supported Query Hints**

| **Hint** | **Description** | **Example** | **When to Use** |
|----------|-----------------|-------------|-----------------|
| **LEADING(table_list)** | Specifies the join order | `/*+ LEADING(t1 t2 t3) */` | When you know the optimal join order |
| **INDEX(table index)** | Specifies the index to use | `/*+ INDEX(t1 idx1) */` | When you know the optimal index |
| **NO_INDEX(table)** | Disables index usage | `/*+ NO_INDEX(t1) */` | When indexes are causing performance issues |
| **HASH_JOIN(table)** | Forces a hash join | `/*+ HASH_JOIN(t2) */` | When you know a hash join is optimal |
| **MERGE_JOIN(table)** | Forces a merge join | `/*+ MERGE_JOIN(t2) */` | When you know a merge join is optimal |
| **BROADCAST_JOIN(table)** | Forces a broadcast join | `/*+ BROADCAST_JOIN(t2) */` | When you know a broadcast join is optimal |
| **NESTED_LOOP_JOIN(table)** | Forces a nested loop join | `/*+ NESTED_LOOP_JOIN(t2) */` | When you know a nested loop join is optimal |
| **PRIORITY(HIGH/MEDIUM/LOW)** | Sets query priority | `/*+ PRIORITY(HIGH) */` | For critical queries |
| **STATEMENT_TIMEOUT(seconds)** | Sets statement timeout | `/*+ STATEMENT_TIMEOUT(300) */` | For long-running queries |
| **MAX_EXECUTION_TIME(seconds)** | Sets maximum execution time | `/*+ MAX_EXECUTION_TIME(300) */` | For long-running queries |
| **USE_HASH(table)** | Uses hash partitioning | `/*+ USE_HASH(t1) */` | For specific partitioning |
| **USE_RANGE(table)** | Uses range partitioning | `/*+ USE_RANGE(t1) */` | For specific partitioning |
| **MATERIALIZE(subquery)** | Materializes a subquery | `/*+ MATERIALIZE(subq) */` | For subqueries referenced multiple times |

#### **3. Query Hint Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use Hints Sparingly** | Only use hints when necessary | Avoid hints unless you've identified a performance issue |
| **Test Hints Before and After** | Compare performance with and without hints | Run EXPLAIN with and without hints |
| **Monitor Hint Usage** | Check if hints are being followed | Check QUERY_PROFILE for execution plan |
| **Use LEADING for Join Order** | Specify join order for complex joins | `/*+ LEADING(t1 t2 t3) */` |
| **Use INDEX for Specific Indexes** | Specify indexes for tables with multiple indexes | `/*+ INDEX(t1 idx1) */` |
| **Use HASH_JOIN for Large Tables** | Force hash join for large tables | `/*+ HASH_JOIN(t2) */` |
| **Use BROADCAST_JOIN for Small Tables** | Force broadcast join for small tables | `/*+ BROADCAST_JOIN(t2) */` |
| **Use PRIORITY for Critical Queries** | Set priority for critical queries | `/*+ PRIORITY(HIGH) */` |
| **Use MATERIALIZE for Subqueries** | Materialize subqueries referenced multiple times | `/*+ MATERIALIZE(subq) */` |
| **Document Hints** | Document why hints are used | Add comments explaining the hint |

**Examples**:

```sql
-- Example 1: LEADING hint for join order
SELECT * FROM t1
/*+ LEADING(t1 t2 t3) */
JOIN t2 ON t1.id = t2.id
JOIN t3 ON t2.id = t3.id;

-- Example 2: HASH_JOIN hint
SELECT * FROM t1
/*+ HASH_JOIN(t2) */
JOIN t2 ON t1.id = t2.id;

-- Example 3: BROADCAST_JOIN hint
SELECT * FROM large_table l
/*+ BROADCAST_JOIN(s) */
JOIN small_table s ON l.id = s.id;

-- Example 4: PRIORITY hint
SELECT * FROM my_table /*+ PRIORITY(HIGH) */ WHERE date > CURRENT_DATE();

-- Example 5: MATERIALIZE hint
SELECT * FROM
    (SELECT * FROM my_table WHERE date > CURRENT_DATE()) /*+ MATERIALIZE(subq) */ subq
JOIN
    other_table o ON subq.id = o.id;

-- Example 6: Multiple hints
SELECT * FROM t1
/*+
  LEADING(t1 t2)
  HASH_JOIN(t2)
  PRIORITY(HIGH)
*/
JOIN t2 ON t1.id = t2.id;
```

### **E. Statistics and Metadata**

#### **1. Statistics in Snowflake**

Snowflake **automatically collects statistics** for:
- **Table row counts**
- **Column min/max values**
- **Column histograms** (for selective columns)
- **Micro-partition metadata** (min/max for each column)

**Statistics Usage**:
- **Partition Pruning**: Uses min/max statistics to skip irrelevant micro-partitions
- **Join Optimization**: Uses histograms to estimate join sizes and choose the best join algorithm
- **Query Rewriting**: Uses statistics to rewrite queries (e.g., predicate pushdown)

#### **2. Managing Statistics**

**Update Statistics Manually**:
```sql
-- Update statistics for a table
ALTER TABLE my_table UPDATE STATISTICS;

-- Update statistics for specific columns
ALTER TABLE my_table UPDATE STATISTICS (col1, col2);
```

**Check Table Statistics**:
```sql
-- Check statistics for a table
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.TABLE_STATISTICS('MY_TABLE'));

-- Check statistics for specific columns
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.COLUMN_STATISTICS('MY_TABLE', 'COL1'));
```

**Check Clustering Information**:
```sql
-- Check clustering information for a table
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.CLUSTERING_INFORMATION('MY_TABLE'));
```

#### **3. Statistics Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Update Statistics for Large Tables** | Ensure statistics are up-to-date for large tables | `ALTER TABLE my_table UPDATE STATISTICS` |
| **Update Statistics After Data Changes** | Update statistics after significant data changes | `ALTER TABLE my_table UPDATE STATISTICS` after bulk loads |
| **Monitor Statistics** | Check TABLE_STATISTICS for missing statistics | `SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.TABLE_STATISTICS('MY_TABLE'))` |
| **Use Clustering for Better Statistics** | Clustering improves partition pruning | `ALTER TABLE my_table CLUSTER BY (date)` |
| **Avoid Skewed Data** | Skewed data can lead to uneven partition sizes | Use RECLUSTER to rebalance data |

## **6. Performance Tuning Workflow**

### **Mermaid: Performance Tuning Workflow**
```mermaid
%% Performance Tuning Workflow
flowchart TD
    A[("Identify Slow Query")] --> B[("Check QUERY_HISTORY")]
    B --> C{High Execution Time?}
    C -->|Yes| D[("Get QUERY_PROFILE")]
    C -->|No| E[("Check Other Metrics")]
    D --> F[("Analyze Bottlenecks")]
    F --> G{High Bytes Scanned?}
    G -->|Yes| H[("Optimize Scanning\n(Clustering, Filters)")]
    G -->|No| I{High CPU Usage?}
    I -->|Yes| J[("Optimize CPU\n(Joins, Aggregations)")]
    I -->|No| K{Spill to Disk/Remote?}
    K -->|Yes| L[("Increase Warehouse Size\n(Reduce Data Volume)")]
    K -->|No| M{High Queue Time?}
    M -->|Yes| N[("Increase Warehouse Size\n(Use Multi-Cluster)")]
    M -->|No| O[("Check Other Issues")]
    E --> P{High Credit Usage?}
    P -->|Yes| Q[("Optimize Queries\n(Right-Size Warehouse)")]
    P -->|No| R{High Latency?}
    R -->|Yes| S[("Check Network\n(Use PrivateLink)")]
    R -->|No| T[("Check External Factors")]

    H --> U[("Add Filters")]
    H --> V[("Use Clustering")]
    H --> W[("Use Partition Pruning")]
    H --> X[("Use Column Pruning")]

    J --> Y[("Optimize Joins")]
    J --> Z[("Use Approximate Functions")]
    J --> AA[("Reduce Data Volume")]

    L --> AB[("Increase Warehouse Size")]
    L --> AC[("Reduce Result Size")]
    L --> AD[("Use LIMIT")]

    N --> AE[("Increase Warehouse Size")]
    N --> AF[("Use Multi-Cluster")]
    N --> AG[("Use Query Prioritization")]

    Q --> AH[("Optimize Queries")]
    Q --> AI[("Use Result Caching")]
    Q --> AJ[("Use Materialized Views")]

    S --> AK[("Use PrivateLink")]
    S --> AL[("Check Cloud Storage")]

    T --> AM[("Check External Systems")]
    T --> AN[("Check Network Connectivity")]

    U --> AO[("Add WHERE Clauses")]
    V --> AP[("Cluster Tables")]
    W --> AQ[("Filter on Partitioned Columns")]
    X --> AR[("Select Only Needed Columns")]
    Y --> AS[("Use Proper Join Types")]
    Z --> AT[("Use APPROX_COUNT_DISTINCT")]
    AA --> AU[("Filter Before Aggregating")]
    AB --> AV[("ALTER WAREHOUSE")]
    AC --> AW[("Use Pagination")]
    AD --> AX[("Add LIMIT Clauses")]
    AE --> AY[("ALTER WAREHOUSE")]
    AF --> AZ[("CREATE WAREHOUSE ... MAX_CLUSTER_COUNT")]
    AG --> BA[("ALTER WAREHOUSE ... QUERY_PRIORITY")]
    AH --> BB[("Rewrite Queries")]
    AI --> BC[("ALTER SESSION SET USE_CACHED_RESULTS")]
    AJ --> BD[("CREATE MATERIALIZED VIEW")]
    AK --> BE[("Configure PrivateLink")]
    AL --> BF[("Check Cloud Storage Performance")]
    AM --> BG[("Check External System Performance")]
    AN --> BH[("Check Network Latency")]

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80,81,82,83,84,85,86,87,88,89,90,91,92,93,94,95,96,97,98,99,100,101,102,103,104,105,106,107,108,109,110,111,112,113,114,115,116,117,118,119,120 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef start fill:#4285f4,stroke:#1976d2;
    classDef history fill:#ff9800,stroke:#f57c00;
    classDef profile fill:#009688,stroke:#00796b;
    classDef metrics fill:#e91e63,stroke:#c2185b;
    classDef bottlenecks fill:#9c27b0,stroke:#7b1fa2;
    classDef scanning fill:#3f51b5,stroke:#303f9f;
    classDef cpu fill:#795548,stroke:#5d4037;
    classDef spill fill:#ff5722,stroke:#e64a19;
    classDef queue fill:#607d8b,stroke:#455a64;
    classDef credits fill:#00bcd4,stroke:#0097a7;
    classDef latency fill:#8bc34a,stroke:#689f38;
    classDef external fill:#f44336,stroke:#d32f2f;
    class A start;
    class B history;
    class D profile;
    class C,E,F,G,I,K,M,P,R bottlenecks;
    class H,I,J,K,L,M,N metrics;
    class U,V,W,X scanning;
    class Y,Z,AA cpu;
    class L spill;
    class N queue;
    class Q credits;
    class S latency;
    class T external;
```

### **Step-by-Step Performance Tuning Workflow**

#### **Step 1: Identify Slow Queries**
Use `QUERY_HISTORY` to identify queries with performance issues.

```sql
-- Get slow queries (>10 seconds) from the last 7 days
SELECT
    query_id,
    query_text,
    warehouse_name,
    user_name,
    start_time,
    execution_time,
    queue_time,
    bytes_scanned,
    partitions_scanned,
    credits_used,
    execution_status
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    execution_time > 10000  -- >10 seconds
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    execution_time DESC
LIMIT 20;
```

#### **Step 2: Analyze Query Profile**
Use `QUERY_PROFILE` to analyze the execution plan and identify bottlenecks.

```sql
-- Get query profile for a specific query
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id_from_step_1'));

-- Get steps with high execution time
SELECT
    step_id,
    operation,
    execution_time,
    rows_produced,
    bytes_scanned,
    spill_to_disk,
    spill_to_remote
FROM
    TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))
WHERE
    execution_time > 1000  -- >1 second
ORDER BY
    execution_time DESC;

-- Get steps with high bytes scanned
SELECT
    step_id,
    operation,
    bytes_scanned,
    rows_produced
FROM
    TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))
WHERE
    bytes_scanned > 100000000  -- >100MB
ORDER BY
    bytes_scanned DESC;
```

#### **Step 3: Diagnose Bottlenecks**
Based on the query profile, diagnose the specific bottlenecks:

1. **High Bytes Scanned**:
   - **Symptoms**: High `bytes_scanned` in `QUERY_PROFILE` or `QUERY_HISTORY`
   - **Root Causes**: Full table scans, missing filters, poor clustering
   - **Solutions**:
     - Add **filters** to reduce scanned data
     - Use **clustering** on frequently filtered columns
     - Use **partition pruning** for partitioned tables
     - Use **column pruning** to only read needed columns

2. **High CPU Usage**:
   - **Symptoms**: High `execution_time` with high CPU usage
   - **Root Causes**: Complex joins, aggregations, window functions
   - **Solutions**:
     - **Optimize joins** (use proper join types, reduce join size)
     - Use **approximate functions** (APPROX_COUNT_DISTINCT, APPROX_QUANTILE)
     - **Reduce data volume** (filter before aggregating)
     - Use **materialized views** for repetitive aggregations

3. **Spill to Disk/Remote**:
   - **Symptoms**: `spill_to_disk > 0` or `spill_to_remote > 0` in `QUERY_PROFILE`
   - **Root Causes**: Large result sets, large joins, large aggregations, small warehouse
   - **Solutions**:
     - **Increase warehouse size** (more memory)
     - **Reduce data volume** (use filters, LIMIT)
     - **Use larger result sets** (avoid small batches)

4. **High Queue Time**:
   - **Symptoms**: High `queue_time` in `QUERY_HISTORY`
   - **Root Causes**: Warehouse overloaded, too many concurrent queries
   - **Solutions**:
     - **Increase warehouse size** (more compute)
     - Use **multi-cluster warehouse** (scale out)
     - Use **query prioritization** (HIGH, MEDIUM, LOW)

5. **High Credit Usage**:
   - **Symptoms**: High `credits_used` in `QUERY_HISTORY`
   - **Root Causes**: Large warehouses, long-running queries, inefficient queries
   - **Solutions**:
     - **Right-size warehouses** (use smallest size that meets requirements)
     - **Optimize queries** (reduce execution time)
     - Use **result caching** for repetitive queries
     - Use **materialized views** for expensive queries

#### **Step 4: Optimize the Query**
Based on the bottleneck diagnosis, apply the appropriate optimizations:

**Example: High Bytes Scanned**
```sql
-- Original query (high bytes scanned)
SELECT * FROM sales JOIN customers ON sales.customer_id = customers.customer_id
WHERE sales.date > '2023-01-01';

-- Optimized query (add filters, use clustering)
SELECT * FROM
    (SELECT * FROM sales WHERE date > '2023-01-01') s
JOIN
    customers c ON s.customer_id = c.customer_id;
```

**Example: High CPU Usage (Joins)**
```sql
-- Original query (inefficient join)
SELECT * FROM large_table JOIN small_table ON large_table.id = small_table.id;

-- Optimized query (filter before join)
SELECT * FROM
    (SELECT * FROM large_table WHERE date > '2023-01-01') l
JOIN
    small_table s ON l.id = s.id;
```

**Example: Spill to Disk**
```sql
-- Original query (spill to disk)
SELECT * FROM my_table
WHERE date > CURRENT_DATE() - 30
ORDER BY value DESC;

-- Optimized query (reduce data volume, increase warehouse size)
-- Option 1: Use LIMIT
SELECT * FROM my_table
WHERE date > CURRENT_DATE() - 30
ORDER BY value DESC
LIMIT 1000;

-- Option 2: Use larger warehouse
ALTER WAREHOUSE my_wh SET WAREHOUSE_SIZE = 'X-LARGE';
```

**Example: High Queue Time**
```sql
-- Original query (high queue time)
-- No changes needed to query, but warehouse is overloaded

-- Solution 1: Increase warehouse size
ALTER WAREHOUSE my_wh SET WAREHOUSE_SIZE = 'X-LARGE';

-- Solution 2: Use multi-cluster warehouse
ALTER WAREHOUSE my_wh SET MAX_CLUSTER_COUNT = 4;

-- Solution 3: Use query prioritization
ALTER WAREHOUSE my_wh SET QUERY_PRIORITY = 'HIGH';
```

#### **Step 5: Test the Optimized Query**
Run the optimized query and compare performance with the original.

```sql
-- Run the optimized query and get query ID
SELECT * FROM my_table WHERE ...;

-- Get the query ID
SELECT LAST_QUERY_ID();

-- Compare performance with original query
SELECT
    query_id,
    query_text,
    execution_time,
    bytes_scanned,
    credits_used
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_id IN ('original_query_id', 'optimized_query_id')
ORDER BY
    query_id;
```

#### **Step 6: Monitor and Iterate**
Monitor the performance of the optimized query over time and iterate as needed.

```sql
-- Set up monitoring for the optimized query
CREATE OR REPLACE ALERT query_performance_alert
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'
AS
  SELECT
    query_id,
    query_text,
    execution_time,
    bytes_scanned,
    credits_used,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE
    query_text LIKE '%optimized_query%'
    AND execution_time > 5000  -- >5 seconds
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP());
```

## **7. Case Studies: Real-World Query Optimization**

### **Case Study 1: Slow Dashboard Queries**

#### **Problem**
- A **dashboard** with 10 queries takes **30 seconds** to load
- Queries scan **100GB+** of data each
- **User complaints** about slow performance

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
   - **Finding**: Queries scan **100GB+** and take **5-10 seconds** each

2. **Check Query Profiles**:
   ```sql
   SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('01a2b3c4-d5e6-78f9'));
   ```
   - **Finding**: Most time is spent in **TableScan** (90% of execution time)

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
   - **Finding**: Tables are **not clustered**

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
   ```sql
   ALTER SESSION SET USE_CACHED_RESULTS = TRUE;
   ```

#### **Results**
| **Metric** | **Before** | **After** | **Improvement** |
|------------|------------|-----------|-----------------|
| **Dashboard Load Time** | 30 seconds | 3 seconds | 10x faster |
| **Bytes Scanned** | 100GB+ | 10GB | 90% reduction |
| **Credit Usage** | 50 credits | 5 credits | 90% reduction |
| **User Satisfaction** | Poor | Excellent | Significant improvement |

### **Case Study 2: High Credit Usage**

#### **Problem**
- **Monthly credit usage** increased from **10,000** to **50,000** credits
- **No obvious increase** in query volume or data size
- **Budget overruns** causing cost concerns

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
   - **Finding**: **ETL_WH** warehouse used **40,000 credits** (80% of total)

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
   - **Finding**: A **daily ETL job** used **30,000 credits** (75% of ETL_WH usage)

3. **Check Query Profile**:
   ```sql
   SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('01a2b3c4-d5e6-78f9'));
   ```
   - **Finding**: Query **spilled to remote storage** (10GB spilled)

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
     JOIN
         LOOKUP_TABLE l ON s.id = l.id
     WHERE
         s.date > CURRENT_DATE() - 1;
     ```

3. **Use Batch Processing**:
   - Break the ETL job into **smaller batches**:
     ```sql
     FOR day IN (SELECT DISTINCT date FROM SOURCE_TABLE WHERE date > CURRENT_DATE() - 30) DO
         INSERT INTO TARGET_TABLE
         SELECT
             s.col1, s.col2, l.col3
         FROM
             SOURCE_TABLE s
         JOIN
             LOOKUP_TABLE l ON s.id = l.id
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
| **Monthly Credit Usage** | 50,000 | 15,000 | 70% reduction |
| **ETL Job Credit Usage** | 30,000 | 5,000 | 83% reduction |
| **Spill to Remote** | 10GB | 0 | Eliminated |
| **ETL Job Duration** | 2 hours | 30 minutes | 4x faster |

### **Case Study 3: Slow JOIN Performance**

#### **Problem**
- A **JOIN query** between **SALES** (100M rows) and **CUSTOMERS** (10M rows) takes **5 minutes**
- **User complaints** about slow report generation

#### **Diagnosis**
1. **Check Query Profile**:
   ```sql
   SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('01a2b3c4-d5e6-78f9'));
   ```
   - **Finding**: **HashJoin** step takes **4 minutes** (80% of total time)

2. **Check Join Size**:
   ```sql
   SELECT
       COUNT(*) AS join_size
   FROM
       SALES JOIN CUSTOMERS ON SALES.customer_id = CUSTOMERS.customer_id;
   ```
   - **Finding**: Join produces **100M rows** (1:1 join)

3. **Check Join Columns**:
   ```sql
   SELECT
       DISTINCT customer_id
   FROM
       SALES;
   ```
   - **Finding**: **10M distinct customer_id** in SALES (matches CUSTOMERS table size)

#### **Solution**
1. **Use Broadcast Join**:
   - Snowflake **automatically uses broadcast join** for small tables
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
| **Join Query Time** | 5 minutes | 30 seconds | 10x faster |
| **Bytes Scanned** | 10GB | 1GB | 90% reduction |
| **Join Size** | 100M rows | 10M rows | 90% reduction |
| **User Satisfaction** | Poor | Excellent | Significant improvement |

## **8. Query Optimization Decision Matrix**

### **Mer
