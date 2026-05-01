# Domain 2.3: Monitoring and Cost Management in Snowflake

```mermaid
graph TD
  MonCost[Monitoring and Cost Management] --> Monitor[Usage Monitoring]
  MonCost --> Cost[Cost Management]
  MonCost --> Alert[Alerting and Notifications]
  MonCost --> Attribute[Usage Attribution]
  MonCost --> Optimize[Optimization Strategies]
  
  Monitor --> AcctUsage[ACCOUNT_USAGE Views]
  Monitor --> InfoSchema[INFORMATION_SCHEMA]
  Monitor --> QueryHist[QUERY_HISTORY]
  Monitor --> WhMetrics[WAREHOUSE_METERING_HISTORY]
  Monitor --> StorageMet[TABLE_STORAGE_METRICS]
  
  Cost --> ResMon[Resource Monitors]
  Cost --> WhSizing[Warehouse Sizing]
  Cost --> AutoSuspend[Auto-Suspend Configuration]
  Cost --> CreditTrack[Credit Tracking]
  
  Alert --> AlertObj[ALERT Object]
  Alert --> Task[Task-Based Notifications]
  Alert --> ExtInt[External Integrations]
  
  Attribute --> Tags[Query Tagging]
  Attribute --> Roles[Role-Based Attribution]
  Attribute --> Chargeback[Chargeback Reporting]
  
  Optimize --> WhRightSize[Warehouse Right-Sizing]
  Optimize --> QueryTune[Query Optimization]
  Optimize --> StorageTier[Storage Tiering]
  Optimize --> IdleRed[Idle Resource Reduction]
```

## Core Monitoring and Cost Principles

| Principle | What It Means | Why It Matters |
|-----------|--------------|----------------|
| Measure before you optimize | You cannot improve what you do not measure | Data-driven decisions prevent wasted effort and credits |
| Attribute to enable accountability | Tag queries and assign costs to teams/projects | Enables chargeback, budgeting, and optimization ownership |
| Alert on anomalies, not absolutes | Baseline normal behavior; alert on deviations | Reduces alert fatigue while catching real issues |
| Automate cost controls | Resource monitors and auto-suspend prevent runaway spend | Manual oversight does not scale; automation enforces guardrails |
| Review regularly | Usage patterns and costs evolve; monitoring should too | Quarterly reviews catch drift before it becomes budget impact |
| Optimize holistically | Compute, storage, and data transfer all contribute to cost | Focusing on one area while ignoring others yields suboptimal results |

```mermaid
flowchart LR
  Q1[Start: Implement monitoring and cost management]
  Q1 --> Q2[Enable ACCOUNT_USAGE views and query tagging]
  Q2 --> Q3[Configure resource monitors and auto-suspend]
  Q3 --> Q4[Set up alerts for cost and usage anomalies]
  Q4 --> Q5[Attribute costs to teams via tags and roles]
  Q5 --> Q6[Review and optimize quarterly]
  Q6 --> Q7[Automate reporting and alerting]
```

---

## Usage Monitoring: ACCOUNT_USAGE and INFORMATION_SCHEMA

### ACCOUNT_USAGE Views Overview

| View | What It Tracks | Retention | Key Use Cases |
|------|---------------|-----------|--------------|
| `QUERY_HISTORY` | All executed queries with details | 365 days | Performance tuning, cost attribution, debugging |
| `WAREHOUSE_METERING_HISTORY` | Credit usage by warehouse | 365 days | Cost tracking, warehouse optimization |
| `TABLE_STORAGE_METRICS` | Storage usage by table over time | 365 days | Storage cost analysis, archival decisions |
| `LOGIN_HISTORY` | Authentication attempts and outcomes | 365 days | Security monitoring, access audits |
| `ACCESS_HISTORY` | Object-level access by users and roles | 365 days | Data governance, compliance audits |
| `CREDIT_USAGE` | Credit consumption by service type | 365 days | Budget tracking, cost allocation |
| `MATERIALIZED_VIEW_REFRESH_HISTORY` | MV refresh timing and credit usage | 365 days | MV optimization, cost analysis |
| `COPY_HISTORY` | Data load/unload operations | 365 days | ETL monitoring, data pipeline auditing |

```sql
-- Query expensive queries for optimization
SELECT
  query_id,
  user_name,
  warehouse_name,
  query_text,
  start_time,
  end_time,
  DATEDIFF(seconds, start_time, end_time) as execution_seconds,
  bytes_scanned,
  credits_used,
  query_tag
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time > DATEADD(day, -7, CURRENT_TIMESTAMP())
  AND credits_used > 10  -- Focus on expensive queries
ORDER BY credits_used DESC
LIMIT 100;

-- Track warehouse credit usage by day
SELECT
  DATE_TRUNC('day', start_time) as usage_date,
  warehouse_name,
  SUM(credits_used) as total_credits,
  SUM(credits_used_cloud_services) as cloud_services_credits,
  COUNT(DISTINCT query_id) as query_count
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time > DATEADD(month, -1, CURRENT_TIMESTAMP())
GROUP BY DATE_TRUNC('day', start_time), warehouse_name
ORDER BY usage_date DESC, total_credits DESC;

-- Monitor storage growth by table
SELECT
  table_schema,
  table_name,
  active_bytes / POWER(1024, 3) as active_gb,
  time_travel_bytes / POWER(1024, 3) as time_travel_gb,
  failsafe_bytes / POWER(1024, 3) as failsafe_gb,
  (active_bytes + time_travel_bytes + failsafe_bytes) / POWER(1024, 3) as total_gb,
  TO_VARCHAR(last_updated, 'YYYY-MM-DD') as last_updated
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE deleted_on IS NULL
  AND table_schema NOT IN ('INFORMATION_SCHEMA', 'SNOWFLAKE')
ORDER BY total_gb DESC
LIMIT 50;
```

