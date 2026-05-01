# Alerts and Notifications in Snowflake Governance

```mermaid
graph TD
  Alerts[Alerts and Notifications] --> AlertObj[ALERT Object]
  Alerts --> Conditions[Alert Conditions]
  Alerts --> Actions[Notification Actions]
  Alerts --> Integrations[External Integrations]
  Alerts --> Lifecycle[Alert Lifecycle]
  
  AlertObj --> Syntax[SQL Syntax]
  AlertObj --> Schedule[Scheduling]
  AlertObj --> State[Alert State Management]
  
  Conditions --> Query[SQL Condition Query]
  Conditions --> Threshold[Threshold Logic]
  Conditions --> Time[Time Window Filtering]
  
  Actions --> Email[Email Notifications]
  Actions --> Slack[Slack/Teams Webhooks]
  Actions --> Task[Trigger Downstream Tasks]
  Actions --> Log[Log to Audit Table]
  
  Integrations --> API[External API Calls]
  Integrations --> SIEM[SIEM Integration]
  Integrations --> PagerDuty[Incident Management]
```

## What Are Alerts in Snowflake

| Concept | Simple Explanation | Example |
|---------|------------------|---------|
| ALERT object | A scheduled SQL query that triggers an action when a condition is met | Alert when failed logins exceed 5 in 15 minutes |
| Condition query | SQL that returns rows when alert should fire | SELECT * FROM LOGIN_HISTORY WHERE success = 'NO' |
| Threshold logic | Logic that determines when condition warrants alert | COUNT(*) >= 5 triggers alert |
| Action | What happens when alert fires: email, webhook, task, or log | Send Slack message to security channel |
| Schedule | CRON expression defining when alert condition is evaluated | Every 15 minutes during business hours |

- Alerts are not real time. They run on a schedule you define
- The condition query runs in a warehouse you assign. It costs credits
- Actions execute only if the condition query returns rows meeting your threshold
- Alerts are governed objects. They require privileges to create and manage
- Alert history is logged. You can audit when alerts fired and what actions ran

```mermaid
flowchart LR
  Schedule[Alert Schedule Triggers] --> Evaluate[Run Condition Query]
  Evaluate --> Check{Does query return rows meeting threshold}
  Check -->|No| Wait[Wait for next schedule]
  Check -->|Yes| Action[Execute configured action]
  Action --> Log[Log alert execution to history]
  Log --> Wait
```

## ALERT Object Syntax and Configuration

### Basic ALERT Creation

```sql
-- Simple alert: notify on failed login attempts
CREATE OR REPLACE ALERT security.failed_login_alert
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON */15 * * * *'  -- Every 15 minutes
  CONDITION = (
    SELECT COUNT(*)
    FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
    WHERE SUCCESS = 'NO'
      AND EVENT_TIMESTAMP > DATEADD(minute, -15, CURRENT_TIMESTAMP)
  ) > 5
  ACTION = (
    SYSTEM$SEND_EMAIL(
      'security-team@company.com',
      'Alert: Multiple failed login attempts detected',
      'Review LOGIN_HISTORY for potential brute force attack'
    )
  );
```

### ALERT Components Explained

| Component | Purpose | Example Values |
|-----------|---------|---------------|
| WAREHOUSE | Compute resource that runs the condition query | admin_wh, monitoring_wh |
| SCHEDULE | CRON expression for evaluation frequency | 'USING CRON 0 */6 * * *' (every 6 hours) |
| CONDITION | SQL expression that evaluates to TRUE to trigger alert | (SELECT COUNT(*) FROM ...) > threshold |
| ACTION | One or more actions to execute when alert fires | SYSTEM$SEND_EMAIL, SYSTEM$SEND_SLACK_MESSAGE, CALL procedure |
| COMMENT | Documentation for alert purpose and ownership | 'Owner: security_team. Review: quarterly' |

```sql
-- Alert with multiple actions and documentation
CREATE OR REPLACE ALERT governance.unmasked_restricted_alert
  WAREHOUSE = governance_wh
  SCHEDULE = 'USING CRON 0 8 * * 1-5'  -- Weekdays at 8 AM
  COMMENT = 'Alert on restricted columns without masking. Owner: security_team. JIRA: GOV-2024'
  CONDITION = (
    SELECT COUNT(*)
    FROM governance.monitor.unmasked_restricted
  ) > 0
  ACTION = (
    -- Action 1: Email security team
    SYSTEM$SEND_EMAIL(
      'security-team@company.com',
      'Alert: Unmasked restricted data detected',
      'Review columns: ' || 
      (SELECT LISTAGG(table_schema || '.' || table_name || '.' || column_name, ', ')
       FROM governance.monitor.unmasked_restricted)
    )
    -- Action 2: Log to audit table
    ; INSERT INTO governance.alert_audit 
      (alert_name, triggered_at, row_count, action_taken)
      SELECT 'unmasked_restricted_alert', CURRENT_TIMESTAMP(), COUNT(*), 'email_sent'
      FROM governance.monitor.unmasked_restricted
  );
```

