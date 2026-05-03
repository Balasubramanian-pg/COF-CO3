## Session and Context

Snowflake session and context objects define how execution state is resolved at runtime. 

This layer controls parameter resolution, role/database/schema/warehouse context, variable substitution, and policy evaluation inputs. Most production defects in Snowflake are not compute issues but context-resolution issues.

<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/b6047774-bc12-437c-acfc-de2b83897c8a" />

>[!Warning]
>**Session** <br>
>A Session in Snowflake represents an active connection between a client and Snowflake, established after successful authentication. <br>
> 1. It maintains connection-level state for the duration of the connection, including authentication tokens, client info, parameter settings, and transaction state.
> 2. A session is terminated by explicit logout, timeout, or connection loss. Each session is identified by a unique SESSION_ID.

### Session lifecycle and context resolution

A session is established when a client connects and is bound to:

* Active role
* Current database and schema
* Current warehouse
* Session parameters
* Session variables

>[!Tip]
>**Context** <br>
>Context in Snowflake refers to the current execution environment settings that determine how SQL statements behave. <br>
> 1. It includes the current role, warehouse, database, and schema.
> 2. Context is scoped to a session but can be changed multiple times within that session using USE commands.
> 3. Context determines object name resolution and privileges for every statement executed.

>[!Note]
>Context resolution is hierarchical and stateful. <br>
>Every SQL statement executes against the current session context unless explicitly overridden.

```sql
SELECT
  CURRENT_ROLE(),
  CURRENT_DATABASE(),
  CURRENT_SCHEMA(),
  CURRENT_WAREHOUSE();
```

Key property: context is **late-bound at execution time**, not at object creation time (except for stored definitions like views).

### Parameter hierarchy and precedence

Snowflake parameters exist at multiple scopes:

* Account
* User
* Session
* Object (warehouse, database, schema, table)

Precedence model:

```
Session > User > Account
```

For warehouse-related parameters:

```
Lowest non-zero value between session-level and warehouse-level applies
```

This is critical for timeout, query limits, and execution controls.

```sql
ALTER SESSION SET STATEMENT_TIMEOUT_IN_SECONDS = 300;
```

Failure mode: mismatched parameter scopes cause non-deterministic behavior across environments.

### Session variables

Session variables are runtime key-value bindings.

Constraints:

* Max size: 256 bytes (string/binary)
* Case-insensitive on set, uppercase internally
* Session-scoped only

```sql
SET env = 'prod';

SELECT $ENV;
```

Variables can be used dynamically with object resolution:

```sql
SET tbl = 'FACT_SALES';

SELECT * FROM IDENTIFIER($TBL);
```

Key production use:

* Dynamic SQL
* Environment abstraction (dev vs prod)
* Parameterized pipelines without external orchestration

Failure mode:

* Silent uppercase conversion breaks case-sensitive identifiers
* Overuse leads to unreadable SQL and hidden dependencies

### Context functions

Context functions expose runtime execution state.

Common ones:

```sql
SELECT
  CURRENT_ROLE(),
  CURRENT_USER(),
  CURRENT_ACCOUNT(),
  CURRENT_REGION(),
  CURRENT_DATABASE(),
  CURRENT_SCHEMA(),
  CURRENT_WAREHOUSE();
```

Special behavior:

* `CURRENT_ROLE()` returns NULL when using database roles
* Context functions behave differently inside:

  * Views
  * UDFs
  * Policies

This matters for security logic.

### SYS_CONTEXT and POLICY_CONTEXT

Snowflake provides structured context access:

```sql
SELECT SYS_CONTEXT('USERENV', 'CURRENT_ROLE');
```

For governance:

```sql
SELECT POLICY_CONTEXT('CURRENT_ROLE');
```

Use cases:

* Row access policies
* Masking policies
* Conditional access logic

Key property:
Policy evaluation context can differ from session context depending on execution path.

### Context in views and compiled objects

Views capture SQL text, not evaluated results.

Implications:

* Context functions are evaluated at query runtime
* Underlying object resolution uses:

  * View owner’s privileges (secure views)
  * Caller context (standard views)

Failure mode:

* Changing session context changes view results unexpectedly
* Role-based filters behave inconsistently if not explicitly controlled

### Context switching

Explicit overrides:

```sql
USE ROLE analyst;
USE DATABASE analytics;
USE SCHEMA core;
USE WAREHOUSE compute_wh;
```

These mutate session state.

Production pattern:
Avoid implicit reliance on session defaults. Always set context explicitly in pipelines.

### Transactional interaction

Session context persists across transactions.

Important distinctions:

* Transactions affect data state
* Session affects execution behavior

Rollback does NOT revert:

* Session variables
* Session parameters
* Context (role, schema, etc.)

This separation is a common source of bugs in procedural workflows.

### Monitoring session behavior

```sql
SELECT *
FROM SNOWFLAKE.ACCOUNT_USAGE.SESSIONS
ORDER BY CREATED_ON DESC;
```

For query-level context:

```sql
SELECT
  QUERY_ID,
  ROLE_NAME,
  DATABASE_NAME,
  SCHEMA_NAME,
  WAREHOUSE_NAME
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
ORDER BY START_TIME DESC;
```

Latency caveat:
ACCOUNT_USAGE views can lag up to ~45–180 minutes.

### Production patterns

1. **Deterministic pipelines**
   Always set:

   * ROLE
   * DATABASE
   * SCHEMA
   * WAREHOUSE

2. **Environment isolation**
   Use session variables + IDENTIFIER()

3. **Policy-safe context**
   Avoid relying on implicit CURRENT_ROLE inside policies

4. **Parameter control**
   Set critical parameters at session start, not ad hoc

5. **Avoid hidden state**
   Treat session like mutable global state, minimize reliance

### Anti-patterns

* Relying on default database/schema
* Mixing session variables with hardcoded identifiers
* Using context functions inside complex view chains without control
* Not resetting context between jobs
* Debugging data issues without checking session state

### Bottom line

Session and context in Snowflake are the hidden execution layer.
Data correctness, security enforcement, and query behavior all depend on it.

If you don’t control session state explicitly, you don’t control your system.