### INFORMATION_SCHEMA for Real-Time Metadata

| View | What It Provides | When To Use |
|------|-----------------|-------------|
| `WAREHOUSES` | Current warehouse state, size, status | Real-time warehouse monitoring |
| `TABLES` | Table metadata including row count, size | Schema inventory, optimization planning |
| `COLUMNS` | Column definitions and data types | Impact analysis, data discovery |
| `VIEWS` | View definitions and dependencies | Lineage tracking, change management |
| `PROCEDURES` | Stored procedure metadata | ETL pipeline monitoring |
| `TASKS` | Task definitions and last run status | Pipeline health monitoring |

```sql
-- Check current warehouse status and load
SELECT
  name as warehouse_name,
  state,
  size,
  type,
  auto_suspend,
  auto_resume,
  min_cluster_count,
  max_cluster_count,
  scaling_policy,
  currently_executing_queries,
  queued_queries
FROM INFORMATION_SCHEMA.WAREHOUSES
WHERE state != 'SUSPENDED'
ORDER BY currently_executing_queries DESC;

-- Find large tables for optimization consideration
SELECT
  table_schema,
  table_name,
  row_count,
  bytes / POWER(1024, 3) as size_gb,
  created,
  last_ddl
FROM INFORMATION_SCHEMA.TABLES
WHERE table_schema NOT IN ('INFORMATION_SCHEMA', 'SNOWFLAKE')
  AND bytes > 10737418240  -- Tables larger than 10 GB
ORDER BY bytes DESC;
```

### Latency Considerations for ACCOUNT_USAGE

| View | Typical Latency | Impact on Monitoring |
|------|----------------|---------------------|
| `QUERY_HISTORY` | ~45 minutes | Not suitable for real-time alerting; use for batch analysis |
| `WAREHOUSE_METERING_HISTORY` | ~45 minutes | Cost reports should account for delay |
| `ACCESS_HISTORY` | ~45 minutes | Compliance audits should use historical windows |
| `LOGIN_HISTORY` | ~45 minutes | Security monitoring may need supplemental real-time logging |
| `CREDIT_USAGE` | ~45 minutes | Budget alerts should have buffer for latency |

```sql
-- Account for latency in real-time monitoring queries
-- Use a buffer window to ensure data is available
SELECT
  warehouse_name,
  SUM(credits_used) as credits_last_hour
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time BETWEEN 
  DATEADD(hour, -2, CURRENT_TIMESTAMP())  -- Buffer for latency
  AND DATEADD(hour, -1, CURRENT_TIMESTAMP())
GROUP BY warehouse_name;
```

---

## Cost Management: Resource Monitors and Warehouse Configuration

### Resource Monitors: Credit Guardrails

```sql
-- Create resource monitor for production warehouse
CREATE OR REPLACE RESOURCE MONITOR prod_wh_monitor
  WITH CREDIT_QUOTA = 1000  -- Monthly credit budget
  FREQUENCY = MONTHLY
  START_TIMESTAMP = IMMEDIATELY
  TRIGGERS
    ON 50 PERCENT DO NOTIFY  -- Alert at 50% usage
    ON 75 PERCENT DO NOTIFY  -- Alert at 75% usage
    ON 90 PERCENT DO NOTIFY  -- Alert at 90% usage
    ON 100 PERCENT DO SUSPEND;  -- Suspend warehouse at 100%

-- Attach monitor to warehouse
ALTER WAREHOUSE reporting_wh SET RESOURCE_MONITOR = prod_wh_monitor;

-- Create separate monitor for development (more lenient)
CREATE OR REPLACE RESOURCE MONITOR dev_wh_monitor
  WITH CREDIT_QUOTA = 200
  FREQUENCY = MONTHLY
  START_TIMESTAMP = IMMEDIATELY
  TRIGGERS
    ON 80 PERCENT DO NOTIFY
    ON 100 PERCENT DO NOTIFY;  -- Notify but do not suspend dev work

ALTER WAREHOUSE dev_wh SET RESOURCE_MONITOR = dev_wh_monitor;
```

| Resource Monitor Setting | Recommended Value | Why |
|-------------------------|------------------|-----|
| Credit quota | Based on historical usage + 20% buffer | Prevents surprise overages while allowing growth |
| Frequency | MONTHLY for budgets; WEEKLY for sprints | Aligns with financial or development cycles |
| Notify at 50% | Enabled | Early warning for proactive adjustment |
| Notify at 75% | Enabled | Time to optimize or request budget increase |
| Notify at 90% | Enabled | Final warning before hard limit |
| Action at 100% | SUSPEND for prod; NOTIFY for dev | Protects budget while allowing dev flexibility |

### Warehouse Sizing and Auto-Suspend Optimization

```sql
-- Right-size warehouse based on historical usage
SELECT
  warehouse_name,
  AVG(credits_used) as avg_credits_per_hour,
  MAX(credits_used) as peak_credits_per_hour,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY credits_used) as p95_credits,
  COUNT(DISTINCT query_id) as query_count
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time > DATEADD(month, -1, CURRENT_TIMESTAMP())
GROUP BY warehouse_name
ORDER BY avg_credits_per_hour DESC;

-- Configure auto-suspend based on usage patterns
-- For interactive workloads: 300 seconds (5 minutes)
ALTER WAREHOUSE reporting_wh SET AUTO_SUSPEND = 300;

-- For batch/ETL workloads: 60 seconds (1 minute)
ALTER WAREHOUSE etl_wh SET AUTO_SUSPEND = 60;

-- For always-on API workloads: disable auto-suspend
ALTER WAREHOUSE api_wh SET AUTO_SUSPEND = 0;

-- Enable auto-resume for all interactive warehouses
ALTER WAREHOUSE reporting_wh SET AUTO_RESUME = TRUE;
ALTER WAREHOUSE dev_wh SET AUTO_RESUME = TRUE;
```