### Scheduling with CRON Expressions

| CRON Expression | Frequency | Use Case |
|----------------|-----------|----------|
| `*/5 * * * *` | Every 5 minutes | High priority security alerts |
| `0 */1 * * *` | Every hour at minute 0 | Operational monitoring |
| `0 8 * * 1-5` | Weekdays at 8 AM | Business hours governance alerts |
| `0 2 * * *` | Daily at 2 AM | Low priority batch alerts |
| `0 0 1 * *` | First day of month at midnight | Monthly compliance reports |

```sql
-- Alert with complex scheduling: business hours only
CREATE OR REPLACE ALERT ops.business_hours_alert
  WAREHOUSE = ops_wh
  SCHEDULE = 'USING CRON 0 9-17 * * 1-5'  -- Mon-Fri, 9 AM to 5 PM, hourly
  CONDITION = (
    SELECT COUNT(*)
    FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE WAREHOUSE_NAME = 'PROD_WH'
      AND BYTES_SCANNED > 500000000000  -- 500 GB
      AND START_TIME > DATEADD(hour, -1, CURRENT_TIMESTAMP)
  ) > 0
  ACTION = (
    SYSTEM$SEND_SLACK_MESSAGE(
      'https://hooks.slack.com/services/XXX',
      '⚠️ Large query detected on PROD_WH. Review immediately.'
    )
  );
```

## Condition Query Patterns for Governance

### Security Monitoring Conditions

```sql
-- Pattern: Brute force detection
CONDITION = (
  SELECT COUNT(DISTINCT user_name)
  FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
  WHERE SUCCESS = 'NO'
    AND EVENT_TIMESTAMP > DATEADD(minute, -15, CURRENT_TIMESTAMP)
  GROUP BY client_ip
  HAVING COUNT(*) >= 5
) IS NOT NULL;

-- Pattern: Privileged role usage outside business hours
CONDITION = (
  SELECT COUNT(*)
  FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE ANY_ROLE_IN_SESSION('ACCOUNTADMIN', 'SECURITYADMIN')
    AND START_TIME > DATEADD(hour, -1, CURRENT_TIMESTAMP)
    AND (EXTRACT(HOUR FROM START_TIME) < 8 OR EXTRACT(HOUR FROM START_TIME) > 18)
) > 0;

-- Pattern: Network policy violations
CONDITION = (
  SELECT COUNT(*)
  FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
  WHERE ERROR_MESSAGE ILIKE '%network policy%'
    AND EVENT_TIMESTAMP > DATEADD(hour, -1, CURRENT_TIMESTAMP)
) >= 3;
```

### Data Protection Conditions

```sql
-- Pattern: Unmasked restricted columns
CONDITION = (
  SELECT COUNT(*)
  FROM INFORMATION_SCHEMA.COLUMNS c
  JOIN SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES tr
    ON c.table_name = tr.object_name
    AND c.column_name = tr.column_name
  LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.MASKING_POLICIES mp
    ON c.column_name = mp.column_name
  WHERE tr.tag_name = 'data_classification'
    AND tr.tag_value = 'restricted'
    AND mp.policy_name IS NULL
) > 0;

-- Pattern: Unauthorized access to sensitive data
CONDITION = (
  SELECT COUNT(DISTINCT user_name)
  FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY ah
  JOIN SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES tr
    ON ah.object_name = tr.object_name
  WHERE tr.tag_value = 'restricted'
    AND ah.start_time > DATEADD(hour, -1, CURRENT_TIMESTAMP)
    AND ah.user_name NOT IN (
      SELECT grantee_name
      FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
      WHERE privilege = 'SELECT'
    )
) > 0;

-- Pattern: Policy evaluation errors
CONDITION = (
  SELECT COUNT(*)
  FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE ERROR_MESSAGE ILIKE '%masking policy%'
     OR ERROR_MESSAGE ILIKE '%row access policy%'
    AND START_TIME > DATEADD(hour, -1, CURRENT_TIMESTAMP)
) > 0;
```

### Compliance and Audit Conditions

```sql
-- Pattern: Missing retention policy tags
CONDITION = (
  SELECT COUNT(*)
  FROM INFORMATION_SCHEMA.TABLES
  WHERE table_schema NOT IN ('INFORMATION_SCHEMA', 'SNOWFLAKE')
    AND table_name NOT IN (
      SELECT object_name
      FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
      WHERE tag_name = 'retention_policy'
    )
) > 10;  -- Alert if more than 10 tables untagged

-- Pattern: Stale access reviews
CONDITION = (
  SELECT COUNT(*)
  FROM governance.policy_registry.definitions
  WHERE status = 'ACTIVE'
    AND next_review_date < CURRENT_DATE()
) > 0;

-- Pattern: External share usage anomalies
CONDITION = (
  SELECT COUNT(DISTINCT consumer_account)
  FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY
  WHERE object_name IN (
    SELECT name FROM SNOWFLAKE.ACCOUNT_USAGE.SHARES WHERE share_type = 'OUTBOUND'
  )
    AND start_time > DATEADD(hour, -1, CURRENT_TIMESTAMP)
) > (
  SELECT AVG(consumer_count) * 2  -- Alert if 2x normal usage
  FROM governance.share_usage_baselines
);
```

