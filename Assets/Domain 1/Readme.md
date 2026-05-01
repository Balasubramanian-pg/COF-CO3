# Snowflake AI Data Cloud Features & Architecture (Module 1 Notes)

## 1.1 Snowflake Architecture  
- **Cloud Services Layer:** Manages metadata, security, infrastructure, query parsing/optimization and global services【62†L139-L148】【62†L193-L202】. Coordinates user requests (login, query planning) across the platform.  
- **Compute Layer (Virtual Warehouses):** Stateless clusters of compute (CPU, memory) that execute queries and data processing【54†L128-L137】. Warehouses can be sized and scaled independently. They support SQL queries, DML (INSERT/UPDATE/DELETE), data loading (COPY), and Snowpark workloads【62†L181-L190】【54†L128-L137】.  
- **Database Storage Layer:** Scalable cloud storage that holds all table data in Snowflake’s optimized columnar format【62†L136-L144】. Data is automatically divided into **micro-partitions** (50–500 MB, compressed)【65†L169-L177】. Snowflake handles file sizing, compression, metadata, statistics, and replication transparently【62†L136-L144】.  

| **Edition**           | **Key Features/Use Cases**                                           |
|-----------------------|---------------------------------------------------------------------|
| *Standard*            | Full access to all standard features (unlimited compute, storage)【5†L94-L100】; basic security and support. Designed for general workloads. |
| *Enterprise*          | Includes Standard + advanced features (e.g. multi-cluster warehouses, data encryption)【5†L100-L105】. Suitable for large-scale analytics. |
| *Business Critical*   | Includes Enterprise + enhanced security/compliance (HIPAA, PCI DSS, encryption-in-transit on compute, etc.)【5†L108-L116】. For highly regulated industries. |
| *Virtual Private Snowflake* | All Business Critical features on isolated hardware in a single-tenant environment【5†L125-L132】. Maximum security/isolation for sensitive workloads. |

**(Snowflake Editions differ by security, compliance, and feature set【5†L94-L100】【5†L108-L116】.)**

## 1.2 Snowflake Interfaces & Tools  
- **Snowsight (Web UI):** Browser-based interface for worksheets, dashboards, and notebooks【7†L184-L187】. Supports writing/executing SQL, visualizing query results, creating dashboards, and running Snowpark or Streamlit apps. (Replaces legacy Classic UI.)  
- **SnowSQL (CLI):** Legacy command-line client for executing SQL and scripts against Snowflake. Useful for batch jobs and scripting.  
- **Snowflake CLI (new):** Open-source CLI for account management (environments, roles, apps, etc.)【8†L209-L216】. Complements SnowSQL by enabling DevOps tasks (applications, data sharing) via terminal.  
- **IDE Integration:** Snowflake provides a VS Code extension for SQL and Snowpark development【10†L143-L148】. It enables writing/executing SQL in VS Code, with Snowpark Python support (debugging, syntax highlighting, autocomplete)【10†L143-L148】. Third-party tools (DBeaver, JDBC/ODBC drivers, Spark/Sybase connectors) are also available.