| Warehouse Type | Recommended Size | Auto-Suspend | Auto-Resume | Scaling Policy |
|---------------|-----------------|--------------|-------------|---------------|
| Interactive analytics | Medium | 300 seconds | TRUE | Standard (for user experience) |
| ETL/Batch processing | Large or XLarge | 60 seconds | TRUE | Economy (cost-sensitive) |
| Development/Testing | Small or XSmall | 60 seconds | TRUE | Economy |
| API/Service backend | Small | 0 (disabled) | TRUE | Economy |
| Executive dashboards | Medium | 300 seconds | TRUE | Standard (zero wait time) |

### Credit Tracking and Attribution

```sql
-- Track credit usage by query tag for project attribution
SELECT
  query_tag,
  SUM(credits_used) as total_credits,
  COUNT(DISTINCT query_id) as query_count,
  AVG(credits_used) as avg_credits_per_query,
  DATE_TRUNC('day', MIN(start_time)) as first_use,
  DATE_TRUNC('day', MAX(start_time)) as last_use
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time > DATEADD(month, -1, CURRENT_TIMESTAMP())
  AND query_tag IS NOT NULL
  AND query_tag != ''
GROUP BY query_tag
ORDER BY total_credits DESC;

-- Attribute costs to roles for team chargeback
SELECT
  r.name as role_name,
  SUM(qh.credits_used) as total_credits,
  COUNT(DISTINCT qh.query_id) as query_count,
  SUM(qh.bytes_scanned) as total_bytes_scanned
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY qh
JOIN SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_USERS g
  ON qh.user_name = g.grantee_name
JOIN SNOWFLAKE.ACCOUNT_USAGE.ROLES r
  ON g.granted_role = r.name
WHERE qh.start_time > DATEADD(month, -1, CURRENT_TIMESTAMP())
  AND r.deleted_on IS NULL
GROUP BY r.name
ORDER BY total_credits DESC;

-- Export credit usage for external billing systems
COPY INTO @finance_exports/credit_usage/
FROM (
  SELECT
    DATE_TRUNC('day', start_time) as usage_date,
    warehouse_name,
    service_type,
    SUM(credits_used) as credits,
    SUM(credits_used) * 3.00 as estimated_cost_usd  -- Enterprise edition rate
  FROM SNOWFLAKE.ACCOUNT_USAGE.CREDIT_USAGE
  WHERE start_time > DATEADD(month, -1, CURRENT_TIMESTAMP())
  GROUP BY DATE_TRUNC('day', start_time), warehouse_name, service_type
)
FILE_FORMAT = (TYPE = CSV HEADER = TRUE);
```

---

## Alerting and Notifications for Cost and Usage

### ALERT Object for Proactive Monitoring

```sql
-- Alert: Warehouse credit usage spike
CREATE OR REPLACE ALERT cost.wh_credit_spike_alert
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON */30 * * * *'  -- Every 30 minutes
  CONDITION = (
    SELECT SUM(credits_used)
    FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
    WHERE warehouse_name = 'PROD_WH'
      AND start_time > DATEADD(hour, -1, CURRENT_TIMESTAMP())
  ) > (
    -- Alert if current hour exceeds 2x the 7-day average
    SELECT AVG(hourly_credits) * 2
    FROM (
      SELECT 
        DATE_TRUNC('hour', start_time) as hour_bucket,
        SUM(credits_used) as hourly_credits
      FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
      WHERE warehouse_name = 'PROD_WH'
        AND start_time > DATEADD(day, -7, CURRENT_TIMESTAMP())
      GROUP BY DATE_TRUNC('hour', start_time)
    )
  )
  ACTION = (
    SYSTEM$SEND_EMAIL(
      'finance-team@company.com',
      'Alert: Unusual credit usage on PROD_WH',
      'Current hour credits: ' || 
      (SELECT SUM(credits_used) FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY WHERE warehouse_name = ''PROD_WH'' AND start_time > DATEADD(hour, -1, CURRENT_TIMESTAMP())) ||
      '\n7-day average: ' ||
      (SELECT AVG(hourly_credits) * 2 FROM (SELECT DATE_TRUNC(''hour'', start_time) as hour_bucket, SUM(credits_used) as hourly_credits FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY WHERE warehouse_name = ''PROD_WH'' AND start_time > DATEADD(day, -7, CURRENT_TIMESTAMP()) GROUP BY DATE_TRUNC(''hour'', start_time))) ||
      '\n\nInvestigate immediately.'
    )
  );

-- Alert: Long-running queries consuming credits
CREATE OR REPLACE ALERT ops.long_query_alert
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON 0 */6 * * *'  -- Every 6 hours
  CONDITION = (
    SELECT COUNT(*)
    FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE DATEDIFF(seconds, start_time, end_time) > 1800  -- Queries > 30 minutes
      AND start_time > DATEADD(hour, -6, CURRENT_TIMESTAMP())
      AND warehouse_name = 'PROD_WH'
  ) > 0
  ACTION = (
    SYSTEM$SEND_SLACK_MESSAGE(
      'https://hooks.slack.com/services/XXX',
      '⚠️ Long-running queries detected on PROD_WH\n' ||
      (SELECT LISTAGG('Query ID: ' || query_id || ' (' || DATEDIFF(seconds, start_time, end_time) || 's)', '\n')
       FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
       WHERE DATEDIFF(seconds, start_time, end_time) > 1800
         AND start_time > DATEADD(hour, -6, CURRENT_TIMESTAMP())
         AND warehouse_name = 'PROD_WH')
    )
  );

-- Alert: Storage growth anomaly
CREATE OR REPLACE ALERT storage.anomalous_growth_alert
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON 0 9 * * 1'  -- Weekly on Monday
  CONDITION = (
    SELECT COUNT(*)
    FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
    WHERE active_bytes > 
      (SELECT AVG(active_bytes) * 1.5  -- 50% growth threshold
       FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
       WHERE table_name = tsm.table_name
         AND last_updated > DATEADD(month, -1, CURRENT_TIMESTAMP()))
      AND last_updated > DATEADD(day, -7, CURRENT_TIMESTAMP())
  ) > 0
  ACTION = (
    SYSTEM$SEND_EMAIL(
      'data-eng-team@company.com',
      'Alert: Unusual storage growth detected',
      'Review tables with >50% storage growth in the last week'
    )
  );
```

