# Resource Monitors and Related Cost Management Components in Snowflake

```mermaid
graph TD
  CostMgmt[Cost Management Components] --> ResMon[Resource Monitors]
  CostMgmt --> QueryTag[Query Tagging]
  CostMgmt --> WhMeter[Warehouse Metering]
  CostMgmt --> Alerts[ALERT Objects]
  CostMgmt --> Tasks[Automated Tasks]
  CostMgmt --> ExtInt[External Integrations]
  CostMgmt --> Attribution[Cost Attribution]
  
  ResMon --> Config[Quota, Frequency, Triggers]
  ResMon --> Actions[Notify, Suspend, Block]
  ResMon --> Scope[Warehouse Assignment]
  
  QueryTag --> Session[Session-Level Tags]
  QueryTag --> Role[Role-Level Tags]
  QueryTag --> Report[Attribution Reporting]
  
  WhMeter --> History[WAREHOUSE_METERING_HISTORY]
  WhMeter --> Credits[CREDIT_USAGE View]
  WhMeter --> Analysis[Utilization Analysis]
  
  Alerts --> Condition[SQL Condition]
  Alerts --> Action[Notification Action]
  Alerts --> Schedule[CRON Schedule]
  
  Tasks --> Report[Automated Reports]
  Tasks --> Cleanup[Resource Cleanup]
  Tasks --> Sync[Configuration Sync]
  
  ExtInt --> Slack[Slack Webhooks]
  ExtInt --> PagerDuty[PagerDuty API]
  ExtInt --> Email[Email Notifications]
  
  Attribution --> Tags[Tag-Based Attribution]
  Attribution --> Roles[Role-Based Attribution]
  Attribution --> Chargeback[Chargeback Export]
```

---

## Resource Monitors: Core Component

### What Resource Monitors Do

| Concept | Simple Explanation | Example |
|---------|------------------|---------|
| Credit quota | Maximum credits allowed per frequency period | 1000 credits per month |
| Frequency | How often the quota resets | MONTHLY, WEEKLY, DAILY, or NEVER |
| Trigger threshold | Percentage of quota that triggers an action | Notify at 50%, Suspend at 100% |
| Action | What happens when threshold is reached | NOTIFY, SUSPEND, or BLOCK |
| Scope | Which warehouses the monitor applies to | One warehouse or multiple |

```sql
-- Create a comprehensive resource monitor
CREATE OR REPLACE RESOURCE MONITOR prod_budget_guardrail
  COMMENT = 'Production budget monitor: 2000 credits/month, notify at 50/75/90%, suspend at 100%. Owner: finance_team. Review: quarterly.'
  WITH CREDIT_QUOTA = 2000
  FREQUENCY = MONTHLY
  START_TIMESTAMP = 'NEXT_MONTH'
  TRIGGERS
    ON 50 PERCENT DO NOTIFY,
    ON 75 PERCENT DO NOTIFY,
    ON 90 PERCENT DO NOTIFY,
    ON 100 PERCENT DO SUSPEND;

-- Assign to production warehouses
ALTER WAREHOUSE reporting_wh SET RESOURCE_MONITOR = prod_budget_guardrail;
ALTER WAREHOUSE etl_wh SET RESOURCE_MONITOR = prod_budget_guardrail;
ALTER WAREHOUSE api_wh SET RESOURCE_MONITOR = prod_budget_guardrail;
```

### Resource Monitor Configuration Reference

| Setting | Allowed Values | Default | Best Practice |
|---------|---------------|---------|--------------|
| `CREDIT_QUOTA` | Positive integer | Required | Historical avg + 20% buffer |
| `FREQUENCY` | MONTHLY, WEEKLY, DAILY, NEVER | MONTHLY | Align with billing or sprint cycle |
| `START_TIMESTAMP` | IMMEDIATELY, timestamp, 'NEXT_DAY/MONTH' | IMMEDIATELY | Use NEXT_MONTH for clean billing alignment |
| `ON X PERCENT DO` | NOTIFY, SUSPEND, BLOCK | None | NOTIFY at 50/75/90; SUSPEND at 100 for prod |
| `NOTIFY_USERS` | Comma-separated role names | None | Include finance and platform admin roles |

### Threshold Strategy by Environment

| Environment | 50% | 75% | 90% | 100% | Rationale |
|------------|-----|-----|-----|------|-----------|
| Production | NOTIFY | NOTIFY | NOTIFY | SUSPEND | Early warnings + hard stop to protect budget |
| Staging | NOTIFY | NOTIFY | NOTIFY | SUSPEND | Allow testing flexibility with safety net |
| Development | NOTIFY | NOTIFY | NOTIFY | NOTIFY | Maximize flexibility; rely on team discipline |
| Shared Pool | NOTIFY | NOTIFY | SUSPEND | SUSPEND | Protect shared budget from runaway workloads |

```sql
-- Production: Strict enforcement
CREATE OR REPLACE RESOURCE MONITOR prod_strict
  WITH CREDIT_QUOTA = 2000
  FREQUENCY = MONTHLY
  TRIGGERS
    ON 50 PERCENT DO NOTIFY
    ON 75 PERCENT DO NOTIFY
    ON 90 PERCENT DO NOTIFY
    ON 100 PERCENT DO SUSPEND;

-- Development: Lenient monitoring
CREATE OR REPLACE RESOURCE MONITOR dev_flexible
  WITH CREDIT_QUOTA = 200
  FREQUENCY = MONTHLY
  TRIGGERS
    ON 80 PERCENT DO NOTIFY
    ON 100 PERCENT DO NOTIFY;  -- No suspend or block
```

---

## Component 1: Query Tagging for Cost Attribution

### What Query Tagging Does

