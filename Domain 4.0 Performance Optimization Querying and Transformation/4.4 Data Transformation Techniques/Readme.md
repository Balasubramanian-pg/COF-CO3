# **Snowflake Data Transformation Techniques: Production-Grade Technical Deep Dive**

---

## **1. Overview of Data Transformation in Snowflake**

Data transformation is the process of **converting data from one format or structure to another** to meet business requirements, improve performance, or enable analytics. In Snowflake, data transformation can be performed using a **variety of techniques**, each optimized for different use cases, performance requirements, and data volumes.

---

### **Mermaid: Snowflake Data Transformation Architecture**
```mermaid
%% Snowflake Data Transformation Architecture
flowchart TD
    subgraph Sources["Data Sources"]
        A[("Internal Tables")] --> B[("Transformation Layer")]
        C[("External Tables")] --> B
        D[("Stages")] --> B
        E[("Streams")] --> B
    end

    subgraph Transformation["Transformation Layer"]
        B --> F[("SQL-Based\nTransformations")]
        B --> G[("Stored Procedures")]
        B --> H[("UDFs")]
        B --> I[("Snowpark")]
        B --> J[("Tasks")]
        B --> K[("Materialized Views")]
    end

    subgraph Execution["Execution Layer"]
        F --> L[("Query Execution Engine")]
        G --> L
        H --> L
        I --> M[("Snowpark Runtime")]
        J --> N[("Task Scheduler")]
        K --> O[("Materialized View\nRefresh Engine")]
    end

    subgraph Targets["Data Targets"]
        L --> P[("Internal Tables")]
        L --> Q[("External Tables")]
        L --> R[("Stages")]
        M --> P
        N --> P
        O --> P
    end

    subgraph Monitoring["Monitoring Layer"]
        S[("QUERY_HISTORY")]
        T[("TASK_HISTORY")]
        U[("ACCOUNT_USAGE")]
    end
    L --> S
    M --> S
    N --> T
    O --> U

    %% --- Transformation Techniques ---
    F --> V[("SELECT Statements")]
    F --> W[("JOIN Operations")]
    F --> X[("GROUP BY")]
    F --> Y[("Window Functions")]
    F --> Z[("PIVOT/UNPIVOT")]
    F --> AA[("JSON Functions")]
    F --> AB[("Semi-Structured Data")]
    F --> AC[("CTEs")]
    F --> AD[("Subqueries")]

    G --> AE[("SQL Procedures")]
    G --> AF[("JavaScript Procedures")]
    G --> AG[("Python Procedures")]

    H --> AH[("SQL UDFs")]
    H --> AI[("JavaScript UDFs")]
    H --> AJ[("Java UDFs")]
    H --> AK[("Python UDFs")]

    I --> AL[("Python DataFrames")]
    I --> AM[("Java DataFrames")]
    I --> AN[("Scala DataFrames")]

    J --> AO[("Scheduled Tasks")]
    J --> AP[("DAG Workflows")]

    K --> AQ[("Pre-Computed Results")]

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80,81,82,83,84,85,86,87,88,89,90,91,92,93,94,95,96,97,98,99,100 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef sources fill:#4285f4,stroke:#1976d2;
    classDef transformation fill:#ff9800,stroke:#f57c00;
    classDef execution fill:#009688,stroke:#00796b;
    classDef targets fill:#e91e63,stroke:#c2185b;
    classDef monitoring fill:#9c27b0,stroke:#7b1fa2;
    class A,C,D,E sources;
    class B,F,G,H,I,J,K transformation;
    class L,M,N,O execution;
    class P,Q,R targets;
    class S,T,U monitoring;
    class V,W,X,Y,Z,AA,AB,AC,AD,F;
    class AE,AF,AG,G;
    class AH,AI,AJ,AK,H;
    class AL,AM,AN,I;
    class AO,AP,J;
    class AQ,K;
```


### **Key Concepts in Snowflake Data Transformation**

#### **1. Transformation Paradigms**
Snowflake supports multiple **transformation paradigms**, each with its own strengths:

| **Paradigm** | **Description** | **Use Cases** | **Pros** | **Cons** |
|--------------|-----------------|--------------|----------|----------|
| **ELT (Extract, Load, Transform)** | Load raw data first, then transform | Data warehousing, analytics | ✅ Best performance, ✅ Scalable, ✅ Separates loading and transformation | ❌ Requires storage for raw data, ❌ Transformation happens after loading |
| **ETL (Extract, Transform, Load)** | Transform data before loading | Data quality, compliance | ✅ Data quality enforced before loading, ✅ Less storage for raw data | ❌ Transformation can be a bottleneck, ❌ More complex pipelines |
| **Stream Processing** | Transform data in real-time as it arrives | Real-time analytics, event processing | ✅ Real-time insights, ✅ Low latency | ❌ More complex, ❌ Higher cost |
| **Batch Processing** | Transform data in scheduled batches | Reporting, data warehousing | ✅ Simple, ✅ Cost-effective | ❌ Latency, ❌ Not real-time |
| **Hybrid** | Combine multiple paradigms | Complex pipelines | ✅ Flexible, ✅ Best of all worlds | ❌ Most complex |

#### **2. Transformation Layers**
Snowflake transformations can occur at **multiple layers**:

| **Layer** | **Description** | **Techniques** | **Performance** |
|-----------|-----------------|----------------|-----------------|
| **Ingestion Layer** | Transform data during loading | COPY INTO with transformations, Snowpipe with transformations | ⭐⭐⭐⭐ High |
| **Storage Layer** | Transform data at rest | Materialized views, clustering, partitioning | ⭐⭐⭐⭐⭐ Very High |
| **Compute Layer** | Transform data during query execution | SQL queries, UDFs, stored procedures | ⭐⭐⭐ Medium |
| **Application Layer** | Transform data in external applications | Snowpark, external functions | ⭐⭐ Low |

#### **3. Transformation Data Flow**
1. **Data Ingestion**:
   - Load data from **sources** (internal tables, external tables, stages, streams)
   - Apply **ingestion-time transformations** (COPY INTO with transformations)

2. **Data Storage**:
   - Store raw or transformed data in **Snowflake tables**
   - Apply **storage-time transformations** (clustering, partitioning, materialized views)

3. **Data Processing**:
   - Apply **compute-time transformations** (SQL queries, UDFs, stored procedures)
   - Use **Snowpark** for complex transformations

4. **Data Output**:
   - Write transformed data to **targets** (internal tables, external tables, stages)
   - Use **Tasks** for scheduled transformations

5. **Data Consumption**:
   - Consume transformed data via **queries, reports, dashboards, applications**

### **Transformation Technique Selection Matrix**

| **Technique** | **Best For** | **Performance** | **Scalability** | **Complexity** | **Cost** | **Real-Time** | **Data Volume** | **Use Case Examples** |
|---------------|-------------|-----------------|----------------|---------------|----------|---------------|-----------------|----------------------|
| **SQL SELECT** | Simple transformations, filtering, projections | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐ | ⭐⭐⭐⭐ | ❌ No | Small - Large | Data filtering, column selection, simple calculations |
| **JOIN Operations** | Combining data from multiple tables | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ❌ No | Small - Large | Data enrichment, star schema joins, fact-dimension joins |
| **GROUP BY** | Aggregations, summarizations | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ❌ No | Small - Large | Sales aggregations, customer analytics, KPI calculations |
| **Window Functions** | Running totals, rankings, moving averages | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ❌ No | Small - Medium | Time-series analysis, customer segmentation, trend analysis |
| **CTEs (WITH)** | Complex multi-step transformations | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ❌ No | Small - Large | Multi-step ETL, data pipelines, complex queries |
| **Subqueries** | Nested transformations | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ❌ No | Small - Medium | Correlated subqueries, EXISTS/NOT EXISTS, derived tables |
| **PIVOT/UNPIVOT** | Data reshaping | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ❌ No | Small - Medium | Reporting, cross-tab reports, data normalization |
| **JSON Functions** | Semi-structured data transformations | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ❌ No | Small - Large | JSON parsing, nested data extraction, API data processing |
| **Semi-Structured Data** | VARIANT, OBJECT, ARRAY transformations | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ❌ No | Small - Large | Nested data processing, array operations, object manipulation |
| **SQL UDFs** | Reusable custom functions | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ❌ No | Small - Medium | Custom calculations, data validation, business logic |
| **JavaScript UDFs** | Complex custom logic | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐ | ❌ No | Small | Complex calculations, string manipulation, custom logic |
| **Python UDFs** | Data science, ML transformations | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐ | ❌ No | Small | Machine learning, statistical analysis, complex data processing |
| **Java UDFs** | High-performance custom logic | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ❌ No | Small - Medium | High-performance calculations, custom algorithms |
| **Stored Procedures (SQL)** | Multi-statement transformations | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ❌ No | Small - Large | ETL pipelines, data validation, complex workflows |
| **Stored Procedures (JS/Python/Java)** | Complex transformation workflows | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ❌ No | Small - Medium | Complex ETL, data quality checks, custom workflows |
| **Snowpark (Python)** | DataFrame-style transformations | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ✅ Yes | Small - Large | Data engineering, ML pipelines, complex transformations |
| **Snowpark (Java/Scala)** | High-performance DataFrame transformations | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ✅ Yes | Medium - Large | High-performance ETL, real-time processing, large-scale transformations |
| **Tasks** | Scheduled transformations | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ❌ No | Small - Large | Scheduled ETL, batch processing, periodic transformations |
| **Streams** | Change data capture, incremental transformations | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ✅ Yes | Medium - Large | CDC, real-time ETL, incremental loading |
| **Materialized Views** | Pre-computed transformations | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐ | ⭐⭐ | ❌ No | Small - Large | Pre-computed aggregations, star schema fact tables, dashboard data |
| **External Functions** | Custom transformations in external systems | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐ | ✅ Yes | Small | Custom business logic, external API calls, specialized processing |

### **Transformation Decision Tree**

```mermaid
%% Transformation Technique Decision Tree
flowchart TD
    A[("Transformation\nRequirement")] --> B{Data Volume?}
    B -->|Small| C{Complexity?}
    B -->|Medium| D{Complexity?}
    B -->|Large| E{Complexity?}

    C -->|Low| F[("SQL SELECT\nGROUP BY\nJOIN")]
    C -->|Medium| G[("CTEs\nSubqueries\nWindow Functions")]
    C -->|High| H[("Stored Procedures\nUDFs\nSnowpark")]

    D -->|Low| I[("SQL SELECT\nJOIN\nGROUP BY")]
    D -->|Medium| J[("CTEs\nMaterialized Views\nTasks")]
    D -->|High| K[("Stored Procedures\nSnowpark\nExternal Functions")]

    E -->|Low| L[("SQL SELECT\nClustering\nPartitioning")]
    E -->|Medium| M[("Materialized Views\nTasks\nStreams")]
    E -->|High| N[("Snowpark\nExternal Functions\nStored Procedures")]

    F --> O[("Use for: Simple filtering, projections, aggregations")]
    G --> P[("Use for: Multi-step transformations, complex queries")]
    H --> Q[("Use for: Reusable logic, complex workflows")]
    I --> R[("Use for: Standard transformations, reporting")]
    J --> S[("Use for: ETL pipelines, pre-computed results")]
    K --> T[("Use for: Complex ETL, custom logic")]
    L --> U[("Use for: Large-scale transformations, analytics")]
    M --> V[("Use for: Batch processing, incremental loading")]
    N --> W[("Use for: High-performance ETL, real-time processing")]

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef start fill:#4285f4,stroke:#1976d2;
    classDef small fill:#ff9800,stroke:#f57c00;
    classDef medium fill:#009688,stroke:#00796b;
    classDef large fill:#e91e63,stroke:#c2185b;
    classDef low fill:#9c27b0,stroke:#7b1fa2;
    classDef mediumc fill:#3f51b5,stroke:#303f9f;
    classDef high fill:#795548,stroke:#5d4037;
    class A start;
    class B start;
    class C,D,E small;
    class F,G,H low;
    class I,J,K mediumc;
    class L,M,N large;
    class O,P,Q,R,S,T,U,V,W high;
```

## **2. SQL-Based Transformation Techniques**

SQL is the **primary transformation language** in Snowflake. Snowflake's **SQL engine** is highly optimized for **data transformation operations**, offering **excellent performance** and **scalability**.

### **A. SELECT Statements**

#### **1. Basic SELECT Transformations**
The `SELECT` statement is the **fundamental transformation tool** in Snowflake, allowing you to:
- **Filter** data (WHERE clause)
- **Project** specific columns (SELECT clause)
- **Join** tables (JOIN clause)
- **Sort** data (ORDER BY clause)
- **Limit** results (LIMIT clause)

**Basic Syntax**:
```sql
SELECT
    column1 [AS alias1],
    column2 [AS alias2],
    ...
FROM
    table_name
[WHERE
    condition]
[GROUP BY
    column1, column2, ...]
[HAVING
    condition]
[ORDER BY
    column1 [ASC|DESC], column2 [ASC|DESC], ...]
[LIMIT
    number_of_rows]
[OFFSET
    number_of_rows];
```

#### **2. Transformation Types with SELECT**

