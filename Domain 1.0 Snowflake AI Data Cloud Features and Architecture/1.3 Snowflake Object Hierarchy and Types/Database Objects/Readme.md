## Database Objects

<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/8075c334-f7c1-42de-a665-54bec09de75b" />


A Snowflake database is the namespace boundary between account-level objects and schema-level objects. 

>[!Note]
>At the database layer, you are managing database creation semantics, database roles, schemas, replication or clone behavior, and the metadata surfaces that expose those objects. Snowflake automatically creates `PUBLIC` and `INFORMATION_SCHEMA` when you create a new database, and `INFORMATION_SCHEMA` is the schema that exposes metadata views and table functions for the database and, in some cases, account-wide objects. 

>[!Caution]
>Databases created from shares are exceptions: they do not get `PUBLIC` or `INFORMATION_SCHEMA` unless those were explicitly granted, they cannot be cloned, and some properties such as `TRANSIENT` and `DATA_RETENTION_TIME_IN_DAYS` do not apply. ([Snowflake Docs][1])

A practical way to think about database objects is this: 
- The database owns schemas,
- Schemas own tables, views, stages, file formats, streams, tasks, pipes, policies, UDFs, and sequences,
- And the database boundary is what lets Snowflake apply privilege scope cleanly.
- Snowflake’s DDL and access-control docs explicitly treat tables, views, stages, file formats, UDFs, and sequences as schema objects,
- And `GET_DDL` confirms the schema-object namespace pattern `database.schema` or `schema` for objects in that family. ([Snowflake Docs][2])

### Database types and object behavior

Standard databases support the usual schema and object lifecycle. `CREATE OR REPLACE` database semantics are atomic, meaning Snowflake deletes the old object and creates the replacement in a single transaction. Snowflake also supports transient databases, which do not have Fail-safe after Time Travel and therefore reduce storage cost, but also reduce recoverability. Snowflake supports shared databases from shares, restored databases from backups, and secondary databases for replication. ([Snowflake Docs][1])

```sql
CREATE TRANSIENT DATABASE raw_ingest;
CREATE DATABASE analytics;
CREATE DATABASE sales_shared FROM SHARE provider_acct.sales_share;
CREATE DATABASE reporting_replica
  AS REPLICA OF org_acct.analytics_db;
```

([Snowflake Docs][1])

### Schema boundary inside the database

Schemas are the real control plane inside the database. 
- Snowflake’s access-control model says that in a regular schema,
- The owner role has all privileges on the object by default, including grant and revoke authority,
- While in a managed access schema the object owners lose the ability to make grant decisions and only the schema owner or a role with `MANAGE GRANTS` can grant privileges on objects in the schema.
- That is the key production distinction between a plain schema and a governed schema. ([Snowflake Docs][3])

>[!Note]
>Database-level metadata is exposed through `INFORMATION_SCHEMA` and `ACCOUNT_USAGE`. `INFORMATION_SCHEMA` is automatically created in every database and is the low-latency metadata surface for the current database.

>[!Note]
>`ACCOUNT_USAGE` gives account-wide object metadata and historical usage, includes dropped objects, and has longer retention but more latency.

>[!Note]
>The `SCHEMATA` view in `ACCOUNT_USAGE` excludes `ACCOUNT_USAGE`, `READER_ACCOUNT_USAGE`, and `INFORMATION_SCHEMA` schemas, so it is useful for inventory but not a complete dump of every schema in the system. ([Snowflake Docs][4])

```sql
SELECT database_name, schema_name, schema_owner, created
FROM SNOWFLAKE.ACCOUNT_USAGE.SCHEMATA
ORDER BY created DESC;

SELECT table_catalog, table_schema, table_name, table_type
FROM <db_name>.INFORMATION_SCHEMA.TABLES
ORDER BY table_schema, table_name;
```

([Snowflake Docs][5])

### Database roles

Database roles are scoped to the database that contains them, and they exist so you can package privileges on that database, its schemas, and its schema objects without widening everything to account roles. 

- Snowflake documents that database roles can receive privileges on the database itself, schemas within it, and schema objects such as tables, views, stages, file formats, UDFs, and sequences inside that database.
- A database role can then be granted to an account role or another role, which gives you a clean least-privilege pattern for access packaging. ([Snowflake Docs][2])

The `SNOWFLAKE` database itself is a special case because it ships with Snowflake-provided database roles such as `OBJECT_VIEWER`, `USAGE_VIEWER`, `GOVERNANCE_VIEWER`, and `SECURITY_VIEWER` for controlled access to shared metadata and system views. 