| Concept | Simple Explanation | Example |
|---------|------------------|---------|
| Query tag | Metadata attached to queries for filtering and attribution | `project=sales_dashboard,team=analytics,env=prod` |
| Session-level tag | Tag set for all queries in a session | `ALTER SESSION SET QUERY_TAG = '...'` |
| Role-level tag | Tag inherited by all users with a role | `ALTER ROLE ANALYST_ROLE SET QUERY_TAG = '...'` |
| Attribution | Grouping costs by tag values for chargeback | Sum credits by `project=` or `team=` tag |

```sql
-- Set query tag at session level
ALTER SESSION SET QUERY_TAG = 'project=sales_dashboard,team=analytics,env=prod,user=jane_doe';

-- Set query tag at role level (applies to all users with role)
ALTER ROLE ANALYST_ROLE SET QUERY_TAG = 'team=analytics,access_level=standard';

-- Query credit usage by tag for chargeback reporting
SELECT
  SPLIT_PART(query_tag, ',', 1) as project,
  SPLIT_PART(query_tag, ',', 2) as team,
  SPLIT_PART(query_tag, ',', 3) as environment,
  SUM(credits_used) as total_credits,
  COUNT(DISTINCT query_id) as query_count,
  ROUND(SUM(credits_used) * 3.00, 2) as estimated_cost_usd
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time > DATEADD(month, -1, CURRENT_TIMESTAMP())
  AND query_tag IS NOT NULL
  AND query_tag != ''
GROUP BY 
  SPLIT_PART(query_tag, ',', 1),
  SPLIT_PART(query_tag, ',', 2),
  SPLIT_PART(query_tag, ',', 3)
ORDER BY total_credits DESC;
```

### Tagging Best Practices

| Practice | Implementation | Benefit |
|----------|---------------|---------|
| Enforce tagging via role settings | `ALTER ROLE X SET QUERY_TAG = '...'` | Ensures consistent attribution without relying on user discipline |
| Use structured key=value format | `project=X,team=Y,env=Z` | Enables easy parsing and filtering in reports |
| Include environment in tag | Always add `env=prod/dev/test` | Enables environment-specific cost analysis |
| Document tag schema | Maintain wiki with allowed keys and values | Prevents inconsistent tagging across teams |
| Validate tags in CI/CD | Check for required tags in deployment pipelines | Catches missing attribution before production |

```sql
-- Create view for standardized tag parsing
CREATE OR REPLACE VIEW governance.query_tag_parsed AS
SELECT
  query_id,
  user_name,
  warehouse_name,
  start_time,
  credits_used,
  query_tag,
  -- Parse structured tags: project=X,team=Y,env=Z
  NULLIF(SPLIT_PART(REGEXP_SUBSTR(query_tag, 'project=[^,]+'), '=', 2), '') as project,
  NULLIF(SPLIT_PART(REGEXP_SUBSTR(query_tag, 'team=[^,]+'), '=', 2), '') as team,
  NULLIF(SPLIT_PART(REGEXP_SUBSTR(query_tag, 'env=[^,]+'), '=', 2), '') as environment,
  NULLIF(SPLIT_PART(REGEXP_SUBSTR(query_tag, 'cost_center=[^,]+'), '=', 2), '') as cost_center
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_tag IS NOT NULL AND query_tag != '';

-- Use parsed view for attribution reports
SELECT
  COALESCE(project, 'UNTAGGED') as project,
  COALESCE(team, 'UNTAGGED') as team,
  COALESCE(environment, 'UNTAGGED') as environment,
  SUM(credits_used) as total_credits,
  ROUND(SUM(credits_used) * 3.00, 2) as estimated_cost_usd
FROM governance.query_tag_parsed
WHERE start_time > DATEADD(month, -1, CURRENT_TIMESTAMP())
GROUP BY project, team, environment
ORDER BY total_credits DESC;
```


## Component 2: Warehouse Metering and Usage Views

### Key ACCOUNT_USAGE Views for Cost Monitoring

| View | What It Tracks | Retention | Primary Use Case |
|------|---------------|-----------|-----------------|
| `WAREHOUSE_METERING_HISTORY` | Credit usage by warehouse over time | 365 days | Warehouse cost analysis, right-sizing decisions |
| `CREDIT_USAGE` | Credit consumption by service type | 365 days | Overall budget tracking, service-level attribution |
| `QUERY_HISTORY` | Query execution details including credits | 365 days | Query-level cost attribution, optimization targeting |
| `TABLE_STORAGE_METRICS` | Storage usage by table over time | 365 days | Storage cost analysis, archival decisions |
| `RESOURCE_MONITOR_EVENTS` | Resource monitor threshold events | 365 days | Audit of budget guardrail triggers and actions |

```sql
-- Analyze warehouse credit usage trends
SELECT
  DATE_TRUNC('day', start_time) as usage_date,
  warehouse_name,
  SUM(credits_used) as daily_credits,
  COUNT(DISTINCT query_id) as query_count,
  ROUND(AVG(credits_used), 2) as avg_credits_per_query
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time > DATEADD(month, -1, CURRENT_TIMESTAMP())
GROUP BY DATE_TRUNC('day', start_time), warehouse_name
ORDER BY usage_date DESC, daily_credits DESC;

-- Track credit usage by service type
SELECT
  DATE_TRUNC('day', start_time) as usage_date,
  service_type,
  SUM(credits_used) as service_credits,
  ROUND(100.0 * SUM(credits_used) / SUM(SUM(credits_used)) OVER (PARTITION BY usage_date), 2) as pct_of_daily_total
FROM SNOWFLAKE.ACCOUNT_USAGE.CREDIT_USAGE
WHERE start_time > DATEADD(month, -1, CURRENT_TIMESTAMP())
GROUP BY DATE_TRUNC('day', start_time), service_type
ORDER BY usage_date DESC, service_credits DESC;

-- Identify expensive queries for optimization
SELECT
  query_id,
  user_name,
  warehouse_name,
  query_text,
  start_time,
  execution_time,
  bytes_scanned,
  credits_used,
  query_tag
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time > DATEADD(day, -7, CURRENT_TIMESTAMP())
  AND credits_used > 5  -- Focus on expensive queries
ORDER BY credits_used DESC
LIMIT 50;
```