| **Transformation Type** | **Description** | **Example** | **Performance** | **Use Cases** |
|-------------------------|-----------------|-------------|-----------------|--------------|
| **Column Selection** | Select specific columns | `SELECT col1, col2 FROM my_table` | ⭐⭐⭐⭐⭐ | Data projection, reducing I/O |
| **Column Renaming** | Rename columns in result | `SELECT col1 AS new_name FROM my_table` | ⭐⭐⭐⭐⭐ | Improving readability, standardizing names |
| **Column Expression** | Compute new columns | `SELECT col1, col2 * col3 AS product FROM my_table` | ⭐⭐⭐⭐ | Calculations, derived fields |
| **Filtering** | Filter rows based on conditions | `SELECT * FROM my_table WHERE col1 > 100` | ⭐⭐⭐⭐ | Data subsetting, conditional extraction |
| **Conditional Logic** | Apply conditional transformations | `SELECT col1, CASE WHEN col2 > 100 THEN 'High' ELSE 'Low' END AS category FROM my_table` | ⭐⭐⭐⭐ | Data categorization, business logic |
| **String Manipulation** | Transform string data | `SELECT col1, UPPER(col2) AS col2_upper FROM my_table` | ⭐⭐⭐⭐ | Data cleaning, formatting |
| **Date/Time Manipulation** | Transform date/time data | `SELECT col1, DATE_TRUNC('DAY', col2) AS day FROM my_table` | ⭐⭐⭐⭐ | Time-series analysis, date bucketing |
| **Type Casting** | Convert data types | `SELECT col1, CAST(col2 AS INTEGER) AS col2_int FROM my_table` | ⭐⭐⭐⭐ | Data type standardization, compatibility |
| **Null Handling** | Handle NULL values | `SELECT col1, COALESCE(col2, 0) AS col2_default FROM my_table` | ⭐⭐⭐⭐ | Data quality, default values |
| **Distinct** | Remove duplicates | `SELECT DISTINCT col1, col2 FROM my_table` | ⭐⭐⭐ | Deduplication, unique value extraction |

#### **3. Best Practices for SELECT Transformations**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Select Only Needed Columns** | Reduce I/O and memory usage | `SELECT col1, col2 FROM my_table` (not `SELECT *`) |
| **Use Column Aliases** | Improve readability and maintainability | `SELECT col1 AS customer_id, col2 AS customer_name` |
| **Push Filters Early** | Reduce data scanned | `SELECT * FROM my_table WHERE date > '2023-01-01'` |
| **Use Parameterized Queries** | Improve performance and security | `SELECT * FROM my_table WHERE id = ?` |
| **Avoid Functions on Filtered Columns** | Enable predicate pushdown | `SELECT * FROM my_table WHERE col1 = 'value'` (not `WHERE UPPER(col1) = 'VALUE'`) |
| **Use WHERE Before GROUP BY/HAVING** | Reduce data before aggregation | `SELECT col1, COUNT(*) FROM (SELECT col1 FROM my_table WHERE ...) GROUP BY col1` |
| **Use LIMIT for Large Result Sets** | Reduce memory usage | `SELECT * FROM my_table LIMIT 1000` |
| **Use OFFSET for Pagination** | Implement efficient pagination | `SELECT * FROM my_table ORDER BY id LIMIT 100 OFFSET 1000` |
| **Use Keyset Pagination** | More efficient than OFFSET for large datasets | `SELECT * FROM my_table WHERE id > last_id ORDER BY id LIMIT 100` |
| **Use Clustering for Filtered Columns** | Improve filter performance | `ALTER TABLE my_table CLUSTER BY (date)` |

#### **4. Performance Optimization for SELECT**

| **Technique** | **Description** | **Example** | **Performance Impact** |
|---------------|-----------------|-------------|-------------------------|
| **Column Pruning** | Only read needed columns | `SELECT col1, col2 FROM my_table` | ⭐⭐⭐⭐ (2-10x faster) |
| **Predicate Pushdown** | Push filters to data source | `SELECT * FROM my_table WHERE date > '2023-01-01'` | ⭐⭐⭐⭐ (2-10x faster) |
| **Partition Pruning** | Skip irrelevant partitions | `SELECT * FROM my_table WHERE date = '2023-01-01'` (with clustering) | ⭐⭐⭐⭐⭐ (10-100x faster) |
| **Join Optimization** | Optimize join operations | `SELECT * FROM t1 JOIN t2 ON t1.id = t2.id` | ⭐⭐⭐⭐ (2-10x faster) |
| **Result Caching** | Cache query results | `ALTER SESSION SET USE_CACHED_RESULTS = TRUE` | ⭐⭐⭐⭐⭐ (10-1000x faster) |
| **Query Rewriting** | Automatic query optimization | Automatic (predicate pushdown, join reordering) | ⭐⭐⭐ (1.5-5x faster) |

#### **5. Examples**

**Example 1: Basic Column Selection and Filtering**
```sql
-- Select specific columns with filtering
SELECT
    customer_id,
    customer_name,
    email,
    signup_date
FROM
    customers
WHERE
    region = 'US'
    AND signup_date > '2023-01-01'
    AND status = 'ACTIVE'
ORDER BY
    signup_date DESC
LIMIT 1000;
```

**Example 2: Column Expressions and Conditional Logic**
```sql
-- Select with expressions and conditional logic
SELECT
    customer_id,
    customer_name,
    total_spend,
    CASE
        WHEN total_spend > 10000 THEN 'Platinum'
        WHEN total_spend > 5000 THEN 'Gold'
        WHEN total_spend > 1000 THEN 'Silver'
        ELSE 'Bronze'
    END AS customer_tier,
    total_spend * 0.1 AS loyalty_points,
    DATEDIFF('day', signup_date, CURRENT_DATE()) AS days_as_customer
FROM
    customers
WHERE
    region IN ('US', 'EU', 'APAC')
ORDER BY
    total_spend DESC;
```

**Example 3: String Manipulation**
```sql
-- String manipulation transformations
SELECT
    customer_id,
    customer_name,
    UPPER(customer_name) AS customer_name_upper,
    LOWER(email) AS email_lower,
    CONCAT(first_name, ' ', last_name) AS full_name,
    SUBSTRING(phone, 1, 3) AS phone_area_code,
    REGEXP_REPLACE(notes, '[^a-zA-Z0-9 ]', '') AS clean_notes
FROM
    customers
WHERE
    customer_name LIKE '%Smith%';
```

**Example 4: Date/Time Manipulation**
```sql
-- Date/time manipulation transformations
SELECT
    order_id,
    customer_id,
    order_date,
    DATE_TRUNC('DAY', order_date) AS order_day,
    DATE_TRUNC('MONTH', order_date) AS order_month,
    DATE_TRUNC('YEAR', order_date) AS order_year,
    DAYOFWEEK(order_date) AS order_day_of_week,
    HOUR(order_date) AS order_hour,
    DATEDIFF('day', order_date, CURRENT_DATE()) AS days_since_order,
    DATEADD('day', 7, order_date) AS order_date_plus_7_days
FROM
    orders
WHERE
    order_date > CURRENT_DATE() - 30
ORDER BY
    order_date DESC;
```

**Example 5: Null Handling**
```sql
-- Null handling transformations
SELECT
    customer_id,
    customer_name,
    COALESCE(phone, 'N/A') AS phone,
    COALESCE(email, 'N/A') AS email,
    NULLIF(total_spend, 0) AS total_spend,
    ZEROIFNULL(discount) AS discount,
    IFF(status IS NULL, 'Unknown', status) AS status
FROM
    customers
WHERE
    signup_date > '2023-01-01';
```

### **B. JOIN Operations**

#### **1. JOIN Types in Snowflake**
Snowflake supports **multiple JOIN types**, each optimized for different use cases:

| **JOIN Type** | **Syntax** | **Description** | **Performance** | **Use Cases** |
|---------------|------------|-----------------|-----------------|--------------|
| **INNER JOIN** | `SELECT * FROM t1 INNER JOIN t2 ON t1.id = t2.id` | Returns rows with matches in both tables | ⭐⭐⭐⭐⭐ | Most common join type, default for JOIN keyword |
| **LEFT JOIN** | `SELECT * FROM t1 LEFT JOIN t2 ON t1.id = t2.id` | Returns all rows from left table + matches from right | ⭐⭐⭐⭐ | Keep all left table rows, optional right table data |
| **RIGHT JOIN** | `SELECT * FROM t1 RIGHT JOIN t2 ON t1.id = t2.id` | Returns all rows from right table + matches from left | ⭐⭐⭐ | Rarely used, can be rewritten as LEFT JOIN |
| **FULL OUTER JOIN** | `SELECT * FROM t1 FULL OUTER JOIN t2 ON t1.id = t2.id` | Returns all rows from both tables | ⭐⭐ | When you need all rows from both tables |
| **CROSS JOIN** | `SELECT * FROM t1 CROSS JOIN t2` | Returns Cartesian product of both tables | ⭐ | Only for small tables, creates row explosion |
| **SEMI JOIN** | `SELECT * FROM t1 WHERE id IN (SELECT id FROM t2)` | Returns rows from left table with matches in right | ⭐⭐⭐⭐⭐ | Filtering (use EXISTS or IN) |
| **ANTI JOIN** | `SELECT * FROM t1 WHERE id NOT IN (SELECT id FROM t2)` | Returns rows from left table without matches in right | ⭐⭐⭐⭐ | Exclusion (use NOT EXISTS or NOT IN) |
| **NATURAL JOIN** | `SELECT * FROM t1 NATURAL JOIN t2` | Joins on columns with the same name | ⭐⭐ | Not recommended (ambiguous, error-prone) |
| **SELF JOIN** | `SELECT * FROM t1 a JOIN t1 b ON a.id = b.parent_id` | Joins a table to itself | ⭐⭐⭐⭐ | Hierarchical data, organizational charts |

#### **2. JOIN Algorithms in Snowflake**
Snowflake **automatically selects** the best join algorithm based on:
- Table sizes
- Join conditions
- Data distribution
- Available memory

| **Algorithm** | **Description** | **When Used** | **Performance** | **Memory Usage** | **Best For** |
|---------------|-----------------|---------------|-----------------|------------------|--------------|
| **Hash Join** | Builds a hash table on one table and probes with the other | Default for most joins | ⭐⭐⭐⭐ | Medium | General-purpose joins |
| **Sort-Merge Join** | Sorts both tables and merges them | Used for large sorted tables | ⭐⭐⭐ | Low | Large tables with sort keys |
| **Nested Loop Join** | Nested loop over rows (rare in Snowflake) | Small tables, specific cases | ⭐ | High | Small dimension tables |
| **Broadcast Join** | Broadcasts the smaller table to all nodes | Small dimension tables | ⭐⭐⭐⭐⭐ | Low | Small tables (<10MB) |

**Note**: Snowflake **automatically uses broadcast join** for small tables (typically <10MB).

#### **3. JOIN Optimization Techniques**

| **Technique** | **Description** | **Example** | **Performance Impact** |
|---------------|-----------------|-------------|-------------------------|
| **Filter Before Joining** | Apply WHERE clauses before joining | `SELECT * FROM (SELECT * FROM t1 WHERE ...) a JOIN t2 b ON a.id = b.id` | ⭐⭐⭐⭐ (2-10x faster) |
| **Join on Indexed Columns** | Join on clustered or partitioned columns | `SELECT * FROM t1 CLUSTER BY (id) JOIN t2 ON t1.id = t2.id` | ⭐⭐⭐⭐ (2-10x faster) |
| **Use Broadcast Join** | Ensure small table is broadcast | `SELECT * FROM large_table JOIN small_table ON ...` | ⭐⭐⭐⭐⭐ (10-100x faster) |
| **Avoid Cartesian Products** | Ensure join conditions are specified | `SELECT * FROM t1 JOIN t2 ON t1.id = t2.id` (not `SELECT * FROM t1, t2`) | ⭐⭐⭐⭐ (10-100x faster) |
| **Use Semi-Join** | Use EXISTS or IN for filtering | `SELECT * FROM t1 WHERE id IN (SELECT id FROM t2)` | ⭐⭐⭐⭐ (2-10x faster) |
| **Use Anti-Join** | Use NOT EXISTS or NOT IN for exclusion | `SELECT * FROM t1 WHERE id NOT IN (SELECT id FROM t2)` | ⭐⭐⭐⭐ (2-10x faster) |
| **Join Small Tables First** | Join small tables before large tables | `SELECT * FROM small_table JOIN medium_table ON ... JOIN large_table ON ...` | ⭐⭐⭐ (1.5-5x faster) |
| **Avoid Redundant Joins** | Remove unnecessary joins | Remove joins to tables not used in SELECT or WHERE | ⭐⭐⭐ (1.5-5x faster) |
| **Use CTEs for Complex Joins** | Break down complex joins into CTEs | `WITH cte AS (SELECT * FROM t1 JOIN t2 ON ...) SELECT * FROM cte JOIN t3 ON ...` | ⭐⭐⭐ (1.5-5x faster) |
| **Use Materialized Views** | Pre-compute join results | `CREATE MATERIALIZED VIEW my_mv AS SELECT * FROM t1 JOIN t2 ON ...` | ⭐⭐⭐⭐ (10-100x faster) |

#### **4. Best Practices for JOIN Operations**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use Proper Join Types** | Choose the right join type for your use case | `INNER JOIN` for most cases, `LEFT JOIN` for optional data |
| **Join on Indexed Columns** | Join on clustered or partitioned columns | `ALTER TABLE my_table CLUSTER BY (join_column)` |
| **Filter Before Joining** | Apply WHERE clauses before joining | `SELECT * FROM (SELECT * FROM t1 WHERE ...) a JOIN t2 b ON a.id = b.id` |
| **Use Broadcast Join for Small Tables** | Snowflake automatically uses broadcast join for small tables | `SELECT * FROM large_table JOIN small_table ON ...` |
| **Avoid Cartesian Products** | Ensure join conditions are specified | `SELECT * FROM t1 JOIN t2 ON t1.id = t2.id` |
| **Use Semi-Join for Filtering** | Use EXISTS or IN for filtering | `SELECT * FROM t1 WHERE id IN (SELECT id FROM t2)` |
| **Use Anti-Join for Exclusion** | Use NOT EXISTS or NOT IN for exclusion | `SELECT * FROM t1 WHERE id NOT IN (SELECT id FROM t2)` |
| **Join Small Tables First** | Join small tables before large tables | `SELECT * FROM small_table JOIN medium_table ON ... JOIN large_table ON ...` |
| **Avoid Redundant Joins** | Remove unnecessary joins | Remove joins to tables not used in SELECT or WHERE |
| **Use CTEs for Complex Joins** | Break down complex joins into CTEs | `WITH cte AS (SELECT * FROM t1 JOIN t2 ON ...) SELECT * FROM cte JOIN t3 ON ...` |
| **Use Materialized Views for Repetitive Joins** | Pre-compute join results | `CREATE MATERIALIZED VIEW my_mv AS SELECT * FROM t1 JOIN t2 ON ...` |
| **Monitor Join Performance** | Check QUERY_PROFILE for join performance | `SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))` |