### Task-Based Monitoring and Notifications

```sql
-- Create task to monitor and report daily credit usage
CREATE OR REPLACE TASK governance.daily_cost_report
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON 0 8 * * *'  -- Daily at 8 AM
AS
  -- Generate and send daily cost summary
  SYSTEM$SEND_EMAIL(
    'finance-team@company.com',
    'Daily Snowflake Cost Report - ' || TO_VARCHAR(DATEADD(day, -1, CURRENT_DATE()), 'YYYY-MM-DD'),
    (SELECT 
      'Total Credits: ' || SUM(credits_used) || '\n' ||
      'Top Warehouse: ' || 
        (SELECT warehouse_name FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY 
         WHERE start_time > DATEADD(day, -1, CURRENT_TIMESTAMP())
         GROUP BY warehouse_name ORDER BY SUM(credits_used) DESC LIMIT 1) || '\n' ||
      'Top Query Tag: ' ||
        (SELECT query_tag FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
         WHERE start_time > DATEADD(day, -1, CURRENT_TIMESTAMP())
           AND query_tag IS NOT NULL
         GROUP BY query_tag ORDER BY SUM(credits_used) DESC LIMIT 1)
     FROM SNOWFLAKE.ACCOUNT_USAGE.CREDIT_USAGE
     WHERE start_time > DATEADD(day, -1, CURRENT_TIMESTAMP()))
  );

-- Create task to identify and alert on idle warehouses
CREATE OR REPLACE TASK ops.idle_warehouse_alert
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON 0 10 * * 1-5'  -- Weekdays at 10 AM
AS
  -- Find warehouses with low utilization
  INSERT INTO governance.idle_warehouse_log
  SELECT
    warehouse_name,
    state,
    auto_suspend,
    COUNT(DISTINCT query_id) as query_count_last_24h,
    SUM(credits_used) as credits_last_24h
  FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
  WHERE start_time > DATEADD(day, -1, CURRENT_TIMESTAMP())
  GROUP BY warehouse_name, state, auto_suspend
  HAVING query_count_last_24h < 10  -- Fewer than 10 queries in 24 hours
    AND credits_last_24h > 5;  -- But still consuming credits

  -- Send alert if idle warehouses found
  SYSTEM$SEND_EMAIL(
    'platform-team@company.com',
    'Alert: Idle warehouses consuming credits',
    (SELECT LISTAGG(warehouse_name || ': ' || credits_last_24h || ' credits, ' || query_count_last_24h || ' queries', '\n')
     FROM governance.idle_warehouse_log
     WHERE logged_date = CURRENT_DATE())
  );
```

### External Integrations for Notifications

```sql
-- Slack notification via webhook
CREATE OR REPLACE ALERT cost.slack_credit_alert
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON 0 */4 * * *'
  CONDITION = (
    SELECT SUM(credits_used)
    FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
    WHERE start_time > DATEADD(hour, -4, CURRENT_TIMESTAMP())
  ) > 50  -- Alert if > 50 credits in 4 hours
  ACTION = (
    SYSTEM$SEND_SLACK_MESSAGE(
      'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX',
      '💰 *Snowflake Cost Alert*\n' ||
      '• Period: Last 4 hours\n' ||
      '• Credits Used: ' || 
        (SELECT SUM(credits_used) FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY WHERE start_time > DATEADD(hour, -4, CURRENT_TIMESTAMP())) || '\n' ||
      '• Threshold: 50 credits\n' ||
      '• Action: Review WAREHOUSE_METERING_HISTORY'
    )
  );

-- PagerDuty integration for critical cost alerts
CREATE OR REPLACE ALERT cost.critical_budget_alert
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON */15 * * * *'
  CONDITION = (
    SELECT SUM(credits_used)
    FROM SNOWFLAKE.ACCOUNT_USAGE.CREDIT_USAGE
    WHERE start_time > DATEADD(day, -1, CURRENT_TIMESTAMP())
  ) > (
    -- Alert if daily usage exceeds 80% of monthly budget / 30
    SELECT (credit_quota * 0.8) / 30
    FROM SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS
    WHERE name = 'prod_budget_monitor'
  )
  ACTION = (
    SYSTEM$HTTP_POST(
      'https://events.pagerduty.com/v2/enqueue',
      OBJECT_CONSTRUCT(
        'routing_key', 'your_pagerduty_integration_key',
        'event_action', 'trigger',
        'payload', OBJECT_CONSTRUCT(
          'summary', 'Critical: Snowflake budget threshold exceeded',
          'source', 'snowflake',
          'severity', 'critical',
          'custom_details', OBJECT_CONSTRUCT(
            'daily_credits', (SELECT SUM(credits_used) FROM SNOWFLAKE.ACCOUNT_USAGE.CREDIT_USAGE WHERE start_time > DATEADD(day, -1, CURRENT_TIMESTAMP())),
            'threshold', (SELECT (credit_quota * 0.8) / 30 FROM SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS WHERE name = 'prod_budget_monitor'),
            'alert_time', CURRENT_TIMESTAMP()
          )
        )
      )::STRING,
      OBJECT_CONSTRUCT('Content-Type', 'application/json')
    )
  );
```