### Latency Considerations

| View | Typical Latency | Impact on Monitoring |
|------|----------------|---------------------|
| `QUERY_HISTORY` | ~45 minutes | Not suitable for real-time alerting; use for batch analysis |
| `WAREHOUSE_METERING_HISTORY` | ~45 minutes | Cost reports should account for delay |
| `RESOURCE_MONITOR_EVENTS` | ~5-15 minutes | More timely for threshold breach alerts |
| `CREDIT_USAGE` | ~45 minutes | Budget tracking should use historical windows |

```sql
-- Account for latency in monitoring queries
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


## Component 3: ALERT Objects for Proactive Notification

### What ALERT Objects Do

| Concept | Simple Explanation | Example |
|---------|------------------|---------|
| ALERT object | Scheduled SQL query that triggers action when condition is met | "Alert if warehouse uses >50 credits in 1 hour" |
| Condition | SQL expression that evaluates to TRUE to trigger alert | `(SELECT SUM(credits) FROM ...) > threshold` |
| Action | What happens when alert fires: email, Slack, task, log | `SYSTEM$SEND_EMAIL(...)`, `SYSTEM$SEND_SLACK_MESSAGE(...)` |
| Schedule | CRON expression defining evaluation frequency | `'USING CRON */30 * * * *'` = every 30 minutes |

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

-- Alert: Resource monitor threshold reached
CREATE OR REPLACE ALERT cost.monitor_threshold_alert
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON */15 * * * *'
  CONDITION = (
    SELECT COUNT(*)
    FROM SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITOR_EVENTS
    WHERE event_timestamp > DATEADD(minute, -15, CURRENT_TIMESTAMP())
      AND event_type IN ('NOTIFY', 'SUSPEND', 'BLOCK')
  ) > 0
  ACTION = (
    SYSTEM$SEND_SLACK_MESSAGE(
      'https://hooks.slack.com/services/XXX',
      '🚨 *Resource Monitor Alert*\n' ||
      (SELECT LISTAGG(
        '• ' || resource_monitor_name || ': ' || event_type || 
        ' at ' || threshold_percent || '% (' || credits_used || '/' || credit_quota || ' credits)',
        '\n')
       FROM SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITOR_EVENTS
       WHERE event_timestamp > DATEADD(minute, -15, CURRENT_TIMESTAMP())
         AND event_type IN ('NOTIFY', 'SUSPEND', 'BLOCK'))
    )
  );
```

### ALERT Action Options

| Action Type | Function | Use Case |
|------------|----------|----------|
| Email notification | `SYSTEM$SEND_EMAIL(recipient, subject, body)` | Formal alerts to finance or management |
| Slack message | `SYSTEM$SEND_SLACK_MESSAGE(webhook_url, message)` | Real-time alerts to engineering channels |
| Task trigger | `CALL procedure_name()` | Trigger downstream remediation workflows |
| Log to table | `INSERT INTO audit_table VALUES (...)` | Persistent audit trail for compliance |
| HTTP POST | `SYSTEM$HTTP_POST(url, body, headers)` | Integrate with PagerDuty, ServiceNow, custom APIs |

```sql
-- Alert: Long-running queries consuming credits
CREATE OR REPLACE ALERT ops.long_query_alert
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON 0 */6 * * *'  -- Every 6 hours
  CONDITION = (
    SELECT COUNT(*)
    FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE execution_time > 1800  -- Queries > 30 minutes
      AND start_time > DATEADD(hour, -6, CURRENT_TIMESTAMP())
      AND warehouse_name = 'PROD_WH'
  ) > 0
  ACTION = (
    -- Log to audit table first
    INSERT INTO governance.long_query_audit
      (query_id, user_name, execution_time, credits_used, alert_time)
    SELECT
      query_id, user_name, execution_time, credits_used, CURRENT_TIMESTAMP()
    FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE execution_time > 1800
      AND start_time > DATEADD(hour, -6, CURRENT_TIMESTAMP())
      AND warehouse_name = 'PROD_WH';
    
    -- Send notification
    SYSTEM$SEND_EMAIL(
      'platform-team@company.com',
      'Alert: Long-running queries detected',
      (SELECT LISTAGG('Query ID: ' || query_id || ' (' || execution_time || 's, ' || credits_used || ' credits)', '\n')
       FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
       WHERE execution_time > 1800
         AND start_time > DATEADD(hour, -6, CURRENT_TIMESTAMP())
         AND warehouse_name = 'PROD_WH')
    )
  );
```


## Component 4: Automated Tasks for Reporting and Cleanup

### What Tasks Do

| Concept | Simple Explanation | Example |
|---------|------------------|---------|
| Task | Scheduled SQL or procedure execution | "Run cost report every Monday at 9 AM" |
| Schedule | CRON expression or interval | `'USING CRON 0 9 * * 1'` = Mondays at 9 AM |
| Warehouse | Compute resource that runs the task | `WAREHOUSE = admin_wh` |
| Dependencies | Tasks can depend on other tasks completing | Build ETL pipelines with task graphs |

