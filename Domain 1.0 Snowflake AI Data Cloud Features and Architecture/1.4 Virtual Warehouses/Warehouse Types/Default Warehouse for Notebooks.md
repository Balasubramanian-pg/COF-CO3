# Default Warehouse for Notebooks

```mermaid
graph TD
  NB[Notebook Session] --> Conn[Connection to Snowflake]
  Conn --> WH[Default Warehouse Assignment]
  WH --> Exec[Query Execution]
  Exec --> Res[Results Returned to Notebook]
```

## What Is a Default Warehouse for Notebooks

| Term | Meaning |
|------|---------|
| Default warehouse | The compute cluster automatically used when a notebook runs a query without explicitly specifying one |
| Notebook session | An interactive coding environment like Snowflake Native Notebooks, Jupyter with Snowpark, or VS Code with Python connector |
| Session context | The set of parameters (role, warehouse, database, schema) active for a notebook connection |

- Notebooks do not run queries on their own. They send SQL or Snowpark code to Snowflake
- Snowflake executes that code on a virtual warehouse
- If the notebook does not specify a warehouse, Snowflake uses the default assigned to the user or session
- Setting a sensible default prevents accidental use of oversized or shared warehouses

```mermaid
flowchart LR
  A[User opens notebook] --> B[Session connects to Snowflake]
  B --> C{Warehouse specified in code}
  C -->|Yes| D[Use specified warehouse]
  C -->|No| E[Use default warehouse for user or session]
  D --> F[Run query]
  E --> F
  F --> G[Return results to notebook]
```

## Configuration Options

| Method | Where To Set | Scope | Best For |
|--------|--------------|-------|----------|
| User default warehouse | ALTER USER username SET DEFAULT_WAREHOUSE = wh_name | All sessions for that user | Individual analysts or data scientists |
| Session parameter | Session configuration in notebook code or connection string | Single notebook session | Temporary overrides or testing |
| Role default warehouse | ALTER ROLE role_name SET DEFAULT_WAREHOUSE = wh_name | All users with that role | Team based workflows |
| Notebook platform setting | Snowflake Native Notebooks UI or partner tool config | All notebooks in that workspace | Centralized control for managed environments |
| Connection profile | Config file or environment variable used by connector | Specific project or script | Reproducible pipelines and shared notebooks |

```mermaid
sequenceDiagram
  participant U as User
  participant NB as Notebook
  participant C as Connector
  participant S as Snowflake
  
  U->>NB: Run query cell
  NB->>C: Send SQL with session context
  C->>S: Execute with default warehouse if none specified
  S->>C: Return results and metadata
  C->>NB: Display results in notebook output
```

## Recommended Warehouse Settings for Notebooks

| Workload Type | Base Size | Auto Suspend | Auto Resume | Statement Timeout | Why |
|---------------|-----------|--------------|-------------|-------------------|-----|
| Learning and tutorials | XSmall | 60 seconds | True | 300 seconds | Low cost, fast start, safe for beginners |
| Exploratory analysis | Small | 300 seconds | True | 600 seconds | Handles moderate queries, avoids interrupting flow |
| Data science with Snowpark | Medium | 600 seconds | True | 1800 seconds | Supports long running Python or Scala logic |
| Shared team notebook | Small to Medium | 300 seconds | True | 600 seconds | Balances cost with responsiveness for multiple users |
| Production notebook job | Medium | 0 during run, 60 after | True | 1800 seconds | Guarantees resources during scheduled execution |

```mermaid
quadrantChart
  title "Notebook Workload Profile"
  x-axis "Short Queries" --> "Long Queries"
  y-axis "Single User" --> "Multiple Users"
  "Tutorial cells": [0.2, 0.2]
  "Ad hoc exploration": [0.5, 0.3]
  "Team collaboration": [0.6, 0.7]
  "ML feature prep": [0.8, 0.4]
```

## How to Set Default Warehouse in Common Scenarios

### Snowflake Native Notebooks

```sql
-- Set at user level (persists across sessions)
ALTER USER analyst_jane SET DEFAULT_WAREHOUSE = NOTEBOOK_WH;

-- Set at role level (applies to all users with role)
ALTER ROLE ANALYST_ROLE SET DEFAULT_WAREHOUSE = NOTEBOOK_WH;

-- Override in notebook cell for one time use
USE WAREHOUSE LARGE_TEMP_WH;
-- Run queries here
USE WAREHOUSE DEFAULT; -- Revert to default
```