## Notification Actions

### Email Notifications

```sql
-- Basic email alert
ACTION = (
  SYSTEM$SEND_EMAIL(
    'recipient@company.com',
    'Subject: Alert - ' || CURRENT_TIMESTAMP(),
    'Body: Alert condition met. Review immediately.'
  )
);

-- Email with dynamic content from condition query
CREATE OR REPLACE ALERT governance.policy_gap_alert
  WAREHOUSE = governance_wh
  SCHEDULE = 'USING CRON 0 9 * * 1'
  CONDITION = (SELECT COUNT(*) FROM governance.monitor.unmasked_restricted) > 0
  ACTION = (
    SYSTEM$SEND_EMAIL(
      'security-team@company.com',
      'Alert: ' || (SELECT COUNT(*) FROM governance.monitor.unmasked_restricted) || ' unmasked restricted columns detected',
      'Columns requiring masking: ' || 
      (SELECT LISTAGG(table_schema || '.' || table_name || '.' || column_name, '\n')
       FROM governance.monitor.unmasked_restricted) ||
      '\n\nReview in Snowflake or respond to this email.'
    )
  );
```

| Email Parameter | Purpose | Example |
|----------------|---------|---------|
| Recipient | Email address or comma separated list | 'security@company.com, compliance@company.com' |
| Subject | Email subject line | 'Alert: Failed login attempts detected' |
| Body | Email content with optional dynamic data | 'Review LOGIN_HISTORY for IP: ' || client_ip |
| CC/BCC | Not directly supported; include in recipient list | Add additional addresses to recipient parameter |

### Slack and Teams Webhook Notifications

```sql
-- Slack message via webhook
ACTION = (
  SYSTEM$SEND_SLACK_MESSAGE(
    'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX',
    '🚨 *Security Alert*: Multiple failed login attempts\n' ||
    '• Time: ' || TO_VARCHAR(CURRENT_TIMESTAMP()) || '\n' ||
    '• Count: ' || (SELECT COUNT(*) FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY WHERE SUCCESS = ''NO'' AND EVENT_TIMESTAMP > DATEADD(minute, -15, CURRENT_TIMESTAMP())) || '\n' ||
    '• Action: Review LOGIN_HISTORY view'
  )
);

-- Teams message via webhook (same function, different payload format)
ACTION = (
  SYSTEM$SEND_SLACK_MESSAGE(
    'https://company.webhook.office.com/webhookb2/XXX/IncomingWebhook/YYY',
    '{
      "@type": "MessageCard",
      "@context": "https://schema.org/extensions",
      "summary": "Governance Alert",
      "sections": [{
        "activityTitle": "Unmasked Restricted Data Detected",
        "facts": [{
          "name": "Count",
          "value": "' || (SELECT COUNT(*) FROM governance.monitor.unmasked_restricted) || '"
        }],
        "potentialAction": [{
          "@type": "OpenUri",
          "name": "View in Snowflake",
          "targets": [{"os": "default", "uri": "https://app.snowflake.com"}]
        }]
      }]
    }'
  )
);
```

| Webhook Parameter | Purpose | Notes |
|------------------|---------|-------|
| Webhook URL | External endpoint that receives the message | Must be pre configured in Slack/Teams |
| Message payload | Text or JSON formatted content | Slack accepts plain text; Teams may require JSON card format |
| Dynamic content | Embed query results in message | Use string concatenation to include alert details |

### Triggering Downstream Tasks

```sql
-- Alert that triggers a remediation task
CREATE OR REPLACE ALERT governance.auto_remediate_masking
  WAREHOUSE = governance_wh
  SCHEDULE = 'USING CRON 0 10 * * 1'
  CONDITION = (SELECT COUNT(*) FROM governance.monitor.unmasked_restricted) > 0
  ACTION = (
    -- Log the alert first
    INSERT INTO governance.alert_audit (alert_name, triggered_at, row_count)
    SELECT 'auto_remediate_masking', CURRENT_TIMESTAMP(), COUNT(*)
    FROM governance.monitor.unmasked_restricted;
    
    -- Trigger remediation procedure
    CALL governance.apply_masking_by_tag();
    
    -- Notify that remediation ran
    SYSTEM$SEND_EMAIL(
      'security-team@company.com',
      'Auto-remediation executed for unmasked restricted columns',
      'Procedure governance.apply_masking_by_tag() executed at ' || CURRENT_TIMESTAMP()
    )
  );
```

### Logging to Audit Tables