```sql
-- Task: Generate weekly cost attribution report
CREATE OR REPLACE TASK governance.weekly_cost_report
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON 0 9 * * 1'  -- Mondays at 9 AM
AS
  -- Generate and email cost attribution summary
  SYSTEM$SEND_EMAIL(
    'finance-team@company.com',
    'Weekly Snowflake Cost Report - Week of ' || TO_VARCHAR(DATEADD(week, -1, CURRENT_DATE()), 'YYYY-MM-DD'),
    (SELECT 
      'Total Credits: ' || SUM(credits_used) || '\n' ||
      'Top Project: ' || 
        (SELECT project FROM governance.query_tag_parsed 
         WHERE start_time > DATEADD(week, -1, CURRENT_TIMESTAMP())
         GROUP BY project ORDER BY SUM(credits_used) DESC LIMIT 1) || '\n' ||
      'Top Warehouse: ' || 
        (SELECT warehouse_name FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY 
         WHERE start_time > DATEADD(week, -1, CURRENT_TIMESTAMP())
         GROUP BY warehouse_name ORDER BY SUM(credits_used) DESC LIMIT 1) || '\n' ||
      'Resource Monitor Alerts: ' ||
        (SELECT COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITOR_EVENTS
         WHERE event_timestamp > DATEADD(week, -1, CURRENT_TIMESTAMP())) || '\n\n' ||
      'View detailed report: https://your-bi-tool.com/snowflake-cost-dashboard'
     FROM SNOWFLAKE.ACCOUNT_USAGE.CREDIT_USAGE
     WHERE start_time > DATEADD(week, -1, CURRENT_TIMESTAMP()))
  );

-- Task: Identify and alert on idle warehouses
CREATE OR REPLACE TASK ops.idle_warehouse_cleanup
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON 0 10 * * 1-5'  -- Weekdays at 10 AM
AS
  -- Log idle warehouses to audit table
  INSERT INTO governance.idle_warehouse_log
  SELECT
    warehouse_name,
    state,
    auto_suspend,
    COUNT(DISTINCT query_id) as query_count_last_24h,
    SUM(credits_used) as credits_last_24h,
    CURRENT_TIMESTAMP() as logged_date
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

-- Task: Archive cold storage data (depends on idle_warehouse_cleanup)
CREATE OR REPLACE TASK governance.archive_cold_data
  WAREHOUSE = admin_wh
  AFTER ops.idle_warehouse_cleanup  -- Run after idle warehouse task completes
  WHEN SYSTEM$GET_PREDECESSOR_RETURN_VALUE('ops.idle_warehouse_cleanup') = 'SUCCESS'
AS
  -- Export tables with no access in 90 days to external storage
  -- (Simplified example; actual implementation would loop through tables)
  COPY INTO @archive_stage/cold_data/
  FROM (
    SELECT * FROM raw.large_table
    WHERE last_accessed < DATEADD(day, -90, CURRENT_TIMESTAMP())
  )
  FILE_FORMAT = (TYPE = PARQUET COMPRESSION = SNAPPY);
```

### Task Dependency Chains

```mermaid
graph TD
  Start[Schedule Trigger] --> IdleCheck[Idle Warehouse Check]
  IdleCheck -->|Success| Archive[Archive Cold Data]
  IdleCheck -->|Failure| Alert[Send Alert]
  Archive --> Report[Update Cost Report]
  Report --> Notify[Email Summary]
```

```sql
-- Example: Task chain for cost optimization workflow
-- Step 1: Check for optimization opportunities
CREATE OR REPLACE TASK governance.cost_optimization_scan
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON 0 8 * * 1'  -- Mondays at 8 AM
AS
  INSERT INTO governance.optimization_recommendations
  SELECT
    'WAREHOUSE_RIGHTSIZE' as recommendation_type,
    warehouse_name,
    current_size,
    recommended_size,
    estimated_monthly_savings,
    CURRENT_TIMESTAMP() as generated_at
  FROM governance.warehouse_utilization_analysis
  WHERE sizing_recommendation != 'Current size may be appropriate';

-- Step 2: Generate recommendations report (depends on scan)
CREATE OR REPLACE TASK governance.optimization_report
  WAREHOUSE = admin_wh
  AFTER governance.cost_optimization_scan
  WHEN SYSTEM$GET_PREDECESSOR_RETURN_VALUE('governance.cost_optimization_scan') = 'SUCCESS'
AS
  SYSTEM$SEND_EMAIL(
    'platform-team@company.com',
    'Weekly Cost Optimization Recommendations',
    (SELECT LISTAGG(
      '• ' || recommendation_type || ': ' || warehouse_name || 
      ' (' || current_size || ' → ' || recommended_size || 
      ', est. savings: $' || estimated_monthly_savings || ')',
      '\n')
     FROM governance.optimization_recommendations
     WHERE generated_at > DATEADD(day, -1, CURRENT_TIMESTAMP()))
  );
```


## Component 5: External Integrations for Notifications

### Slack Integration via Webhook

```sql
-- Alert: Send cost summary to Slack channel
CREATE OR REPLACE ALERT cost.slack_daily_summary
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON 0 9 * * *'  -- Daily at 9 AM
  CONDITION = TRUE  -- Always run for daily summary
  ACTION = (
    SYSTEM$SEND_SLACK_MESSAGE(
      'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX',
      '💰 *Snowflake Daily Cost Summary*\n' ||
      '• Date: ' || TO_VARCHAR(CURRENT_DATE(), 'YYYY-MM-DD') || '\n' ||
      '• Total Credits: ' || 
        (SELECT SUM(credits_used) FROM SNOWFLAKE.ACCOUNT_USAGE.CREDIT_USAGE 
         WHERE start_time > DATEADD(day, -1, CURRENT_TIMESTAMP())) || '\n' ||
      '• Top Warehouse: ' || 
        (SELECT warehouse_name FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY 
         WHERE start_time > DATEADD(day, -1, CURRENT_TIMESTAMP())
         GROUP BY warehouse_name ORDER BY SUM(credits_used) DESC LIMIT 1) || '\n' ||
      '• Active Resource Monitors: ' ||
        (SELECT COUNT(DISTINCT resource_monitor_name) FROM SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITOR_EVENTS
         WHERE event_timestamp > DATEADD(day, -1, CURRENT_TIMESTAMP())) || '\n\n' ||
      '🔗 <https://your-bi-tool.com/snowflake-dashboard|View Detailed Dashboard>'
    )
  );
```

### PagerDuty Integration for Critical Alerts

