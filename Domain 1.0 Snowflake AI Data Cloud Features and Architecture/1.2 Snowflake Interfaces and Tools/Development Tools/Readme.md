# Snowflake Development Tools Overview

```mermaid
graph TD
  A[Development Tools] --> B[Snowsight Web UI]
  A --> C[SnowSQL CLI]
  A --> D[IDE Extensions]
  A --> E[Snowpark Python Java]
  A --> F[dbt Core]
```

| Tool | Primary Use Case | Where It Struggles |
|------|------------------|-------------------|
| Snowsight Web UI | Quick queries, data preview, account settings | Poor offline work, weak git tracking |
| SnowSQL CLI | Automation, CI/CD pipelines, server scripts | No visual editor, hard to debug large files |
| IDE Extensions VS Code JetBrains | Daily coding, version control, debugging | Requires local config, depends on API limits |
| Snowpark | Custom data apps, UDFs, lightweight ML | Needs programming knowledge, extra setup |
| dbt Core | Data transformations, testing, documentation | SQL only, not for raw ingestion or admin tasks |

```mermaid
flowchart LR
  A[Write Script] --> B[Commit to Git]
  B --> C[Run in IDE or CLI]
  C --> D[Test in Dev Warehouse]
  D --> E[Promote to Prod]
  E --> F[Monitor in Snowsight]
```

- Install the tool that matches your daily task. Do not force one tool to do everything.
- Keep connection files out of git. Add local config folders to gitignore.
- Use separate warehouses for dev and prod. Dev can run small. Prod runs at scale.
- Store SQL and Python files in a single repo. Track every change.
- Switch to key pair or OAuth auth. Password sessions expire and break automation.
- Run explain before heavy joins. Check execution plan to avoid wasted credits.
- Test role permissions in dev first. Wrong role blocks queries in prod.

| Issue | Root Cause | Fix |
|-------|------------|-----|
| Query stalls | Warehouse too small or suspended | Increase size or disable auto suspend for testing |
| Auth fails mid run | Short lived token | Switch to key pair or configure OAuth refresh |
| Missing columns in autocomplete | Metadata cache outdated | Run metadata refresh command in IDE |
| dbt models fail | Target database mismatch | Update profiles yaml to point to correct schema |
| Snowpark package error | Missing dependencies | List required packages in session configuration |

```mermaid
flowchart TD
  Q1[Start: What are you building] 
  Q1 --> Q2[Quick check or data preview]
  Q1 --> Q3[Pipeline automation]
  Q1 --> Q4[Daily coding and git workflow]
  Q1 --> Q5[Python or Java data logic]
  Q1 --> Q6[SQL transformations and tests]
  
  Q2 --> A[Snowsight]
  Q3 --> B[SnowSQL CLI]
  Q4 --> C[IDE Extension]
  Q5 --> D[Snowpark]
  Q6 --> E[dbt Core]
```

- Match the tool to the job. Web UI for quick looks, CLI for scripts, IDE for code, Snowpark for apps, dbt for transforms.
- Never hardcode credentials. Use environment variables or secure vaults.
- Keep warehouses on auto suspend. Turn them on only when running tasks.
- Version control every script. Local files break when machines change.
- Start small. Test logic on sample data before running full pipelines.
