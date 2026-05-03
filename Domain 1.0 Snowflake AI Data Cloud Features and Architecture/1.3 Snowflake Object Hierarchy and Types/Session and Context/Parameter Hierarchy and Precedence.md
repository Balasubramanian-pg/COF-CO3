## Parameter Hierarchy and Precedence

Snowflake parameters control execution behavior across compute, storage interaction, query limits, and session semantics. The system is multi-layered and override-driven. Most production inconsistencies come from misunderstanding which layer actually wins.

### Parameter scopes

Snowflake parameters exist at these scopes:

* Account
* User
* Session
* Object:

  * Warehouse
  * Database
  * Schema
  * Table

Each parameter is not universally applicable to all scopes. Scope support is parameter-specific.

### Precedence model

For most parameters:

```
Session > User > Account
```

This is strict override precedence. The closest scope to execution wins.

Example:

```sql
ALTER ACCOUNT SET STATEMENT_TIMEOUT_IN_SECONDS = 3600;
ALTER USER my_user SET STATEMENT_TIMEOUT_IN_SECONDS = 600;
ALTER SESSION SET STATEMENT_TIMEOUT_IN_SECONDS = 60;
```

Effective value at runtime = **60 seconds**

### Warehouse interaction model

Warehouse-scoped parameters introduce a different rule:

For parameters like:

* `STATEMENT_TIMEOUT_IN_SECONDS`
* `STATEMENT_QUEUED_TIMEOUT_IN_SECONDS`

Snowflake applies:

```
Effective value = MIN(non-zero(session-level, warehouse-level))
```

This is not override. This is constraint intersection.

Example:

```sql
ALTER SESSION SET STATEMENT_TIMEOUT_IN_SECONDS = 300;
ALTER WAREHOUSE compute_wh SET STATEMENT_TIMEOUT_IN_SECONDS = 120;
```

Effective timeout = **120 seconds**

If one value is 0 (unlimited), the other value applies.

### Object-level parameters

Object parameters apply when the object is used, not when the session starts.

Examples:

* Table: `DATA_RETENTION_TIME_IN_DAYS`
* Schema: default retention inheritance
* Database: default parameter propagation

Inheritance pattern:

```
Database → Schema → Table
```

Overrides at lower levels replace inherited values.

### Resolution timing

Critical detail:

* Parameter resolution happens at **statement execution time**
* Not at session creation
* Not at object creation (except defaults captured at creation)

Implication:
Changing a session parameter mid-session immediately affects subsequent queries.

### Default propagation

When creating objects:

* Schema inherits database-level defaults
* Table inherits schema-level defaults
* Explicit values override inheritance

Example:

```sql
ALTER DATABASE analytics SET DATA_RETENTION_TIME_IN_DAYS = 7;

CREATE SCHEMA analytics.core;

CREATE TABLE analytics.core.fact_sales (...);
```

Table retention = **7 days** unless overridden.

### Inspection and debugging

Check effective parameters at each scope:

```sql
SHOW PARAMETERS IN SESSION;
SHOW PARAMETERS IN USER my_user;
SHOW PARAMETERS IN ACCOUNT;
SHOW PARAMETERS IN WAREHOUSE compute_wh;
```

For object-level:

```sql
SHOW PARAMETERS IN TABLE analytics.core.fact_sales;
```

For current effective session:

```sql
SELECT *
FROM TABLE(INFORMATION_SCHEMA.SESSION_PARAMETERS());
```

### Failure modes

1. **Hidden overrides**
   User-level parameter silently overrides account baseline

2. **Warehouse constraint collisions**
   Session expects higher timeout but warehouse enforces lower

3. **Environment drift**
   Dev vs prod mismatch due to different account-level defaults

4. **Implicit inheritance**
   Table inherits retention or behavior unintentionally

5. **Mid-session mutation**
   Parameter changes affect subsequent queries unpredictably

### Performance implications

* Lower `STATEMENT_TIMEOUT_IN_SECONDS` reduces long-running query risk but increases failure rate under heavy joins or large scans
* Low `STATEMENT_QUEUED_TIMEOUT_IN_SECONDS` increases query rejection under concurrency pressure
* Warehouse-level limits act as hard caps, preventing runaway workloads

There is no direct credit cost for parameters, but indirect cost impact is significant:

* Timeouts → retries → higher compute usage
* Queue timeouts → failed workloads → reprocessing overhead

### Production patterns

1. **Pin critical parameters at session start**

```sql
ALTER SESSION SET
  STATEMENT_TIMEOUT_IN_SECONDS = 600,
  STATEMENT_QUEUED_TIMEOUT_IN_SECONDS = 60;
```

2. **Use warehouse as safety guardrail**
   Enforce upper bounds at warehouse level

3. **Avoid user-level overrides**
   They create invisible drift

4. **Standardize database defaults**
   Especially retention and time travel

5. **Audit regularly**

```sql
SELECT *
FROM SNOWFLAKE.ACCOUNT_USAGE.PARAMETER_HISTORY
ORDER BY START_TIME DESC;
```

### Anti-patterns

* Relying on account defaults for production workloads
* Setting conflicting session and warehouse limits without understanding min-rule
* Allowing uncontrolled user-level parameter overrides
* Not validating parameter state in pipelines

### Bottom line

Snowflake parameter behavior is not simple override logic.
It is a layered resolution system with mixed precedence and constraint rules.

If you do not explicitly control parameter scope and resolution, you will get inconsistent execution behavior across users, warehouses, and environments.