```sql
-- Create audit table for alert history
CREATE OR REPLACE TABLE governance.alert_execution_log (
  alert_name STRING,
  triggered_at TIMESTAMP_LTZ,
  condition_row_count NUMBER,
  action_status STRING,  -- SUCCESS, FAILED, SKIPPED
  error_message STRING,
  executed_by STRING DEFAULT CURRENT_USER()
);

-- Alert that logs execution details
CREATE OR REPLACE ALERT ops.query_cost_alert
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON 0 */6 * * *'
  CONDITION = (
    SELECT SUM(credits_used)
    FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE WAREHOUSE_NAME = 'PROD_WH'
      AND START_TIME > DATEADD(hour, -6, CURRENT_TIMESTAMP)
  ) > 100  -- Alert if > 100 credits in 6 hours
  ACTION = (
    -- Log alert execution
    INSERT INTO governance.alert_execution_log (
      alert_name, triggered_at, condition_row_count, action_status
    ) VALUES (
      'query_cost_alert',
      CURRENT_TIMESTAMP(),
      (SELECT SUM(credits_used) FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY WHERE WAREHOUSE_NAME = 'PROD_WH' AND START_TIME > DATEADD(hour, -6, CURRENT_TIMESTAMP())),
      'email_sent'
    );
    
    -- Send notification
    SYSTEM$SEND_EMAIL(
      'finance-team@company.com',
      'Alert: High compute cost detected on PROD_WH',
      'Credits used in last 6 hours: ' || 
      (SELECT SUM(credits_used) FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY WHERE WAREHOUSE_NAME = 'PROD_WH' AND START_TIME > DATEADD(hour, -6, CURRENT_TIMESTAMP()))
    )
  );
```

## External Integrations for Notifications

### PagerDuty Integration via Webhook

```sql
-- Alert that triggers PagerDuty incident
CREATE OR REPLACE ALERT security.critical_breach_alert
  WAREHOUSE = security_wh
  SCHEDULE = 'USING CRON */5 * * * *'
  CONDITION = (
    SELECT COUNT(*)
    FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY
    WHERE object_name IN (
      SELECT object_name FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
      WHERE tag_name = 'data_classification' AND tag_value = 'restricted'
    )
    AND user_name NOT IN (SELECT grantee_name FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES WHERE privilege = 'SELECT')
    AND start_time > DATEADD(minute, -5, CURRENT_TIMESTAMP)
  ) > 0
  ACTION = (
    SYSTEM$HTTP_POST(
      'https://events.pagerduty.com/v2/enqueue',
      OBJECT_CONSTRUCT(
        'routing_key', 'your_pagerduty_integration_key',
        'event_action', 'trigger',
        'payload', OBJECT_CONSTRUCT(
          'summary', 'Critical: Unauthorized access to restricted data',
          'source', 'snowflake',
          'severity', 'critical',
          'custom_details', OBJECT_CONSTRUCT(
            'alert_name', 'critical_breach_alert',
            'triggered_at', CURRENT_TIMESTAMP(),
            'unauthorized_users', (SELECT LISTAGG(DISTINCT user_name, ', ') FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY WHERE ...)
          )
        )
      )::STRING,
      OBJECT_CONSTRUCT('Content-Type', 'application/json')
    )
  );
```

### SIEM Integration via External Stage

```sql
-- Create external stage for SIEM export
CREATE OR REPLACE EXTERNAL STAGE siem_exports
  URL = 's3://siem-bucket/snowflake-alerts/'
  CREDENTIALS = (AWS_KEY_ID = '***' AWS_SECRET_KEY = '***')
  FILE_FORMAT = (TYPE = JSON);

-- Alert that exports to SIEM
CREATE OR REPLACE ALERT compliance.audit_export_alert
  WAREHOUSE = compliance_wh
  SCHEDULE = 'USING CRON 0 0 * * *'  -- Daily at midnight
  CONDITION = (
    SELECT COUNT(*)
    FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY
    WHERE start_time > DATEADD(day, -1, CURRENT_TIMESTAMP)
      AND object_name IN (
        SELECT object_name FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
        WHERE tag_name = 'data_classification' AND tag_value IN ('confidential', 'restricted')
      )
  ) > 0
  ACTION = (
    -- Export access history to SIEM
    COPY INTO @siem_exports/restricted_access/
    FROM (
      SELECT
        event_timestamp,
        user_name,
        object_name,
        query_text,
        bytes_scanned,
        'restricted_data_access' as alert_type
      FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY
      WHERE start_time > DATEADD(day, -1, CURRENT_TIMESTAMP)
        AND object_name IN (
          SELECT object_name FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
          WHERE tag_name = 'data_classification' AND tag_value IN ('confidential', 'restricted')
        )
    )
    FILE_FORMAT = (TYPE = JSON);
    
    -- Notify that export completed
    SYSTEM$SEND_EMAIL(
      'compliance-team@company.com',
      'Daily restricted access audit exported to SIEM',
      'Export completed at ' || CURRENT_TIMESTAMP()
    )
  );
```

### Custom API Integration