```sql
-- Alert: Critical budget threshold exceeded
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
            'alert_time', CURRENT_TIMESTAMP(),
            'runbook_url', 'https://wiki.company.com/snowflake-budget-runbook'
          )
        )
      )::STRING,
      OBJECT_CONSTRUCT('Content-Type', 'application/json')
    )
  );
```

### Email Integration with Dynamic Content

```sql
-- Alert: Resource monitor threshold with detailed context
CREATE OR REPLACE ALERT cost.monitor_threshold_detailed
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON */15 * * * *'
  CONDITION = (
    SELECT COUNT(*)
    FROM SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITOR_EVENTS
    WHERE event_timestamp > DATEADD(minute, -15, CURRENT_TIMESTAMP())
      AND event_type IN ('NOTIFY', 'SUSPEND', 'BLOCK')
  ) > 0
  ACTION = (
    SYSTEM$SEND_EMAIL(
      'finance-team@company.com, platform-team@company.com',
      'Resource Monitor Threshold Triggered',
      'Monitor Events in Last 15 Minutes:\n\n' ||
      (SELECT LISTAGG(
        '• Monitor: ' || resource_monitor_name || '\n' ||
        '  Event: ' || event_type || '\n' ||
        '  Threshold: ' || threshold_percent || '%\n' ||
        '  Credits: ' || credits_used || '/' || credit_quota || '\n' ||
        '  Time: ' || TO_VARCHAR(event_timestamp, 'YYYY-MM-DD HH24:MI') || '\n' ||
        '  Warehouses: ' || COALESCE(warehouse_name, 'All attached') || '\n',
        '\n---\n\n')
       FROM SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITOR_EVENTS
       WHERE event_timestamp > DATEADD(minute, -15, CURRENT_TIMESTAMP())
         AND event_type IN ('NOTIFY', 'SUSPEND', 'BLOCK')) ||
      '\n\nRecommended Actions:\n' ||
      '1. Review WAREHOUSE_METERING_HISTORY for usage patterns\n' ||
      '2. Check QUERY_HISTORY for expensive new queries\n' ||
      '3. Verify resource monitor configuration if unexpected\n' ||
      '4. Resume suspended warehouses if appropriate: ALTER WAREHOUSE X RESUME;\n\n' ||
      'Dashboard: https://your-bi-tool.com/snowflake-cost-monitoring'
    )
  );
```


## Component 6: Cost Attribution and Chargeback Mechanisms

### Tag-Based Attribution

```sql
-- Create view for standardized tag-based attribution
CREATE OR REPLACE VIEW governance.cost_attribution_by_tag AS
SELECT
  DATE_TRUNC('month', qh.start_time) as billing_month,
  NULLIF(SPLIT_PART(REGEXP_SUBSTR(qh.query_tag, 'project=[^,]+'), '=', 2), '') as project,
  NULLIF(SPLIT_PART(REGEXP_SUBSTR(qh.query_tag, 'team=[^,]+'), '=', 2), '') as team,
  NULLIF(SPLIT_PART(REGEXP_SUBSTR(qh.query_tag, 'env=[^,]+'), '=', 2), '') as environment,
  NULLIF(SPLIT_PART(REGEXP_SUBSTR(qh.query_tag, 'cost_center=[^,]+'), '=', 2), '') as cost_center,
  SUM(qh.credits_used) as total_credits,
  COUNT(DISTINCT qh.query_id) as query_count,
  SUM(qh.bytes_scanned) as total_bytes_scanned,
  ROUND(SUM(qh.credits_used) * 3.00, 2) as estimated_cost_usd  -- Enterprise edition rate
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY qh
WHERE qh.start_time > DATEADD(month, -3, CURRENT_TIMESTAMP())
  AND qh.query_tag IS NOT NULL
  AND qh.query_tag != ''
GROUP BY 
  DATE_TRUNC('month', qh.start_time),
  NULLIF(SPLIT_PART(REGEXP_SUBSTR(qh.query_tag, 'project=[^,]+'), '=', 2), ''),
  NULLIF(SPLIT_PART(REGEXP_SUBSTR(qh.query_tag, 'team=[^,]+'), '=', 2), ''),
  NULLIF(SPLIT_PART(REGEXP_SUBSTR(qh.query_tag, 'env=[^,]+'), '=', 2), ''),
  NULLIF(SPLIT_PART(REGEXP_SUBSTR(qh.query_tag, 'cost_center=[^,]+'), '=', 2), '');

-- Export for external billing systems
COPY INTO @finance_exports/chargeback/
FROM governance.cost_attribution_by_tag
WHERE billing_month = DATE_TRUNC('month', CURRENT_DATE())
FILE_FORMAT = (TYPE = CSV HEADER = TRUE);
```

### Role-Based Attribution

```sql
-- Create view for role-based cost attribution
CREATE OR REPLACE VIEW governance.cost_attribution_by_role AS
SELECT
  DATE_TRUNC('month', qh.start_time) as billing_month,
  r.name as role_name,
  SUM(qh.credits_used) as total_credits,
  COUNT(DISTINCT qh.query_id) as query_count,
  SUM(qh.bytes_scanned) as total_bytes_scanned,
  ROUND(SUM(qh.credits_used) * 3.00, 2) as estimated_cost_usd
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
FROM governance.cost_attribution_by_role
WHERE billing_month = DATE_TRUNC('month', CURRENT_DATE())
ORDER BY estimated_cost_usd DESC;
```

### Storage Cost Attribution