#### **5. Examples**

**Example 1: Basic INNER JOIN**
```sql
-- Join customers and orders
SELECT
    c.customer_id,
    c.customer_name,
    c.region,
    o.order_id,
    o.order_date,
    o.amount,
    o.status AS order_status
FROM
    customers c
INNER JOIN
    orders o ON c.customer_id = o.customer_id
WHERE
    o.order_date > CURRENT_DATE() - 30
ORDER BY
    o.order_date DESC;
```

**Example 2: LEFT JOIN with Filtering**
```sql
-- Left join to include all customers, even without orders
SELECT
    c.customer_id,
    c.customer_name,
    c.region,
    c.signup_date,
    o.order_id,
    o.order_date,
    o.amount
FROM
    customers c
LEFT JOIN
    orders o ON c.customer_id = o.customer_id
    AND o.order_date > CURRENT_DATE() - 30
WHERE
    c.region = 'US'
ORDER BY
    c.signup_date DESC;
```

**Example 3: Multiple JOINs with CTEs**
```sql
-- Multiple joins with CTEs for better readability
WITH customer_orders AS (
    SELECT
        c.customer_id,
        c.customer_name,
        c.region,
        o.order_id,
        o.order_date,
        o.amount,
        o.product_id
    FROM
        customers c
    INNER JOIN
        orders o ON c.customer_id = o.customer_id
    WHERE
        o.order_date > CURRENT_DATE() - 30
),
order_products AS (
    SELECT
        co.*,
        p.product_name,
        p.category,
        p.price
    FROM
        customer_orders co
    INNER JOIN
        products p ON co.product_id = p.product_id
)
SELECT
    customer_id,
    customer_name,
    region,
    product_name,
    category,
    SUM(amount) AS total_spend,
    COUNT(order_id) AS order_count
FROM
    order_products
GROUP BY
    customer_id, customer_name, region, product_name, category
ORDER BY
    total_spend DESC;
```

**Example 4: Broadcast Join Optimization**
```sql
-- Ensure small table is broadcast (Snowflake does this automatically)
SELECT
    l.*,
    d.department_name,
    d.location
FROM
    large_table l  -- 100M rows
JOIN
    small_table d ON l.department_id = d.department_id  -- 100 rows
WHERE
    l.date > CURRENT_DATE() - 7;

-- Check if broadcast join was used
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'));
-- Look for "BroadcastJoin" in the plan
```

**Example 5: Semi-Join for Filtering**
```sql
-- Use semi-join for filtering (EXISTS)
SELECT
    c.customer_id,
    c.customer_name,
    c.region
FROM
    customers c
WHERE
    EXISTS (
        SELECT 1
        FROM orders o
        WHERE o.customer_id = c.customer_id
        AND o.order_date > CURRENT_DATE() - 30
    );

-- Alternative: Use IN
SELECT
    c.customer_id,
    c.customer_name,
    c.region
FROM
    customers c
WHERE
    c.customer_id IN (
        SELECT DISTINCT customer_id
        FROM orders
        WHERE order_date > CURRENT_DATE() - 30
    );
```

**Example 6: Anti-Join for Exclusion**
```sql
-- Use anti-join for exclusion (NOT EXISTS)
SELECT
    c.customer_id,
    c.customer_name,
    c.region
FROM
    customers c
WHERE
    NOT EXISTS (
        SELECT 1
        FROM orders o
        WHERE o.customer_id = c.customer_id
    );

-- Alternative: Use NOT IN
SELECT
    c.customer_id,
    c.customer_name,
    c.region
FROM
    customers c
WHERE
    c.customer_id NOT IN (
        SELECT DISTINCT customer_id
        FROM orders
    );
```

### **C. GROUP BY Aggregations**

#### **1. Basic GROUP BY Syntax**
The `GROUP BY` clause is used to **aggregate data** by one or more columns.

**Basic Syntax**:
```sql
SELECT
    column1 [AS alias1],
    aggregate_function(column2) [AS alias2],
    ...
FROM
    table_name
[WHERE
    condition]
GROUP BY
    column1, column3, ...
[HAVING
    condition]
[ORDER BY
    column1 [ASC|DESC], aggregate_function(column2) [ASC|DESC], ...]
[LIMIT
    number_of_rows];
```

#### **2. Aggregate Functions**

| **Function** | **Description** | **Example** | **Performance** | **Use Cases** |
|--------------|-----------------|-------------|-----------------|--------------|
| **COUNT** | Count rows or values | `COUNT(*)`, `COUNT(col1)` | ⭐⭐⭐⭐⭐ | Row counting, distinct counting |
| **SUM** | Sum of values | `SUM(col1)` | ⭐⭐⭐⭐⭐ | Total calculations, financial aggregations |
| **AVG** | Average of values | `AVG(col1)` | ⭐⭐⭐⭐ | Mean calculations, statistical analysis |
| **MIN** | Minimum value | `MIN(col1)` | ⭐⭐⭐⭐⭐ | Finding minimum values |
| **MAX** | Maximum value | `MAX(col1)` | ⭐⭐⭐⭐⭐ | Finding maximum values |
| **STDDEV** | Standard deviation | `STDDEV(col1)` | ⭐⭐⭐ | Statistical analysis |
| **VARIANCE** | Variance | `VARIANCE(col1)` | ⭐⭐⭐ | Statistical analysis |
| **APPROX_COUNT_DISTINCT** | Approximate count of distinct values | `APPROX_COUNT_DISTINCT(col1)` | ⭐⭐⭐⭐⭐ | Distinct counting on large datasets |
| **APPROX_QUANTILE** | Approximate quantile | `APPROX_QUANTILE(col1, 0.5)` | ⭐⭐⭐⭐ | Percentile calculations on large datasets |
| **APPROX_TOP_K** | Approximate top K values | `APPROX_TOP_K(col1, 10)` | ⭐⭐⭐⭐ | Top K queries on large datasets |
| **APPROX_TOP_SUM** | Approximate top K values by sum | `APPROX_TOP_SUM(col1, 10, col2)` | ⭐⭐⭐⭐ | Top K by sum on large datasets |
| **LISTAGG** | Concatenate values into a list | `LISTAGG(col1, ', ') WITHIN GROUP (ORDER BY col1)` | ⭐⭐⭐ | String aggregation |
| **ARRAY_AGG** | Aggregate values into an array | `ARRAY_AGG(col1) WITHIN GROUP (ORDER BY col1)` | ⭐⭐⭐ | Array aggregation |

#### **3. GROUP BY Optimization Techniques**

| **Technique** | **Description** | **Example** | **Performance Impact** |
|---------------|-----------------|-------------|-------------------------|
| **Filter Before GROUP BY** | Apply WHERE before GROUP BY | `SELECT col1, COUNT(*) FROM (SELECT col1 FROM my_table WHERE ...) GROUP BY col1` | ⭐⭐⭐⭐ (2-10x faster) |
| **Use HAVING for Filtering Groups** | Filter groups after aggregation | `SELECT col1, COUNT(*) FROM my_table GROUP BY col1 HAVING COUNT(*) > 100` | ⭐⭐⭐ (1.5-5x faster) |
| **Use Approximate Functions** | Use approximate functions for large datasets | `SELECT APPROX_COUNT_DISTINCT(col1) FROM my_table` | ⭐⭐⭐⭐ (10-100x faster) |
| **Use Materialized Views** | Pre-compute aggregations | `CREATE MATERIALIZED VIEW my_mv AS SELECT col1, COUNT(*) FROM my_table GROUP BY col1` | ⭐⭐⭐⭐ (10-100x faster) |
| **Use Clustering** | Cluster on GROUP BY columns | `ALTER TABLE my_table CLUSTER BY (col1)` | ⭐⭐⭐⭐ (2-10x faster) |
| **Use ROLLUP** | Multi-level aggregations | `SELECT col1, col2, SUM(col3) FROM my_table GROUP BY ROLLUP(col1, col2)` | ⭐⭐⭐ (1.5-5x faster) |
| **Use CUBE** | All possible groupings | `SELECT col1, col2, SUM(col3) FROM my_table GROUP BY CUBE(col1, col2)` | ⭐⭐ (1.1-2x faster) |
| **Use GROUPING SETS** | Multiple GROUP BY clauses | `SELECT col1, col2, SUM(col3) FROM my_table GROUP BY GROUPING SETS ((col1), (col2))` | ⭐⭐⭐ (1.5-5x faster) |
| **Avoid Unnecessary Aggregations** | Only aggregate what's needed | `SELECT col1, col2 FROM my_table` (not `SELECT col1, COUNT(*) FROM my_table GROUP BY col1, col2`) | ⭐⭐⭐ (1.5-5x faster) |

#### **4. Best Practices for GROUP BY**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Filter Before GROUP BY** | Reduce data volume before aggregation | `SELECT col1, COUNT(*) FROM (SELECT col1 FROM my_table WHERE ...) GROUP BY col1` |
| **Use Proper Data Types** | Use appropriate data types for aggregations | `SUM(amount)` (not `SUM(CAST(amount AS STRING))`) |
| **Use Approximate Functions for Large Datasets** | Use APPROX_COUNT_DISTINCT, APPROX_QUANTILE | `SELECT APPROX_COUNT_DISTINCT(col1) FROM large_table` |
| **Use Materialized Views for Repetitive Aggregations** | Pre-compute aggregations | `CREATE MATERIALIZED VIEW my_mv AS SELECT col1, COUNT(*) FROM my_table GROUP BY col1` |
| **Use Clustering on GROUP BY Columns** | Improve aggregation performance | `ALTER TABLE my_table CLUSTER BY (col1)` |
| **Use ROLLUP for Hierarchical Aggregations** | Multi-level aggregations | `SELECT col1, col2, SUM(col3) FROM my_table GROUP BY ROLLUP(col1, col2)` |
| **Use CUBE for Multi-Dimensional Aggregations** | All possible groupings | `SELECT col1, col2, SUM(col3) FROM my_table GROUP BY CUBE(col1, col2)` |
| **Use GROUPING SETS for Multiple Aggregations** | Multiple GROUP BY clauses | `SELECT col1, col2, SUM(col3) FROM my_table GROUP BY GROUPING SETS ((col1), (col2))` |
| **Avoid Unnecessary Aggregations** | Only aggregate what's needed | `SELECT col1, col2 FROM my_table` (not `SELECT col1, COUNT(*) FROM my_table GROUP BY col1, col2`) |
| **Use HAVING for Filtering Groups** | Filter groups after aggregation | `SELECT col1, COUNT(*) FROM my_table GROUP BY col1 HAVING COUNT(*) > 100` |
| **Monitor GROUP BY Performance** | Check QUERY_PROFILE for aggregation performance | `SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))` |

#### **5. Examples**

**Example 1: Basic Aggregations**
```sql
-- Basic aggregations by region
SELECT
    region,
    COUNT(*) AS customer_count,
    SUM(total_spend) AS total_revenue,
    AVG(total_spend) AS avg_revenue,
    MIN(total_spend) AS min_revenue,
    MAX(total_spend) AS max_revenue
FROM
    customers
WHERE
    signup_date > '2023-01-01'
GROUP BY
    region
ORDER BY
    total_revenue DESC;
```

**Example 2: Aggregations with Filtering**
```sql
-- Aggregations with WHERE and HAVING
SELECT
    product_category,
    COUNT(*) AS order_count,
    SUM(amount) AS total_sales,
    AVG(amount) AS avg_sale_amount
FROM
    orders
WHERE
    order_date > CURRENT_DATE() - 30
GROUP BY
    product_category
HAVING
    COUNT(*) > 100
    AND SUM(amount) > 10000
ORDER BY
    total_sales DESC;
```

**Example 3: ROLLUP for Hierarchical Aggregations**
```sql
-- ROLLUP for hierarchical aggregations (region -> total)
SELECT
    region,
    product_category,
    SUM(amount) AS total_sales
FROM
    sales
WHERE
    sale_date > CURRENT_DATE() - 30
GROUP BY
    ROLLUP(region, product_category)
ORDER BY
    region, product_category;
```

**Example 4: CUBE for Multi-Dimensional Aggregations**
```sql
-- CUBE for all possible groupings (region, product_category, both, none)
SELECT
    region,
    product_category,
    SUM(amount) AS total_sales
FROM
    sales
WHERE
    sale_date > CURRENT_DATE() - 30
GROUP BY
    CUBE(region, product_category)
ORDER BY
    region, product_category;
```

**Example 5: GROUPING SETS for Multiple Aggregations**
```sql
-- GROUPING SETS for multiple GROUP BY clauses
SELECT
    region,
    product_category,
    SUM(amount) AS total_sales
FROM
    sales
WHERE
    sale_date > CURRENT_DATE() - 30
GROUP BY
    GROUPING SETS (
        (region),
        (product_category),
        (region, product_category),
        ()
    )
ORDER BY
    region, product_category;
```

**Example 6: Approximate Aggregations for Large Datasets**
```sql
-- Approximate aggregations for large datasets
SELECT
    region,
    APPROX_COUNT_DISTINCT(customer_id) AS unique_customers,
    APPROX_QUANTILE(amount, 0.5) AS median_sale_amount,
    APPROX_TOP_K(product_id, 10, amount) AS top_products
FROM
    sales
WHERE
    sale_date > CURRENT_DATE() - 30
GROUP BY
    region
ORDER BY
    unique_customers DESC;
```

### **D. Window Functions**