## 1.3 Snowflake Object Hierarchy & Types  
- **Organization & Accounts:** An *Organization* groups multiple Snowflake accounts under one umbrella for unified management (billing, replication, sharing)【30†L132-L141】. Accounts are independent environments containing compute (warehouses), databases, and metadata. Organization administrators manage all accounts and can view usage across accounts【30†L132-L141】.  
- **Database Objects:** Within each account, objects are organized as:  
  - *Databases* → *Schemas* → *Tables/Views/Other Objects*.  
  - **Stages:** Named pointers to data locations. **Internal stages** (user, table, or named) store files in Snowflake storage (internal or transient). **External stages** store URL and credentials for cloud storage (S3, Azure, GCS)【33†L277-L284】【33†L311-L318】. Stages are used for bulk loading/unloading.  
  - **Schemas:** Containers within databases that group tables, views, and other schema-level objects.  
  - **Tables:** Physical data storage objects. See Table Types (permanent, transient, temporary, external, Iceberg) below.  
  - **Views:** Virtual tables defined by SELECT queries. (See View Types below.)  
  - **User-Defined Functions (UDFs):** Custom scalar or table functions. Can be written in SQL, JavaScript, Python, or Scala (Snowpark) to encapsulate reusable logic【35†L214-L223】【35†L232-L240】. Called like built-in functions in SQL. Variations include scalar UDFs, UDTFs, and UDAFs【35†L232-L240】.  
  - **File Formats:** Named format objects defining how to parse staged files (CSV, JSON, PARQUET, etc.) for loading/unloading【37†L207-L211】. They specify delimiters, compression, etc., and can be reused across loads【37†L207-L211】.  
  - **Stored Procedures:** Scripted routines (JavaScript, Snowflake Scripting, or Snowpark code) that execute procedural logic on the server. Created with `CREATE PROCEDURE` and called with `CALL`【43†L197-L206】. Unlike UDFs, procedures can run multiple SQL statements, control flow, and access Snowflake APIs. Procedures can be JavaScript (sync API), or use the SQL-like Snowflake Scripting language. They are useful for complex transformations, loops, and admin tasks.  
  - **Pipes:** Objects (Snowpipe) that continuously load data from stages into tables. A pipe encapsulates a COPY statement and stage; Snowflake monitors the stage and auto-loads new files【45†L591-L600】. (Use `CREATE PIPE`, `ALTER PIPE`, etc.)【45†L591-L600】.  
  - **Shares:** Objects representing a data share. A *Share* encapsulates which databases, tables, or other objects are shared with consumer accounts. Data providers create a share and grant privileges on objects to it. Consumers then create a database from that share. Shared data is never copied; it's accessed live via Snowflake’s global metadata【47†L429-L438】. Shares are read-only to consumers and do not consume storage in the consumer account【47†L387-L394】【47†L429-L438】.  
  - **Sequences:** Incrementing counters used to generate unique numeric values (e.g. for primary keys). All values are unique (no repeats across sessions)【49†L150-L157】, but gaps can occur. Sequences can be altered (range, increment) and used in expressions (e.g. `NEXTVAL()`). They provide 64-bit integers and support CONSECUTIVE or NOORDER modes for concurrency.  
  - **ML Models & Applications:** Snowflake allows storing and sharing of machine learning models (e.g. auto-trained models, user models, or external models) within the database as first-class objects. The Native App Framework lets developers bundle data and logic (SQL, UDFs, notebooks, Streamlit apps) into an application package that can be published on the Marketplace【52†L281-L290】. Apps encapsulate database objects plus code and metadata for versioning and deployment.  

- **Session and Context Variables:** Snowflake supports session parameters and user-defined session variables (`SET var = value;`). Parameters can be set at multiple levels (organization, account, user, session, object) with defined precedence: **session-level** overrides **user-level**, which overrides **account-level** (and **organization-level**) settings【73†L132-L140】【72†L1-L4】. For example, a parameter or context variable set in a session will override the account default for that session.

## 1.4 Virtual Warehouses  
- **Types:** Snowflake warehouses are either **Standard** or **Snowpark-Optimized**【54†L128-L137】. Standard warehouses (Gen1 or Gen2) use a mix of CPU/memory for general SQL workloads. Gen2 (next-gen) is faster for analytics (optimized hardware, query engine)【55†L128-L137】【55†L138-L148】. Snowpark-Optimized warehouses allocate more memory per node for Python/ML workloads and are recommended for heavy in-memory tasks (e.g. large model training)【54†L162-L170】.  
- **Default (Notebooks):** Each account has a default multi-cluster X-Small warehouse (`SYSTEM$STREAMLIT_NOTEBOOK_WH`) for running notebook/Streamlit code. This warehouse auto-scales to optimize resource usage and reduce idle clusters【60†L354-L358】.  
- **Scaling Policies:** Multi-cluster warehouses can auto-scale clusters based on workload. Two policies exist【59†L696-L704】【59†L717-L724】:  
  - *Standard (default):* Prioritize minimizing query queuing by adding clusters early as needed【59†L696-L704】. New clusters start when queries queue or load increases. Idle clusters shut down gradually after low load.  
  - *Economy:* Prioritize cost-saving by keeping existing clusters busy before adding new ones【59†L717-L724】. Clusters shut down sooner and additional clusters only start when existing ones are very busy.  
