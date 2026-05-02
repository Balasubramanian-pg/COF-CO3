## Tables and Views

### Table families and when they matter

Snowflake’s default table type is permanent. If you do not specify `TEMPORARY` or `TRANSIENT`, the table is permanent. Temporary tables exist only for the session that created them and are not visible to other sessions. Transient tables exist until dropped, but they have a lower protection level than permanent tables because they do not carry the same recoverability guarantees after Time Travel. Snowflake explicitly recommends transient tables only for data that can be recreated externally. ([Snowflake Docs][1])

Hybrid tables are the transactional outlier in the table family. Snowflake describes them as low-latency, high-throughput tables with index-based random reads and writes, row locking, and support for unique and referential integrity constraints. Snowflake also states you cannot create hybrid tables as temporary or transient, and you cannot place them inside transient schemas or databases. ([Snowflake Docs][2])

External tables are read-only tables over files in an external stage. Snowflake stores metadata for those files inside Snowflake, but the stage itself is external. External tables can be queried and joined, but DML is not supported. Snowflake also says query performance can be slower than native tables, and materialized views can be used to improve performance over external tables. ([Snowflake Docs][3])

Dynamic tables are managed table objects that Snowflake refreshes automatically from a definition query. Snowflake runs the definition query and merges changes from the base objects using compute resources associated with the table. Creation requires that the base objects have change tracking enabled, and the dynamic table refresh and scheduler behavior is controlled through the dynamic-table DDL surface. ([Snowflake Docs][4])

Apache Iceberg tables also appear in the table family, but they are a separate table type with their own lifecycle and storage semantics. The `INFORMATION_SCHEMA.TABLES` view exposes `IS_ICEBERG`, `IS_DYNAMIC`, `IS_IMMUTABLE`, and `IS_HYBRID` flags so you can distinguish these variants programmatically instead of guessing from naming conventions. ([Snowflake Docs][5])

### Creation patterns that matter in production

`CREATE TABLE` supports the practical deployment patterns you actually use: CTAS, `LIKE`, `CLONE`, and `USING TEMPLATE`. CTAS can copy data into a new table, `LIKE` copies only the column definitions, `CLONE` creates a zero-copy clone, and `USING TEMPLATE` derives schema from staged files through `INFER_SCHEMA`. Snowflake also documents that `CREATE OR REPLACE <object>` is atomic, so queries concurrent with the operation see either the old version or the new version, not a half-built object. ([Snowflake Docs][1])

For table replacement, `COPY GRANTS` preserves privileges on the replaced table rather than inheriting from source tables in the query. Snowflake also notes that `COPY TAGS` can propagate tags during `CREATE OR REPLACE TABLE`, `LIKE`, and `CLONE` workflows, with precedence rules if the same tag exists in multiple sources. That matters for controlled promotion because governance metadata may survive replacement if you design for it, or disappear if you do not. ([Snowflake Docs][1])

```sql
CREATE OR REPLACE TABLE analytics.fact_sales
  CLUSTER BY (sale_date)
AS
SELECT *
FROM staging.fact_sales_raw;

CREATE OR REPLACE TABLE analytics.fact_sales_snapshot
  LIKE analytics.fact_sales;

CREATE OR REPLACE TABLE analytics.fact_sales_clone
  CLONE analytics.fact_sales;
```

### Views as logical dependencies

A view is a logical query object based on one or more tables, views, or any other valid `SELECT` expression. Snowflake supports `CREATE OR ALTER VIEW`, which is useful for idempotent deployment workflows when you want a single statement to create or update a view definition. The view definition is stored as text and is visible in `SHOW VIEWS` and `INFORMATION_SCHEMA.VIEWS`. ([Snowflake Docs][6])

Materialized views are different. Snowflake describes them as being based on a query of an existing table and populated with data. They require Enterprise Edition, and the initial creation behaves like a CTAS-style build. Snowflake also says `CREATE OR REPLACE` materialized view is atomic, so concurrent queries see either the old or the new version. ([Snowflake Docs][7])

