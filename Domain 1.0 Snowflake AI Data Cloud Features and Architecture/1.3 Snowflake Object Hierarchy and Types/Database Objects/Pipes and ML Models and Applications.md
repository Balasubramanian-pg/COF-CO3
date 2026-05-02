## Pipes and ML Models and Applications

### Pipes

A pipe is the persisted `COPY INTO <table>` definition used by Snowpipe to load data from an ingestion queue, and by Snowpipe Streaming with high-performance architecture to load streaming data directly into tables. 
Snowflake exposes the standard lifecycle commands `CREATE PIPE`, `ALTER PIPE`, `DROP PIPE`, `SHOW PIPES`, and `DESCRIBE PIPE`, and the operational status surface is `SYSTEM$PIPE_STATUS(<pipe_name>)`. 
In production, the two useful monitoring pivots are `SYSTEM$PIPE_STATUS` for current state and `INFORMATION_SCHEMA.PIPES` for metadata, with the caveat that the `PIPES` view returns rows only to the pipe owner or a role with `MONITOR` privilege. ([Snowflake Docs][1])

```sql
CREATE OR REPLACE PIPE raw_ingest_pipe
  AUTO_INGEST = TRUE
AS
COPY INTO raw_events
FROM @raw_stage
FILE_FORMAT = (FORMAT_NAME = raw_csv_ff);
```

For incident work, `DESCRIBE PIPE` is the fastest way to inspect the stored definition, and Snowflake’s pipe-management guidance says you can use `CREATE OR REPLACE PIPE` to change the COPY statement, internally dropping and recreating the pipe. 
That matters because the pipe object is the control plane, not just a convenience wrapper around COPY. 
If the definition changes, treat it like a controlled deployment, not a harmless metadata edit. ([Snowflake Docs][2])

```sql
SELECT SYSTEM$PIPE_STATUS('RAW_INGEST_PIPE');

DESC PIPE raw_ingest_pipe;
SHOW PIPES;
```

### ML models

<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/18a06766-8ca8-42f4-a8e9-436723ac44be" />

Snowflake ML models are versioned schema objects. `CREATE MODEL` creates a model in the current or specified schema, but Snowflake is explicit that SQL can only create models from other models, while creation from scratch is done through the Snowflake Model Registry Python API. Every model must have at least one version, and one version must be designated as the default. ([Snowflake Docs][3])

The operational consequence is that model promotion is version-centric. `ALTER MODEL … ADD VERSION` adds a new version, and `ALTER MODEL` can set the default version. 
`SHOW VERSIONS IN MODEL` exposes the version inventory, including which version is default. 
In other words, the stable object name stays fixed while the deployable artifact moves behind it. ([Snowflake Docs][4])

```sql
CREATE OR REPLACE MODEL fraud_model
FROM some_other_model;

ALTER MODEL fraud_model
  SET DEFAULT_VERSION = 'v2';

SHOW VERSIONS IN MODEL fraud_model;
```

For production controls, treat the default version as the live pointer and version creation as the release event. 
That gives you rollback by pointer switch rather than object replacement, which is the safer pattern when downstream consumers are pinned to a model name rather than a version. ([Snowflake Docs][5])

### Applications
<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/f2126897-4909-4bac-a298-942d0e1f3510" />

A Snowflake Native App is created from an application package or a listing. 
When `CREATE APPLICATION` runs, Snowflake executes the setup script, so installation is a real initialization workflow, not just object registration. 
The command also supports release-channel selection, telemetry authorization, tags, feature policy attachment, and background install for listing-based installs. ([Snowflake Docs][6])

The provider-side container for a Native App is the application package. 
Snowflake defines an application package as the container that encapsulates the app’s data content and application logic, plus version and patch information. 
Each version requires its own manifest and setup script. Release channels are enabled by default on new application packages, and Snowflake recommends them for new development. ([Snowflake Docs][7])

```sql
CREATE APPLICATION PACKAGE my_application_package;

CREATE APPLICATION my_app
  FROM APPLICATION PACKAGE my_application_package
  USING RELEASE CHANNEL DEFAULT;
```

Privilege boundaries matter here. 
Snowflake says creating an application package requires the global `CREATE APPLICATION PACKAGE` privilege, and application package privileges include `DEVELOP`, `INSTALL`, `MANAGE RELEASES`, `MANAGE VERSIONS`, and `OWNERSHIP`. 
That is the right control surface for separating app development, release management, and consumer installation responsibilities. ([Snowflake Docs][7])

### Production takeaway

Use pipes for deterministic ingest control, ML models for versioned inference artifacts, and Native Apps for packaged application deployment with explicit release management. The shared pattern is the same: the object name is the stable interface, while the operational behavior is driven by status, version, or setup-script execution behind that name. ([Snowflake Docs][1])

[1]: https://docs.snowflake.com/en/sql-reference/sql/create-pipe "CREATE PIPE | Snowflake Documentation"
[2]: https://docs.snowflake.com/en/sql-reference/sql/desc-pipe?utm_source=chatgpt.com "DESCRIBE PIPE"
[3]: https://docs.snowflake.com/en/sql-reference/sql/create-model "CREATE MODEL | Snowflake Documentation"
[4]: https://docs.snowflake.com/en/sql-reference/sql/alter-model-add-version?utm_source=chatgpt.com "ALTER MODEL … ADD VERSION"
[5]: https://docs.snowflake.com/en/sql-reference/sql/alter-model?utm_source=chatgpt.com "ALTER MODEL"
[6]: https://docs.snowflake.com/en/sql-reference/sql/create-application "CREATE APPLICATION | Snowflake Documentation"
[7]: https://docs.snowflake.com/en/developer-guide/native-apps/creating-app-package "Create and manage an application package | Snowflake Documentation"