```sql
-- Attribute storage costs using table tags
CREATE OR REPLACE VIEW governance.storage_cost_attribution AS
SELECT
  DATE_TRUNC('month', tsm.last_updated) as billing_month,
  COALESCE(tr.tag_value, 'UNTAGGED') as cost_center,
  tsm.table_schema,
  tsm.table_name,
  tsm.active_bytes / POWER(1024, 3) as active_gb,
  tsm.time_travel_bytes / POWER(1024, 3) as time_travel_gb,
  tsm.failsafe_bytes / POWER(1024, 3) as failsafe_gb,
  (tsm.active_bytes + tsm.time_travel_bytes + tsm.failsafe_bytes) / POWER(1024, 3) as total_gb,
  -- Estimate monthly cost: $23/TB/month for storage
  (tsm.active_bytes + tsm.time_travel_bytes + tsm.failsafe_bytes) / POWER(1024, 4) * 23.00 as estimated_monthly_cost_usd
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS tsm
LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES tr
  ON tsm.table_name = tr.object_name
  AND tr.tag_name = 'cost_center'
  AND tr.column_name IS NULL  -- Table-level tag
WHERE tsm.deleted_on IS NULL
  AND tsm.table_schema NOT IN ('INFORMATION_SCHEMA', 'SNOWFLAKE')
  AND tsm.last_updated > DATEADD(month, -1, CURRENT_TIMESTAMP());

-- Export storage attribution for finance
COPY INTO @finance_exports/storage_chargeback/
FROM governance.storage_cost_attribution
WHERE billing_month = DATE_TRUNC('month', CURRENT_DATE())
FILE_FORMAT = (TYPE = CSV HEADER = TRUE);
```


## Integration Patterns: Combining Components

### Pattern: Budget Guardrail with Attribution

```mermaid
graph TD
  Usage[Credit Usage] --> Monitor[Resource Monitor]
  Monitor --> Threshold{Threshold Reached}
  Threshold -->|No| Continue[Continue Tracking]
  Threshold -->|Yes| Alert[Trigger ALERT]
  
  Alert --> Notify[Send Notification]
  Alert --> Log[Log to Audit Table]
  
  Usage --> Tag[Query Tag Attached]
  Tag --> Attribute[Attribute to Project/Team]
  Attribute --> Report[Generate Chargeback Report]
  
  Notify --> Finance[Finance Team Reviews]
  Report --> Finance
  Finance --> Optimize[Optimize or Adjust Budget]
```

```sql
-- Step 1: Create resource monitor with attribution-aware alerting
CREATE OR REPLACE RESOURCE MONITOR prod_attributed_monitor
  WITH CREDIT_QUOTA = 2000
  FREQUENCY = MONTHLY
  TRIGGERS
    ON 50 PERCENT DO NOTIFY
    ON 75 PERCENT DO NOTIFY
    ON 90 PERCENT DO NOTIFY
    ON 100 PERCENT DO SUSPEND;

-- Step 2: Create alert that includes attribution context
CREATE OR REPLACE ALERT cost.attributed_threshold_alert
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON */15 * * * *'
  CONDITION = (
    SELECT COUNT(*)
    FROM SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITOR_EVENTS
    WHERE event_timestamp > DATEADD(minute, -15, CURRENT_TIMESTAMP())
      AND resource_monitor_name = 'prod_attributed_monitor'
  ) > 0
  ACTION = (
    SYSTEM$SEND_EMAIL(
      'finance-team@company.com',
      'Resource Monitor Threshold: Attribution Context Included',
      'Monitor: prod_attributed_monitor\n' ||
      'Events in last 15 minutes:\n\n' ||
      (SELECT LISTAGG(
        '• Event: ' || event_type || ' at ' || threshold_percent || '%\n' ||
        '  Credits: ' || credits_used || '/' || credit_quota || '\n' ||
        '  Top Projects by Usage:\n' ||
        (SELECT LISTAGG('    - ' || project || ': ' || proj_credits || ' credits', '\n')
         FROM (
           SELECT 
             NULLIF(SPLIT_PART(REGEXP_SUBSTR(qh.query_tag, 'project=[^,]+'), '=', 2), '') as project,
             SUM(qh.credits_used) as proj_credits
           FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY qh
           JOIN SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITOR_EVENTS rme
             ON qh.warehouse_name IN (
               SELECT name FROM INFORMATION_SCHEMA.WAREHOUSES 
               WHERE resource_monitor = rme.resource_monitor_name
             )
           WHERE qh.start_time > DATEADD(hour, -1, CURRENT_TIMESTAMP())
             AND qh.query_tag IS NOT NULL
           GROUP BY project
           ORDER BY proj_credits DESC
           LIMIT 5
         )),
        '\n---\n\n')
       FROM SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITOR_EVENTS
       WHERE event_timestamp > DATEADD(minute, -15, CURRENT_TIMESTAMP())
         AND resource_monitor_name = 'prod_attributed_monitor')
    )
  );
```

### Pattern: Automated Optimization Workflow

```mermaid
graph TD
  Schedule[Weekly Schedule] --> Scan[Scan for Optimization Opportunities]
  Scan --> WhRight[Identify Warehouse Right-Sizing]
  Scan --> QueryOpt[Identify Expensive Queries]
  Scan --> StorageOpt[Identify Storage Optimization]
  
  WhRight --> RecWh[Generate Warehouse Recommendations]
  QueryOpt --> RecQuery[Generate Query Optimization Recommendations]
  StorageOpt --> RecStorage[Generate Storage Recommendations]
  
  RecWh --> Report[Compile Optimization Report]
  RecQuery --> Report
  RecStorage --> Report
  
  Report --> Alert[Send to Platform Team]
  Alert --> Review[Team Reviews and Implements]
  Review --> Measure[Measure Impact]
  Measure --> Adjust[Adjust Recommendations]
```