```sql
-- Alert that calls custom governance API
CREATE OR REPLACE ALERT governance.policy_violation_api
  WAREHOUSE = governance_wh
  SCHEDULE = 'USING CRON 0 9 * * 1-5'
  CONDITION = (SELECT COUNT(*) FROM governance.monitor.policy_violations) > 0
  ACTION = (
    SYSTEM$HTTP_POST(
      'https://governance-api.company.com/alerts',
      OBJECT_CONSTRUCT(
        'alert_type', 'policy_violation',
        'severity', 'medium',
        'details', (
          SELECT ARRAY_AGG(OBJECT_CONSTRUCT(
            'table', table_name,
            'column', column_name,
            'issue', issue
          ))
          FROM governance.monitor.policy_violations
        ),
        'timestamp', CURRENT_TIMESTAMP()
      )::STRING,
      OBJECT_CONSTRUCT(
        'Content-Type', 'application/json',
        'Authorization', 'Bearer ' || (SELECT api_token FROM governance.secrets WHERE name = 'governance_api')
      )
    )
  );
```

## Alert Lifecycle Management

### Alert Registry and Documentation

```sql
-- Create alert registry schema
CREATE SCHEMA IF NOT EXISTS governance.alert_registry
  COMMENT = 'Central registry for all governance alerts';

-- Create alert definition table
CREATE TABLE IF NOT EXISTS governance.alert_registry.definitions (
  alert_name STRING NOT NULL,
  alert_type STRING NOT NULL,  -- SECURITY, COMPLIANCE, OPERATIONAL, COST
  description STRING,
  owner_role STRING,
  severity STRING,  -- LOW, MEDIUM, HIGH, CRITICAL
  condition_summary STRING,
  action_summary STRING,
  schedule_cron STRING,
  warehouse_name STRING,
  created_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
  created_by STRING DEFAULT CURRENT_USER(),
  last_modified_date TIMESTAMP,
  last_modified_by STRING,
  status STRING DEFAULT 'ACTIVE',  -- ACTIVE, PAUSED, DEPRECATED
  next_review_date DATE,
  false_positive_rate FLOAT,  -- Track alert quality
  mean_time_to_resolve FLOAT,  -- Track response efficiency
  CONSTRAINT pk_alert PRIMARY KEY (alert_name)
);

-- Register a new alert
INSERT INTO governance.alert_registry.definitions (
  alert_name, alert_type, description, owner_role, severity,
  condition_summary, action_summary, schedule_cron, warehouse_name, next_review_date
) VALUES (
  'failed_login_alert',
  'SECURITY',
  'Alert on multiple failed login attempts indicating potential brute force',
  'SECURITYADMIN',
  'HIGH',
  'COUNT of failed logins > 5 in 15 minute window',
  'Email security team with details',
  '*/15 * * * *',
  'admin_wh',
  DATEADD(month, 3, CURRENT_DATE())
);
```

### Alert Testing and Validation

```sql
-- Test alert condition without triggering action
-- Run the condition query manually to verify logic
SELECT COUNT(*) as failed_logins
FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE SUCCESS = 'NO'
  AND EVENT_TIMESTAMP > DATEADD(minute, -15, CURRENT_TIMESTAMP);

-- Test alert with synthetic data
-- Create test scenario that should trigger alert
INSERT INTO governance.test_login_history (event_timestamp, user_name, success, client_ip)
VALUES
  (DATEADD(minute, -5, CURRENT_TIMESTAMP()), 'test_user', 'NO', '192.168.1.100'),
  (DATEADD(minute, -4, CURRENT_TIMESTAMP()), 'test_user', 'NO', '192.168.1.100'),
  (DATEADD(minute, -3, CURRENT_TIMESTAMP()), 'test_user', 'NO', '192.168.1.100'),
  (DATEADD(minute, -2, CURRENT_TIMESTAMP()), 'test_user', 'NO', '192.168.1.100'),
  (DATEADD(minute, -1, CURRENT_TIMESTAMP()), 'test_user', 'NO', '192.168.1.100'),
  (CURRENT_TIMESTAMP(), 'test_user', 'NO', '192.168.1.100');

-- Run alert condition to verify it triggers
SELECT COUNT(*) >= 5 as should_trigger
FROM governance.test_login_history
WHERE success = 'NO'
  AND event_timestamp > DATEADD(minute, -15, CURRENT_TIMESTAMP());

-- Clean up test data
DELETE FROM governance.test_login_history;
```

### Alert Performance Monitoring