- **Use-case Sizing:** Warehouse size and configuration depend on workload:  
  - *Ad-hoc/small queries:* Smaller sizes (X-Small, Small, Medium) often suffice【60†L281-L290】. Very small queries don’t need very large warehouses.  
  - *Large-scale/complex queries:* Larger sizes (Large, X-Large, etc.) or Gen2 yield faster scans and joins【60†L281-L290】【55†L128-L137】.  
  - *Data loading:* Choose a size matching file count/size for parallel load throughput【60†L281-L290】.  
  - *BI/Reporting:* Use multi-cluster warehouses or workload management (multi warehouses by team) to handle concurrency. Query Acceleration Service (QAS) can also speed up complex BI queries.  
- **Best Practices:**  
  - **Auto-Suspend:** Enable auto-suspend (e.g. 5–10 minutes) to stop warehouses when idle and save credits【60†L369-L378】. Choose suspension time to match workload gaps.  
  - **Auto-Resume:** Enable if instant availability is needed; disable for tight cost control.  
  - **Scaling:** Scale *up* (resize) to improve performance of individual queries【60†L423-L430】. Scale *out* (multi-cluster) to handle concurrency peaks (requires Enterprise+ edition)【60†L416-L424】.  
  - **Assign Workloads:** Use separate warehouses per team/workload to isolate resource usage. For high concurrency, use multi-cluster (auto-scale); for complex single queries, a larger single warehouse may suffice.  
  - **Monitoring:** Regularly monitor warehouse load (queries running/queued) and resize or adjust scaling policies accordingly.  

## 1.5 Snowflake Storage Concepts  
- **Micro-partitions:** Snowflake tables are automatically divided into small, contiguous **micro-partitions** (50–500 MB uncompressed)【65†L169-L177】. Each micro-partition stores columnar data plus metadata (value ranges, distinct counts) for all columns. This enables **pruning**: queries skip partitions that don’t match filters, greatly reducing I/O【65†L169-L177】【65†L246-L254】. Micro-partitions overlap on values, avoiding hotspots and allowing fine-grained access.  
- **Automatic Clustering:** As data is loaded, Snowflake captures clustering metadata (min/max values per partition) for key columns【65†L278-L285】. Even without user-defined keys, natural sort order is recorded so queries can skip irrelevant partitions. For large tables with evolving data, explicit clustering keys and automatic clustering (on Enterprise+) can be used to maintain optimal data order. Clustering metadata (including *depth*) indicates how well-ordered a table is.  
- **Table Types:** Snowflake supports various table types with different persistence and time-travel policies:  
  - *Permanent Tables:* Default type. Persistent data until dropped. Standard Time Travel retention up to **1 day** (Standard Edition) or up to **90 days** (Enterprise+)【67†L118-L122】. Data is recoverable via Time Travel queries (up to the retention period). After Time Travel expires, data goes to Fail-safe (7 days, for disaster recovery)【20†L408-L416】.  
  - *Temporary Tables:* Session-lifespan tables (declared `TEMPORARY`) that persist only for the user session. No Time Travel or Fail-safe (data gone when session ends)【18†L152-L160】【20†L378-L386】. Used for intermediate/one-time data processing.  
  - *Transient Tables:* Persistent like permanent tables, but with minimal data protection. No Fail-safe and only minimal (1 day) Time Travel【20†L378-L386】. Good for staging or intermediate data where long-term recovery isn’t needed.  
  - *External Tables:* Represent external data stored outside Snowflake (in S3/GCS/ADLS). Snowflake stores only metadata (file paths/schema); queries load data on the fly【23†L148-L156】. External tables are **read-only** and do not incur Snowflake storage. Ideal for querying data in a data lake without copying it.  
  - *Apache Iceberg Tables:* Externally-managed tables in Snowflake using the Apache Iceberg format【22†L144-L153】. Data lives in external storage but Iceberg tables behave like Snowflake tables (ACID, schema evolution, performance optimizations)【22†L144-L153】. Best for large data lake use cases when you want Snowflake SQL on top of external data.  
  - *Dynamic Tables:* Continuously maintained tables defined by a query (similar to “materialized view with auto-refresh”). Snowflake automatically updates dynamic tables to keep them fresh according to user-defined refresh policies【24†L175-L183】. Useful for real-time pipelines (e.g. streaming or regularly updated transformations).  
