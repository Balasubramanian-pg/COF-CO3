# Virtual Warehouse Best Practices
This document covers the additional best practices which we did not cover in the previous [document]([url](https://github.com/Balasubramanian-pg/COF-CO3/blob/main/Domain%201.0%20Snowflake%20AI%20Data%20Cloud%20Features%20and%20Architecture/1.4%20Virtual%20Warehouses/Readme.md)). 

Let us first start from understanding the bigger picture.

```mermaid
graph TD
  WB[Warehouse Best Practices] --> RM[Resource Monitors]
  WB --> QT[Query Tagging]
  WB --> NC[Naming Conventions]
  WB --> SW[Workload Separation]
  WB --> ST[Statement Timeouts]
  WB --> AC[Access Control]
  WB --> SCH[Scheduling]
  WB --> MON[Monitoring]
```

## Resource Monitors and Credit Control

| Setting | What It Does | Recommended Use |
|---------|-------------|-----------------|
| Credit quota | Set maximum credits per billing period | Apply to every production warehouse |
| Notify at thresholds | Send alert at 50 percent, 75 percent, 90 percent usage | Gives team time to adjust before hitting limit |
| Action at limit | Suspend warehouse or block queries | Suspend for dev, block for prod with on call alert |
| Frequency | Monthly or weekly reset | Match to your billing cycle or sprint cadence |

```mermaid
flowchart LR
  A[Create resource monitor] --> B[Set credit quota]
  B --> C[Add notification thresholds]
  C --> D[Define action at limit]
  D --> E[Attach to warehouse]
  E --> F[Monitor alerts and adjust]
```

## Query Tagging for Cost Attribution

| Tag Field | Example Value | Why Tag It |
|-----------|---------------|------------|
| team | analytics, finance, engineering | Track cost by owning team |
| project | dashboard_refresh, ml_training, etl_v2 | Allocate spend to initiative |
| environment | dev, test, prod | Separate cost buckets for each stage |
| owner | jane_doe, etl_service | Know who to contact for optimization |
| cost_center | CC_12345 | Map to finance system for chargeback |

```mermaid
sequenceDiagram
  participant U as User or App
  participant W as Warehouse
  participant S as Snowflake Metering
  participant R as Reporting View
  
  U->>W: Submit query with tag project=etl_v2
  W->>S: Execute and record credits with tag
  S->>R: Write to METERING_HISTORY with tag
  R->>U: Query view to see cost by project
```

## Naming Conventions That Scale

| Pattern | Example | Benefit |
|---------|---------|---------|
| Purpose first | REPORTING_WH, ETL_WH, DEV_WH | Clear what the warehouse is for |
| Environment suffix | _DEV, _TEST, _PROD | Prevent accidental cross environment use |
| Team prefix | ANALYTICS_REPORTING_WH, FINANCE_ETL_WH | Quick ownership identification |
| Avoid size in name | Do not use MEDIUM_WH | Size changes, name should not |
| Lowercase with underscores | reporting_prod_wh | Consistent, easy to script and search |

```mermaid
flowchart LR
  A[New warehouse request] --> B[Follow naming pattern]
  B --> C[Document purpose and owner]
  C --> D[Add to central inventory]
  D --> E[Review quarterly for cleanup]
```

## Workload Separation Strategy

| Workload Type | Dedicated Warehouse | Why Separate |
|---------------|---------------------|--------------|
| Development | DEV_WH (XSmall, 60s suspend) | Prevent dev queries from blocking prod |
| Testing | TEST_WH (Small, 60s suspend) | Isolate test data loads from reporting |
| Production reporting | REPORTING_WH (Medium, multi cluster) | Guarantee performance for business users |
| ETL and transforms | ETL_WH (Large, scheduled) | Heavy jobs do not compete with interactive queries |
| Data science | DS_WH (Medium with acceleration) | Long exploratory queries get needed resources |
| Executive dashboards | EXEC_WH (Medium, Standard policy) | Priority handling for leadership facing tools |

```mermaid
graph TD
  Users[User Groups] --> Dev[Developers]
  Users --> Analysts[Analysts]
  Users --> Execs[Executives]
  Users --> ETL[ETL Service]
  
  Dev --> WH1[DEV_WH]
  Analysts --> WH2[REPORTING_WH]
  Execs --> WH3[EXEC_WH]
  ETL --> WH4[ETL_WH]
  
  WH1 --> Cost1[Low cost, flexible]
  WH2 --> Cost2[Balanced cost and speed]
  WH3 --> Cost3[Priority performance]
  WH4 --> Cost4[High throughput, scheduled]
```

## Statement Timeout Configuration

| Timeout Value | Best For | Risk If Too Short | Risk If Too Long |
|---------------|----------|-------------------|------------------|
| 60 seconds | Simple lookups, API backed queries | Legitimate queries get cancelled | Minimal, but may hide inefficient queries |
| 300 seconds default | General reporting and transforms | Complex joins may fail mid execution | Runaway queries waste more credits |
| 1800 seconds | Large batch transforms, data science | Rarely needed, may indicate poor query design | Very high cost if query loops or scans unnecessarily |
| No timeout | Only for controlled maintenance windows | Not recommended for regular use | Unlimited credit exposure |

```mermaid
flowchart TD
  Q[Query starts] --> T[Check timeout setting]
  T --> R[Run query]
  R --> C{Completes before timeout}
  C -->|Yes| S[Return results]
  C -->|No| X[Cancel query, return error]
  X --> L[Log for review and optimization]
```

## Access Control and Role Assignment

| Practice | How To Implement | Why It Matters |
|----------|-----------------|----------------|
| Grant warehouse usage to roles, not users | GRANT USAGE ON WAREHOUSE x TO ROLE y | Easier to audit and update when team changes |
| Use separate roles for different warehouses | REPORTING_ROLE, ETL_ROLE, DEV_ROLE | Prevent accidental use of wrong warehouse |
| Limit OPERATE privilege to admins | Only grant to platform team | Prevent users from resizing or suspending warehouses |
| Review warehouse grants quarterly | Query SNOWFLAKE.ACCOUNT_USAGE.GRANTS | Catch stale permissions before they cause issues |
| Document who can use each warehouse | Central wiki or config repo | New team members know where to run their work |

```mermaid
graph LR
  Role[Role Based Access] --> Grant[Grant USAGE to functional role]
  Grant --> Assign[Assign role to user or service]
  Assign --> Audit[Review grants quarterly]
  Audit --> Update[Add or remove as team changes]
```

## Warehouse Scheduling for Batch Jobs

| Scenario | Scheduling Approach | Tool Options |
|----------|--------------------|--------------|
| Nightly ETL | Start warehouse 1 hour before job, stop after | Snowflake Tasks, Airflow, cron |
| Weekly report refresh | Resume warehouse 30 minutes before, suspend after | Snowflake Tasks with dependency chain |
| Month end close | Keep warehouse running for 48 hour window | Manual start stop with calendar reminder |
| Ad hoc maintenance | On demand start by admin only | Snowsight or CLI with approval workflow |

```mermaid
sequenceDiagram
  participant Sched as Scheduler
  participant WH as Warehouse
  participant Job as ETL Job
  participant Mon as Monitor
  
  Sched->>WH: Resume warehouse at 2 AM
  WH->>Job: Signal ready
  Job->>WH: Run transform queries
  Job->>Mon: Report completion
  Sched->>WH: Suspend warehouse at 4 AM
```

## Monitoring and Alerting Setup

| Metric | Where To Query | Alert Threshold | Action |
|--------|---------------|-----------------|--------|
| Credits per hour | WAREHOUSE_METERING_HISTORY | 2x baseline for 2 hours | Investigate workload change or runaway query |
| Queue time > 30 seconds | QUERY_HISTORY with QUEUED_PROVISIONING_TIME | Sustained during business hours | Scale out or adjust policy |
| Warehouse start failures | WAREHOUSE_EVENTS_HISTORY | Any failure in prod | Alert on call engineer |
| Idle time > auto suspend | WAREHOUSE_LOAD_HISTORY | Frequent short idle periods | Consider lowering auto suspend |
| Query timeout errors | QUERY_HISTORY with ERROR_MESSAGE | Spike in cancellations | Review timeout setting or query patterns |

```mermaid
flowchart LR
  M[Collect metrics] --> A[Analyze trends]
  A --> T[Set thresholds]
  T --> N[Configure notifications]
  N --> R[Review and adjust monthly]
```

## Cost Attribution and Chargeback

| Step | Action | Output |
|------|--------|--------|
| 1 | Tag every query with team and project | Consistent metadata in metering views |
| 2 | Aggregate credits by tag weekly | Cost breakdown by team or initiative |
| 3 | Share report with stakeholders | Visibility into spend drivers |
| 4 | Set budgets per team via resource monitors | Guardrails before overspend |
| 5 | Review and adjust allocations quarterly | Align spend with business priorities |

```mermaid
graph TD
  Tags[Query Tags] --> Meter[Metering Data]
  Meter --> Agg[Aggregate by Team or Project]
  Agg --> Report[Weekly Cost Report]
  Report --> Budget[Budget Planning]
  Budget --> Optimize[Optimization Actions]
```

## Testing and Validation Before Production Changes

| Change Type | Test Approach | Rollback Plan |
|-------------|---------------|---------------|
| Resize warehouse | Run sample queries on new size, compare runtime and credits | Revert to previous size if no improvement |
| Add multi cluster | Enable with min 1 max 2, monitor queue times for 1 week | Reduce max back to 1 if cost not justified |
| Change scaling policy | Switch one non critical warehouse first, measure user impact | Revert to previous policy if wait times increase |
| Update auto suspend | Lower by 60 seconds, monitor for start delay complaints | Increase back if users report friction |
| New query tag | Add to dev warehouse first, verify tags appear in metering views | Remove tag if not populating correctly |

```mermaid
flowchart TD
  C[Propose change] --> D[Test in dev or test warehouse]
  D --> M[Measure impact for 3-7 days]
  M --> R[Review with stakeholders]
  R --> A[Approve and apply to prod]
  A --> Mo[Monitor for 1 week post change]
```

## Quick Reference Checklist

```mermaid
graph TD
  Start[New or Existing Warehouse] --> Check1[Named by purpose, not size]
  Start --> Check2[Auto suspend set, auto resume true]
  Start --> Check3[Resource monitor attached]
  Start --> Check4[Query tagging enabled]
  Start --> Check5[Access granted to roles, not users]
  Start --> Check6[Statement timeout configured]
  Start --> Check7[Documented in central inventory]
  
  Check1 --> Review[Quarterly review scheduled]
  Check2 --> Review
  Check3 --> Review
  Check4 --> Review
  Check5 --> Review
  Check6 --> Review
  Check7 --> Review
```

## Common Anti Patterns to Avoid

| Anti Pattern | What Happens | Better Approach |
|--------------|--------------|-----------------|
| One warehouse for everything | Dev queries block prod reports, cost attribution impossible | Separate warehouses by workload type |
| Setting size based on guess | Over provision wastes credits, under provision frustrates users | Test with real queries before finalizing |
| Ignoring query tags | Cannot tell which team or project drove spend | Tag every query at submission |
| Never reviewing settings | Workloads change but config stays static | Schedule quarterly warehouse reviews |
| Granting OPERATE to all users | Anyone can resize or suspend, causing outages | Limit to platform or admin roles |
| Disabling auto suspend to avoid start delay | Warehouse bills 24/7 even when idle | Use 300 second suspend for user facing, 60 for others |

## Bottom Line

- Separate workloads into separate warehouses. One size and policy cannot fit all patterns
- Tag everything. Without tags, you cannot attribute cost or optimize effectively
- Start small and measure. Increase size or clusters only when data shows you need to
- Automate guardrails. Resource monitors and timeouts prevent surprises before they happen
- Document decisions. Future you and your teammates need context for why settings exist
- Review regularly. Workloads evolve, and your warehouse config should evolve with them
- Control access by role. Make it easy to grant and revoke access as teams change
- Test changes in non prod first. Warehouse changes can impact cost and performance immediately

Think of warehouse management like managing a fleet of vehicles:
- Name each vehicle by its job, not its engine size
- Turn off engines when parked to save fuel
- Assign drivers to vehicles by role, not by name
- Track fuel use by trip purpose to spot waste
- Service vehicles on a schedule before they break down
- Keep a log of changes so you know what was done and why

Apply these habits consistently. Small improvements compound. Cost savings add up. Performance stays predictable. Teams stay unblocked.