```sql
-- Query to monitor alert execution history
SELECT
  alert_name,
  scheduled_time,
  actual_execution_time,
  condition_evaluation_time_ms,
  action_execution_time_ms,
  status,
  error_message
FROM TABLE(INFORMATION_SCHEMA.ALERT_HISTORY())
WHERE scheduled_time > DATEADD(day, -7, CURRENT_TIMESTAMP())
ORDER BY scheduled_time DESC;

-- Query to identify alerts with high false positive rates
SELECT
  d.alert_name,
  d.severity,
  COUNT(CASE WHEN h.status = 'TRIGGERED' THEN 1 END) as times_triggered,
  COUNT(CASE WHEN h.status = 'TRIGGERED' AND a.resolved_as = 'false_positive' THEN 1 END) as false_positives,
  ROUND(100.0 * COUNT(CASE WHEN h.status = 'TRIGGERED' AND a.resolved_as = 'false_positive' THEN 1 END) / 
        NULLIF(COUNT(CASE WHEN h.status = 'TRIGGERED' THEN 1 END), 0), 2) as false_positive_rate
FROM governance.alert_registry.definitions d
LEFT JOIN TABLE(INFORMATION_SCHEMA.ALERT_HISTORY()) h
  ON d.alert_name = h.alert_name
LEFT JOIN governance.alert_resolution_log a
  ON h.alert_history_id = a.alert_history_id
WHERE h.scheduled_time > DATEADD(month, -1, CURRENT_TIMESTAMP())
GROUP BY d.alert_name, d.severity
HAVING COUNT(CASE WHEN h.status = 'TRIGGERED' THEN 1 END) > 0
ORDER BY false_positive_rate DESC;
```

### Alert Review and Retirement

```sql
-- Query to identify alerts needing review
SELECT
  alert_name,
  alert_type,
  severity,
  next_review_date,
  DATEDIFF(day, CURRENT_DATE(), next_review_date) as days_until_review,
  status
FROM governance.alert_registry.definitions
WHERE status = 'ACTIVE'
  AND next_review_date <= DATEADD(month, 1, CURRENT_DATE())
ORDER BY next_review_date;

-- Query to identify unused alerts for retirement consideration
SELECT
  d.alert_name,
  d.alert_type,
  COUNT(h.alert_history_id) as executions_last_90_days,
  COUNT(CASE WHEN h.status = 'TRIGGERED' THEN 1 END) as times_triggered,
  MAX(h.scheduled_time) as last_execution
FROM governance.alert_registry.definitions d
LEFT JOIN TABLE(INFORMATION_SCHEMA.ALERT_HISTORY()) h
  ON d.alert_name = h.alert_name
  AND h.scheduled_time > DATEADD(day, -90, CURRENT_TIMESTAMP())
WHERE d.status = 'ACTIVE'
GROUP BY d.alert_name, d.alert_type
HAVING COUNT(h.alert_history_id) = 0
   OR COUNT(CASE WHEN h.status = 'TRIGGERED' THEN 1 END) = 0
ORDER BY last_execution;

-- Pause an alert (soft disable)
ALTER ALERT security.failed_login_alert SUSPEND;

-- Update alert registry to reflect pause
UPDATE governance.alert_registry.definitions
SET
  status = 'PAUSED',
  last_modified_date = CURRENT_TIMESTAMP(),
  last_modified_by = CURRENT_USER()
WHERE alert_name = 'failed_login_alert';

-- Resume an alert
ALTER ALERT security.failed_login_alert RESUME;

-- Deprecate and eventually drop an alert
UPDATE governance.alert_registry.definitions
SET
  status = 'DEPRECATED',
  last_modified_date = CURRENT_TIMESTAMP()
WHERE alert_name = 'legacy_policy_alert';

-- After 30 days, drop the deprecated alert
DROP ALERT governance.legacy_policy_alert;
```

## Best Practices for Alert Design

### Threshold Tuning

| Practice | Why It Matters | Implementation |
|----------|---------------|---------------|
| Start conservative | Avoid alert fatigue from false positives | Begin with higher thresholds, lower based on actual patterns |
| Use time windows | Prevent alert storms from sustained issues | Filter condition queries with EVENT_TIMESTAMP > DATEADD(...) |
| Baseline normal behavior | Alert on anomalies not absolute values | Compare to historical averages or percentiles |
| Tier by severity | Route critical alerts to on call, low priority to digest | Use severity field to determine notification channel |
| Document rationale | Future you needs to know why threshold was chosen | Add COMMENT with threshold justification |

```sql
-- Example: Baseline-based alert
CREATE OR REPLACE ALERT ops.anomalous_query_volume
  WAREHOUSE = ops_wh
  SCHEDULE = 'USING CRON 0 */6 * * *'
  CONDITION = (
    SELECT SUM(credits_used)
    FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE WAREHOUSE_NAME = 'PROD_WH'
      AND START_TIME > DATEADD(hour, -6, CURRENT_TIMESTAMP)
  ) > (
    -- Alert if current 6-hour usage exceeds 2x the 7-day average
    SELECT AVG(six_hour_credits) * 2
    FROM (
      SELECT 
        DATE_TRUNC('hour', START_TIME) as hour_bucket,
        SUM(credits_used) as six_hour_credits
      FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
      WHERE WAREHOUSE_NAME = 'PROD_WH'
        AND START_TIME > DATEADD(day, -7, CURRENT_TIMESTAMP)
      GROUP BY DATE_TRUNC('hour', START_TIME)
    )
  )
  ACTION = (
    SYSTEM$SEND_EMAIL(
      'ops-team@company.com',
      'Alert: Anomalous query volume on PROD_WH',
      'Current 6-hour usage exceeds 2x the 7-day average. Investigate.'
    )
  );
```

