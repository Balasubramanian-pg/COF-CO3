## Database Objects

A Snowflake database is the namespace boundary between account-level objects and schema-level objects. At the database layer, you are managing database creation semantics, database roles, schemas, replication or clone behavior, and the metadata surfaces that expose those objects. Snowflake automatically creates `PUBLIC` and `INFORMATION_SCHEMA` when you create a new database, and `INFORMATION_SCHEMA` is the schema that exposes metadata views and table functions for the database and, in some cases, account-wide objects. Databases created from shares are exceptions: they do not get `PUBLIC` or `INFORMATION_SCHEMA` unless those were explicitly granted, they cannot be cloned, and some properties such as `TRANSIENT` and `DATA_RETENTION_TIME_IN_DAYS` do not apply. ([Snowflake Docs][1])

A practical way to think about database objects is this: the database owns schemas, schemas own tables, views, stages, file formats, streams, tasks, pipes, policies, UDFs, and sequences, and the database boundary is what lets Snowflake apply privilege scope cleanly. Snowflake’s DDL and access-control docs explicitly treat tables, views, stages, file formats, UDFs, and sequences as schema objects, and `GET_DDL` confirms the schema-object namespace pattern `database.schema` or `schema` for objects in that family. ([Snowflake Docs][2])

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

Schemas are the real control plane inside the database. Snowflake’s access-control model says that in a regular schema, the owner role has all privileges on the object by default, including grant and revoke authority, while in a managed access schema the object owners lose the ability to make grant decisions and only the schema owner or a role with `MANAGE GRANTS` can grant privileges on objects in the schema. That is the key production distinction between a plain schema and a governed schema. ([Snowflake Docs][3])

Database-level metadata is exposed through `INFORMATION_SCHEMA` and `ACCOUNT_USAGE`. `INFORMATION_SCHEMA` is automatically created in every database and is the low-latency metadata surface for the current database. `ACCOUNT_USAGE` gives account-wide object metadata and historical usage, includes dropped objects, and has longer retention but more latency. The `SCHEMATA` view in `ACCOUNT_USAGE` excludes `ACCOUNT_USAGE`, `READER_ACCOUNT_USAGE`, and `INFORMATION_SCHEMA` schemas, so it is useful for inventory but not a complete dump of every schema in the system. ([Snowflake Docs][4])

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

Database roles are scoped to the database that contains them, and they exist so you can package privileges on that database, its schemas, and its schema objects without widening everything to account roles. Snowflake documents that database roles can receive privileges on the database itself, schemas within it, and schema objects such as tables, views, stages, file formats, UDFs, and sequences inside that database. A database role can then be granted to an account role or another role, which gives you a clean least-privilege pattern for access packaging. ([Snowflake Docs][2])

The `SNOWFLAKE` database itself is a special case because it ships with Snowflake-provided database roles such as `OBJECT_VIEWER`, `USAGE_VIEWER`, `GOVERNANCE_VIEWER`, and `SECURITY_VIEWER` for controlled access to shared metadata and system views. Administrators can use `GRANT DATABASE ROLE` to assign those database roles to custom roles, and then grant those custom roles to users. That is the supported pattern for delegating read access to Snowflake metadata without opening broad privileges. ([Snowflake Docs][6])

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
