# Session and Context Variables

## 1.1 Session Variables

Session variables are runtime key-value bindings scoped to a single session. They are not persisted and are lost when the session ends.

**Constraints**

* Max size: 256 bytes (string/binary)
* Case-insensitive on assignment, stored as uppercase
* Session-scoped only

```sql
SET env = 'prod';
SET run_id = '20260503_01';

SELECT $ENV, $RUN_ID;
```

**Dynamic object resolution**

```sql
SET tbl = 'FACT_SALES';

SELECT * FROM IDENTIFIER($TBL);
```

**Operational notes**

* Variables are resolved at execution time
* Useful for environment abstraction and parameterized SQL
* Overuse creates hidden dependencies and reduces query readability

## 1.2 Context Functions

Context functions expose runtime session state. These are evaluated at query execution time.

```sql
SELECT
  CURRENT_USER(),
  CURRENT_ROLE(),
  CURRENT_DATABASE(),
  CURRENT_SCHEMA(),
  CURRENT_WAREHOUSE(),
  CURRENT_ACCOUNT(),
  CURRENT_REGION();
```

**Key behaviors**

* `CURRENT_ROLE()` returns NULL when using database roles
* Values reflect current session state, not object definition state
* Behavior can differ inside views, policies, and UDFs

## 1.3 SYS_CONTEXT

Provides structured access to session environment attributes.

```sql
SELECT SYS_CONTEXT('USERENV', 'CURRENT_ROLE');
```

**Use cases**

* Fine-grained access control logic
* Dynamic query behavior based on session attributes

## 1.4 POLICY_CONTEXT

Exposes context specifically for governance policies.

```sql
SELECT POLICY_CONTEXT('CURRENT_ROLE');
```

**Use cases**

* Row access policies
* Masking policies
* Conditional data exposure

**Important**
Policy evaluation context may differ from session context depending on execution path and object type.

## 1.5 Context in Views and Compiled Objects

Views store SQL text, not evaluated results.

**Implications**

* Context functions are evaluated at runtime
* Output depends on the executing session

**Resolution model**

* Standard views: use caller context
* Secure views: use view owner’s privileges

**Failure modes**

* Same query returns different results under different roles
* Role-based filters behave inconsistently if not explicitly controlled
* Session variables are not captured in view definitions

## 1.6 Session State Mutation

Session context can be modified explicitly during execution.

```sql
USE ROLE analyst;
USE DATABASE analytics;
USE SCHEMA core;
USE WAREHOUSE compute_wh;
```

**Behavior**

* Changes apply immediately to subsequent statements
* Does not affect already running queries

**Risk**

* Implicit context switching causes non-deterministic behavior in pipelines

## 1.7 Variable Interaction with SQL Engine

Variables are substituted before execution but resolved within session context.

**Execution model**

* Variables → substituted
* Identifiers → resolved via IDENTIFIER()
* Context → applied at runtime

```sql
SET schema_name = 'CORE';

SELECT *
FROM IDENTIFIER('ANALYTICS.' || $SCHEMA_NAME || '.FACT_SALES');
```

## 1.8 Limitations and Edge Cases

* No cross-session persistence
* No native support for complex data structures
* Cannot be used directly in object DDL definitions (without IDENTIFIER)
* Uppercase normalization can break case-sensitive logic
* Not suitable for large payloads or state management

## 1.9 Monitoring and Debugging

Session-level visibility:

```sql
SELECT *
FROM TABLE(INFORMATION_SCHEMA.SESSION_PARAMETERS());
```

Query-level context inspection:

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

Session tracking:

```sql
SELECT *
FROM SNOWFLAKE.ACCOUNT_USAGE.SESSIONS
ORDER BY CREATED_ON DESC;
```

**Latency**

* ACCOUNT_USAGE views: ~45–180 minutes delay

## 1.10 Production Patterns

1. Always set explicit context at session start
2. Use variables for environment abstraction, not logic control
3. Avoid mixing variables and hardcoded identifiers
4. Validate context in every pipeline execution
5. Use IDENTIFIER() for dynamic object resolution

## 1.11 Anti-Patterns

* Relying on default database/schema
* Using session variables as hidden configuration
* Embedding context-dependent logic in views without controls
* Ignoring role context in security policies
* Not resetting session state between jobs

## 1.12 Bottom Line

Session and context variables form the execution control layer of Snowflake.

They do not store data, but they determine how data is accessed, filtered, and processed.

If session state is not explicitly controlled, system behavior becomes non-deterministic.
