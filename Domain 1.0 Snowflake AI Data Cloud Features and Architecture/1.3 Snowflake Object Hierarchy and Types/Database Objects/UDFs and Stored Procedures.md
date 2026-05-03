## UDFs and Stored Procedures

User-defined functions and stored procedures occupy different layers in Snowflake’s object model. 

A UDF is the expression-level object

1. it evaluates to a scalar or tabular result and can be used where a general SQL expression is valid.
2. A stored procedure is the _orchestration object_: it supports branching, looping, multi-statement logic, and can execute DDL and DML.
3. Snowflake treats them as different invocation models, not interchangeable wrappers. ([Snowflake Docs][1])

### UDFs

A `CREATE FUNCTION` object creates a UDF, and Snowflake documents that it can return either scalar results or tabular results. 
1. UDF handlers can be written in SQL, JavaScript, Python, Java, or Scala, and depending on the language the handler can be inlined in the DDL or referenced from staged or precompiled code. 
2. That makes UDFs the right fit for reusable computation, policy logic, and query-time transformations. ([Snowflake Docs][1])
3. For scalar SQL UDFs, Snowflake supports the `MEMOIZABLE` keyword. Snowflake’s docs describe memoizable functions as a way to cache deterministic scalar SQL UDF results, which is especially relevant when the same expression is repeatedly invoked in policies or repetitive query paths. ([Snowflake Docs][2])

```sql
CREATE OR REPLACE FUNCTION util.norm_email(v VARCHAR)
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
  LOWER(TRIM(v))
$$;
```

### Stored procedures

Stored procedures are designed for procedural code. Snowflake documents that they support branching, looping, and other programmatic constructs, and they are commonly used to automate multiple database operations or dynamically create and execute database operations. They can also run with owner’s rights or caller’s rights, which makes them the boundary object for controlled privilege delegation. ([Snowflake Docs][3])

A stored procedure does not behave like a scalar expression in SQL. Snowflake explicitly says you cannot use a stored procedure directly in an expression the way you would use a function. The normal invocation is `CALL`, although a procedure that returns tabular data can also be called in the `FROM` clause, and Snowflake Scripting can capture returned values inside a block. ([Snowflake Docs][4])

Snowflake also supports anonymous stored procedures via `WITH ... CALL ...`, and those do not require a role with `CREATE PROCEDURE` schema privileges. For SQL stored procedures, Snowflake Scripting is the body format, and Snowflake notes a recommended source size ceiling of about 100 KB for the procedure body. ([Snowflake Docs][5])

```sql
CREATE OR REPLACE PROCEDURE ops.refresh_dim()
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
BEGIN
  -- multi-step orchestration logic
  RETURN 'ok';
END;
$$;
```
### **UDF vs Stored Procedure: Key Differences**

| Aspect | **User-Defined Function (UDF)** | **Stored Procedure** |
| --- | --- | --- |
| **Primary Purpose** | Extend SQL with custom calculations/transformations | Encapsulate multi-step business logic and workflows |
| **Invocation** | Used in SQL expressions: `SELECT my_udf(col) FROM table` | Called explicitly: `CALL my_proc(arg1, arg2)` |
| **Return Value** | Must return: Scalar value OR Table for UDTFs | Returns single value, usually VARCHAR status message |
| **SQL Usage Context** | Can be used in `SELECT`, `WHERE`, `JOIN`, `GROUP BY` | Cannot be used directly in queries; standalone call only |
| **Side Effects** | No side effects allowed. Cannot run DML/DDL | Allowed. Can `INSERT`, `UPDATE`, `CREATE`, `DROP`, etc |
| **Transaction Control** | Not allowed. No `COMMIT`/`ROLLBACK` | Allowed. Can explicitly manage transactions |
| **Dynamic SQL** | Not supported | Supported. Can build and execute SQL strings |
| **Error Handling** | Limited. Error aborts statement | Full support via `EXCEPTION` blocks in SQL Scripting |
| **Execution Rights** | Always owner’s rights | Caller’s rights by default. `EXECUTE AS OWNER` optional |
| **State** | Stateless per row for scalar, per partition for UDTF | Can maintain state across multiple SQL statements |
| **Types** | Scalar UDF, Table UDF/UDTF, Aggregate, Window | No sub-types. All are procedural |
| **Recursion** | Not allowed | Allowed with nesting depth limits |
| **Typical Use Case** | `clean_email(email_col)`, `haversine_dist(lat1,lon1,lat2,lon2)` | `load_staging_to_prod()`, `clone_db_with_grants()`, `loop_and_merge()` |

**Rule of thumb to remember:**  
Use a **UDF** when you need to compute a value inside a query.  
Use a **Stored Procedure** when you need to orchestrate actions, especially DDL/DML or multi-step logic.

### Null handling and arguments

Stored procedures have explicit null-handling behavior in their DDL. Snowflake documents `CALLED ON NULL INPUT` and `RETURNS NULL ON NULL INPUT`, with `CALLED ON NULL INPUT` as the default. Snowflake Scripting procedures also support `IN` and `OUT` arguments, with output values passed back through variables rather than as multiple return values. ([Snowflake Docs][6])

### Practical decision rule

Use a UDF when the logic must behave like a function inside a query and return a value inline. Use a stored procedure when the logic needs multiple statements, procedural flow, control over execution order, or controlled privilege delegation. That is the cleanest separation Snowflake’s own docs draw between the two objects. ([Snowflake Docs][4])

### Production pattern

1. For platform code, keep UDFs small, deterministic, and composable.
2. Use stored procedures for orchestration, DDL, DML, and cross-step workflows.
3. If a routine starts needing control flow, state, or side effects, promote it from UDF to stored procedure instead of overloading a function with procedural behavior.
4. That matches Snowflake’s invocation and capability model. ([Snowflake Docs][3])

If useful, I can do the same treatment for `Session and Context Variables.md` next.

[1]: https://docs.snowflake.com/en/sql-reference/sql/create-function "CREATE FUNCTION | Snowflake Documentation"
[2]: https://docs.snowflake.com/en/developer-guide/udf/sql/udf-sql-scalar-functions "Scalar SQL UDFs | Snowflake Documentation"
[3]: https://docs.snowflake.com/en/developer-guide/stored-procedure/stored-procedures-overview "Stored procedures overview | Snowflake Documentation"
[4]: https://docs.snowflake.com/en/developer-guide/stored-procedures-vs-udfs "Choosing whether to write a stored procedure or a user-defined function | Snowflake Documentation"
[5]: https://docs.snowflake.com/en/developer-guide/stored-procedure/stored-procedures-usage "Working with stored procedures | Snowflake Documentation"
[6]: https://docs.snowflake.com/en/sql-reference/sql/create-procedure "CREATE PROCEDURE | Snowflake Documentation"