>[!Tip]
>Administrators can use `GRANT DATABASE ROLE` to assign those database roles to custom roles, and then grant those custom roles to users. That is the supported pattern for delegating read access to Snowflake metadata without opening broad privileges. ([Snowflake Docs][6])

```sql
CREATE ROLE can_viewmd;
GRANT DATABASE ROLE OBJECT_VIEWER TO ROLE can_viewmd;
GRANT ROLE can_viewmd TO USER smith;
```

([Snowflake Docs][6])

### Object dependencies and change impact

If you are doing impact analysis, use `ACCOUNT_USAGE.OBJECT_DEPENDENCIES`. Snowflake defines an object dependency as a case where one object references a base object without materializing or copying data, such as a view referencing a table. That means dependency tracking is about logical references, not data movement. It is the right source for determining what will break if you rename, replace, or drop a database object. ([Snowflake Docs][7])

```sql
SELECT *
FROM SNOWFLAKE.ACCOUNT_USAGE.OBJECT_DEPENDENCIES
WHERE referencing_object_domain IN ('VIEW', 'MATERIALIZED VIEW', 'STREAM')
ORDER BY referencing_object_domain, referencing_object_name;
```

([Snowflake Docs][7])

### Production notes that matter

Database replication has its own failure surface, especially when objects in one database reference objects in another database. Snowflake explicitly tells you to inspect `OBJECT_DEPENDENCIES` before replicating, because cross-database references can change replication behavior or make a secondary object invalid until dependencies are resolved. That is the part to check before promoting a database between accounts or regions. ([Snowflake Docs][8])

The most reliable inventory queries are still `SHOW DATABASES`, `SHOW SCHEMAS`, and the relevant `INFORMATION_SCHEMA` and `ACCOUNT_USAGE` views. `SHOW` is good for operational visibility, `INFORMATION_SCHEMA` is good for current-database metadata, and `ACCOUNT_USAGE` is where you go for historical and cross-object auditing. ([Snowflake Docs][9])

Use these as add-on sections only.

## Readme.md