#### **1. Window Function Overview**
Window functions perform **calculations across a set of table rows** that are **related to the current row**. Unlike GROUP BY, which **collapses rows**, window functions **retain the individual rows** while adding aggregated or computed values.

**Basic Syntax**:
```sql
SELECT
    column1,
    window_function(column2) OVER (
        [PARTITION BY column3, column4, ...]
        [ORDER BY column5 [ASC|DESC], ...]
        [frame_clause]
    ) [AS alias],
    ...
FROM
    table_name
[WHERE
    condition]
[ORDER BY
    column1 [ASC|DESC], ...];
```

**Frame Clause**:
```sql
-- Frame specification
[ROWS | RANGE] BETWEEN
    {UNBOUNDED PRECEDING | offset PRECEDING | CURRENT ROW | offset FOLLOWING | UNBOUNDED FOLLOWING}
    AND
    {UNBOUNDED PRECEDING | offset PRECEDING | CURRENT ROW | offset FOLLOWING | UNBOUNDED FOLLOWING}
```

#### **2. Window Function Types**

| **Category** | **Function** | **Description** | **Example** | **Performance** | **Use Cases** |
|--------------|--------------|-----------------|-------------|-----------------|--------------|
| **Ranking** | ROW_NUMBER() | Assigns a unique sequential integer to rows | `ROW_NUMBER() OVER (ORDER BY sales)` | ⭐⭐⭐⭐ | Pagination, top N queries |
| **Ranking** | RANK() | Assigns a rank with gaps for ties | `RANK() OVER (ORDER BY sales DESC)` | ⭐⭐⭐⭐ | Ranking with ties |
| **Ranking** | DENSE_RANK() | Assigns a rank without gaps for ties | `DENSE_RANK() OVER (ORDER BY sales DESC)` | ⭐⭐⭐⭐ | Ranking without gaps |
| **Ranking** | NTILE(n) | Divides rows into n groups | `NTILE(4) OVER (ORDER BY sales)` | ⭐⭐⭐⭐ | Quartiles, percentiles |
| **Aggregate** | SUM() | Running or window sum | `SUM(sales) OVER (PARTITION BY region ORDER BY date)` | ⭐⭐⭐⭐ | Running totals, cumulative sums |
| **Aggregate** | AVG() | Running or window average | `AVG(sales) OVER (PARTITION BY region ORDER BY date)` | ⭐⭐⭐⭐ | Moving averages |
| **Aggregate** | COUNT() | Running or window count | `COUNT(*) OVER (PARTITION BY region ORDER BY date)` | ⭐⭐⭐⭐ | Running counts |
| **Aggregate** | MIN() | Running or window minimum | `MIN(sales) OVER (PARTITION BY region ORDER BY date)` | ⭐⭐⭐⭐ | Running minimums |
| **Aggregate** | MAX() | Running or window maximum | `MAX(sales) OVER (PARTITION BY region ORDER BY date)` | ⭐⭐⭐⭐ | Running maximums |
| **Value** | LAG(n) | Accesses data from a previous row | `LAG(sales, 1) OVER (ORDER BY date)` | ⭐⭐⭐⭐ | Previous period comparisons |
| **Value** | LEAD(n) | Accesses data from a subsequent row | `LEAD(sales, 1) OVER (ORDER BY date)` | ⭐⭐⭐⭐ | Next period comparisons |
| **Value** | FIRST_VALUE() | Returns the first value in the window | `FIRST_VALUE(sales) OVER (PARTITION BY region ORDER BY date)` | ⭐⭐⭐⭐ | First value in group |
| **Value** | LAST_VALUE() | Returns the last value in the window | `LAST_VALUE(sales) OVER (PARTITION BY region ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)` | ⭐⭐⭐⭐ | Last value in group |
| **Value** | NTH_VALUE(n) | Returns the nth value in the window | `NTH_VALUE(sales, 2) OVER (PARTITION BY region ORDER BY date)` | ⭐⭐⭐ | Nth value in group |
| **Analytic** | PERCENT_RANK() | Returns the relative rank of a row | `PERCENT_RANK() OVER (ORDER BY sales)` | ⭐⭐⭐ | Percentile ranking |
| **Analytic** | CUME_DIST() | Returns the cumulative distribution | `CUME_DIST() OVER (ORDER BY sales)` | ⭐⭐⭐ | Cumulative distribution |
| **Analytic** | PERCENTILE_CONT(n) | Returns the nth percentile | `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY sales) OVER (PARTITION BY region)` | ⭐⭐⭐ | Median, quartiles |
| **Analytic** | PERCENTILE_DISC(n) | Returns the nth percentile (discrete) | `PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY sales) OVER (PARTITION BY region)` | ⭐⭐⭐ | Discrete percentiles |

#### **3. Window Function Optimization Techniques**

| **Technique** | **Description** | **Example** | **Performance Impact** |
|---------------|-----------------|-------------|-------------------------|
| **Partition by Clustered Columns** | Partition by clustered columns for better performance | `SELECT *, SUM(sales) OVER (PARTITION BY region ORDER BY date) FROM my_table CLUSTER BY (region)` | ⭐⭐⭐⭐ (2-10x faster) |
| **Limit Window Frame** | Use RANGE or ROWS to limit the window frame | `SUM(sales) OVER (PARTITION BY region ORDER BY date ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING)` | ⭐⭐⭐⭐ (2-10x faster) |
| **Avoid Unbounded Windows** | Unbounded windows can be expensive | `SUM(sales) OVER (PARTITION BY region ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` | ⭐⭐⭐ (1.5-5x faster) |
| **Use Materialized Views** | Pre-compute window function results | `CREATE MATERIALIZED VIEW my_mv AS SELECT *, SUM(sales) OVER (PARTITION BY region ORDER BY date) FROM my_table` | ⭐⭐⭐⭐ (10-100x faster) |
| **Use QUALIFY for Filtering Window Results** | Filter window function results | `SELECT *, ROW_NUMBER() OVER (PARTITION BY region ORDER BY sales DESC) AS rn FROM my_table QUALIFY rn <= 10` | ⭐⭐⭐⭐ (2-10x faster) |
| **Use INDEX OFF for Large Windows** | Disable index usage for large windows | `SELECT *, SUM(sales) OVER (PARTITION BY region ORDER BY date) FROM my_table /*+ INDEX(OFF) */` | ⭐⭐⭐ (1.5-5x faster) |
| **Avoid Redundant Window Functions** | Remove window functions that are not needed | `SELECT *, ROW_NUMBER() OVER (ORDER BY id) AS rn FROM my_table` (if rn is not used) | ⭐⭐⭐ (1.5-5x faster) |
| **Use Simple Window Definitions** | Use simple PARTITION BY and ORDER BY | `SUM(sales) OVER (PARTITION BY region ORDER BY date)` | ⭐⭐⭐ (1.5-5x faster) |
| **Monitor Window Function Performance** | Check QUERY_PROFILE for window function performance | `SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))` | N/A |

#### **4. Best Practices for Window Functions**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Partition by Clustered Columns** | Partition by clustered columns for better performance | `SELECT *, SUM(sales) OVER (PARTITION BY region ORDER BY date) FROM my_table CLUSTER BY (region)` |
| **Limit Window Frame** | Use RANGE or ROWS to limit the window frame | `SUM(sales) OVER (PARTITION BY region ORDER BY date ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING)` |
| **Avoid Unbounded Windows** | Unbounded windows can be expensive | `SUM(sales) OVER (PARTITION BY region ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` |
| **Use Materialized Views for Window Functions** | Pre-compute window function results | `CREATE MATERIALIZED VIEW my_mv AS SELECT *, SUM(sales) OVER (PARTITION BY region ORDER BY date) FROM my_table` |
| **Use QUALIFY for Filtering Window Results** | Filter window function results | `SELECT *, ROW_NUMBER() OVER (PARTITION BY region ORDER BY sales DESC) AS rn FROM my_table QUALIFY rn <= 10` |
| **Avoid Redundant Window Functions** | Remove window functions that are not needed | `SELECT col1, col2 FROM my_table` (not `SELECT col1, col2, ROW_NUMBER() OVER (ORDER BY id) AS rn FROM my_table` if rn is not used) |
| **Use Simple Window Definitions** | Use simple PARTITION BY and ORDER BY | `SUM(sales) OVER (PARTITION BY region ORDER BY date)` |
| **Use INDEX OFF for Large Windows** | Disable index usage for large windows | `SELECT *, SUM(sales) OVER (PARTITION BY region ORDER BY date) FROM my_table /*+ INDEX(OFF) */` |
| **Monitor Window Function Performance** | Check QUERY_PROFILE for window function performance | `SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))` |
| **Use Window Functions for Running Totals** | Calculate running totals efficiently | `SELECT date, sales, SUM(sales) OVER (ORDER BY date) AS running_total FROM my_table` |
| **Use Window Functions for Rankings** | Calculate rankings efficiently | `SELECT customer_id, sales, RANK() OVER (ORDER BY sales DESC) AS sales_rank FROM my_table` |
| **Use Window Functions for Moving Averages** | Calculate moving averages efficiently | `SELECT date, sales, AVG(sales) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS moving_avg FROM my_table` |

#### **5. Examples**

**Example 1: Running Totals**
```sql
-- Running totals by date
SELECT
    date,
    sales,
    SUM(sales) OVER (ORDER BY date) AS running_total,
    SUM(sales) OVER (ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total_explicit
FROM
    sales
ORDER BY
    date;
```

**Example 2: Rankings**
```sql
-- Rankings by sales
SELECT
    customer_id,
    customer_name,
    sales,
    RANK() OVER (ORDER BY sales DESC) AS sales_rank,
    DENSE_RANK() OVER (ORDER BY sales DESC) AS dense_rank,
    ROW_NUMBER() OVER (ORDER BY sales DESC) AS row_num,
    NTILE(4) OVER (ORDER BY sales DESC) AS quartile
FROM
    customers
ORDER BY
    sales DESC;
```

**Example 3: Moving Averages**
```sql
-- Moving averages (7-day)
SELECT
    date,
    sales,
    AVG(sales) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7day
FROM
    sales
ORDER BY
    date;
```

**Example 4: Partitioned Window Functions**
```sql
-- Window functions with PARTITION BY
SELECT
    region,
    date,
    sales,
    SUM(sales) OVER (PARTITION BY region ORDER BY date) AS region_running_total,
    AVG(sales) OVER (PARTITION BY region ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS region_moving_avg,
    RANK() OVER (PARTITION BY region ORDER BY sales DESC) AS region_sales_rank
FROM
    sales
ORDER BY
    region, date;
```

**Example 5: LAG and LEAD**
```sql
-- LAG and LEAD for period-over-period comparisons
SELECT
    date,
    sales,
    LAG(sales, 1) OVER (ORDER BY date) AS previous_day_sales,
    LEAD(sales, 1) OVER (ORDER BY date) AS next_day_sales,
    sales - LAG(sales, 1) OVER (ORDER BY date) AS day_over_day_change,
    (sales - LAG(sales, 1) OVER (ORDER BY date)) / LAG(sales, 1) OVER (ORDER BY date) * 100 AS day_over_day_pct_change
FROM
    sales
ORDER BY
    date;
```

**Example 6: FIRST_VALUE and LAST_VALUE**
```sql
-- FIRST_VALUE and LAST_VALUE
SELECT
    region,
    date,
    sales,
    FIRST_VALUE(sales) OVER (PARTITION BY region ORDER BY date) AS first_sale_in_region,
    LAST_VALUE(sales) OVER (
        PARTITION BY region
        ORDER BY date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_sale_in_region
FROM
    sales
ORDER BY
    region, date;
```

**Example 7: QUALIFY for Filtering Window Results**
```sql
-- QUALIFY for filtering window results (top 3 sales by region)
SELECT
    region,
    customer_id,
    customer_name,
    sales,
    RANK() OVER (PARTITION BY region ORDER BY sales DESC) AS sales_rank
FROM
    customers
QUALIFY
    sales_rank <= 3
ORDER BY
    region, sales_rank;
```

**Example 8: Complex Window Function Query**
```sql
-- Complex window function query with multiple window functions
SELECT
    region,
    product_category,
    date,
    sales,
    -- Running totals
    SUM(sales) OVER (PARTITION BY region ORDER BY date) AS region_running_total,
    SUM(sales) OVER (PARTITION BY product_category ORDER BY date) AS category_running_total,
    -- Moving averages
    AVG(sales) OVER (
        PARTITION BY region
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS region_moving_avg_7day,
    -- Rankings
    RANK() OVER (PARTITION BY region ORDER BY sales DESC) AS region_sales_rank,
    RANK() OVER (PARTITION BY product_category ORDER BY sales DESC) AS category_sales_rank,
    -- Period-over-period
    LAG(sales, 7) OVER (PARTITION BY region ORDER BY date) AS sales_7_days_ago,
    sales - LAG(sales, 7) OVER (PARTITION BY region ORDER BY date) AS week_over_week_change
FROM
    sales
WHERE
    date > CURRENT_DATE() - 30
ORDER BY
    region, product_category, date;
```

### **D. PIVOT and UNPIVOT**

#### **1. PIVOT: Reshaping Data from Rows to Columns**
The `PIVOT` function **transforms rows into columns**, which is useful for **cross-tab reports** and **data reshaping**.

**Basic Syntax**:
```sql
SELECT *
FROM
    (
        SELECT
            row_grouping_columns,
            pivot_column,
            value_column
        FROM
            source_table
    )
PIVOT (
    aggregate_function(value_column)
    FOR pivot_column IN (pivot_column_value1, pivot_column_value2, ...)
) [AS alias];
```

#### **2. UNPIVOT: Reshaping Data from Columns to Rows**
The `UNPIVOT` function **transforms columns into rows**, which is useful for **normalizing data** and **preparing for analysis**.

**Basic Syntax**:
```sql
SELECT *
FROM
    source_table
UNPIVOT (
    value_column
    FOR pivot_column IN (column1, column2, ...)
) [AS alias];
```