### Jupyter with Snowpark Python

```python
# In connection parameters
connection_parameters = {
    "account": "myaccount",
    "user": "myuser",
    "warehouse": "NOTEBOOK_WH",  # Default for this session
    "database": "ANALYTICS",
    "schema": "PUBLIC"
}

# Or set after connection
session = Session.builder.configs(connection_parameters).create()
session.sql("USE WAREHOUSE NOTEBOOK_WH").collect()
```

### VS Code with Snowflake Extension

```yaml
# In .snowflake/connection.yml
profiles:
  notebook_dev:
    account: myaccount
    user: myuser
    warehouse: NOTEBOOK_WH  # Default for this profile
    role: ANALYST_ROLE
    database: ANALYTICS
```

### Airflow or Orchestration Tool

```python
# In task definition
SnowflakeOperator(
    task_id="run_notebook_query",
    warehouse="NOTEBOOK_WH",  # Explicit, no reliance on user default
    sql="SELECT * FROM my_table"
)
```

```mermaid
flowchart TD
  Q1[Start: Who uses this notebook]
  Q1 --> Q2[One person learning]
  Q1 --> Q3[Team exploring data]
  Q1 --> Q4[Automated job or pipeline]
  
  Q2 --> A[User default: XSmall, 60s suspend]
  Q3 --> B[Role default: Small, 300s suspend, multi cluster if needed]
  Q4 --> C[Explicit warehouse in code: Medium, no suspend during job]
  
  A --> D[Tag queries with environment=learning]
  B --> E[Tag queries with team=analytics]
  C --> F[Tag queries with project=etl_job]
```

## Cost Control for Notebook Warehouses

| Practice | How To Implement | Impact |
|----------|-----------------|--------|
| Dedicated notebook warehouse | Create NOTEBOOK_WH separate from REPORTING_WH or ETL_WH | Prevents notebook queries from blocking business critical work |
| Auto suspend at 300 seconds | Set in warehouse config for interactive use | Saves credits during thinking time between cells |
| Resource monitor attached | Set credit quota and alert thresholds | Catches runaway notebook sessions before budget impact |
| Query tagging enforced | Set session query_tag at notebook start | Enables cost attribution by user, project, or experiment |
| Statement timeout at 600 seconds | Prevents accidental long running scans | Fails fast on missing filters or cartesian joins |

```mermaid
graph LR
  Notebook[Notebook Query] --> Tag[Apply query_tag]
  Tag --> Exec[Run on default warehouse]
  Exec --> Meter[Record credits with tag]
  Meter --> Report[Aggregate by project or user]
  Report --> Optimize[Identify expensive notebooks for tuning]
```

## Common Issues and Fixes

| Symptom | Likely Cause | Resolution |
|---------|--------------|------------|
| Query fails with warehouse suspended error | Default warehouse has auto resume false | Set auto resume true for interactive notebook warehouses |
| Notebook runs slowly | Default warehouse too small for query complexity | Increase size or enable query acceleration for large scans |
| Credits spike unexpectedly | Notebook left running with idle warehouse | Lower auto suspend to 60 seconds for dev notebooks |
| Multiple users block each other | Shared default warehouse with single cluster | Use multi cluster or assign team specific warehouses |
| Query tags not appearing in reports | Tag set after query execution | Set query_tag at session start, before any queries |
| Warehouse not found error | Default warehouse name typo or missing grant | Verify warehouse exists and role has USAGE privilege |

```mermaid
flowchart TD
  Prob[Notebook issue] --> Q1[Query fails to start]
  Prob --> Q2[Query runs but slow]
  Prob --> Q3[Cost higher than expected]
  
  Q1 --> A[Check warehouse exists and auto resume true]
  Q2 --> B[Check warehouse size and query plan]
  Q3 --> C[Check auto suspend and query tagging]
  
  A --> Fix1[Grant USAGE or fix connection config]
  B --> Fix2[Resize warehouse or optimize query]
  C --> Fix3[Lower suspend or add resource monitor]
```

## Security and Access Considerations