Snowflake database objects are best treated as a layered contract, not a flat catalog. The database is the namespace and governance boundary, schemas are the containment and privilege boundary, and the object types inside schemas determine runtime behavior, durability, and operational risk. In production, the fastest way to break a deployment is to ignore object type differences and assume that all objects participate in the same lifecycle, which is false for pipes, models, external tables, materialized views, and procedures. ([docs.snowflake.com](https://docs.snowflake.com/en/sql-reference/sql/create-database), [docs.snowflake.com](https://docs.snowflake.com/en/user-guide/security-access-control-overview))

For inventory and impact analysis, use `INFORMATION_SCHEMA` for low-latency current-state inspection and `ACCOUNT_USAGE` for historical, cross-database, and cross-account governance analysis. For dependency management, `OBJECT_DEPENDENCIES` is the right source when you need to know what will break if a base object is replaced or dropped. For access packaging, use database roles inside the database and managed access schemas where you want grant authority centralized. ([docs.snowflake.com](https://docs.snowflake.com/en/sql-reference/info-schema), [docs.snowflake.com](https://docs.snowflake.com/en/sql-reference/account-usage/object_dependencies), [docs.snowflake.com](https://docs.snowflake.com/en/sql-reference/snowflake-db-roles))

## Pipes and ML Models and Applications.md

### Pipes

A pipe is the persisted definition of a `COPY INTO <table>` statement used by Snowpipe or Snowpipe Streaming to load data into a target table. The pipe stores the ingest contract, including whether auto-ingest is enabled, and Snowflake exposes pipe lifecycle commands such as `CREATE PIPE`, `ALTER PIPE`, `DROP PIPE`, `SHOW PIPES`, and `DESCRIBE PIPE`. For production troubleshooting, `SYSTEM$PIPE_STATUS` is the high-signal operational check because it exposes timestamps for the last successful ingest and the last pipe error. Snowpipe documentation also notes that `PURGE` is not part of the pipe object behavior. ([docs.snowflake.com](https://docs.snowflake.com/en/sql-reference/sql/create-pipe), [docs.snowflake.com](https://docs.snowflake.com/en/user-guide/data-load-snowpipe-intro))

```sql
CREATE OR REPLACE PIPE raw_ingest_pipe
  AUTO_INGEST = TRUE
AS
COPY INTO raw_events
FROM @raw_stage
FILE_FORMAT = (FORMAT_NAME = raw_csv_ff);
```

### ML models

A Snowflake model is a schema object with versioning built in. `CREATE MODEL` creates or replaces a model in the current or specified schema, but Snowflake is explicit that SQL creation from scratch is not the model-authoring path. For SQL, you create models from other models. The object must have at least one version, and one version must be the default. That versioned object model is the core operational distinction: promotion is version-driven, not object-name-driven. ([docs.snowflake.com](https://docs.snowflake.com/en/sql-reference/sql/create-model))

For release workflows, model versioning gives you rollback and controlled promotion without renaming the object itself. That means consumers can remain pointed at the model name while version selection changes underneath them. The practical consequence is that your CI/CD controls should validate version creation, default-version assignment, and downstream consumer compatibility, not just the presence of a model object. ([docs.snowflake.com](https://docs.snowflake.com/en/sql-reference/sql/create-model), [docs.snowflake.com](https://docs.snowflake.com/en/developer-guide/snowflake-ml/modeling))

```sql
CREATE OR REPLACE MODEL fraud_model
FROM some_other_model;
```

### Applications

A Snowflake Native App is created with `CREATE APPLICATION` from an application package or listing. When the command runs, Snowflake executes the app setup script, which means app installation is not just metadata registration, it is an initialization workflow with side effects. The command supports telemetry-event authorization, release channels, and feature-policy attachment, which are the levers you use for controlled deployment and governance. ([docs.snowflake.com](https://docs.snowflake.com/en/sql-reference/sql/create-application))

Native Apps are the application-layer object type in the hierarchy. They are how Snowflake packages data apps that can create objects, expose services, and participate in Snowflake governance rather than living as external orchestration glue. In operational terms, treat app installation like a deployment transaction, not a simple `CREATE` statement. ([docs.snowflake.com](https://docs.snowflake.com/en/developer-guide/native-apps/native-apps-about), [docs.snowflake.com](https://docs.snowflake.com/en/sql-reference/sql/create-application))

## Stages and File Formats.md

A stage is the object boundary for file ingress and egress. `CREATE STAGE` supports either a named file format reference or an inline file format type definition, and those are mutually exclusive. Snowflake documents `FORMAT_NAME` as the preferred route when you want the stage to inherit a reusable parsing contract, while `TYPE` defines the file type directly on the stage. Default stage file type is CSV if not specified. ([docs.snowflake.com](https://docs.snowflake.com/en/sql-reference/sql/create-stage))

Named file formats exist so the parsing contract is explicit and reusable. `CREATE FILE FORMAT` creates a named description of staged data for access or loading into Snowflake tables, and Snowflake also supports `CREATE OR ALTER FILE FORMAT` for idempotent configuration management. This is the correct abstraction when multiple stages or pipelines must share the same parsing semantics, especially for delimiter, quoting, null handling, and compression behavior. ([docs.snowflake.com](https://docs.snowflake.com/en/sql-reference/sql/create-file-format))

User stages behave differently from named stages. Snowflake explicitly says user stages cannot be altered or dropped, and they do not support file format options on the stage itself, so load-time copy options must carry the parsing rules instead. That makes user stages fit for per-user scratch space, not for governed ingestion contracts. ([docs.snowflake.com](https://docs.snowflake.com/en/user-guide/data-load-local-file-system-create-stage), [docs.snowflake.com](https://docs.snowflake.com/en/user-guide/data-load-local-file-system-stage-ui))

```sql
CREATE OR REPLACE FILE FORMAT raw_csv_ff
  TYPE = CSV
  FIELD_DELIMITER = ','
  SKIP_HEADER = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"';

CREATE OR REPLACE STAGE raw_stage
  FILE_FORMAT = raw_csv_ff;
```

For loading, `COPY INTO <table>` can consume all supported stage file types, while unloading to a stage has a narrower supported set. Snowflake documents that unloading to a stage supports CSV, JSON, or PARQUET. If you use `CUSTOM`, Snowflake treats the stage as unstructured and requires `FILE_PROCESSOR`. That is a hard boundary worth preserving in platform standards. ([docs.snowflake.com](https://docs.snowflake.com/en/sql-reference/sql/create-stage))

## Tables and Views.md

Snowflake tables are not one thing. Permanent, temporary, and transient tables differ in durability and recovery semantics, external tables read from files in an external stage, dynamic tables automate refresh, materialized views persist query results for fast reuse, and hybrid tables are optimized for transactional workloads with row locking and integrity constraints. The practical consequence is that table type is an architectural choice, not a storage preference. ([docs.snowflake.com](https://docs.snowflake.com/en/guides-overview-db), [docs.snowflake.com](https://docs.snowflake.com/en/user-guide/tables-temp-transient))

Temporary tables are session-scoped. Transient tables remove Fail-safe after Time Travel expires. External tables are read-only and read files from an external stage. Hybrid tables cannot be temporary or transient and cannot live inside transient schemas or databases. That matters for lifecycle design, because the recoverability and governance model of each table type is materially different. ([docs.snowflake.com](https://docs.snowflake.com/en/sql-reference/sql/create-table), [docs.snowflake.com](https://docs.snowflake.com/en/guides-overview-db))

Views are logical query objects, not data containers. `CREATE VIEW` defines a query over one or more tables or any valid query expression. Materialized views are physically maintained by Snowflake and require Enterprise Edition. Snowflake also documents that `CREATE OR REPLACE` materialized view operations are atomic, so concurrent queries see either the old or the new definition, not a partial replacement. ([docs.snowflake.com](https://docs.snowflake.com/en/sql-reference/sql/create-view), [docs.snowflake.com](https://docs.snowflake.com/en/sql-reference/sql/create-materialized-view))

```sql
CREATE OR REPLACE TABLE stage_fact
  DATA_RETENTION_TIME_IN_DAYS = 1
AS
SELECT * FROM raw_fact;

CREATE OR REPLACE VIEW v_fact AS
SELECT col1, col2
FROM stage_fact;
```

For dependency-sensitive work, prefer `OBJECT_DEPENDENCIES` over guesswork. Snowflake defines a dependency as a logical reference where one object points to another without copying data. That makes it the right source for impact analysis before drop, replace, clone promotion, or replication. ([docs.snowflake.com](https://docs.snowflake.com/en/sql-reference/account-usage/object_dependencies))

## UDFs and Stored Procedures.md

A UDF is the expression-level abstraction. Snowflake says `CREATE FUNCTION` creates a user-defined function that can return scalar or tabular results, and the handler may be inline or referenced from staged or precompiled code depending on language. Use UDFs when you need reusable logic inside SQL expressions, not when you need orchestration, side effects, or multi-step control flow. ([docs.snowflake.com](https://docs.snowflake.com/en/sql-reference/sql/create-function))

Stored procedures are the execution-level abstraction. Snowflake’s `CREATE PROCEDURE` documentation makes clear that the `Session` object is created automatically and passed to the handler, optional arguments use `DEFAULT`, and the procedure return type can be scalar or table-shaped. The procedure object is for orchestration and imperative control flow, while the UDF object is for computation embedded in SQL. ([docs.snowflake.com](https://docs.snowflake.com/en/sql-reference/sql/create-procedure))

```sql
CREATE OR REPLACE FUNCTION util.norm_email(v VARCHAR)
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
  LOWER(TRIM(v))
$$;

CREATE OR REPLACE PROCEDURE ops.refresh_dim()
RETURNS STRING
LANGUAGE SQL
AS
$$
BEGIN
  RETURN 'ok';
END;
$$;
```

Operationally, the main difference is side effects and session access. A procedure can manage session-aware execution, argument defaults, and multi-step logic. A UDF should stay deterministic, compact, and composable. That distinction is what keeps SQL pipelines predictable under change control. ([docs.snowflake.com](https://docs.snowflake.com/en/sql-reference/sql/create-function), [docs.snowflake.com](https://docs.snowflake.com/en/sql-reference/sql/create-procedure))


### Bottom line

A Snowflake database is not just a folder. It is the privilege boundary, clone and replication unit, and metadata root for its schemas and schema objects. In production, the things to care about are database type, schema governance mode, database roles, dependency graphs, and which metadata plane you are querying. If you get those wrong, you get broken grants, broken clones, or broken dependency promotion. ([Snowflake Docs][3])

[1]: https://docs.snowflake.com/en/sql-reference/sql/create-database "CREATE DATABASE | Snowflake Documentation"
[2]: https://docs.snowflake.com/en/sql-reference/sql/grant-privilege "GRANT <privileges> … TO ROLE | Snowflake Documentation"
[3]: https://docs.snowflake.com/en/user-guide/security-access-control-overview "Overview of Access Control | Snowflake Documentation"
[4]: https://docs.snowflake.com/en/sql-reference/info-schema "Snowflake Information Schema | Snowflake Documentation"
[5]: https://docs.snowflake.com/en/sql-reference/account-usage/schemata?utm_source=chatgpt.com "SCHEMATA view"
[6]: https://docs.snowflake.com/en/sql-reference/snowflake-db-roles "SNOWFLAKE database roles | Snowflake Documentation"
[7]: https://docs.snowflake.com/en/sql-reference/account-usage/object_dependencies?utm_source=chatgpt.com "OBJECT_DEPENDENCIES view"
[8]: https://docs.snowflake.com/en/user-guide/database-replication-considerations?utm_source=chatgpt.com "Database replication considerations"
[9]: https://docs.snowflake.com/en/sql-reference/sql/show?utm_source=chatgpt.com "SHOW <objects>"