- **Time Travel & Fail-safe:** Snowflake retains historical data (deleted/updated) for recovery: standard Time Travel allows querying past data **1 day** (Standard) up to **90 days** (Enterprise)【67†L118-L122】. After Time Travel retention, Fail-safe retains data an additional **7 days** for Snowflake-managed recovery (charged storage).  

## 1.6 Table and View Types (AI/ML & Dev Features)  
- **Additional Table Types:** Snowflake also supports:  
  - *Event Tables:* For time-based data retention (configurable) with built-in time tracking. (Not covered by Time Travel beyond retention.)  
  - *Hybrid Tables:* High-throughput tables for transactional/analytical (Unistore) workloads (row-locking, constraints).  
  - *(See Domain resources for Streams, Tasks, Snowpark, ML, etc.)*  
- **View Types:**  
  - *Standard Views:* Simply store a query; results are computed on access. Useful for abstraction and reuse.  
  - *Materialized Views:* Physically store precomputed query results for faster access【27†L170-L177】. Useful when query reuse is high and underlying data changes infrequently. Only supports single-table queries.  
  - *Secure Views:* Hide view definition and underlying data details. Only the owner can see the definition, and end users cannot see base tables through the view【27†L183-L187】. Used for sensitive data masking/sharing.  
- **AI/ML & Data Science:** Snowflake provides built-in ML support:  
  - *Snowpark API:* Enables writing data pipelines and UDFs in Python/Java/Scala. Snowpark-optimized warehouses enhance large-scale model training.  
  - *Built-in ML:* Snowflake AutoML and ML modeling tools for common algorithms.  
  - *Cortex/AI SQL:* In-database AI features (e.g. search indexing, text functions) for unstructured analytics.  
  - *Streamlit Apps:* Native support to build and share data apps in Snowflake using Streamlit.  
  - *Native App Framework:* Package data, notebooks, UDFs, Streamlit apps into a deployable application for Marketplace sharing【52†L281-L290】.  

## Question Bank  
1. **Layers of Snowflake:** Name Snowflake’s three logical layers.  
2. **Snowflake Editions:** What additional security/compliance features does Business Critical edition provide over Standard?  
3. **Virtual Warehouses:** What is a Snowpark-optimized warehouse and when should it be used?  
4. **Snowsight:** List two features of the Snowsight web interface.  
5. **Schemas vs Databases:** In Snowflake’s hierarchy, which contains the other: database or schema?  
6. **Stages:** What is the difference between a named internal stage and an external stage?  
7. **Snowflake CLI vs SnowSQL:** For what tasks would you use the Snowflake CLI instead of SnowSQL?  
8. **UDFs:** In what languages can Snowflake UDFs be written?  
9. **File Formats:** What is the purpose of a file format object?  
10. **Pipes:** What is a Snowflake Pipe (Snowpipe) used for?  
11. **Shares:** Explain the basic concept of a Snowflake data share.  
12. **Sequences:** What guarantee do Snowflake sequences provide for generated values?  
13. **Stored Procedures:** How do Snowflake stored procedures differ from UDFs?  
14. **Warehouse Auto-Suspend:** Why should you enable auto-suspend on a warehouse, and what is a typical setting?  
15. **Scaling Policy:** What is the difference between Standard and Economy scaling policies for a multi-cluster warehouse?  
16. **Time Travel:** How long can you time-travel data in Enterprise vs Standard edition?  
17. **Table Types:** Compare Temporary vs Transient tables in terms of persistence and data protection.  
18. **External vs Iceberg Tables:** How does an Iceberg table differ from a regular external table?  
19. **Views:** What is a secure view and when is it used?  
20. **Parameters:** If a parameter is set both at the account level and the session level, which value takes effect?  