| Practice | Why It Matters | How To Apply |
|----------|---------------|--------------|
| Grant USAGE on notebook warehouse to role, not user | Easier to manage as team changes | GRANT USAGE ON WAREHOUSE NOTEBOOK_WH TO ROLE ANALYST_ROLE |
| Limit OPERATE privilege to platform team | Prevents users from resizing or suspending shared warehouses | Only grant OPERATE to SNOWFLAKE_ADMIN or PLATFORM_ROLE |
| Use row level security with notebook roles | Ensures notebook users only see data they are allowed to see | Apply masking policies and secure views before granting access |
| Audit notebook warehouse usage | Detect misuse or unexpected patterns | Query ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY filtered by notebook tags |
| Isolate prod data access | Prevents accidental changes from exploratory notebooks | Use separate databases or schemas for dev exploration vs production |

```mermaid
graph TD
  Role[Role Based Access] --> Grant[Grant USAGE on NOTEBOOK_WH]
  Grant --> Policy[Apply data masking or row level security]
  Policy --> Audit[Monitor usage via ACCOUNT_USAGE views]
  Audit --> Review[Quarterly access review]
```

## Testing and Validation Checklist

| Step | Action | Success Signal |
|------|--------|----------------|
| 1 | Set default warehouse at user or role level | New notebook sessions pick up warehouse without code change |
| 2 | Run representative query from notebook | Query completes within expected time |
| 3 | Leave notebook idle for 5 minutes | Warehouse suspends after auto suspend period |
| 4 | Submit query after idle period | Warehouse resumes automatically, query runs |
| 5 | Check metering views for query tags | Credits attributed to correct project or user |
| 6 | Simulate concurrent notebook users | No queue time over 30 seconds for interactive work |
| 7 | Verify resource monitor alerts fire | Guardrails activate at defined thresholds |

```mermaid
flowchart LR
  Setup[Configure default warehouse] --> Test[Run test queries]
  Test --> Measure[Check runtime and credits]
  Measure --> Adjust[Tweak size or suspend if needed]
  Adjust --> Doc[Document config and rationale]
```

## Quick Reference: Default Warehouse Decision Flow

```mermaid
flowchart TD
  Q1[Start: What is the notebook for]
  Q1 --> Q2[Learning or personal exploration]
  Q1 --> Q3[Team collaboration or shared analysis]
  Q1 --> Q4[Automated job or scheduled pipeline]
  
  Q2 --> A[User default: XSmall or Small, 60-300s suspend]
  Q3 --> B[Role default: Small or Medium, 300s suspend, multi cluster if concurrent]
  Q4 --> C[Explicit in code: Medium or Large, no suspend during job window]
  
  A --> D[Tag: environment=learning]
  B --> E[Tag: team=analytics, project=exploration]
  C --> F[Tag: project=etl, job=nightly_load]
  
  D --> G[Review quarterly, right size if usage changes]
  E --> G
  F --> G
```

## Key Practices

- Set a default warehouse at the role level for team notebooks. Reduces config drift and onboarding friction
- Use a dedicated warehouse for notebooks. Do not share with dashboards or ETL to avoid resource contention
- Keep auto resume true for interactive notebooks. Users should not see warehouse suspended errors during exploration
- Start with Small size for new notebook warehouses. Scale up only after measuring real query patterns
- Tag every notebook query at session start. Without tags, you cannot attribute cost or optimize effectively
- Attach a resource monitor to notebook warehouses. Catches runaway sessions before they impact budget
- Document the expected workload for each notebook warehouse. Helps teammates understand sizing and policy choices
- Review notebook warehouse usage monthly. Right size or retire warehouses that are no longer used

## Bottom Line

- The default warehouse for notebooks is a safety net and a cost control lever
- Set it intentionally. Do not rely on Snowflake defaults or user preferences
- Match the warehouse config to the notebook workload. Learning, exploration, and production jobs need different settings
- Separate notebook work from other workloads. Shared warehouses create contention and obscure cost attribution
- Measure before you change. One week of notebook query history tells you more than guessing
- Tag everything. Without attribution, you cannot optimize or explain spend
- Review regularly. Notebook usage patterns change as teams learn and projects evolve

Think of default warehouse for notebooks like a home office setup:
- The default desk is where you sit unless you choose otherwise
- A small desk works for quick tasks. A large desk helps with complex projects
- Turning off the light when you leave saves power. Leaving it on avoids fumbling in the dark
- Labeling your files helps you find work later. Tagging queries helps you track cost later
- Upgrading your setup is easy. Downgrading feels like losing progress

Set your default wisely. Adjust when your work changes. Save effort without sacrificing output.