---

## Usage Attribution and Chargeback

### Query Tagging for Cost Attribution

```sql
-- Set query tag at session level for automatic attribution
ALTER SESSION SET QUERY_TAG = 'project=sales_dashboard,team=analytics,env=prod';

-- Or set at role level for all users with that role
ALTER ROLE ANALYST_ROLE SET QUERY_TAG = 'team=analytics,access_level=standard';

-- Query credit usage by tag for chargeback reporting
SELECT
  SPLIT_PART(query_tag, ',', 1) as project,
  SPLIT_PART(query_tag, ',', 2) as team,
  SPLIT_PART(query_tag, ',', 3) as environment,
  SUM(credits_used) as total_credits,
  COUNT(DISTINCT query_id) as query_count,
  DATE_TRUNC('day', MIN(start_time)) as first_use,
  DATE_TRUNC('day', MAX(start_time)) as last_use
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time > DATEADD(month, -1, CURRENT_TIMESTAMP())
  AND query_tag IS NOT NULL
  AND query_tag != ''
GROUP BY 
  SPLIT_PART(query_tag, ',', 1),
  SPLIT_PART(query_tag, ',', 2),
  SPLIT_PART(query_tag, ',', 3)
ORDER BY total_credits DESC;

-- Export attribution data for external billing systems
COPY INTO @finance_exports/chargeback/
FROM (
  SELECT
    DATE_TRUNC('month', start_time) as billing_month,
    SPLIT_PART(query_tag, ',', 1) as project,
    SPLIT_PART(query_tag, ',', 2) as team,
    SUM(credits_used) as credits,
    SUM(credits_used) * 3.00 as estimated_cost_usd,
    COUNT(DISTINCT query_id) as query_count
  FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE start_time > DATEADD(month, -2, CURRENT_TIMESTAMP())
    AND query_tag IS NOT NULL
  GROUP BY 
    DATE_TRUNC('month', start_time),
    SPLIT_PART(query_tag, ',', 1),
    SPLIT_PART(query_tag, ',', 2)
)
FILE_FORMAT = (TYPE = CSV HEADER = TRUE);
```

### Role-Based Cost Attribution

```sql
-- Attribute costs to roles for team-level chargeback
CREATE OR REPLACE VIEW governance.role_cost_attribution AS
SELECT
  DATE_TRUNC('month', qh.start_time) as billing_month,
  r.name as role_name,
  SUM(qh.credits_used) as total_credits,
  COUNT(DISTINCT qh.query_id) as query_count,
  SUM(qh.bytes_scanned) as total_bytes_scanned,
  SUM(qh.credits_used) * 3.00 as estimated_cost_usd  -- Enterprise rate
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY qh
JOIN SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_USERS g
  ON qh.user_name = g.grantee_name
JOIN SNOWFLAKE.ACCOUNT_USAGE.ROLES r
  ON g.granted_role = r.name
WHERE qh.start_time > DATEADD(month, -3, CURRENT_TIMESTAMP())
  AND r.deleted_on IS NULL
GROUP BY DATE_TRUNC('month', qh.start_time), r.name;

-- Generate monthly chargeback report
SELECT
  billing_month,
  role_name,
  total_credits,
  estimated_cost_usd,
  ROUND(100.0 * total_credits / SUM(total_credits) OVER (PARTITION BY billing_month), 2) as pct_of_total
FROM governance.role_cost_attribution
WHERE billing_month = DATE_TRUNC('month', CURRENT_DATE())
ORDER BY estimated_cost_usd DESC;
```

### Storage Cost Attribution by Tag

```sql
-- Attribute storage costs using table tags
CREATE OR REPLACE VIEW governance.storage_cost_by_tag AS
SELECT
  tr.tag_value as cost_center,
  SUM(tsm.active_bytes) / POWER(1024, 3) as active_gb,
  SUM(tsm.time_travel_bytes) / POWER(1024, 3) as time_travel_gb,
  SUM(tsm.failsafe_bytes) / POWER(1024, 3) as failsafe_gb,
  (SUM(tsm.active_bytes) + SUM(tsm.time_travel_bytes) + SUM(tsm.failsafe_bytes)) / POWER(1024, 3) as total_gb,
  (SUM(tsm.active_bytes) + SUM(tsm.time_travel_bytes) + SUM(tsm.failsafe_bytes)) / POWER(1024, 3) * 23.00 / 1024 as estimated_monthly_cost_usd  -- $23/TB/month
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS tsm
LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES tr
  ON tsm.table_name = tr.object_name
  AND tr.tag_name = 'cost_center'
  AND tr.column_name IS NULL  -- Table-level tag
WHERE tsm.deleted_on IS NULL
  AND tsm.table_schema NOT IN ('INFORMATION_SCHEMA', 'SNOWFLAKE')
GROUP BY tr.tag_value;

-- Export storage attribution for finance
COPY INTO @finance_exports/storage_chargeback/
FROM governance.storage_cost_by_tag
FILE_FORMAT = (TYPE = CSV HEADER = TRUE);
```

---

## Optimization Strategies for Cost Reduction

### Warehouse Right-Sizing