#### **3. PIVOT/UNPIVOT Optimization Techniques**

| **Technique** | **Description** | **Example** | **Performance Impact** |
|---------------|-----------------|-------------|-------------------------|
| **Filter Before PIVOT** | Reduce data volume before pivoting | `SELECT * FROM (SELECT * FROM my_table WHERE ...) PIVOT (...)` | ⭐⭐⭐⭐ (2-10x faster) |
| **Use Materialized Views for PIVOT** | Pre-compute pivot results | `CREATE MATERIALIZED VIEW my_mv AS SELECT * FROM my_table PIVOT (...)` | ⭐⭐⭐⭐ (10-100x faster) |
| **Limit PIVOT Columns** | Only pivot necessary columns | `PIVOT (SUM(sales) FOR region IN ('US', 'EU', 'APAC'))` | ⭐⭐⭐⭐ (2-10x faster) |
| **Use Conditional Aggregation** | Alternative to PIVOT for simple cases | `SELECT region, SUM(CASE WHEN product = 'A' THEN sales ELSE 0 END) AS sales_A, ...` | ⭐⭐⭐⭐ (2-10x faster) |
| **Avoid PIVOT on Large Datasets** | PIVOT can be expensive for large datasets | Use filtering or sampling | ⭐⭐⭐ (1.5-5x faster) |

#### **4. Best Practices for PIVOT/UNPIVOT**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Filter Before PIVOT** | Reduce data volume before pivoting | `SELECT * FROM (SELECT * FROM my_table WHERE date > '2023-01-01') PIVOT (...)` |
| **Use Materialized Views for PIVOT** | Pre-compute pivot results | `CREATE MATERIALIZED VIEW my_mv AS SELECT * FROM my_table PIVOT (...)` |
| **Limit PIVOT Columns** | Only pivot necessary columns | `PIVOT (SUM(sales) FOR region IN ('US', 'EU'))` |
| **Use Conditional Aggregation for Simple PIVOTs** | Alternative to PIVOT for simple cases | `SELECT region, SUM(CASE WHEN product = 'A' THEN sales ELSE 0 END) AS sales_A` |
| **Avoid PIVOT on Large Datasets** | PIVOT can be expensive for large datasets | Use filtering or sampling |
| **Use UNPIVOT for Data Normalization** | Normalize denormalized data | `SELECT * FROM my_table UNPIVOT (...)` |
| **Monitor PIVOT/UNPIVOT Performance** | Check QUERY_PROFILE for performance | `SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))` |
| **Use PIVOT for Reporting** | Create cross-tab reports | `SELECT * FROM sales PIVOT (SUM(amount) FOR region IN ('US', 'EU', 'APAC'))` |
| **Use UNPIVOT for Data Preparation** | Prepare data for analysis | `SELECT * FROM denormalized_table UNPIVOT (value FOR metric IN (col1, col2, col3))` |

#### **5. Examples**

**Example 1: Basic PIVOT**
```sql
-- Pivot sales by region
SELECT *
FROM
    (
        SELECT
            product_category,
            region,
            SUM(amount) AS sales
        FROM
            sales
        WHERE
            sale_date > CURRENT_DATE() - 30
        GROUP BY
            product_category, region
    )
PIVOT (
    SUM(sales)
    FOR region IN ('US', 'EU', 'APAC', 'Other')
) AS pivot_table
ORDER BY
    product_category;
```

**Example 2: PIVOT with Multiple Aggregations**
```sql
-- Pivot with multiple aggregations
SELECT *
FROM
    (
        SELECT
            product_category,
            region,
            SUM(amount) AS sales,
            COUNT(*) AS order_count,
            AVG(amount) AS avg_sale
        FROM
            sales
        WHERE
            sale_date > CURRENT_DATE() - 30
        GROUP BY
            product_category, region
    )
PIVOT (
    SUM(sales) AS total_sales,
    SUM(order_count) AS total_orders,
    AVG(avg_sale) AS avg_sale_amount
    FOR region IN ('US', 'EU', 'APAC')
) AS pivot_table
ORDER BY
    product_category;
```

**Example 3: Conditional Aggregation (Alternative to PIVOT)**
```sql
-- Conditional aggregation as alternative to PIVOT
SELECT
    product_category,
    SUM(CASE WHEN region = 'US' THEN amount ELSE 0 END) AS sales_us,
    SUM(CASE WHEN region = 'EU' THEN amount ELSE 0 END) AS sales_eu,
    SUM(CASE WHEN region = 'APAC' THEN amount ELSE 0 END) AS sales_apac,
    SUM(CASE WHEN region NOT IN ('US', 'EU', 'APAC') THEN amount ELSE 0 END) AS sales_other
FROM
    sales
WHERE
    sale_date > CURRENT_DATE() - 30
GROUP BY
    product_category
ORDER BY
    product_category;
```

**Example 4: Basic UNPIVOT**
```sql
-- Unpivot denormalized data
SELECT *
FROM
    sales_by_region
UNPIVOT (
    sales
    FOR region IN (us_sales, eu_sales, apac_sales)
) AS unpivot_table
ORDER BY
    product_category, region;
```

**Example 5: UNPIVOT with Multiple Columns**
```sql
-- Unpivot with multiple columns
SELECT *
FROM
    sales_metrics
UNPIVOT (
    value
    FOR metric IN (total_sales, total_orders, avg_sale_amount)
) AS unpivot_table
ORDER BY
    region, metric;
```

**Example 6: PIVOT and UNPIVOT in ETL Pipeline**
```sql
-- ETL pipeline with PIVOT and UNPIVOT
-- Step 1: Extract and normalize data with UNPIVOT
CREATE TEMPORARY TABLE temp_normalized_sales AS
SELECT *
FROM
    denormalized_sales
UNPIVOT (
    sales
    FOR region IN (us_sales, eu_sales, apac_sales, other_sales)
);

-- Step 2: Transform data
CREATE TEMPORARY TABLE temp_transformed_sales AS
SELECT
    product_category,
    region,
    date,
    sales,
    sales * 1.1 AS sales_with_tax
FROM
    temp_normalized_sales;

-- Step 3: Pivot data for reporting
CREATE TEMPORARY TABLE temp_pivoted_sales AS
SELECT *
FROM
    (
        SELECT
            product_category,
            date,
            region,
            SUM(sales_with_tax) AS sales
        FROM
            temp_transformed_sales
        GROUP BY
            product_category, date, region
    )
PIVOT (
    SUM(sales)
    FOR region IN ('us_sales', 'eu_sales', 'apac_sales', 'other_sales')
);

-- Step 4: Load to target
INSERT INTO target_sales_report
SELECT * FROM temp_pivoted_sales;
```

### **E. JSON and Semi-Structured Data Transformation**

#### **1. JSON Data in Snowflake**
Snowflake provides **extensive support** for **JSON data** through:
- **VARIANT** data type: Stores semi-structured data (JSON, Avro, Parquet)
- **OBJECT** data type: Stores key-value pairs
- **ARRAY** data type: Stores arrays
- **JSON functions**: Extract and manipulate JSON data

#### **2. JSON Functions**

| **Function** | **Description** | **Example** | **Performance** | **Use Cases** |
|--------------|-----------------|-------------|-----------------|--------------|
| **PARSE_JSON** | Parses a JSON string into a VARIANT | `PARSE_JSON(json_string)` | ⭐⭐⭐⭐ | Converting JSON strings to VARIANT |
| **TO_JSON** | Converts a VARIANT to a JSON string | `TO_JSON(variant_column)` | ⭐⭐⭐⭐ | Converting VARIANT to JSON string |
| **OBJECT_CONSTRUCT** | Creates an OBJECT from key-value pairs | `OBJECT_CONSTRUCT('key1', value1, 'key2', value2)` | ⭐⭐⭐⭐ | Creating JSON objects |
| **ARRAY_CONSTRUCT** | Creates an ARRAY from values | `ARRAY_CONSTRUCT(value1, value2, value3)` | ⭐⭐⭐⭐ | Creating JSON arrays |
| **OBJECT_INSERT** | Inserts a key-value pair into an OBJECT | `OBJECT_INSERT(object, 'key', value)` | ⭐⭐⭐ | Modifying JSON objects |
| **OBJECT_DELETE** | Deletes a key from an OBJECT | `OBJECT_DELETE(object, 'key')` | ⭐⭐⭐ | Modifying JSON objects |
| **ARRAY_APPEND** | Appends a value to an ARRAY | `ARRAY_APPEND(array, value)` | ⭐⭐⭐ | Modifying JSON arrays |
| **ARRAY_REMOVE** | Removes a value from an ARRAY | `ARRAY_REMOVE(array, value)` | ⭐⭐⭐ | Modifying JSON arrays |
| **GET_PATH** | Extracts a value from a VARIANT using a path | `GET_PATH(variant_column, '$.path.to.value')` | ⭐⭐⭐⭐ | Extracting nested JSON values |
| **GET_OBJECT** | Extracts an OBJECT from a VARIANT | `GET_OBJECT(variant_column, 'key')` | ⭐⭐⭐⭐ | Extracting JSON objects |
| **GET_ARRAY** | Extracts an ARRAY from a VARIANT | `GET_ARRAY(variant_column, 'key')` | ⭐⭐⭐⭐ | Extracting JSON arrays |
| **FLATTEN** | Flattens a VARIANT containing an ARRAY of OBJECTs | `FLATTEN(variant_column)` | ⭐⭐⭐ | Flattening nested JSON |
| **LATERAL FLATTEN** | Flattens a VARIANT in a LATERAL JOIN | `SELECT f.value FROM my_table, LATERAL FLATTEN(input => json_column) f` | ⭐⭐⭐⭐ | Flattening nested JSON in joins |
| **JSON_EXTRACT_PATH_TEXT** | Extracts a string value from a JSON path | `JSON_EXTRACT_PATH_TEXT(json_column, '$.path.to.value')` | ⭐⭐⭐⭐ | Extracting string values from JSON |
| **JSON_EXTRACT** | Extracts a VARIANT value from a JSON path | `JSON_EXTRACT(json_column, '$.path.to.value')` | ⭐⭐⭐⭐ | Extracting VARIANT values from JSON |

#### **3. Semi-Structured Data Functions**

| **Function** | **Description** | **Example** | **Performance** | **Use Cases** |
|--------------|-----------------|-------------|-----------------|--------------|
| **ARRAY_SIZE** | Returns the size of an ARRAY | `ARRAY_SIZE(array_column)` | ⭐⭐⭐⭐ | Getting array length |
| **ARRAY_CONTAINS** | Checks if an ARRAY contains a value | `ARRAY_CONTAINS(array_column, value)` | ⭐⭐⭐⭐ | Checking for values in arrays |
| **ARRAY_POSITION** | Returns the position of a value in an ARRAY | `ARRAY_POSITION(array_column, value)` | ⭐⭐⭐ | Finding value positions in arrays |
| **ARRAY_SLICE** | Returns a slice of an ARRAY | `ARRAY_SLICE(array_column, start, end)` | ⭐⭐⭐ | Extracting array slices |
| **ARRAY_TO_STRING** | Converts an ARRAY to a string | `ARRAY_TO_STRING(array_column, delimiter)` | ⭐⭐⭐ | Converting arrays to strings |
| **STRING_TO_ARRAY** | Converts a string to an ARRAY | `STRING_TO_ARRAY(string_column, delimiter)` | ⭐⭐⭐ | Converting strings to arrays |
| **OBJECT_KEYS** | Returns the keys of an OBJECT | `OBJECT_KEYS(object_column)` | ⭐⭐⭐ | Getting object keys |
| **OBJECT_AGG** | Aggregates OBJECTs | `OBJECT_AGG(key, value) WITHIN GROUP (ORDER BY key)` | ⭐⭐⭐ | Aggregating key-value pairs |
| **ARRAY_AGG** | Aggregates values into an ARRAY | `ARRAY_AGG(value) WITHIN GROUP (ORDER BY value)` | ⭐⭐⭐ | Aggregating values into arrays |

#### **4. JSON Transformation Optimization Techniques**

| **Technique** | **Description** | **Example** | **Performance Impact** |
|---------------|-----------------|-------------|-------------------------|
| **Use VARIANT Data Type** | Store JSON in VARIANT columns | `CREATE TABLE my_table (json_data VARIANT)` | ⭐⭐⭐⭐ (2-10x faster) |
| **Use FLATTEN for Nested Data** | Flatten nested JSON structures | `SELECT f.value FROM my_table, LATERAL FLATTEN(input => json_column) f` | ⭐⭐⭐⭐ (2-10x faster) |
| **Use JSON Path Expressions** | Extract specific values from JSON | `SELECT GET_PATH(json_column, '$.path.to.value') FROM my_table` | ⭐⭐⭐⭐ (2-10x faster) |
| **Use Materialized Views for JSON** | Pre-compute JSON transformations | `CREATE MATERIALIZED VIEW my_mv AS SELECT GET_PATH(json_column, '$.path') FROM my_table` | ⭐⭐⭐⭐ (10-100x faster) |
| **Use Clustering on JSON Columns** | Cluster on frequently accessed JSON paths | `ALTER TABLE my_table CLUSTER BY (GET_PATH(json_column, '$.path'))` | ⭐⭐⭐⭐ (2-10x faster) |
| **Avoid Parsing Large JSON** | Parse JSON in smaller chunks | Use FLATTEN or limit data | ⭐⭐⭐ (1.5-5x faster) |
| **Use OBJECT and ARRAY Functions** | Use built-in functions for JSON manipulation | `SELECT OBJECT_INSERT(json_column, 'key', value) FROM my_table` | ⭐⭐⭐⭐ (2-10x faster) |
| **Monitor JSON Transformation Performance** | Check QUERY_PROFILE for performance | `SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))` | N/A |