```sql
-- Task: Weekly optimization scan
CREATE OR REPLACE TASK governance.weekly_optimization_scan
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON 0 8 * * 1'  -- Mondays at 8 AM
AS
  -- Clear previous recommendations
  TRUNCATE TABLE governance.optimization_recommendations;
  
  -- Warehouse right-sizing recommendations
  INSERT INTO governance.optimization_recommendations
  SELECT
    'WAREHOUSE_RIGHTSIZE' as recommendation_type,
    warehouse_name as object_name,
    current_size as current_config,
    sizing_recommendation as recommended_config,
    estimated_monthly_savings_usd as estimated_savings,
    'Review WAREHOUSE_METERING_HISTORY for usage patterns' as action_steps,
    CURRENT_TIMESTAMP() as generated_at
  FROM governance.warehouse_utilization_analysis
  WHERE sizing_recommendation != 'Current size may be appropriate';
  
  -- Query optimization recommendations
  INSERT INTO governance.optimization_recommendations
  SELECT
    'QUERY_OPTIMIZATION' as recommendation_type,
    query_id as object_name,
    'bytes_scanned=' || bytes_scanned || ', execution_time=' || execution_time as current_config,
    'Consider clustering, filtering, or materialization' as recommended_config,
    credits_used * 0.5 * 3.00 as estimated_savings,  -- Assume 50% savings
    'Review query plan; add clustering keys if filtering on non-clustered columns' as action_steps,
    CURRENT_TIMESTAMP() as generated_at
  FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE start_time > DATEADD(week, -1, CURRENT_TIMESTAMP())
    AND bytes_scanned > 100000000000  -- > 100 GB
    AND credits_used > 10;
  
  -- Storage optimization recommendations
  INSERT INTO governance.optimization_recommendations
  SELECT
    'STORAGE_OPTIMIZATION' as recommendation_type,
    table_schema || '.' || table_name as object_name,
    'time_travel_pct=' || ROUND(100.0 * time_travel_bytes / NULLIF(active_bytes, 0), 2) || '%' as current_config,
    retention_recommendation as recommended_config,
    potential_monthly_savings_usd as estimated_savings,
    'Review Time Travel retention; archive cold data to external storage' as action_steps,
    CURRENT_TIMESTAMP() as generated_at
  FROM governance.time_travel_cost_analysis
  WHERE potential_monthly_savings_usd > 10;

-- Task: Generate and send optimization report
CREATE OR REPLACE TASK governance.optimization_report
  WAREHOUSE = admin_wh
  AFTER governance.weekly_optimization_scan
  WHEN SYSTEM$GET_PREDECESSOR_RETURN_VALUE('governance.weekly_optimization_scan') = 'SUCCESS'
AS
  SYSTEM$SEND_EMAIL(
    'platform-team@company.com',
    'Weekly Cost Optimization Recommendations',
    'Optimization Opportunities Identified:\n\n' ||
    (SELECT LISTAGG(
      '• ' || recommendation_type || ': ' || object_name || '\n' ||
      '  Current: ' || current_config || '\n' ||
      '  Recommended: ' || recommended_config || '\n' ||
      '  Est. Savings: $' || estimated_savings || '/month\n' ||
      '  Action: ' || action_steps || '\n',
      '\n---\n\n')
     FROM governance.optimization_recommendations
     WHERE generated_at > DATEADD(day, -1, CURRENT_TIMESTAMP())
     ORDER BY estimated_savings DESC) ||
    '\n\nReview and implement recommendations via:\n' ||
    'https://your-bi-tool.com/snowflake-optimization-dashboard'
  );
```


## Best Practices Summary

### Resource Monitor Configuration

| Practice | Implementation | Benefit |
|----------|---------------|---------|
| Base quota on historical usage | Query `WAREHOUSE_METERING_HISTORY` for past 3 months; add 20% buffer | Prevents unnecessary suspensions while controlling runaway spend |
| Separate monitors by environment | Prod, staging, dev get different quotas and enforcement | Aligns guardrails with business criticality |
| Use progressive thresholds | NOTIFY at 50/75/90%; SUSPEND at 100% for prod | Gives teams time to optimize before hard enforcement |
| Document quota rationale | Add COMMENT with calculation method and review date | Enables audits and future adjustments |

### Query Tagging for Attribution

| Practice | Implementation | Benefit |
|----------|---------------|---------|
| Enforce tagging via role settings | `ALTER ROLE X SET QUERY_TAG = '...'` | Ensures consistent attribution without relying on user discipline |
| Use structured key=value format | `project=X,team=Y,env=Z` | Enables easy parsing and filtering in reports |
| Include environment in tag | Always add `env=prod/dev/test` | Enables environment-specific cost analysis |
| Validate tags in CI/CD | Check for required tags in deployment pipelines | Catches missing attribution before production |

### ALERT and Task Configuration

| Practice | Implementation | Benefit |
|----------|---------------|---------|
| Alert on anomalies, not absolutes | Compare to baselines; alert on deviations | Reduces alert fatigue while catching real issues |
| Use appropriate notification channels | Email for formal reports; Slack for real-time; PagerDuty for critical | Matches urgency to communication method |
| Log alert actions to audit table | `INSERT INTO audit_table VALUES (...)` | Creates persistent record for compliance and analysis |
| Test alert conditions in non-prod | Verify logic before production deployment | Prevents false positives or missed alerts |

### Integration and Attribution

| Practice | Implementation | Benefit |
|----------|---------------|---------|
| Combine resource monitors with query tagging | Monitors track credits; tags enable attribution | Enables both budget protection and chargeback |
| Export attribution data for external systems | `COPY INTO @external_stage/...` | Integrates with finance billing and reporting tools |
| Use tasks for automated reporting | Schedule weekly/monthly cost reports | Ensures consistent, timely reporting without manual effort |
| Monitor the monitors | Track `RESOURCE_MONITOR_EVENTS` for threshold breaches | Ensures guardrails are working as intended |


## Common Pitfalls and Mitigations