```sql
-- Analyze warehouse utilization to identify right-sizing opportunities
CREATE OR REPLACE VIEW governance.warehouse_utilization_analysis AS
SELECT
  wmh.warehouse_name,
  w.size as current_size,
  AVG(wmh.credits_used) as avg_credits_per_hour,
  MAX(wmh.credits_used) as peak_credits_per_hour,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY wmh.credits_used) as p95_credits,
  COUNT(DISTINCT qh.query_id) as query_count_30d,
  AVG(qh.execution_time) as avg_query_duration_seconds,
  CASE
    WHEN AVG(wmh.credits_used) < 1 THEN 'Consider XSmall'
    WHEN AVG(wmh.credits_used) < 2 THEN 'Consider Small'
    WHEN AVG(wmh.credits_used) < 4 THEN 'Consider Medium'
    WHEN AVG(wmh.credits_used) < 8 THEN 'Current size may be appropriate'
    ELSE 'Consider scaling up or optimizing queries'
  END as sizing_recommendation
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY wmh
JOIN INFORMATION_SCHEMA.WAREHOUSES w ON wmh.warehouse_name = w.name
LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY qh
  ON wmh.warehouse_name = qh.warehouse_name
  AND qh.start_time > DATEADD(month, -1, CURRENT_TIMESTAMP())
WHERE wmh.start_time > DATEADD(month, -1, CURRENT_TIMESTAMP())
GROUP BY wmh.warehouse_name, w.size;

-- Generate right-sizing recommendations
SELECT
  warehouse_name,
  current_size,
  avg_credits_per_hour,
  sizing_recommendation,
  -- Estimated monthly savings if downsized
  CASE
    WHEN sizing_recommendation LIKE '%XSmall%' THEN (avg_credits_per_hour - 1) * 720 * 3.00
    WHEN sizing_recommendation LIKE '%Small%' THEN (avg_credits_per_hour - 2) * 720 * 3.00
    WHEN sizing_recommendation LIKE '%Medium%' THEN (avg_credits_per_hour - 4) * 720 * 3.00
    ELSE 0
  END as estimated_monthly_savings_usd
FROM governance.warehouse_utilization_analysis
WHERE sizing_recommendation != 'Current size may be appropriate'
ORDER BY estimated_monthly_savings_usd DESC;
```

### Query Optimization for Cost Reduction

```sql
-- Identify queries with high bytes scanned for optimization
CREATE OR REPLACE VIEW governance.expensive_queries AS
SELECT
  query_id,
  user_name,
  warehouse_name,
  query_text,
  start_time,
  execution_time,
  bytes_scanned,
  credits_used,
  query_tag,
  -- Flag queries that scan > 100 GB
  CASE WHEN bytes_scanned > 100000000000 THEN 'HIGH_SCAN' ELSE 'NORMAL' END as scan_category
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time > DATEADD(day, -7, CURRENT_TIMESTAMP())
  AND bytes_scanned > 10000000000  -- > 10 GB
ORDER BY bytes_scanned DESC;

-- Find queries that could benefit from clustering
SELECT
  qh.query_id,
  qh.query_text,
  qh.bytes_scanned,
  t.cluster_by,
  -- Suggest clustering if query filters on non-clustered column
  CASE
    WHEN t.cluster_by IS NULL AND qh.query_text ILIKE '%WHERE%' THEN 'Consider clustering on filter columns'
    WHEN t.cluster_by IS NOT NULL AND qh.query_text NOT ILIKE '%' || t.cluster_by || '%' THEN 'Filter columns may not align with clustering'
    ELSE 'Clustering may be appropriate'
  END as clustering_recommendation
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY qh
JOIN INFORMATION_SCHEMA.TABLES t
  ON qh.query_text ILIKE '%FROM ' || t.table_name || '%'
  OR qh.query_text ILIKE '%JOIN ' || t.table_name || '%'
WHERE qh.start_time > DATEADD(day, -7, CURRENT_TIMESTAMP())
  AND qh.bytes_scanned > 50000000000  -- > 50 GB
  AND t.table_schema NOT IN ('INFORMATION_SCHEMA', 'SNOWFLAKE');
```

### Storage Optimization: Time Travel and Archival

```sql
-- Identify tables with high Time Travel costs for optimization
CREATE OR REPLACE VIEW governance.time_travel_cost_analysis AS
SELECT
  table_schema,
  table_name,
  active_bytes / POWER(1024, 3) as active_gb,
  time_travel_bytes / POWER(1024, 3) as time_travel_gb,
  failsafe_bytes / POWER(1024, 3) as failsafe_gb,
  ROUND(100.0 * time_travel_bytes / NULLIF(active_bytes, 0), 2) as time_travel_pct,
  -- Estimate monthly cost savings if Time Travel reduced
  time_travel_bytes / POWER(1024, 3) * 23.00 / 1024 * 0.9 as potential_monthly_savings_usd,  -- Assuming 90% reduction
  -- Recommend retention based on usage
  CASE
    WHEN last_updated < DATEADD(month, -6, CURRENT_TIMESTAMP()) THEN 'Consider reducing to 1 day'
    WHEN time_travel_pct > 50 THEN 'Review if 90-day retention is needed'
    ELSE 'Current retention may be appropriate'
  END as retention_recommendation
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE deleted_on IS NULL
  AND table_schema NOT IN ('INFORMATION_SCHEMA', 'SNOWFLAKE')
  AND time_travel_bytes > 0
ORDER BY time_travel_bytes DESC;

-- Archive cold data to external storage
-- Step 1: Identify tables with low access frequency
SELECT
  tsm.table_schema,
  tsm.table_name,
  tsm.active_bytes / POWER(1024, 3) as size_gb,
  MAX(ah.start_time) as last_accessed,
  DATEDIFF(day, MAX(ah.start_time), CURRENT_DATE()) as days_since_access
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS tsm
LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY ah
  ON tsm.table_name = ah.object_name
WHERE tsm.deleted_on IS NULL
  AND tsm.table_schema NOT IN ('INFORMATION_SCHEMA', 'SNOWFLAKE')
GROUP BY tsm.table_schema, tsm.table_name, tsm.active_bytes
HAVING MAX(ah.start_time) < DATEADD(month, -3, CURRENT_TIMESTAMP())  -- No access in 3 months
ORDER BY size_gb DESC;

-- Step 2: Export to external stage then drop
-- COPY INTO @archive_stage/cold_data/table_name/ FROM schema.table_name;
-- DROP TABLE schema.table_name;
```