#### **5. Best Practices for JSON and Semi-Structured Data**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use VARIANT for JSON Data** | Store JSON in VARIANT columns for best performance | `CREATE TABLE my_table (json_data VARIANT)` |
| **Use FLATTEN for Nested Data** | Flatten nested JSON structures for querying | `SELECT f.value FROM my_table, LATERAL FLATTEN(input => json_column) f` |
| **Use JSON Path Expressions** | Extract specific values from JSON using paths | `SELECT GET_PATH(json_column, '$.path.to.value') FROM my_table` |
| **Use Materialized Views for JSON** | Pre-compute JSON transformations | `CREATE MATERIALIZED VIEW my_mv AS SELECT GET_PATH(json_column, '$.path') FROM my_table` |
| **Use Clustering on JSON Columns** | Cluster on frequently accessed JSON paths | `ALTER TABLE my_table CLUSTER BY (GET_PATH(json_column, '$.path'))` |
| **Avoid Parsing Large JSON** | Parse JSON in smaller chunks | Use FLATTEN or limit data |
| **Use OBJECT and ARRAY Functions** | Use built-in functions for JSON manipulation | `SELECT OBJECT_INSERT(json_column, 'key', value) FROM my_table` |
| **Use LATERAL FLATTEN for Complex JSON** | Flatten complex JSON structures in joins | `SELECT f.value FROM my_table, LATERAL FLATTEN(input => json_column) f` |
| **Use JSON Schema Validation** | Validate JSON data on ingestion | `CREATE FILE FORMAT my_json_format TYPE = 'JSON' STRIP_OUTER_ARRAY = TRUE` |
| **Monitor JSON Transformation Performance** | Check QUERY_PROFILE for performance | `SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))` |

#### **6. Examples**

**Example 1: Basic JSON Extraction**
```sql
-- Extract values from JSON
SELECT
    json_data,
    GET_PATH(json_data, '$.customer.id') AS customer_id,
    GET_PATH(json_data, '$.customer.name') AS customer_name,
    GET_PATH(json_data, '$.order.total') AS order_total,
    GET_PATH(json_data, '$.order.items[0].product_id') AS first_product_id
FROM
    json_orders;
```

**Example 2: FLATTEN for Nested Arrays**
```sql
-- Flatten nested JSON arrays
SELECT
    o.order_id,
    o.customer_id,
    f.value:product_id AS product_id,
    f.value:product_name AS product_name,
    f.value:quantity AS quantity,
    f.value:price AS price
FROM
    orders o,
    LATERAL FLATTEN(input => o.json_data:items) f
WHERE
    o.order_date > CURRENT_DATE() - 30;
```

**Example 3: JSON Aggregation**
```sql
-- Aggregate JSON data
SELECT
    region,
    OBJECT_AGG(
        customer_id,
        OBJECT_CONSTRUCT(
            'name', customer_name,
            'total_spend', total_spend
        )
    ) AS customers
FROM
    customers
WHERE
    signup_date > '2023-01-01'
GROUP BY
    region;
```

**Example 4: JSON Transformation Pipeline**
```sql
-- JSON transformation pipeline
-- Step 1: Extract and flatten JSON data
CREATE TEMPORARY TABLE temp_flattened_orders AS
SELECT
    o.order_id,
    o.order_date,
    o.customer_id,
    f.value:product_id AS product_id,
    f.value:product_name AS product_name,
    f.value:quantity AS quantity,
    f.value:price AS price,
    f.value:quantity * f.value:price AS line_total
FROM
    json_orders o,
    LATERAL FLATTEN(input => o.json_data:items) f;

-- Step 2: Aggregate data
CREATE TEMPORARY TABLE temp_aggregated_orders AS
SELECT
    customer_id,
    order_id,
    order_date,
    SUM(line_total) AS order_total,
    COUNT(*) AS item_count,
    ARRAY_AGG(product_name) AS products
FROM
    temp_flattened_orders
GROUP BY
    customer_id, order_id, order_date;

-- Step 3: Join with customer data
CREATE TEMPORARY TABLE temp_enriched_orders AS
SELECT
    a.*,
    c.customer_name,
    c.region,
    c.signup_date
FROM
    temp_aggregated_orders a
JOIN
    customers c ON a.customer_id = c.customer_id;

-- Step 4: Load to target
INSERT INTO target_orders
SELECT * FROM temp_enriched_orders;
```

**Example 5: JSON to Relational Transformation**
```sql
-- Transform JSON to relational tables
-- Step 1: Create customers table from JSON
INSERT INTO customers (customer_id, customer_name, email, signup_date)
SELECT
    GET_PATH(json_data, '$.customer.id') AS customer_id,
    GET_PATH(json_data, '$.customer.name') AS customer_name,
    GET_PATH(json_data, '$.customer.email') AS email,
    PARSE_JSON(GET_PATH(json_data, '$.customer.signup_date')) AS signup_date
FROM
    json_orders
GROUP BY
    customer_id, customer_name, email, signup_date;

-- Step 2: Create orders table from JSON
INSERT INTO orders (order_id, customer_id, order_date, total_amount)
SELECT
    GET_PATH(json_data, '$.order.id') AS order_id,
    GET_PATH(json_data, '$.customer.id') AS customer_id,
    PARSE_JSON(GET_PATH(json_data, '$.order.date')) AS order_date,
    GET_PATH(json_data, '$.order.total') AS total_amount
FROM
    json_orders
GROUP BY
    order_id, customer_id, order_date, total_amount;

-- Step 3: Create order_items table from JSON
INSERT INTO order_items (order_id, product_id, product_name, quantity, price)
SELECT
    o.order_id,
    f.value:product_id AS product_id,
    f.value:product_name AS product_name,
    f.value:quantity AS quantity,
    f.value:price AS price
FROM
    json_orders o,
    LATERAL FLATTEN(input => o.json_data:items) f;
```

**Example 6: Complex JSON Query**
```sql
-- Complex JSON query with multiple extractions and aggregations
SELECT
    region,
    COUNT(DISTINCT customer_id) AS unique_customers,
    SUM(order_total) AS total_sales,
    AVG(order_total) AS avg_order_value,
    ARRAY_AGG(DISTINCT product_category) AS popular_categories,
    OBJECT_AGG(
        product_category,
        SUM(line_total)
    ) AS sales_by_category
FROM
    (
        SELECT
            c.region,
            o.customer_id,
            o.order_id,
            o.order_date,
            o.order_total,
            f.value:product_id AS product_id,
            f.value:product_category AS product_category,
            f.value:quantity * f.value:price AS line_total
        FROM
            customers c
        JOIN
            json_orders o ON c.customer_id = GET_PATH(o.json_data, '$.customer.id')
        JOIN
            LATERAL FLATTEN(input => o.json_data:items) f
        WHERE
            o.order_date > CURRENT_DATE() - 30
    ) subq
GROUP BY
    region
ORDER BY
    total_sales DESC;
```

### **F. CTEs (Common Table Expressions) and Subqueries**

#### **1. CTEs (WITH Clause)**
CTEs (Common Table Expressions) are **temporary result sets** defined within a query that can be **referenced multiple times** in the same query. CTEs improve **readability**, **maintainability**, and **performance** (through query optimization).

**Basic Syntax**:
```sql
WITH
    cte_name1 AS (
        SELECT ... FROM ...
    ),
    cte_name2 AS (
        SELECT ... FROM cte_name1 JOIN ...
    ),
    ...
SELECT ... FROM cte_name1, cte_name2, ...
```

#### **2. CTE Types**

| **Type** | **Description** | **Example** | **Performance** | **Use Cases** |
|----------|-----------------|-------------|-----------------|--------------|
| **Non-Recursive CTE** | Standard CTE that can be referenced multiple times | `WITH cte AS (SELECT * FROM my_table) SELECT * FROM cte` | ⭐⭐⭐⭐ | Query readability, performance optimization |
| **Recursive CTE** | CTE that references itself | `WITH RECURSIVE cte AS (SELECT 1 AS n UNION ALL SELECT n + 1 FROM cte WHERE n < 10) SELECT * FROM cte` | ⭐⭐ | Hierarchical data, graph traversal |
| **Materialized CTE** | CTE that is materialized during query execution | `WITH MATERIALIZE cte AS (SELECT * FROM my_table) SELECT * FROM cte` | ⭐⭐⭐⭐⭐ | Performance optimization for complex queries |
| **Non-Materialized CTE** | CTE that is inlined during query execution (default) | `WITH cte AS (SELECT * FROM my_table) SELECT * FROM cte` | ⭐⭐⭐ | Query readability |

#### **3. Subqueries**
Subqueries are **queries nested within other queries**. They can be used in:
- **SELECT clause** (scalar subqueries)
- **FROM clause** (inline views)
- **WHERE clause** (filter subqueries)
- **HAVING clause** (filter subqueries)
- **WITH clause** (CTEs)

**Subquery Types**:

| **Type** | **Description** | **Example** | **Performance** | **Use Cases** |
|----------|-----------------|-------------|-----------------|--------------|
| **Scalar Subquery** | Returns a single value | `SELECT col1, (SELECT MAX(col2) FROM t2) AS max_col2 FROM t1` | ⭐⭐⭐ | Single value lookups |
| **Row Subquery** | Returns a single row | `SELECT * FROM t1 WHERE (col1, col2) = (SELECT col1, col2 FROM t2 LIMIT 1)` | ⭐⭐ | Single row lookups |
| **Table Subquery** | Returns multiple rows and columns | `SELECT * FROM (SELECT col1, col2 FROM t1) AS subq` | ⭐⭐⭐ | Inline views, derived tables |
| **Correlated Subquery** | References columns from the outer query | `SELECT * FROM t1 WHERE col1 IN (SELECT col2 FROM t2 WHERE t2.col3 = t1.col3)` | ⭐ | Filtering based on related data |
| **EXISTS Subquery** | Checks for existence of rows | `SELECT * FROM t1 WHERE EXISTS (SELECT 1 FROM t2 WHERE t2.col1 = t1.col1)` | ⭐⭐⭐⭐ | Existence checks |
| **NOT EXISTS Subquery** | Checks for non-existence of rows | `SELECT * FROM t1 WHERE NOT EXISTS (SELECT 1 FROM t2 WHERE t2.col1 = t1.col1)` | ⭐⭐⭐⭐ | Exclusion checks |
| **IN Subquery** | Checks if value is in a list | `SELECT * FROM t1 WHERE col1 IN (SELECT col2 FROM t2)` | ⭐⭐⭐ | Membership checks |
| **NOT IN Subquery** | Checks if value is not in a list | `SELECT * FROM t1 WHERE col1 NOT IN (SELECT col2 FROM t2)` | ⭐⭐ | Exclusion checks |

#### **4. CTE and Subquery Optimization Techniques**

| **Technique** | **Description** | **Example** | **Performance Impact** |
|---------------|-----------------|-------------|-------------------------|
| **Use CTEs for Complex Queries** | Break down complex queries into CTEs | `WITH cte1 AS (...), cte2 AS (...) SELECT * FROM cte1 JOIN cte2 ON ...` | ⭐⭐⭐⭐ (2-10x faster) |
| **Use MATERIALIZE for CTEs** | Materialize CTEs that are referenced multiple times | `WITH MATERIALIZE cte AS (SELECT * FROM my_table) SELECT * FROM cte JOIN cte ON ...` | ⭐⭐⭐⭐⭐ (10-100x faster) |
| **Use Subquery Unnesting** | Convert subqueries to joins | `SELECT * FROM t1 WHERE id IN (SELECT id FROM t2)` → `SELECT * FROM t1 JOIN t2 ON t1.id = t2.id` | ⭐⭐⭐⭐ (2-10x faster) |
| **Use EXISTS Instead of IN for Large Subqueries** | EXISTS is often more efficient than IN | `SELECT * FROM t1 WHERE EXISTS (SELECT 1 FROM t2 WHERE t2.id = t1.id)` | ⭐⭐⭐⭐ (2-10x faster) |
| **Use NOT EXISTS Instead of NOT IN** | NOT EXISTS handles NULLs better than NOT IN | `SELECT * FROM t1 WHERE NOT EXISTS (SELECT 1 FROM t2 WHERE t2.id = t1.id)` | ⭐⭐⭐⭐ (2-10x faster) |
| **Filter Early in CTEs** | Apply filters in CTE definitions | `WITH cte AS (SELECT * FROM my_table WHERE date > '2023-01-01') SELECT * FROM cte` | ⭐⭐⭐⭐ (2-10x faster) |
| **Limit Data in CTEs** | Limit data in CTEs to reduce processing | `WITH cte AS (SELECT * FROM my_table LIMIT 1000) SELECT * FROM cte` | ⭐⭐⭐ (1.5-5x faster) |
| **Avoid Redundant CTEs** | Remove CTEs that are not used | Remove unused CTEs | ⭐⭐ (1.1-2x faster) |
| **Use CTEs for Query Reuse** | Reuse CTEs in multiple parts of the query | `WITH cte AS (SELECT * FROM my_table) SELECT COUNT(*) FROM cte, SELECT SUM(col1) FROM cte` | ⭐⭐⭐ (1.5-5x faster) |
| **Monitor CTE Performance** | Check QUERY_PROFILE for CTE performance | `SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))` | N/A |