### Avoiding Alert Fatigue

| Strategy | Implementation | Expected Outcome |
|----------|---------------|-----------------|
| Deduplicate alerts | Track last alert time; suppress repeats within window | Reduce noise from sustained conditions |
| Aggregate related alerts | Combine multiple conditions into single alert | Fewer notifications, clearer signal |
| Use digest emails | Send daily summary instead of real time for low priority | Reduce interruption for non critical issues |
| Escalate by severity | Critical alerts page on call; low priority goes to ticket | Right alert to right person at right time |
| Review and tune quarterly | Adjust thresholds based on false positive rates | Maintain alert relevance over time |

```sql
-- Example: Deduplication via state tracking
CREATE OR REPLACE TABLE governance.alert_state (
  alert_name STRING,
  last_triggered TIMESTAMP_LTZ,
  suppression_window_minutes NUMBER DEFAULT 60,
  CONSTRAINT pk_alert_state PRIMARY KEY (alert_name)
);

CREATE OR REPLACE ALERT security.deduped_failed_login_alert
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON */15 * * * *'
  CONDITION = (
    -- Only trigger if condition met AND not suppressed
    SELECT COUNT(*)
    FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
    WHERE SUCCESS = 'NO'
      AND EVENT_TIMESTAMP > DATEADD(minute, -15, CURRENT_TIMESTAMP)
  ) > 5
  AND NOT EXISTS (
    SELECT 1
    FROM governance.alert_state s
    WHERE s.alert_name = 'failed_login_alert'
      AND s.last_triggered > DATEADD(minute, -s.suppression_window_minutes, CURRENT_TIMESTAMP())
  )
  ACTION = (
    -- Send alert
    SYSTEM$SEND_EMAIL('security-team@company.com', 'Alert: Failed logins', 'Review LOGIN_HISTORY');
    
    -- Update state to suppress repeats
    MERGE INTO governance.alert_state s
    USING (SELECT 'failed_login_alert' as alert_name) src
    ON s.alert_name = src.alert_name
    WHEN MATCHED THEN UPDATE SET last_triggered = CURRENT_TIMESTAMP()
    WHEN NOT MATCHED THEN INSERT (alert_name, last_triggered) VALUES ('failed_login_alert', CURRENT_TIMESTAMP())
  );
```

### Alert Testing Checklist

```mermaid
flowchart TD
  Start[New Alert Created] --> Unit[Test condition query logic]
  Unit --> Threshold[Verify threshold triggers correctly]
  Threshold --> Action[Test action executes as expected]
  Action --> Schedule[Confirm schedule evaluates at expected times]
  Schedule --> Integration[Verify external integrations work]
  Integration --> Document[Update alert registry with test results]
  Document --> Monitor[Enable monitoring and review after 30 days]
```

```sql
-- Example: Comprehensive alert test script
-- File: test_failed_login_alert.sql

-- Step 1: Verify condition query returns expected rows
SELECT 'Condition test' as test_step,
       CASE WHEN COUNT(*) >= 5 THEN 'PASS' ELSE 'FAIL' END as result
FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE SUCCESS = 'NO'
  AND EVENT_TIMESTAMP > DATEADD(minute, -15, CURRENT_TIMESTAMP());

-- Step 2: Verify action can execute (test email send)
-- Note: SYSTEM$SEND_EMAIL returns success/failure status
SELECT SYSTEM$SEND_EMAIL(
  'test-recipient@company.com',
  'Test: Failed login alert',
  'This is a test alert. No action required.'
) as email_send_result;

-- Step 3: Verify alert is registered correctly
SELECT alert_name, status, schedule, warehouse
FROM TABLE(INFORMATION_SCHEMA.ALERTS())
WHERE name = 'FAILED_LOGIN_ALERT';

-- Step 4: Document test results
INSERT INTO governance.alert_test_log (
  alert_name, test_date, tester, condition_result, action_result, overall_status
) VALUES (
  'failed_login_alert',
  CURRENT_TIMESTAMP(),
  CURRENT_USER(),
  'PASS',  -- or 'FAIL' based on step 1
  'PASS',  -- or 'FAIL' based on step 2
  'PASS'   -- or 'FAIL' based on all steps
);
```

## Common Alert Pitfalls

| Pitfall | What Happens | How To Fix |
|---------|--------------|------------|
| Too sensitive thresholds | Alert fires constantly causing fatigue | Start with conservative thresholds; tune based on baseline |
| No time window filtering | Alert evaluates entire history every run | Always filter condition queries with EVENT_TIMESTAMP > DATEADD(...) |
| Action failures silently | Alert condition met but notification never sent | Log action execution status; monitor alert history |
| Alert without documentation | Future team cannot understand alert purpose | Add COMMENT and register in alert_registry |
| Testing only happy path | Alert fails in production edge cases | Test condition, action, schedule, and integrations |
| No suppression logic | Sustained issue triggers alert every schedule interval | Implement deduplication via alert_state tracking |
| Ignoring alert history | Cannot measure alert effectiveness or false positive rate | Query ALERT_HISTORY view; track resolution outcomes |
| Over alerting on low priority | Critical alerts buried in noise | Tier by severity; route to appropriate channels |