| Pitfall | What Happens | How To Fix |
|---------|--------------|------------|
| Setting resource monitor quota too low | Warehouses suspend unexpectedly, disrupting business | Base quota on historical usage + buffer; test in non-prod first |
| Not using query tags | Cannot attribute costs to projects or teams | Enforce tagging via role settings or governance policy |
| Ignoring ACCOUNT_USAGE latency | Real-time alerts fail because data is not yet available | Add buffer window to queries; use for batch analysis not real-time |
| Over-provisioning warehouse size | Paying for unused compute capacity | Analyze `WAREHOUSE_METERING_HISTORY`; right-size based on p95 usage |
| Disabling auto-suspend for all warehouses | Paying for idle compute 24/7 | Set auto-suspend based on workload pattern; disable only for always-on APIs |
| Assuming actions are instantaneous | Queries continue for minutes after threshold breach | Design applications to handle "warehouse suspended" errors gracefully |
| Not testing alert conditions | False positives or missed alerts in production | Test alert logic in staging environment before production deployment |
| Forgetting to assign monitor to new warehouses | New warehouses have no budget guardrails | Include monitor assignment in warehouse provisioning automation |

```mermaid
flowchart TD
  Prob[Cost management issue] --> Q1[Unexpected credit spike]
  Prob --> Q2[Cannot attribute costs]
  Prob --> Q3[Warehouse suspended unexpectedly]
  Prob --> Q4[Alerts not firing or too noisy]
  
  Q1 --> A[Check WAREHOUSE_METERING_HISTORY for usage patterns]
  Q1 --> B[Review QUERY_HISTORY for expensive new queries]
  Q1 --> C[Verify resource monitor triggers and actions]
  
  Q2 --> D[Enforce query tagging at session or role level]
  Q2 --> E[Use role-based attribution as fallback]
  
  Q3 --> F[Check RESOURCE_MONITOR_EVENTS for trigger details]
  Q3 --> G[Verify quota and threshold configuration]
  Q3 --> H[Review recent usage spikes]
  
  Q4 --> I[Adjust alert condition thresholds or baseline logic]
  Q4 --> J[Verify notification channel configuration]
  Q4 --> K[Add deduplication or suppression logic]
  
  A --> L[Implement optimization; measure impact]
  B --> L
  C --> L
  D --> L
  E --> L
  F --> L
  G --> L
  H --> L
  I --> L
  J --> L
  K --> L
```


## Decision Framework: Cost Management Component Selection

```mermaid
flowchart TD
  Q1[Start: Design cost management strategy]
  Q1 --> Q2[What is your primary goal]
  
  Q2 -->|Prevent budget overruns| A[Resource Monitors with SUSPEND/BLOCK]
  Q2 -->|Attribute costs to teams| B[Query Tagging + Attribution Views]
  Q2 -->|Proactive anomaly detection| C[ALERT Objects with baseline comparison]
  Q2 -->|Automated reporting| D[Tasks with scheduled exports]
  Q2 -->|Real-time notifications| E[External Integrations: Slack/PagerDuty]
  
  A --> F[Set quota: historical avg + 20% buffer]
  B --> G[Enforce tagging via role settings]
  C --> H[Define baseline from historical usage]
  D --> I[Schedule reports aligned with billing cycle]
  E --> J[Configure webhooks and API integrations]
  
  F --> K[Combine components for comprehensive coverage]
  G --> K
  H --> K
  I --> K
  J --> K
  
  K --> L[Test in non-prod before production rollout]
  L --> M[Document configuration and review schedule]
  M --> N[Monitor and adjust quarterly]
```

| Requirement | Recommended Components | Implementation Priority |
|------------|----------------------|----------------------|
| Prevent budget overruns | Resource Monitors + ALERT objects | High: Configure before production workloads |
| Attribute costs to teams | Query Tagging + Attribution Views | High: Enable at project kickoff |
| Detect cost anomalies | ALERT Objects + baseline queries | Medium: After baseline usage established |
| Automate reporting | Tasks + External Exports | Medium: After attribution is working |
| Real-time notifications | Slack/PagerDuty Integrations | Low: After core monitoring is stable |
| Optimize costs | Warehouse analysis + Query optimization tasks | Ongoing: After baseline monitoring established |


## Key Principles to Remember

- **Measure before you optimize.** You cannot improve what you do not measure. Start with baseline metrics from `ACCOUNT_USAGE` views.
- **Attribute to enable accountability.** Tag queries and assign costs to teams; ownership drives optimization.
- **Alert on anomalies, not absolutes.** Baseline normal behavior; alert on deviations to reduce noise.
- **Automate cost controls.** Resource monitors and auto-suspend enforce guardrails without manual intervention.
- **Review regularly.** Usage patterns and costs evolve; quarterly reviews catch drift before budget impact.
- **Optimize holistically.** Compute, storage, and data transfer all contribute to cost; address all three.
- **Document your strategy.** Future you and your team need context for monitoring and cost decisions.

## Bottom Line

- Resource monitors are your budget guardrails. Configure quotas based on historical usage, set progressive thresholds, and assign to appropriate warehouses.
- Query tagging enables cost attribution. Enforce structured tags via role settings; parse and report for chargeback.
- ALERT objects provide proactive notification. Compare to baselines; alert on anomalies; integrate with Slack, PagerDuty, or email.
- Tasks automate reporting and cleanup. Schedule weekly cost reports, idle resource identification, and optimization recommendations.
- External integrations extend notifications. Use webhooks and APIs to connect Snowflake alerts to your existing incident management tools.
- Attribution views enable chargeback. Combine tag-based and role-based attribution; export to external billing systems.
- Integration patterns combine components. Use resource monitors with attribution, alerts with tasks, and exports with dashboards.
- Review and adjust quarterly. Workloads evolve; your cost management strategy should evolve with them.

Think of cost management like managing a household budget:
- **Resource monitors are your spending limits.** They prevent you from exceeding your budget before it is too late.
- **Query tags are your itemized receipts.** They show which project or team incurred which costs.
- **ALERT objects are your budget alerts.** They warn you when spending is unusual or approaching limits.
- **Tasks are your monthly budget review.** They automate reporting so you do not forget to check.
- **External integrations are your family group chat.** They notify the right people when action is needed.
- **Attribution views are your expense categories.** They enable chargeback and accountability across the household.

Set your limits based on actual income. Track your spending by category. Alert on unusual patterns. Review monthly. Adjust as needs change. That is how cost management works in Snowflake.