#### **5. Best Practices for CTEs and Subqueries**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use CTEs for Complex Queries** | Break down complex queries into CTEs for better readability | `WITH cte1 AS (...), cte2 AS (...) SELECT * FROM cte1 JOIN cte2 ON ...` |
| **Use MATERIALIZE for CTEs Referenced Multiple Times** | Materialize CTEs that are used multiple times | `WITH MATERIALIZE cte AS (SELECT * FROM my_table) SELECT * FROM cte JOIN cte ON ...` |
| **Use Subquery Unnesting** | Convert subqueries to joins for better performance | `SELECT * FROM t1 WHERE id IN (SELECT id FROM t2)` → `SELECT * FROM t1 JOIN t2 ON t1.id = t2.id` |
| **Use EXISTS Instead of IN** | EXISTS is often more efficient than IN | `SELECT * FROM t1 WHERE EXISTS (SELECT 1 FROM t2 WHERE t2.id = t1.id)` |
| **Use NOT EXISTS Instead of NOT IN** | NOT EXISTS handles NULLs better than NOT IN | `SELECT * FROM t1 WHERE NOT EXISTS (SELECT 1 FROM t2 WHERE t2.id = t1.id)` |
| **Filter Early in CTEs** | Apply filters in CTE definitions to reduce data volume | `WITH cte AS (SELECT * FROM my_table WHERE date > '2023-01-01') SELECT * FROM cte` |
| **Limit Data in CTEs** | Limit data in CTEs to reduce processing | `WITH cte AS (SELECT * FROM my_table LIMIT 1000) SELECT * FROM cte` |
| **Avoid Redundant CTEs** | Remove CTEs that are not used | Remove unused CTEs |
| **Use CTEs for Query Reuse** | Reuse CTEs in multiple parts of the query | `WITH cte AS (SELECT * FROM my_table) SELECT COUNT(*) FROM cte, SELECT SUM(col1) FROM cte` |
| **Use CTEs for Recursive Queries** | Use RECURSIVE CTEs for hierarchical data | `WITH RECURSIVE org_hierarchy AS (...) SELECT * FROM org_hierarchy` |
| **Monitor CTE Performance** | Check QUERY_PROFILE for CTE performance | `SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'))` |

#### **6. Examples**

**Example 1: Basic CTE Usage**
```sql
-- Use CTEs for better readability
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
)
SELECT
    c.customer_id,
    c.customer_name,
    cs.total_spend,
    RANK() OVER (ORDER BY cs.total_spend DESC) AS spend_rank
FROM
    customers c
JOIN
    customer_sales cs ON c.customer_id = cs.customer_id
ORDER BY
    spend_rank;
```

**Example 2: Materialized CTE**
```sql
-- Use MATERIALIZE for CTEs referenced multiple times
WITH MATERIALIZE customer_data AS (
    SELECT
        customer_id,
        customer_name,
        region,
        signup_date
    FROM
        customers
    WHERE
        signup_date > '2023-01-01'
)
SELECT
    region,
    COUNT(*) AS customer_count,
    AVG(DATEDIFF('day', signup_date, CURRENT_DATE())) AS avg_days_as_customer
FROM
    customer_data
GROUP BY
    region

UNION ALL

SELECT
    'All Regions' AS region,
    COUNT(*) AS customer_count,
    AVG(DATEDIFF('day', signup_date, CURRENT_DATE())) AS avg_days_as_customer
FROM
    customer_data;
```

**Example 3: Subquery Unnesting**
```sql
-- Convert subquery to join (unnesting)
-- Before (subquery)
SELECT
    c.customer_id,
    c.customer_name,
    c.region
FROM
    customers c
WHERE
    c.customer_id IN (
        SELECT customer_id
        FROM orders
        WHERE order_date > CURRENT_DATE() - 30
    );

-- After (join)
SELECT
    c.customer_id,
    c.customer_name,
    c.region
FROM
    customers c
JOIN
    orders o ON c.customer_id = o.customer_id
WHERE
    o.order_date > CURRENT_DATE() - 30;
```

**Example 4: EXISTS vs IN**
```sql
-- EXISTS vs IN performance comparison
-- Using IN
SELECT
    c.customer_id,
    c.customer_name
FROM
    customers c
WHERE
    c.customer_id IN (
        SELECT customer_id
        FROM orders
        WHERE order_date > CURRENT_DATE() - 30
    );

-- Using EXISTS (often faster)
SELECT
    c.customer_id,
    c.customer_name
FROM
    customers c
WHERE
    EXISTS (
        SELECT 1
        FROM orders o
        WHERE o.customer_id = c.customer_id
        AND o.order_date > CURRENT_DATE() - 30
    );
```

**Example 5: NOT EXISTS vs NOT IN**
```sql
-- NOT EXISTS vs NOT IN performance comparison
-- Using NOT IN (handles NULLs poorly)
SELECT
    c.customer_id,
    c.customer_name
FROM
    customers c
WHERE
    c.customer_id NOT IN (
        SELECT customer_id
        FROM orders
    );

-- Using NOT EXISTS (handles NULLs better)
SELECT
    c.customer_id,
    c.customer_name
FROM
    customers c
WHERE
    NOT EXISTS (
        SELECT 1
        FROM orders o
        WHERE o.customer_id = c.customer_id
    );
```

**Example 6: Recursive CTE for Hierarchical Data**
```sql
-- Recursive CTE for organizational hierarchy
WITH RECURSIVE org_hierarchy AS (
    -- Base case: top-level employees
    SELECT
        employee_id,
        employee_name,
        manager_id,
        1 AS level,
        employee_name AS path
    FROM
        employees
    WHERE
        manager_id IS NULL

    UNION ALL

    -- Recursive case: employees with managers
    SELECT
        e.employee_id,
        e.employee_name,
        e.manager_id,
        h.level + 1,
        h.path || ' -> ' || e.employee_name AS path
    FROM
        employees e
    JOIN
        org_hierarchy h ON e.manager_id = h.employee_id
)
SELECT
    employee_id,
    employee_name,
    level,
    path
FROM
    org_hierarchy
ORDER BY
    level, employee_name;
```

**Example 7: Complex CTE Query**
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
        s.*,
        c.customer_name,
        c.region,
        c.segment
    FROM
        recent_sales s
    JOIN
        customers c ON s.customer_id = c.customer_id
),

-- CTE 3: Aggregate by region and segment
region_segment_sales AS (
    SELECT
        region,
        segment,
        SUM(amount) AS total_sales,
        COUNT(*) AS transaction_count,
        AVG(amount) AS avg_sale_amount
    FROM
        sales_with_customers
    GROUP BY
        region, segment
),

-- CTE 4: Calculate regional totals
region_totals AS (
    SELECT
        region,
        SUM(total_sales) AS region_total_sales,
        SUM(transaction_count) AS region_transaction_count
    FROM
        region_segment_sales
    GROUP BY
        region
)

-- Final query: sales by region and segment with regional totals
SELECT
    rs.region,
    rs.segment,
    rs.total_sales,
    rs.transaction_count,
    rs.avg_sale_amount,
    rt.region_total_sales,
    rs.total_sales * 100.0 / rt.region_total_sales AS pct_of_region
FROM
    region_segment_sales rs
JOIN
    region_totals rt ON rs.region = rt.region
ORDER BY
    rs.region, rs.total_sales DESC;
```

## **3. Programmatic Transformation Techniques**

### **A. Stored Procedures**

#### **1. Stored Procedure Overview**
Stored procedures in Snowflake are **reusable SQL or script-based routines** that can:
- **Encapsulate complex logic**
- **Accept parameters**
- **Return results**
- **Execute multiple statements**
- **Include transaction control**

Snowflake supports stored procedures in **multiple languages**:
- **SQL** (Snowflake Scripting)
- **JavaScript**
- **Python**
- **Java**

#### **2. Stored Procedure Types**

| **Type** | **Language** | **Description** | **Performance** | **Use Cases** |
|----------|--------------|-----------------|-----------------|--------------|
| **SQL Procedure** | SQL (Snowflake Scripting) | Procedure written in SQL with Snowflake Scripting | ⭐⭐⭐⭐ | ETL pipelines, data validation, complex workflows |
| **JavaScript Procedure** | JavaScript | Procedure written in JavaScript | ⭐⭐⭐ | Complex logic, string manipulation, custom algorithms |
| **Python Procedure** | Python | Procedure written in Python | ⭐⭐ | Data science, ML, complex data processing |
| **Java Procedure** | Java | Procedure written in Java | ⭐⭐⭐⭐⭐ | High-performance processing, custom algorithms |

#### **3. SQL Stored Procedures (Snowflake Scripting)**
Snowflake Scripting is a **procedural extension** to SQL that enables:
- **Variable declarations**
- **Control structures** (IF, CASE, LOOP, etc.)
- **Exception handling**
- **Transaction control**

**Basic Syntax**:
```sql
CREATE OR REPLACE PROCEDURE procedure_name(
    [parameter1 data_type [DEFAULT default_value] [, ...]]
)
RETURNS return_type
LANGUAGE SQL
[EXECUTE AS {CALLER | OWNER}]
[STRICT | NOT STRICT]
AS
$$
DECLARE
    variable1 data_type [DEFAULT default_value];
    variable2 data_type [DEFAULT default_value];
    ...
BEGIN
    -- Procedure logic
    statement1;
    statement2;
    ...
    RETURN return_value;
EXCEPTION
    WHEN error_type THEN
        -- Exception handling
        statement;
    [WHEN ... THEN ...]
END;
$$;
```

**Example: Simple SQL Procedure**
```sql
CREATE OR REPLACE PROCEDURE get_customer_orders(
    customer_id INT
)
RETURNS TABLE (
    order_id INT,
    order_date DATE,
    amount FLOAT
)
LANGUAGE SQL
AS
$$
BEGIN
    RETURN TABLE (
        SELECT
            order_id,
            order_date,
            amount
        FROM
            orders
        WHERE
            customer_id = :customer_id
        ORDER BY
            order_date DESC
    );
END;
$$;
```

**Example: SQL Procedure with Variables and Control Flow**
```sql
CREATE OR REPLACE PROCEDURE process_customer_orders(
    customer_id INT,
    start_date DATE DEFAULT CURRENT_DATE() - 30
)
RETURNS STRING
LANGUAGE SQL
AS
$$
DECLARE
    order_count INT;
    total_spend FLOAT;
    result STRING;
BEGIN
    -- Get order count and total spend
    SELECT
        COUNT(*) INTO :order_count,
        SUM(amount) INTO :total_spend
    FROM
        orders
    WHERE
        customer_id = :customer_id
        AND order_date >= :start_date;

    -- Process based on order count
    IF (:order_count = 0) THEN
        result := 'No orders found for customer ' || :customer_id;
    ELSIF (:order_count > 0 AND :order_count <= 5) THEN
        result := 'Customer ' || :customer_id || ' has ' || :order_count || ' orders with total spend of $' || :total_spend;
    ELSE
        result := 'Customer ' || :customer_id || ' is a high-value customer with ' || :order_count || ' orders and $' || :total_spend || ' spend';
    END IF;

    RETURN result;
END;
$$;
```

**Example: SQL Procedure with Exception Handling**
```sql
CREATE OR REPLACE PROCEDURE safe_delete_customer(
    customer_id INT
)
RETURNS STRING
LANGUAGE SQL
AS
$$
DECLARE
    result STRING;
    error_message STRING;
BEGIN
    BEGIN
        -- Try to delete customer
        DELETE FROM customers WHERE customer_id = :customer_id;

        -- Check if customer was deleted
        IF (SQL%ROWCOUNT = 0) THEN
            result := 'No customer found with ID: ' || :customer_id;
        ELSE
            result := 'Successfully deleted customer with ID: ' || :customer_id;
        END IF;
    EXCEPTION
        WHEN STATEMENT_ERROR THEN
            error_message := SQLERRM;
            result := 'Error deleting customer: ' || error_message;
    END;

    RETURN result;
END;
$$;
```

**Example: SQL Procedure with Transaction Control**
```sql
CREATE OR REPLACE PROCEDURE transfer_funds(
    from_account INT,
    to_account INT,
    amount FLOAT
)
RETURNS STRING
LANGUAGE SQL
AS
$$
DECLARE
    result STRING;
    from_balance FLOAT;
    to_balance FLOAT;
BEGIN
    -- Start transaction
    BEGIN TRANSACTION;

    BEGIN
        -- Get current balances
        SELECT balance INTO :from_balance FROM accounts WHERE account_id = :from_account;
        SELECT balance INTO :to_balance FROM accounts WHERE account_id = :to_account;

        -- Check if from account has sufficient funds
        IF (:from_balance < :amount) THEN
            result := 'Insufficient funds in account ' || :from_account;
            ROLLBACK;
            RETURN result;
        END IF;

        -- Update balances
        UPDATE accounts SET balance = balance - :amount WHERE account_id = :from_account;
        UPDATE accounts SET balance = balance + :amount WHERE account_id = :to_account;

        -- Commit transaction
        COMMIT;
        result := 'Successfully transferred $' || :amount || ' from account ' || :from_account || ' to account ' || :to_account;
    EXCEPTION
        WHEN STATEMENT_ERROR THEN
            ROLLBACK;
            result := 'Error transferring funds: ' || SQLERRM;
    END;

    RETURN result;
END;
$$;
```

#### **4. JavaScript Stored Procedures**
JavaScript procedures allow you to **write custom logic** in JavaScript that can be executed in Snowflake.

**Basic Syntax**:
```sql
CREATE OR REPLACE PROCEDURE procedure_name(
    [parameter1 data_type [, ...]]
)
RETURNS return_type
LANGUAGE JAVASCRIPT
[EXECUTE AS {CALLER | OWNER}]
[STRICT | NOT STRICT]
AS
$$
    // JavaScript code
    function (parameter1, ...) {
        // Procedure logic
        return return_value;
    }
$$;
```

**Example: Simple JavaScript Procedure**
```sql
CREATE OR REPLACE PROCEDURE calculate_discount(
    order_amount FLOAT
)
RETURNS FLOAT
LANGUAGE JAVASCRIPT
AS
$$
    // Calculate discount based on order amount
    if (ORDER_AMOUNT > 1000) {
        return ORDER_AMOUNT * 0.1;  // 10% discount
    } else if (ORDER_AMOUNT > 500) {
        return ORDER_AMOUNT * 0.05; // 5% discount
    } else {
        return 0;                  // No discount
    }