```mermaid
flowchart TD
  Prob[Alert issue] --> Q1[Alert not firing when expected]
  Prob --> Q2[Alert firing too frequently]
  Prob --> Q3[Action not executing]
  
  Q1 --> A[Verify condition query returns rows]
  Q1 --> B[Check alert status is RESUME not SUSPEND]
  Q1 --> C[Confirm schedule CRON expression is correct]
  
  Q2 --> D[Review threshold logic and baseline]
  Q2 --> E[Add time window filtering to condition]
  Q2 --> F[Implement deduplication suppression]
  
  Q3 --> G[Check warehouse is running and sized appropriately]
  Q3 --> H[Verify external integration credentials are valid]
  Q3 --> I[Log action execution status for debugging]
  
  A --> J[Test with representative data]
  B --> J
  C --> J
  D --> J
  E --> J
  F --> J
  G --> J
  H --> J
  I --> J
```

## Decision Framework for Alert Design

```mermaid
flowchart TD
  Q1[Start: Design new alert]
  Q1 --> Q2[What is the alert purpose]
  Q2 --> Q3[Security incident detection]
  Q2 --> Q4[Compliance policy enforcement]
  Q2 --> Q5[Operational monitoring]
  Q2 --> Q6[Cost anomaly detection]
  
  Q3 --> A[High severity, frequent schedule, page on call]
  Q4 --> B[Medium severity, daily schedule, email digest]
  Q5 --> C[Low severity, hourly schedule, Slack channel]
  Q6 --> D[Medium severity, 6-hour schedule, ticket system]
  
  A --> E[Condition: real time indicators with tight thresholds]
  B --> F[Condition: policy gap queries with moderate thresholds]
  C --> G[Condition: operational metrics with baseline comparison]
  D --> H[Condition: cost aggregation with anomaly detection]
  
  E --> I[Action: email + PagerDuty + audit log]
  F --> J[Action: email digest + registry update]
  G --> K[Action: Slack message + dashboard update]
  H --> L[Action: ticket creation + finance notification]
  
  I --> M[Test condition action and integrations]
  J --> M
  K --> M
  L --> M
  M --> N[Register in alert_registry with documentation]
  N --> O[Monitor for 30 days then tune thresholds]
```

| Question | If Yes | If No |
|----------|--------|-------|
| Is this a security incident alert | High severity, frequent schedule, page on call | Consider lower severity or different channel |
| Does alert require immediate action | Trigger PagerDuty or SMS | Email or Slack digest may suffice |
| Is condition based on real time data | Use short time windows (5-15 min) | Longer windows (1-6 hours) may be appropriate |
| Will alert fire frequently | Implement deduplication suppression | Standard alert logic may suffice |
| Does alert integrate with external systems | Test webhook/API credentials thoroughly | Internal actions (email/log) are simpler to validate |

## Key Principles to Remember

- Alerts are scheduled not real time. Design condition queries with appropriate time windows
- Thresholds require tuning. Start conservative and adjust based on actual patterns
- Actions can fail. Log execution status and monitor alert history for reliability
- Documentation enables maintenance. Register alerts with purpose owner and review date
- Testing prevents production failures. Verify condition action schedule and integrations
- Suppression reduces fatigue. Implement deduplication for sustained conditions
- Review quarterly. Alert relevance drifts as systems and threats evolve

## Bottom Line

- Alerts in Snowflake enable proactive governance through automated monitoring
- The ALERT object combines scheduled SQL conditions with configurable actions
- Condition queries should filter by time window and use thresholds based on baselines
- Actions can notify via email Slack webhooks or trigger downstream tasks
- External integrations extend alerts to PagerDuty SIEM or custom governance APIs
- Alert lifecycle management includes testing documentation monitoring and quarterly review
- Avoid alert fatigue through conservative thresholds deduplication and severity tiering
- Measure alert effectiveness through false positive rates and mean time to resolve

Think of alerts like a security system for your data:
- Condition queries are the sensors. They detect when something unusual happens
- Thresholds are the sensitivity settings. Too sensitive and you get false alarms. Too loose and you miss real threats
- Actions are the response. Email Slack or PagerDuty depending on how urgent the alert is
- Schedule is the patrol frequency. High risk areas get checked more often
- Documentation is the system manual. Future you needs to know what each alert does and why
- Review is the system maintenance. Sensors drift. Threats evolve. Your alerts should too

Design alerts that catch what matters. Suppress what does not. Document why each alert exists. Review and tune regularly. That is how alerts work in Snowflake governance.