For dependency analysis, Snowflake distinguishes object dependencies from data movement. A view that references a table creates an object dependency, but CTAS, INSERT, and MERGE do not. That distinction matters when you are deciding whether a table change will break dependent views, because the right question is not “what data moved,” it is “what object references exist.” ([Snowflake Docs][8])

```sql
CREATE OR REPLACE VIEW analytics.v_sales AS
SELECT
  sale_date,
  customer_id,
  amount
FROM analytics.fact_sales;

CREATE MATERIALIZED VIEW analytics.mv_sales_daily
AS
SELECT
  sale_date,
  SUM(amount) AS total_amount
FROM analytics.fact_sales
GROUP BY sale_date;
```

### Operational inventory and impact analysis

`INFORMATION_SCHEMA.TABLES` is the fastest low-latency way to classify table objects in a database. The documented columns include `IS_TEMPORARY`, `IS_ICEBERG`, `IS_DYNAMIC`, `IS_IMMUTABLE`, and `IS_HYBRID`, which lets you audit table families directly instead of relying on naming conventions or metadata in other systems. Snowflake also notes that `INFORMATION_SCHEMA` only shows objects visible to the current role and does not honor `MANAGE GRANTS` the way `SHOW` commands can. ([Snowflake Docs][5])

`OBJECT_DEPENDENCIES` is the control surface for blast-radius analysis. Snowflake says it records cases where one object references another without copying or materializing data, and that the view has up to three hours of latency. If you are changing a base table, query that view first to identify dependent views and other objects that will be affected. ([Snowflake Docs][8])

```sql
SELECT
  table_catalog,
  table_schema,
  table_name,
  table_type,
  is_temporary,
  is_iceberg,
  is_dynamic,
  is_immutable,
  is_hybrid
FROM my_db.information_schema.tables
ORDER BY table_schema, table_name;

SELECT
  referencing_object_domain,
  referencing_object_name,
  referenced_object_domain,
  referenced_object_name
FROM snowflake.account_usage.object_dependencies
WHERE referenced_object_name = 'FACT_SALES';
```

### Production patterns

Use permanent tables for durable curated data. Use transient tables only when external reconstruction is possible. Use temporary tables for session-local staging. Use external tables when the source of truth is outside Snowflake and you are willing to accept read-only access over external storage. Use hybrid tables only when transactional behavior and integrity constraints are the actual requirement, not just a preference for a different storage format. ([Snowflake Docs][1])

Use views when you want a stable logical contract over changing physical tables. Use materialized views when query latency matters enough to pay for maintained storage and refresh work. Use dynamic tables when you want Snowflake to own the refresh pipeline and you can express the transformation as a deterministic definition query over change-tracked sources. ([Snowflake Docs][6])

### Bottom line

Tables define physical persistence and concurrency semantics. Views define logical dependency semantics. Materialized views and dynamic tables add managed refresh behavior on top. The engineering mistake is to treat them as the same object with different syntax. Snowflake does not. Neither should your platform design. ([Snowflake Docs][6])

[1]: https://docs.snowflake.com/en/sql-reference/sql/create-table "CREATE TABLE | Snowflake Documentation"
[2]: https://docs.snowflake.com/en/user-guide/tables-hybrid "Hybrid tables | Snowflake Documentation"
[3]: https://docs.snowflake.com/en/user-guide/tables-external-intro "Introduction to external tables | Snowflake Documentation"
[4]: https://docs.snowflake.com/en/user-guide/dynamic-tables-about?utm_source=chatgpt.com "Dynamic tables"
[5]: https://docs.snowflake.com/en/sql-reference/info-schema/tables "TABLES view | Snowflake Documentation"
[6]: https://docs.snowflake.com/en/sql-reference/sql/create-view "CREATE VIEW | Snowflake Documentation"
[7]: https://docs.snowflake.com/en/sql-reference/sql/create-materialized-view "CREATE MATERIALIZED VIEW | Snowflake Documentation"
[8]: https://docs.snowflake.com/en/sql-reference/account-usage/object_dependencies "OBJECT_DEPENDENCIES view | Snowflake Documentation"