$$;
```

**Example: JavaScript Procedure with Complex Logic**
```sql
CREATE OR REPLACE PROCEDURE process_customer_data(
    customer_id INT
)
RETURNS OBJECT
LANGUAGE JAVASCRIPT
AS
$$
    // Get customer data
    var customer_stmt = snowflake.createStatement({
        sqlText: `SELECT * FROM customers WHERE customer_id = ${CUSTOMER_ID}`
    });
    var customer_result = customer_stmt.execute();

    if (!customer_result.next()) {
        return { error: 'Customer not found' };
    }

    var customer = {
        id: customer_result.getColumnValue(1),
        name: customer_result.getColumnValue(2),
        region: customer_result.getColumnValue(3),
        signup_date: customer_result.getColumnValue(4),
        total_spend: customer_result.getColumnValue(5)
    };

    // Get customer orders
    var orders_stmt = snowflake.createStatement({
        sqlText: `SELECT order_id, order_date, amount FROM orders WHERE customer_id = ${CUSTOMER_ID} ORDER BY order_date DESC LIMIT 10`
    });
    var orders_result = orders_stmt.execute();

    var orders = [];
    while (orders_result.next()) {
        orders.push({
            order_id: orders_result.getColumnValue(1),
            order_date: orders_result.getColumnValue(2),
            amount: orders_result.getColumnValue(3)
        });
    }

    // Calculate customer tier
    var tier;
    if (customer.total_spend > 10000) {
        tier = 'Platinum';
    } else if (customer.total_spend > 5000) {
        tier = 'Gold';
    } else if (customer.total_spend > 1000) {
        tier = 'Silver';
    } else {
        tier = 'Bronze';
    }

    // Return processed data
    return {
        customer: customer,
        orders: orders,
        tier: tier,
        processed_at: new Date().toISOString()
    };
$$;
```

**Example: JavaScript Procedure with Snowflake API**
```sql
CREATE OR REPLACE PROCEDURE analyze_sales_data(
    start_date DATE,
    end_date DATE
)
RETURNS OBJECT
LANGUAGE JAVASCRIPT
AS
$$
    // Create a statement to get sales data
    var stmt = snowflake.createStatement({
        sqlText: `SELECT region, SUM(amount) AS total_sales, COUNT(*) AS order_count
                  FROM sales
                  WHERE sale_date BETWEEN ${START_DATE} AND ${END_DATE}
                  GROUP BY region`
    });

    // Execute the statement
    var result = stmt.execute();

    // Process the results
    var regions = [];
    while (result.next()) {
        regions.push({
            region: result.getColumnValue(1),
            total_sales: result.getColumnValue(2),
            order_count: result.getColumnValue(3)
        });
    }

    // Calculate totals
    var total_sales = 0;
    var total_orders = 0;
    for (var i = 0; i < regions.length; i++) {
        total_sales += regions[i].total_sales;
        total_orders += regions[i].order_count;
    }

    // Return the analysis
    return {
        start_date: START_DATE,
        end_date: END_DATE,
        regions: regions,
        total_sales: total_sales,
        total_orders: total_orders,
        avg_order_value: total_sales / total_orders,
        analyzed_at: new Date().toISOString()
    };
$$;
```

#### **5. Python Stored Procedures**
Python procedures allow you to **leverage Python's extensive libraries** for data transformation, including **pandas**, **NumPy**, **scikit-learn**, and more.

**Basic Syntax**:
```sql
CREATE OR REPLACE PROCEDURE procedure_name(
    [parameter1 data_type [, ...]]
)
RETURNS return_type
LANGUAGE PYTHON
[RUNTIME_VERSION = '3.8' | '3.9' | '3.10']
[HANDLER = handler_name]
[EXECUTE AS {CALLER | OWNER}]
[STRICT | NOT STRICT]
AS
$$
    # Python code
    def procedure_name(parameter1, ...):
        # Procedure logic
        return return_value
$$;
```

**Example: Simple Python Procedure**
```sql
CREATE OR REPLACE PROCEDURE calculate_statistics(
    table_name STRING,
    column_name STRING
)
RETURNS OBJECT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.8'
AS
$$
    # Import pandas
    import pandas as pd
    import snowflake.snowpark as snowpark

    # Create a Snowpark session
    session = snowpark.Session.builder.getOrCreate()

    # Read data into a DataFrame
    df = session.table(table_name)

    # Calculate statistics
    stats = {
        'count': df.count(),
        'mean': df[col1].mean(),
        'std': df[col1].std(),
        'min': df[col1].min(),
        'max': df[col1].max(),
        '25%': df[col1].quantile(0.25),
        '50%': df[col1].quantile(0.50),
        '75%': df[col1].quantile(0.75)
    }

    # Close the session
    session.close()

    return stats
$$;
```

**Note**: The above example uses **Snowpark**, which is covered in more detail in the [Snowpark section](#7-snowpark).

**Example: Python Procedure with Snowflake Connector**
```sql
CREATE OR REPLACE PROCEDURE process_sales_data(
    start_date DATE,
    end_date DATE
)
RETURNS STRING
LANGUAGE PYTHON
RUNTIME_VERSION = '3.8'
HANDLER = 'process_sales'
AS
$$
    import snowflake.connector
    import pandas as pd

    def process_sales(start_date, end_date):
        # Connect to Snowflake
        conn = snowflake.connector.connect(
            user='...',
            password='...',
            account='...'
        )

        try:
            # Execute query
            query = f"""
                SELECT region, product_category, SUM(amount) AS total_sales
                FROM sales
                WHERE sale_date BETWEEN '{start_date}' AND '{end_date}'
                GROUP BY region, product_category
            """
            df = pd.read_sql(query, conn)

            # Process data
            df['pct_of_total'] = df['total_sales'] / df['total_sales'].sum() * 100

            # Return summary
            return f"Processed {len(df)} records from {start_date} to {end_date}"

        finally:
            conn.close()
$$;
```

#### **6. Java Stored Procedures**
Java procedures allow you to **write high-performance custom logic** in Java.

**Basic Syntax**:
```sql
CREATE OR REPLACE PROCEDURE procedure_name(
    [parameter1 data_type [, ...]]
)
RETURNS return_type
LANGUAGE JAVA
[EXECUTE AS {CALLER | OWNER}]
[STRICT | NOT STRICT]
[IMPORTS = ('package1.Class1', 'package2.Class2', ...)]
[HANDLER = 'handler_method']
AS
$$
    // Java code
    public class ProcedureClass {
        public static return_type handler_method(parameter1_type parameter1, ...) {
            // Procedure logic
            return return_value;
        }
    }
$$;
```

**Example: Simple Java Procedure**
```sql
CREATE OR REPLACE PROCEDURE calculate_factorial(
    n INT
)
RETURNS BIGINT
LANGUAGE JAVA
IMPORTS = ('java.math.BigInteger')
HANDLER = 'calculateFactorial'
AS
$$
    import java.math.BigInteger;

    public class Factorial {
        public static BigInteger calculateFactorial(int n) {
            BigInteger result = BigInteger.ONE;
            for (int i = 2; i <= n; i++) {
                result = result.multiply(BigInteger.valueOf(i));
            }
            return result;
        }
    }
$$;
```

#### **7. Stored Procedure Optimization Techniques**

| **Technique** | **Description** | **Example** | **Performance Impact** |
|---------------|-----------------|-------------|-------------------------|
| **Use SQL Procedures for Simple Logic** | SQL procedures are fastest for simple logic | `CREATE PROCEDURE ... LANGUAGE SQL ...` | ⭐⭐⭐⭐⭐ (Best performance) |
| **Use Java Procedures for High-Performance Logic** | Java procedures are fastest for complex logic | `CREATE PROCEDURE ... LANGUAGE JAVA ...` | ⭐⭐⭐⭐ (Best for CPU-intensive tasks) |
| **Use Python Procedures for Data Science** | Python procedures leverage data science libraries | `CREATE PROCEDURE ... LANGUAGE PYTHON ...` | ⭐⭐⭐ (Best for ML, stats) |
| **Use Parameterized Procedures** | Accept parameters for reusability | `CREATE PROCEDURE my_proc(param1 INT) ...` | ⭐⭐⭐⭐ (Improves reusability) |
| **Use Transaction Control** | Manage transactions for data consistency | `BEGIN TRANSACTION; ... COMMIT;` | ⭐⭐⭐⭐ (Ensures data consistency) |
| **Use Exception Handling** | Handle errors gracefully | `EXCEPTION WHEN STATEMENT_ERROR THEN ...` | ⭐⭐⭐⭐ (Improves reliability) |
| **Use Materialized Results** | Cache results for repetitive calls | Use temporary tables or materialized views | ⭐⭐⭐⭐ (Improves performance) |
| **Avoid Long-Running Procedures** | Break long procedures into smaller ones | Split into multiple procedures | ⭐⭐⭐ (Improves maintainability) |
| **Use Snowpark for Data-Intensive Procedures** | Use Snowpark for data-intensive operations | `CREATE PROCEDURE ... LANGUAGE PYTHON ...` with Snowpark | ⭐⭐⭐⭐ (Best for large datasets) |
| **Monitor Procedure Performance** | Check QUERY_HISTORY for procedure performance | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY WHERE query_text LIKE '%my_proc%'` | N/A |

#### **8. Best Practices for Stored Procedures**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use SQL Procedures for Simple Logic** | SQL is fastest for simple data operations | `CREATE PROCEDURE ... LANGUAGE SQL ...` |
| **Use Java Procedures for High-Performance Logic** | Java is fastest for CPU-intensive tasks | `CREATE PROCEDURE ... LANGUAGE JAVA ...` |
| **Use Python Procedures for Data Science** | Python has best data science libraries | `CREATE PROCEDURE ... LANGUAGE PYTHON ...` |
| **Use Parameterized Procedures** | Accept parameters for reusability | `CREATE PROCEDURE my_proc(param1 INT, param2 STRING) ...` |
| **Use Transaction Control** | Manage transactions for data consistency | `BEGIN TRANSACTION; ... COMMIT;` |
| **Use Exception Handling** | Handle errors gracefully | `EXCEPTION WHEN STATEMENT_ERROR THEN ...` |
| **Use Materialized Results** | Cache results for repetitive calls | Use temporary tables or materialized views |
| **Avoid Long-Running Procedures** | Break long procedures into smaller ones | Split into multiple procedures |
| **Use Snowpark for Data-Intensive Procedures** | Use Snowpark for large datasets | `CREATE PROCEDURE ... LANGUAGE PYTHON ...` with Snowpark |
| **Monitor Procedure Performance** | Check QUERY_HISTORY for performance | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY WHERE query_text LIKE '%my_proc%'` |
| **Document Procedure Logic** | Document procedure purpose and parameters | Add comments in procedure code |
| **Use Appropriate Runtime Version** | Use the right Python runtime version | `RUNTIME_VERSION = '3.8'` |
| **Use EXECUTE AS CALLER for Security** | Execute with caller's permissions | `EXECUTE AS CALLER` |
| **Use EXECUTE AS OWNER for Performance** | Execute with owner's permissions | `EXECUTE AS OWNER` |
| **Use STRICT for Validation** | Validate input parameters | `STRICT` |
| **Use NOT STRICT for Flexibility** | Allow NULL inputs | `NOT STRICT` |

#### **9. Examples**

**Example 1: ETL Pipeline with SQL Procedure**
```sql
CREATE OR REPLACE PROCEDURE etl_sales_pipeline(
    start_date DATE,
    end_date DATE
)
RETURNS STRING
LANGUAGE SQL
AS
$$
DECLARE
    result STRING;
    row_count INT;
BEGIN
    -- Step 1: Extract data from source
    CREATE TEMPORARY TABLE temp_source_sales AS
    SELECT * FROM source_sales
    WHERE sale_date BETWEEN :start_date AND :end_date;

    -- Check if data was extracted
    SELECT COUNT(*) INTO :row_count FROM temp_source_sales;
    IF (:row_count = 0) THEN
        result := 'No data found for date range: ' || :start_date || ' to ' || :end_date;
        RETURN result;
    END IF;

    -- Step 2: Transform data
    CREATE TEMPORARY TABLE temp_transformed_sales AS
    SELECT
        sale_id,
        customer_id,
        product_id,
        sale_date,
        amount,
        region,
        product_category,
        amount * 1.1 AS amount_with_tax
    FROM
        temp_source_sales;

    -- Step 3: Load data to target
    INSERT INTO target_sales
    SELECT * FROM temp_transformed_sales;

    -- Step 4: Clean up
    DROP TABLE IF EXISTS temp_source_sales;
    DROP TABLE IF EXISTS temp_transformed_sales;

    -- Return success
    result := 'Successfully processed ' || :row_count || ' sales records from ' || :start_date || ' to ' || :end_date;
    RETURN result;
EXCEPTION
    WHEN STATEMENT_ERROR THEN
        -- Clean up on error
        DROP TABLE IF EXISTS temp_source_sales;
        DROP TABLE IF EXISTS temp_transformed_sales;
        RETURN 'Error processing sales data: ' || SQLERRM;
END;
$$;
```

**Example 2: Data Validation with JavaScript Procedure**
```sql
CREATE OR REPLACE PROCEDURE validate_customer_data(
    customer_id INT
)
RETURNS OBJECT
LANGUAGE JAVASCRIPT
AS
$$
    // Get customer data
    var customer_stmt = snowflake.createStatement({
        sqlText: `SELECT * FROM customers WHERE customer_id = ${CUSTOMER_ID}`
    });
    var customer_result = customer_stmt.execute();

    if (!customer_result.next()) {
        return { valid: false, error: 'Customer not found' };
    }

    // Validate required fields
    var customer = {
        id: customer_result.getColumnValue(1),
        name: customer_result.getColumnValue(2),
        email: customer_result.getColumnValue(3),
        region: customer_result.getColumnValue(4),
        signup_date: customer_result.getColumnValue(5)
    };

    var errors = [];

    // Validate name
    if (!customer.name || customer.name.trim() === '') {
        errors.push('Name is required');
    }

    // Validate email
    if (!customer.email || !