### Idle Resource Reduction

```sql
-- Identify warehouses with low utilization
CREATE OR REPLACE VIEW governance.idle_warehouse_analysis AS
SELECT
  wmh.warehouse_name,
  w.state,
  w.auto_suspend,
  w.size,
  COUNT(DISTINCT qh.query_id) as queries_last_7d,
  SUM(wmh.credits_used) as credits_last_7d,
  AVG(wmh.credits_used) as avg_credits_per_hour,
  -- Flag warehouses consuming credits but with few queries
  CASE
    WHEN COUNT(DISTINCT qh.query_id) < 10 AND SUM(wmh.credits_used) > 10 THEN 'IDLE_HIGH_COST'
    WHEN COUNT(DISTINCT qh.query_id) < 50 AND SUM(wmh.credits_used) > 20 THEN 'LOW_UTILIZATION'
    ELSE 'NORMAL'
  END as utilization_flag
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY wmh
JOIN INFORMATION_SCHEMA.WAREHOUSES w ON wmh.warehouse_name = w.name
LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY qh
  ON wmh.warehouse_name = qh.warehouse_name
  AND qh.start_time > DATEADD(day, -7, CURRENT_TIMESTAMP())
WHERE wmh.start_time > DATEADD(day, -7, CURRENT_TIMESTAMP())
GROUP BY wmh.warehouse_name, w.state, w.auto_suspend, w.size
ORDER BY credits_last_7d DESC;

-- Generate recommendations for idle warehouses
SELECT
  warehouse_name,
  state,
  auto_suspend,
  queries_last_7d,
  credits_last_7d,
  utilization_flag,
  CASE
    WHEN utilization_flag = 'IDLE_HIGH_COST' THEN 'Reduce auto-suspend to 60s or suspend manually'
    WHEN utilization_flag = 'LOW_UTILIZATION' THEN 'Consider downsizing or consolidating workloads'
    ELSE 'Monitor'
  END as recommendation,
  -- Estimated savings if optimized
  CASE
    WHEN utilization_flag = 'IDLE_HIGH_COST' THEN credits_last_7d * 0.8 * 3.00  -- 80% of current cost
    WHEN utilization_flag = 'LOW_UTILIZATION' THEN credits_last_7d * 0.3 * 3.00  -- 30% savings from downsizing
    ELSE 0
  END as estimated_weekly_savings_usd
FROM governance.idle_warehouse_analysis
WHERE utilization_flag IN ('IDLE_HIGH_COST', 'LOW_UTILIZATION')
ORDER BY estimated_weekly_savings_usd DESC;
```

---

## Best Practices for Monitoring and Cost Management

### Monitoring Best Practices

| Practice | Implementation | Benefit |
|----------|---------------|---------|
| Enable query tagging at session or role level | `ALTER SESSION SET QUERY_TAG = 'project=X,team=Y'` | Enables automatic cost attribution and filtering |
| Monitor ACCOUNT_USAGE with latency buffer | Query with `start_time > DATEADD(hour, -2, CURRENT_TIMESTAMP())` | Ensures data is available for analysis |
| Create dashboards for key metrics | Use BI tools or Streamlit to visualize credit usage, query performance | Enables proactive management and stakeholder visibility |
| Set up alerts for anomalies, not absolutes | Compare to baselines; alert on deviations | Reduces alert fatigue while catching real issues |
| Review monitoring queries quarterly | Ensure queries still reflect current usage patterns | Keeps monitoring relevant as workloads evolve |

### Cost Management Best Practices

| Practice | Implementation | Benefit |
|----------|---------------|---------|
| Configure resource monitors for all production warehouses | Set credit quotas and triggers | Prevents runaway spend; enables proactive budget management |
| Right-size warehouses based on actual usage | Analyze `WAREHOUSE_METERING_HISTORY`; adjust size | Reduces compute costs without impacting performance |
| Set appropriate auto-suspend values | 60s for batch, 300s for interactive, 0 for always-on | Eliminates idle billing while maintaining responsiveness |
| Attribute costs via tags and roles | Use `QUERY_TAG` and role-based attribution | Enables chargeback, budgeting, and optimization ownership |
| Archive cold data to external storage | `COPY INTO @external_stage` then `DROP TABLE` | Reduces Snowflake storage costs for rarely accessed data |
| Review Time Travel retention by environment | 1 day for dev/test, 7-90 days for prod based on need | Balances recovery capability with storage cost |

### Optimization Checklist

```mermaid
flowchart TD
  Start[Start: Optimize Snowflake costs] --> Compute[Review compute usage]
  Start --> Storage[Review storage usage]
  Start --> Transfer[Review data transfer]
  
  Compute --> WhSize[Right-size warehouses]
  Compute --> AutoSusp[Configure auto-suspend]
  Compute --> QueryOpt[Optimize expensive queries]
  
  Storage --> TimeTravel[Adjust Time Travel retention]
  Storage --> Archive[Archive cold data externally]
  Storage --> Dedup[Remove duplicate or unused tables]
  
  Transfer --> Region[Keep compute and storage in same region]
  Transfer --> External[Minimize external data movement]
  
  WhSize --> Measure[Measure impact after changes]
  AutoSusp --> Measure
  QueryOpt --> Measure
  TimeTravel --> Measure
  Archive --> Measure
  Dedup --> Measure
  Region --> Measure
  External --> Measure
  
  Measure --> Review[Review quarterly; adjust as needed]
```

---

## Common Pitfalls and Mitigations

| Pitfall | What Happens | How To Fix |
|---------|--------------|------------|
| Not using query tags | Cannot attribute costs to projects or teams | Enforce tagging via role settings or governance policy |
| Ignoring ACCOUNT_USAGE latency | Real-time alerts fail because data is not yet available | Add buffer window to queries; use for batch analysis not real-time |
| Setting resource monitor quotas too low | Warehouses suspend unexpectedly, disrupting work | Base quotas on historical usage + 20% buffer; review quarterly |
| Over-provisioning warehouse size | Paying for unused compute capacity | Analyze `WAREHOUSE_METERING_HISTORY`; right-size based on p95 usage |
| Disabling auto-suspend for all warehouses | Paying for idle compute 24/7 | Set auto-suspend based on workload pattern; disable only for always-on APIs |
| Keeping 90-day Time Travel on dev tables | Paying for historical storage on data that changes daily | Set `DATA_RETENTION_TIME_IN_DAYS = 1` for non-production environments |
| Not monitoring storage growth | Surprise bills from uncontrolled data growth | Set up alerts on `TABLE_STORAGE_METRICS`; archive cold data proactively |
| Ignoring data transfer costs | Cross-region queries or external exports add up | Keep compute and storage in same region; minimize external data movement |

```mermaid
flowchart TD
  Prob[Cost or monitoring issue] --> Q1[Unexpected credit spike]
  Prob --> Q2[Cannot attribute costs]
  Prob --> Q3[Storage costs growing unexpectedly]
  
  Q1 --> A[Check WAREHOUSE_METERING_HISTORY for usage patterns]
  Q1 --> B[Review QUERY_HISTORY for expensive new queries]
  Q1 --> C[Verify resource monitor triggers and actions]
  
  Q2 --> D[Enforce query tagging at session or role level]
  Q2 --> E[Use role-based attribution as fallback]
  
  Q3 --> F[Query TABLE_STORAGE_METRICS for growth sources]
  Q3 --> G[Review Time Travel retention settings]
  Q3 --> H[Archive cold data to external storage]
  
  A --> I[Implement optimization; measure impact]
  B --> I
  C --> I
  D --> I
  E --> I
  F --> I
  G --> I
  H --> I
```

---

## Decision Framework: Monitoring and Cost Strategy

```mermaid
flowchart TD
  Q1[Start: Design monitoring and cost strategy]
  Q1 --> Q2[What is your primary goal]
  
  Q2 -->|Cost control| A[Configure resource monitors + auto-suspend]
  Q2 -->|Cost attribution| B[Enable query tagging + role-based attribution]
  Q2 -->|Performance optimization| C[Monitor QUERY_HISTORY + warehouse metrics]
  Q2 -->|Compliance auditing| D[Enable ACCESS_HISTORY + export to external system]
  
  A --> E[Set credit quotas based on historical usage + buffer]
  B --> F[Enforce tagging via role settings or governance policy]
  C --> G[Create dashboards for key performance metrics]
  D --> H[Schedule regular exports for audit evidence]
  
  E --> I[Review and adjust quarterly]
  F --> I
  G --> I
  H --> I
  
  I --> J[Automate reporting and alerting]
  J --> K[Document strategy for team alignment]
```

| Requirement | Recommended Approach | Validation Method |
|------------|---------------------|------------------|
| Prevent budget overruns | Resource monitors with SUSPEND at 100% | Test with simulated high-usage scenario |
| Attribute costs to teams | Query tagging + role-based attribution | Verify chargeback reports match expectations |
| Optimize query performance | Monitor `QUERY_HISTORY` for bytes scanned, execution time | Track improvement after optimization |
| Meet compliance audit requirements | Export `ACCESS_HISTORY` + `QUERY_HISTORY` to external system | Validate with internal audit team |
| Reduce storage costs | Adjust Time Travel retention + archive cold data | Measure storage cost reduction post-optimization |
| Minimize idle compute | Configure auto-suspend based on workload pattern | Monitor warehouse utilization post-change |

---

## Key Principles to Remember

- **Measure before you optimize.** You cannot improve what you do not measure. Start with baseline metrics.
- **Attribute to enable accountability.** Tag queries and assign costs to teams; ownership drives optimization.
- **Alert on anomalies, not absolutes.** Baseline normal behavior; alert on deviations to reduce noise.
- **Automate cost controls.** Resource monitors and auto-suspend enforce guardrails without manual intervention.
- **Review regularly.** Usage patterns and costs evolve; quarterly reviews catch drift before budget impact.
- **Optimize holistically.** Compute, storage, and data transfer all contribute to cost; address all three.
- **Document your strategy.** Future you and your team need context for monitoring and cost decisions.

## Bottom Line

- Monitoring in Snowflake is comprehensive but requires intentional configuration. Enable `ACCOUNT_USAGE` views, query tagging, and external exports for full visibility.
- Cost management is proactive, not reactive. Resource monitors, auto-suspend, and right-sizing prevent surprises before they happen.
- Attribution enables accountability. Tag queries, assign roles, and export data to enable chargeback and optimization ownership.
- Optimization is iterative. Measure baseline, implement changes, measure impact, and adjust. Repeat quarterly.
- Automation scales. Manual oversight does not. Use ALERT objects, Tasks, and external integrations to enforce guardrails at scale.
- Documentation enables continuity. Record your monitoring queries, cost policies, and optimization decisions for future reference.

Think of monitoring and cost management like managing a utility budget:
- **ACCOUNT_USAGE views are your meter readings.** They tell you what you used, when, and how much it cost.
- **Resource monitors are your budget alerts.** They warn you before you exceed your spending limit.
- **Query tagging is your itemized receipt.** It shows which project or team incurred which costs.
- **Auto-suspend is turning off lights when you leave the room.** It eliminates waste without impacting functionality.
- **Right-sizing is choosing the right appliance for the job.** A small fridge for a studio; a large one for a family.
- **Archival is moving seasonal items to storage.** Keep what you need accessible; store the rest cost-effectively.

Measure your usage. Set your budget. Attribute your costs. Automate your controls. Review and adjust regularly. That is how monitoring and cost management work in Snowflake.
