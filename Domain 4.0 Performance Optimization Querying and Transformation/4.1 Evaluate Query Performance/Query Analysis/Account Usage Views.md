# Account Usage Views for Query Analysis in Snowflake

## Overview of Account Usage Views

Account Usage views in Snowflake provide historical data about queries, warehouses, storage, and other account-level activities. These views are part of the SNOWFLAKE.ACCOUNT_USAGE schema and are designed for monitoring, troubleshooting, and analyzing workload performance over time.

### Key Characteristics of Account Usage Views

1. **Historical Data**: Account Usage views retain data for 365 days (1 year), enabling long-term trend analysis and historical comparisons.

2. **Account-Level Scope**: These views provide data across the entire Snowflake account, not limited to a specific database or schema.

3. **Read-Only**: Account Usage views are read-only and cannot be modified.

4. **Shared Data**: The data in these views is shared across all users with appropriate privileges in the account.

5. **Latency**: Data in Account Usage views may have a latency of up to 3 hours. For real-time monitoring, use the corresponding Information Schema views.

6. **Access Control**: Access to Account Usage views requires the ACCOUNTADMIN role or a custom role with the IMPORTED PRIVILEGES privilege.

### Account Usage vs. Information Schema Views

| Feature | Account Usage Views | Information Schema Views |
|---------|---------------------|---------------------------|
| Data Retention | 365 days | Session duration |
| Scope | Account-level | Session-level |
| Latency | Up to 3 hours | Real-time |
| Use Case | Historical analysis, trend monitoring | Real-time monitoring, current session analysis |
| Access | ACCOUNTADMIN or custom role with IMPORTED PRIVILEGES | Any role with USAGE privilege on the database |
| Example | SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY | SNOWFLAKE.INFORMATION_SCHEMA.QUERY_HISTORY |

## Enabling Access to Account Usage Views

By default, Account Usage views are not visible to all users. To enable access:

```sql
-- Grant IMPORTED PRIVILEGES to a role
GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE my_analyst_role;

-- Grant the role to users
GRANT ROLE my_analyst_role TO USER analyst1;
GRANT ROLE my_analyst_role TO USER analyst2;
```

Alternatively, users with the ACCOUNTADMIN role can access all Account Usage views.

## Complete Reference: Account Usage Views for Query Analysis

### 1. QUERY_HISTORY

**Purpose**: Provides detailed historical information about all queries executed in the account.

**Retention**: 365 days

**Key Columns**:
- QUERY_ID: Unique identifier for the query
- QUERY_TEXT: The SQL text of the query
- DATABASE_NAME: Name of the database used
- SCHEMA_NAME: Name of the schema used
- WAREHOUSE_NAME: Name of the warehouse used
- WAREHOUSE_SIZE: Size of the warehouse
- USER_NAME: Name of the user who executed the query
- ROLE_NAME: Name of the role used
- START_TIME: Timestamp when the query started
- END_TIME: Timestamp when the query ended
- EXECUTION_STATUS: Status of the query (RUNNING, SUCCESS, FAILED, etc.)
- ERROR_MESSAGE: Error message if the query failed
- ERROR_NUMBER: Error number if the query failed
- BYTES_SCANNED: Number of bytes scanned
- PARTITIONS_SCANNED: Number of partitions scanned
- ROWS_PRODUCED: Number of rows produced
- CREDITS_USED: Number of credits used
- EXECUTION_TIME: Execution time in milliseconds
- QUEUE_TIME: Time spent in queue in milliseconds
- COMPILATION_TIME: Time spent compiling the query in milliseconds
- TOTAL_ELAPSED_TIME: Total elapsed time in milliseconds
- WAREHOUSE_TYPE: Type of warehouse (STANDARD, MULTI_CLUSTER)
- QUERY_TAG: Query tag if specified
- QUERY_TYPE: Type of query (SELECT, INSERT, UPDATE, DELETE, etc.)
- SESSION_ID: ID of the session
- PRIORITY: Query priority (HIGH, MEDIUM, LOW)
- RESOURCE_MONITOR: Name of the resource monitor if assigned

**Example Queries**:

```sql
-- Get all queries from the last 24 hours
SELECT
    query_id,
    query_text,
    warehouse_name,
    user_name,
    start_time,
    end_time,
    execution_status,
    credits_used,
    execution_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;

-- Get slow queries (>10 seconds) from the last 7 days
SELECT
    query_id,
    query_text,
    warehouse_name,
    user_name,
    start_time,
    execution_time,
    bytes_scanned,
    credits_used
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    execution_time > 10000  -- 10 seconds in milliseconds
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    execution_time DESC;

-- Get queries with high bytes scanned (>1GB) from the last 7 days
SELECT
    query_id,
    query_text,
    warehouse_name,
    user_name,
    start_time,
    bytes_scanned / 1024 / 1024 / 1024 AS bytes_scanned_gb,
    credits_used
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    bytes_scanned > 1024 * 1024 * 1024  -- 1GB
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    bytes_scanned DESC;

-- Get failed queries from the last 7 days
SELECT
    query_id,
    query_text,
    warehouse_name,
    user_name,
    start_time,
    error_message,
    error_number
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    execution_status = 'FAILED'
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;

-- Get queries by user with credit usage
SELECT
    user_name,
    COUNT(*) AS query_count,
    SUM(credits_used) AS total_credits_used,
    AVG(credits_used) AS avg_credits_per_query,
    SUM(credits_used) * 0.028 AS estimated_cost_usd  -- Assuming $0.028 per credit
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('month', -1, CURRENT_TIMESTAMP())
GROUP BY
    user_name
ORDER BY
    total_credits_used DESC;

-- Get queries by warehouse with credit usage
SELECT
    warehouse_name,
    COUNT(*) AS query_count,
    SUM(credits_used) AS total_credits_used,
    AVG(credits_used) AS avg_credits_per_query,
    SUM(credits_used) * 0.028 AS estimated_cost_usd
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('month', -1, CURRENT_TIMESTAMP())
GROUP BY
    warehouse_name
ORDER BY
    total_credits_used DESC;

-- Get queries by query tag
SELECT
    query_tag,
    COUNT(*) AS query_count,
    SUM(credits_used) AS total_credits_used,
    AVG(credits_used) AS avg_credits_per_query
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_tag IS NOT NULL
    AND start_time > DATEADD('month', -1, CURRENT_TIMESTAMP())
GROUP BY
    query_tag
ORDER BY
    total_credits_used DESC;

-- Get queries with high queue time (>1 minute)
SELECT
    query_id,
    query_text,
    warehouse_name,
    user_name,
    start_time,
    queue_time / 1000 AS queue_time_seconds,
    execution_time / 1000 AS execution_time_seconds
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    queue_time > 60000  -- 1 minute in milliseconds
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    queue_time DESC;
```

**Best Practices for QUERY_HISTORY**:
1. Use START_TIME and END_TIME filters to limit the time range and improve query performance.
2. Filter by specific columns (e.g., WAREHOUSE_NAME, USER_NAME) to reduce the result set size.
3. Use aggregation to summarize data when analyzing large time ranges.
4. Combine with other views (e.g., WAREHOUSE_LOAD_HISTORY) for comprehensive analysis.
5. Consider creating materialized views for frequently run queries against QUERY_HISTORY.

---

### 2. WAREHOUSE_LOAD_HISTORY

**Purpose**: Provides historical information about warehouse utilization, including running and queued queries.

**Retention**: 365 days

**Key Columns**:
- WAREHOUSE_NAME: Name of the warehouse
- WAREHOUSE_SIZE: Size of the warehouse
- START_TIME: Timestamp when the load sample was taken
- END_TIME: Timestamp when the load sample ended
- RUNNING_QUERIES: Number of queries currently running
- QUEUED_QUERIES: Number of queries currently queued
- CREDIT_USAGE: Number of credits used during the sample period

**Example Queries**:

```sql
-- Get warehouse load history for the last 24 hours
SELECT
    warehouse_name,
    warehouse_size,
    start_time,
    end_time,
    running_queries,
    queued_queries,
    credit_usage,
    DATEDIFF('second', start_time, end_time) AS sample_duration_seconds
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE
    start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;

-- Get average warehouse utilization for the last 7 days
SELECT
    warehouse_name,
    warehouse_size,
    AVG(running_queries) AS avg_running_queries,
    AVG(queued_queries) AS avg_queued_queries,
    AVG(credit_usage) AS avg_credit_usage,
    MAX(running_queries) AS max_running_queries,
    MAX(queued_queries) AS max_queued_queries
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    warehouse_name, warehouse_size
ORDER BY
    avg_credit_usage DESC;

-- Get warehouse load by hour for the last 7 days
SELECT
    warehouse_name,
    HOUR(start_time) AS hour_of_day,
    AVG(running_queries) AS avg_running_queries,
    AVG(queued_queries) AS avg_queued_queries,
    AVG(credit_usage) AS avg_credit_usage
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    warehouse_name, HOUR(start_time)
ORDER BY
    warehouse_name, hour_of_day;

-- Get peak warehouse utilization for each warehouse
SELECT
    warehouse_name,
    MAX(running_queries) AS peak_running_queries,
    MAX(queued_queries) AS peak_queued_queries,
    MAX(credit_usage) AS peak_credit_usage,
    start_time AS peak_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    warehouse_name, start_time
ORDER BY
    peak_credit_usage DESC;

-- Get warehouses with high queue times
SELECT
    warehouse_name,
    start_time,
    running_queries,
    queued_queries,
    credit_usage
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE
    queued_queries > 0
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    queued_queries DESC, start_time DESC;
```

**Best Practices for WAREHOUSE_LOAD_HISTORY**:
1. Use this view to identify periods of high warehouse utilization.
2. Correlate with QUERY_HISTORY to understand which queries are causing high load.
3. Use the data to right-size warehouses and determine if multi-cluster warehouses are needed.
4. Monitor queued_queries to identify resource contention.
5. Set up alerts for warehouses that frequently have queued queries.

---

### 3. WAREHOUSE_METERING_HISTORY

**Purpose**: Provides detailed historical information about credit usage by warehouse.

**Retention**: 365 days

**Key Columns**:
- WAREHOUSE_NAME: Name of the warehouse
- WAREHOUSE_SIZE: Size of the warehouse
- START_TIME: Timestamp when the metering period started
- END_TIME: Timestamp when the metering period ended
- CREDITS_USED: Number of credits used during the period
- QUERY_TYPE: Type of query (SELECT, INSERT, etc.)
- SERVICE_TYPE: Type of service (QUERY, LOAD, etc.)

**Example Queries**:

```sql
-- Get credit usage by warehouse for the last 7 days
SELECT
    warehouse_name,
    warehouse_size,
    SUM(credits_used) AS total_credits_used,
    COUNT(*) AS metering_periods,
    SUM(credits_used) / COUNT(*) AS avg_credits_per_period
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    warehouse_name, warehouse_size
ORDER BY
    total_credits_used DESC;

-- Get daily credit usage by warehouse for the last 30 days
SELECT
    warehouse_name,
    DATE_TRUNC('DAY', start_time) AS day,
    SUM(credits_used) AS daily_credits_used,
    SUM(credits_used) * 0.028 AS estimated_daily_cost_usd  -- Assuming $0.028 per credit
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE
    start_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY
    warehouse_name, DATE_TRUNC('DAY', start_time)
ORDER BY
    day DESC, warehouse_name;

-- Get credit usage by query type for the last 7 days
SELECT
    warehouse_name,
    query_type,
    SUM(credits_used) AS total_credits_used,
    COUNT(*) AS query_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    warehouse_name, query_type
ORDER BY
    warehouse_name, total_credits_used DESC;

-- Get hourly credit usage for a specific warehouse
SELECT
    warehouse_name,
    HOUR(start_time) AS hour_of_day,
    SUM(credits_used) AS hourly_credits_used
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE
    warehouse_name = 'MY_WH'
    AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
GROUP BY
    warehouse_name, HOUR(start_time)
ORDER BY
    hour_of_day;

-- Get credit usage trend for the last 30 days
SELECT
    DATE_TRUNC('DAY', start_time) AS day,
    SUM(credits_used) AS daily_credits_used,
    SUM(credits_used) - LAG(SUM(credits_used), 1) OVER (ORDER BY DATE_TRUNC('DAY', start_time)) AS day_over_day_change,
    (SUM(credits_used) - LAG(SUM(credits_used), 1) OVER (ORDER BY DATE_TRUNC('DAY', start_time))) /
        NULLIF(LAG(SUM(credits_used), 1) OVER (ORDER BY DATE_TRUNC('DAY', start_time)), 0) * 100 AS day_over_day_change_percent
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE
    start_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY
    DATE_TRUNC('DAY', start_time)
ORDER BY
    day DESC;
```

**Best Practices for WAREHOUSE_METERING_HISTORY**:
1. Use this view to track credit usage over time and identify cost trends.
2. Correlate with QUERY_HISTORY to understand which queries are consuming the most credits.
3. Use the data for capacity planning and budget forecasting.
4. Set up alerts for unusual credit usage patterns.
5. Compare actual credit usage with resource monitor quotas to ensure you're staying within budget.


### 4. WAREHOUSE_EVENTS_HISTORY

**Purpose**: Provides historical information about warehouse events, such as resizing, suspension, and resumption.

**Retention**: 365 days

**Key Columns**:
- WAREHOUSE_NAME: Name of the warehouse
- EVENT_TIME: Timestamp when the event occurred
- EVENT_TYPE: Type of event (RESIZE, SUSPEND, RESUME, etc.)
- OLD_SIZE: Previous size of the warehouse (for RESIZE events)
- NEW_SIZE: New size of the warehouse (for RESIZE events)
- USER_NAME: Name of the user who initiated the event
- ROLE_NAME: Name of the role used

**Example Queries**:

```sql
-- Get all warehouse events for the last 7 days
SELECT
    warehouse_name,
    event_time,
    event_type,
    old_size,
    new_size,
    user_name,
    role_name
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_EVENTS_HISTORY
WHERE
    event_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    event_time DESC;

-- Get warehouse resize events
SELECT
    warehouse_name,
    event_time,
    old_size,
    new_size,
    user_name,
    role_name
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_EVENTS_HISTORY
WHERE
    event_type = 'RESIZE'
    AND event_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
ORDER BY
    event_time DESC;

-- Get warehouse suspension events
SELECT
    warehouse_name,
    event_time,
    user_name,
    role_name
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_EVENTS_HISTORY
WHERE
    event_type = 'SUSPEND'
    AND event_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
ORDER BY
    event_time DESC;

-- Get warehouse resumption events
SELECT
    warehouse_name,
    event_time,
    user_name,
    role_name
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_EVENTS_HISTORY
WHERE
    event_type = 'RESUME'
    AND event_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
ORDER BY
    event_time DESC;

-- Get warehouse events by user
SELECT
    user_name,
    event_type,
    COUNT(*) AS event_count,
    warehouse_name
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_EVENTS_HISTORY
WHERE
    event_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY
    user_name, event_type, warehouse_name
ORDER BY
    user_name, event_count DESC;
```

**Best Practices for WAREHOUSE_EVENTS_HISTORY**:
1. Use this view to track changes to warehouse configurations.
2. Monitor suspension and resumption events to understand warehouse usage patterns.
3. Correlate resize events with performance data to evaluate the impact of warehouse size changes.
4. Use the data for audit trails and compliance reporting.
5. Set up alerts for unexpected warehouse events (e.g., frequent suspensions).


### 5. RESOURCE_MONITORS

**Purpose**: Provides information about resource monitors, including their configuration and current usage.

**Retention**: 365 days

**Key Columns**:
- MONITOR_NAME: Name of the resource monitor
- MONITOR_TYPE: Type of monitor (ACCOUNT, WAREHOUSE, USER, ROLE)
- CREDIT_QUOTA: Credit quota for the monitor
- USED_CREDITS: Credits used so far in the current quota period
- REMAINING_CREDITS: Remaining credits in the current quota period
- START_TIME: Start time of the current quota period
- END_TIME: End time of the current quota period
- FREQUENCY: Quota period frequency (DAILY, WEEKLY, MONTHLY, etc.)
- NOTIFY_THRESHOLD: Threshold for warning notifications (percentage)
- SUSPEND_THRESHOLD: Threshold for suspension (percentage)
- NOTIFY_USERS: List of users to notify

**Example Queries**:

```sql
-- Get all resource monitors
SELECT
    monitor_name,
    monitor_type,
    credit_quota,
    used_credits,
    remaining_credits,
    used_credits * 100.0 / NULLIF(credit_quota, 0) AS percent_used,
    start_time,
    end_time,
    frequency,
    notify_threshold,
    suspend_threshold
FROM
    SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS
ORDER BY
    monitor_name;

-- Get resource monitors with high usage (>80%)
SELECT
    monitor_name,
    monitor_type,
    credit_quota,
    used_credits,
    remaining_credits,
    used_credits * 100.0 / credit_quota AS percent_used,
    start_time,
    end_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS
WHERE
    used_credits * 100.0 / credit_quota > 80
ORDER BY
    percent_used DESC;

-- Get resource monitors that are near or at their limit
SELECT
    monitor_name,
    monitor_type,
    credit_quota,
    used_credits,
    remaining_credits,
    used_credits * 100.0 / credit_quota AS percent_used
FROM
    SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS
WHERE
    used_credits * 100.0 / credit_quota > 90
ORDER BY
    percent_used DESC;

-- Get resource monitors by type
SELECT
    monitor_type,
    COUNT(*) AS monitor_count,
    SUM(credit_quota) AS total_quota,
    SUM(used_credits) AS total_used
FROM
    SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS
GROUP BY
    monitor_type
ORDER BY
    monitor_count DESC;
```

**Best Practices for RESOURCE_MONITORS**:
1. Use this view to monitor credit usage against quotas.
2. Set up alerts for monitors that are approaching their limits.
3. Regularly review and adjust credit quotas based on actual usage.
4. Use the data to enforce budget controls and prevent bill shocks.
5. Correlate with WAREHOUSE_METERING_HISTORY to understand credit usage patterns.


### 6. RESOURCE_MONITOR_HISTORY

**Purpose**: Provides historical information about resource monitor notifications and suspension events.

**Retention**: 365 days

**Key Columns**:
- MONITOR_NAME: Name of the resource monitor
- NOTIFICATION_TIME: Timestamp when the notification was sent
- NOTIFICATION_TYPE: Type of notification (WARNING, SUSPENSION, etc.)
- THRESHOLD_REACHED: Threshold that was reached (percentage)
- CURRENT_USAGE: Credit usage at the time of notification
- CREDIT_QUOTA: Credit quota for the monitor

**Example Queries**:

```sql
-- Get all resource monitor notifications for the last 7 days
SELECT
    monitor_name,
    notification_time,
    notification_type,
    threshold_reached,
    current_usage,
    credit_quota
FROM
    SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITOR_HISTORY
WHERE
    notification_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    notification_time DESC;

-- Get warning notifications
SELECT
    monitor_name,
    notification_time,
    threshold_reached,
    current_usage,
    credit_quota
FROM
    SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITOR_HISTORY
WHERE
    notification_type = 'WARNING'
    AND notification_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
ORDER BY
    notification_time DESC;

-- Get suspension notifications
SELECT
    monitor_name,
    notification_time,
    threshold_reached,
    current_usage,
    credit_quota
FROM
    SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITOR_HISTORY
WHERE
    notification_type = 'SUSPENSION'
    AND notification_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
ORDER BY
    notification_time DESC;

-- Get resource monitor notifications by monitor
SELECT
    monitor_name,
    notification_type,
    COUNT(*) AS notification_count,
    AVG(threshold_reached) AS avg_threshold,
    AVG(current_usage) AS avg_usage
FROM
    SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITOR_HISTORY
WHERE
    notification_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY
    monitor_name, notification_type
ORDER BY
    monitor_name, notification_count DESC;

-- Get frequent resource monitor notifications
SELECT
    monitor_name,
    notification_type,
    COUNT(*) AS notification_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITOR_HISTORY
WHERE
    notification_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY
    monitor_name, notification_type
HAVING
    COUNT(*) > 5  -- More than 5 notifications in 30 days
ORDER BY
    notification_count DESC;
```

**Best Practices for RESOURCE_MONITOR_HISTORY**:
1. Use this view to track resource monitor notifications and suspension events.
2. Monitor for frequent warnings, which may indicate that quotas are set too low.
3. Analyze suspension events to understand their impact on workloads.
4. Use the data to fine-tune resource monitor configurations.
5. Set up alerts for resource monitor notifications to enable proactive management.


### 7. LOGIN_HISTORY

**Purpose**: Provides historical information about login attempts to the Snowflake account.

**Retention**: 365 days

**Key Columns**:
- EVENT_TIME: Timestamp when the login attempt occurred
- USER_NAME: Name of the user attempting to log in
- CLIENT_IP: IP address of the client
- CLIENT_TYPE: Type of client (SNOWFLAKE UI, JDBC, ODBC, etc.)
- EVENT_TYPE: Type of event (LOGIN, LOGOUT)
- STATUS: Status of the login attempt (SUCCESS, FAILED)
- ERROR_MESSAGE: Error message if login failed
- ERROR_NUMBER: Error number if login failed
- MFA_USED: Whether MFA was used for the login
- SECOND_FACTOR_TYPE: Type of second factor if MFA was used

**Example Queries**:

```sql
-- Get all login attempts for the last 7 days
SELECT
    event_time,
    user_name,
    client_ip,
    client_type,
    event_type,
    status,
    error_message,
    mfa_used,
    second_factor_type
FROM
    SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE
    event_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    event_time DESC;

-- Get failed login attempts
SELECT
    event_time,
    user_name,
    client_ip,
    client_type,
    error_message,
    error_number
FROM
    SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE
    status = 'FAILED'
    AND event_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    event_time DESC;

-- Get successful logins by user
SELECT
    user_name,
    COUNT(*) AS login_count,
    MIN(event_time) AS first_login,
    MAX(event_time) AS last_login,
    COUNT(DISTINCT client_ip) AS unique_ips
FROM
    SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE
    status = 'SUCCESS'
    AND event_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY
    user_name
ORDER BY
    login_count DESC;

-- Get logins by client type
SELECT
    client_type,
    COUNT(*) AS login_count,
    COUNT(DISTINCT user_name) AS unique_users
FROM
    SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE
    event_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY
    client_type
ORDER BY
    login_count DESC;

-- Get logins from suspicious IPs
SELECT
    client_ip,
    COUNT(*) AS login_count,
    COUNT(DISTINCT user_name) AS unique_users,
    MIN(event_time) AS first_attempt,
    MAX(event_time) AS last_attempt
FROM
    SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE
    status = 'FAILED'
    AND event_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    client_ip
HAVING
    COUNT(*) > 5  -- More than 5 failed attempts from the same IP
ORDER BY
    login_count DESC;

-- Get MFA usage statistics
SELECT
    mfa_used,
    second_factor_type,
    COUNT(*) AS login_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE
    event_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY
    mfa_used, second_factor_type
ORDER BY
    login_count DESC;
```

**Best Practices for LOGIN_HISTORY**:
1. Use this view to monitor authentication activity and detect suspicious login attempts.
2. Set up alerts for repeated failed login attempts from the same IP address.
3. Monitor MFA usage to ensure compliance with security policies.
4. Use the data for security audits and compliance reporting.
5. Analyze login patterns to understand user activity and identify potential security issues.


### 8. USER_LOGIN_HISTORY

**Purpose**: Similar to LOGIN_HISTORY but provides additional details about user login sessions.

**Retention**: 365 days

**Key Columns**:
- USER_NAME: Name of the user
- EVENT_TIME: Timestamp when the login event occurred
- EVENT_TYPE: Type of event (LOGIN, LOGOUT, SESSION_EXPIRED)
- CLIENT_IP: IP address of the client
- CLIENT_TYPE: Type of client
- STATUS: Status of the event
- SESSION_ID: ID of the session
- WAREHOUSE_NAME: Default warehouse for the session
- DATABASE_NAME: Default database for the session
- SCHEMA_NAME: Default schema for the session
- ROLE_NAME: Default role for the session

**Example Queries**:

```sql
-- Get all user login events for the last 7 days
SELECT
    user_name,
    event_time,
    event_type,
    client_ip,
    client_type,
    status,
    session_id,
    warehouse_name,
    database_name,
    schema_name,
    role_name
FROM
    SNOWFLAKE.ACCOUNT_USAGE.USER_LOGIN_HISTORY
WHERE
    event_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    event_time DESC;

-- Get active sessions
SELECT
    user_name,
    session_id,
    client_ip,
    client_type,
    warehouse_name,
    database_name,
    schema_name,
    role_name,
    event_time AS login_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.USER_LOGIN_HISTORY
WHERE
    event_type = 'LOGIN'
    AND event_time > DATEADD('hour', -24, CURRENT_TIMESTAMP())
    AND session_id NOT IN (
        SELECT session_id
        FROM SNOWFLAKE.ACCOUNT_USAGE.USER_LOGIN_HISTORY
        WHERE event_type = 'LOGOUT'
          AND event_time > DATEADD('hour', -24, CURRENT_TIMESTAMP())
    )
ORDER BY
    login_time DESC;

-- Get session duration statistics
SELECT
    user_name,
    AVG(DATEDIFF('second', login.event_time, logout.event_time)) AS avg_session_duration_seconds,
    MAX(DATEDIFF('second', login.event_time, logout.event_time)) AS max_session_duration_seconds,
    COUNT(*) AS session_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.USER_LOGIN_HISTORY login
JOIN
    SNOWFLAKE.ACCOUNT_USAGE.USER_LOGIN_HISTORY logout
    ON login.session_id = logout.session_id
    AND logout.event_type = 'LOGOUT'
WHERE
    login.event_type = 'LOGIN'
    AND login.event_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY
    user_name
ORDER BY
    avg_session_duration_seconds DESC;

-- Get concurrent sessions by user
SELECT
    user_name,
    event_time,
    COUNT(*) AS concurrent_sessions
FROM
    SNOWFLAKE.ACCOUNT_USAGE.USER_LOGIN_HISTORY
WHERE
    event_type = 'LOGIN'
    AND event_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    user_name, event_time
HAVING
    COUNT(*) > 1  -- Users with multiple concurrent sessions
ORDER BY
    user_name, concurrent_sessions DESC;
```

**Best Practices for USER_LOGIN_HISTORY**:
1. Use this view to monitor user session activity and detect unusual patterns.
2. Set up alerts for users with multiple concurrent sessions if this violates your security policies.
3. Monitor session durations to identify potential performance issues or misuse.
4. Use the data for capacity planning and understanding user behavior.
5. Correlate with QUERY_HISTORY to understand what users are doing during their sessions.


### 9. TABLE_STORAGE_METRICS

**Purpose**: Provides historical information about table storage usage.

**Retention**: 365 days

**Key Columns**:
- TABLE_NAME: Name of the table
- SCHEMA_NAME: Name of the schema
- DATABASE_NAME: Name of the database
- STORAGE_BYTES: Number of bytes used by the table
- ROW_COUNT: Number of rows in the table
- PARTITION_COUNT: Number of partitions in the table
- LAST_ALTERED: Timestamp when the table was last altered

**Example Queries**:

```sql
-- Get storage metrics for all tables
SELECT
    database_name,
    schema_name,
    table_name,
    storage_bytes / 1024 / 1024 / 1024 AS storage_gb,
    row_count,
    partition_count,
    last_altered
FROM
    SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
ORDER BY
    storage_gb DESC;

-- Get largest tables
SELECT
    database_name,
    schema_name,
    table_name,
    storage_bytes / 1024 / 1024 / 1024 AS storage_gb,
    row_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE
    storage_bytes > 100 * 1024 * 1024 * 1024  -- >100 GB
ORDER BY
    storage_gb DESC;

-- Get storage growth over time for a specific table
SELECT
    table_name,
    last_altered,
    storage_bytes / 1024 / 1024 / 1024 AS storage_gb,
    row_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE
    table_name = 'MY_LARGE_TABLE'
    AND last_altered > DATEADD('month', -6, CURRENT_TIMESTAMP())
ORDER BY
    last_altered;

-- Get storage by database
SELECT
    database_name,
    SUM(storage_bytes) / 1024 / 1024 / 1024 AS total_storage_gb,
    COUNT(*) AS table_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
GROUP BY
    database_name
ORDER BY
    total_storage_gb DESC;

-- Get storage by schema
SELECT
    database_name,
    schema_name,
    SUM(storage_bytes) / 1024 / 1024 / 1024 AS total_storage_gb,
    COUNT(*) AS table_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
GROUP BY
    database_name, schema_name
ORDER BY
    total_storage_gb DESC;

-- Get tables with high partition counts
SELECT
    database_name,
    schema_name,
    table_name,
    partition_count,
    storage_bytes / 1024 / 1024 / 1024 AS storage_gb
FROM
    SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE
    partition_count > 1000  -- More than 1000 partitions
ORDER BY
    partition_count DESC;
```

**Best Practices for TABLE_STORAGE_METRICS**:
1. Use this view to monitor storage usage and identify large tables.
2. Set up alerts for tables that are growing rapidly.
3. Use the data for capacity planning and storage optimization.
4. Identify tables that may benefit from clustering or partitioning.
5. Correlate with QUERY_HISTORY to understand which tables are being scanned most frequently.


### 10. MATERIALIZED_VIEW_REFRESH_HISTORY

**Purpose**: Provides historical information about materialized view refreshes.

**Retention**: 365 days

**Key Columns**:
- VIEW_NAME: Name of the materialized view
- SCHEMA_NAME: Name of the schema
- DATABASE_NAME: Name of the database
- REFRESH_TIME: Timestamp when the refresh started
- END_TIME: Timestamp when the refresh ended
- STATUS: Status of the refresh (SUCCESS, FAILED, etc.)
- ROWS_REFRESHED: Number of rows refreshed
- BYTES_REFRESHED: Number of bytes refreshed
- ERROR_MESSAGE: Error message if refresh failed
- ERROR_NUMBER: Error number if refresh failed

**Example Queries**:

```sql
-- Get refresh history for all materialized views
SELECT
    database_name,
    schema_name,
    view_name,
    refresh_time,
    end_time,
    status,
    rows_refreshed,
    bytes_refreshed / 1024 / 1024 AS bytes_refreshed_mb,
    DATEDIFF('second', refresh_time, end_time) AS refresh_duration_seconds
FROM
    SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_REFRESH_HISTORY
ORDER BY
    refresh_time DESC;

-- Get failed refreshes
SELECT
    database_name,
    schema_name,
    view_name,
    refresh_time,
    end_time,
    error_message,
    error_number
FROM
    SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_REFRESH_HISTORY
WHERE
    status = 'FAILED'
    AND refresh_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    refresh_time DESC;

-- Get refresh statistics by view
SELECT
    database_name,
    schema_name,
    view_name,
    COUNT(*) AS refresh_count,
    AVG(DATEDIFF('second', refresh_time, end_time)) AS avg_refresh_duration_seconds,
    AVG(rows_refreshed) AS avg_rows_refreshed,
    AVG(bytes_refreshed) AS avg_bytes_refreshed,
    SUM(CASE WHEN status = 'FAILED' THEN 1 ELSE 0 END) AS failed_refresh_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_REFRESH_HISTORY
WHERE
    refresh_time > DATEADD('month', -1, CURRENT_TIMESTAMP())
GROUP BY
    database_name, schema_name, view_name
ORDER BY
    refresh_count DESC;

-- Get refresh duration trend for a specific view
SELECT
    view_name,
    refresh_time,
    DATEDIFF('second', refresh_time, end_time) AS refresh_duration_seconds
FROM
    SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_REFRESH_HISTORY
WHERE
    view_name = 'MY_MV'
    AND refresh_time > DATEADD('month', -1, CURRENT_TIMESTAMP())
ORDER BY
    refresh_time;

-- Get views with long refresh times
SELECT
    database_name,
    schema_name,
    view_name,
    AVG(DATEDIFF('second', refresh_time, end_time)) AS avg_refresh_duration_seconds,
    COUNT(*) AS refresh_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_REFRESH_HISTORY
WHERE
    DATEDIFF('second', refresh_time, end_time) > 300  -- >5 minutes
    AND refresh_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    database_name, schema_name, view_name
ORDER BY
    avg_refresh_duration_seconds DESC;
```

**Best Practices for MATERIALIZED_VIEW_REFRESH_HISTORY**:
1. Use this view to monitor materialized view refresh performance.
2. Set up alerts for failed refreshes or long-running refreshes.
3. Use the data to optimize materialized view definitions and refresh schedules.
4. Monitor refresh durations to identify performance bottlenecks.
5. Correlate with QUERY_HISTORY to understand the impact of refreshes on warehouse performance.

### 11. MATERIALIZED_VIEW_STORAGE

**Purpose**: Provides information about storage usage for materialized views.

**Retention**: 365 days

**Key Columns**:
- VIEW_NAME: Name of the materialized view
- SCHEMA_NAME: Name of the schema
- DATABASE_NAME: Name of the database
- STORAGE_BYTES: Number of bytes used by the materialized view

**Example Queries**:

```sql
-- Get storage usage for all materialized views
SELECT
    database_name,
    schema_name,
    view_name,
    storage_bytes / 1024 / 1024 / 1024 AS storage_gb
FROM
    SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_STORAGE
ORDER BY
    storage_gb DESC;

-- Get largest materialized views
SELECT
    database_name,
    schema_name,
    view_name,
    storage_bytes / 1024 / 1024 / 1024 AS storage_gb
FROM
    SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_STORAGE
WHERE
    storage_bytes > 10 * 1024 * 1024 * 1024  -- >10 GB
ORDER BY
    storage_gb DESC;

-- Get storage by database for materialized views
SELECT
    database_name,
    SUM(storage_bytes) / 1024 / 1024 / 1024 AS total_storage_gb,
    COUNT(*) AS mv_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_STORAGE
GROUP BY
    database_name
ORDER BY
    total_storage_gb DESC;

-- Get storage by schema for materialized views
SELECT
    database_name,
    schema_name,
    SUM(storage_bytes) / 1024 / 1024 / 1024 AS total_storage_gb,
    COUNT(*) AS mv_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_STORAGE
GROUP BY
    database_name, schema_name
ORDER BY
    total_storage_gb DESC;
```

**Best Practices for MATERIALIZED_VIEW_STORAGE**:
1. Use this view to monitor storage usage for materialized views.
2. Identify large materialized views that may be consuming excessive storage.
3. Set up alerts for materialized views that are growing rapidly.
4. Use the data for capacity planning and storage optimization.
5. Consider the trade-off between storage costs and query performance when using materialized views.

### 12. STORAGE_USAGE

**Purpose**: Provides information about storage usage at the account level.

**Retention**: 365 days

**Key Columns**:
- USAGE_DATE: Date of the storage usage
- STORAGE_BYTES: Total storage bytes used on that date
- ACTIVE_BYTES: Storage bytes for active data
- TIME_TRAVEL_BYTES: Storage bytes for time travel data
- FAILSAFE_BYTES: Storage bytes for fail-safe data
- STAGE_BYTES: Storage bytes for stage data

**Example Queries**:

```sql
-- Get daily storage usage for the last 30 days
SELECT
    usage_date,
    storage_bytes / 1024 / 1024 / 1024 AS total_storage_gb,
    active_bytes / 1024 / 1024 / 1024 AS active_storage_gb,
    time_travel_bytes / 1024 / 1024 / 1024 AS time_travel_storage_gb,
    failsafe_bytes / 1024 / 1024 / 1024 AS failsafe_storage_gb,
    stage_bytes / 1024 / 1024 / 1024 AS stage_storage_gb
FROM
    SNOWFLAKE.ACCOUNT_USAGE.STORAGE_USAGE
WHERE
    usage_date > DATEADD('day', -30, CURRENT_DATE())
ORDER BY
    usage_date DESC;

-- Get storage growth trend
SELECT
    usage_date,
    storage_bytes / 1024 / 1024 / 1024 AS total_storage_gb,
    storage_bytes / 1024 / 1024 / 1024 - LAG(storage_bytes / 1024 / 1024 / 1024, 1) OVER (ORDER BY usage_date) AS day_over_day_growth_gb,
    (storage_bytes - LAG(storage_bytes, 1) OVER (ORDER BY usage_date)) /
        NULLIF(LAG(storage_bytes, 1) OVER (ORDER BY usage_date), 0) * 100 AS day_over_day_growth_percent
FROM
    SNOWFLAKE.ACCOUNT_USAGE.STORAGE_USAGE
WHERE
    usage_date > DATEADD('day', -30, CURRENT_DATE())
ORDER BY
    usage_date DESC;

-- Get storage breakdown by component
SELECT
    AVG(storage_bytes / 1024 / 1024 / 1024) AS avg_total_storage_gb,
    AVG(active_bytes / 1024 / 1024 / 1024) AS avg_active_storage_gb,
    AVG(time_travel_bytes / 1024 / 1024 / 1024) AS avg_time_travel_storage_gb,
    AVG(failsafe_bytes / 1024 / 1024 / 1024) AS avg_failsafe_storage_gb,
    AVG(stage_bytes / 1024 / 1024 / 1024) AS avg_stage_storage_gb
FROM
    SNOWFLAKE.ACCOUNT_USAGE.STORAGE_USAGE
WHERE
    usage_date > DATEADD('day', -30, CURRENT_DATE());

-- Get storage usage by day of week
SELECT
    DAYNAME(usage_date) AS day_of_week,
    AVG(storage_bytes / 1024 / 1024 / 1024) AS avg_storage_gb,
    MAX(storage_bytes / 1024 / 1024 / 1024) AS max_storage_gb,
    MIN(storage_bytes / 1024 / 1024 / 1024) AS min_storage_gb
FROM
    SNOWFLAKE.ACCOUNT_USAGE.STORAGE_USAGE
WHERE
    usage_date > DATEADD('day', -30, CURRENT_DATE())
GROUP BY
    DAYNAME(usage_date), DAYOFWEEK(usage_date)
ORDER BY
    DAYOFWEEK(usage_date);
```

**Best Practices for STORAGE_USAGE**:
1. Use this view to monitor overall storage usage and growth trends.
2. Set up alerts for unusual storage growth patterns.
3. Use the data for capacity planning and budget forecasting.
4. Monitor the breakdown between active, time travel, fail-safe, and stage storage.
5. Correlate with other views (e.g., TABLE_STORAGE_METRICS) to identify the sources of storage growth.

### 13. DATABASE_STORAGE_USAGE

**Purpose**: Provides information about storage usage by database.

**Retention**: 365 days

**Key Columns**:
- DATABASE_NAME: Name of the database
- USAGE_DATE: Date of the storage usage
- STORAGE_BYTES: Total storage bytes used by the database on that date

**Example Queries**:

```sql
-- Get daily storage usage by database for the last 30 days
SELECT
    database_name,
    usage_date,
    storage_bytes / 1024 / 1024 / 1024 AS storage_gb
FROM
    SNOWFLAKE.ACCOUNT_USAGE.DATABASE_STORAGE_USAGE
WHERE
    usage_date > DATEADD('day', -30, CURRENT_DATE())
ORDER BY
    usage_date DESC, storage_gb DESC;

-- Get storage growth by database
SELECT
    database_name,
    usage_date,
    storage_bytes / 1024 / 1024 / 1024 AS storage_gb,
    storage_bytes / 1024 / 1024 / 1024 - LAG(storage_bytes / 1024 / 1024 / 1024, 1) OVER (PARTITION BY database_name ORDER BY usage_date) AS day_over_day_growth_gb
FROM
    SNOWFLAKE.ACCOUNT_USAGE.DATABASE_STORAGE_USAGE
WHERE
    usage_date > DATEADD('day', -30, CURRENT_DATE())
ORDER BY
    database_name, usage_date DESC;

-- Get average storage by database for the last 30 days
SELECT
    database_name,
    AVG(storage_bytes / 1024 / 1024 / 1024) AS avg_storage_gb,
    MAX(storage_bytes / 1024 / 1024 / 1024) AS max_storage_gb,
    MIN(storage_bytes / 1024 / 1024 / 1024) AS min_storage_gb
FROM
    SNOWFLAKE.ACCOUNT_USAGE.DATABASE_STORAGE_USAGE
WHERE
    usage_date > DATEADD('day', -30, CURRENT_DATE())
GROUP BY
    database_name
ORDER BY
    avg_storage_gb DESC;

-- Get databases with high storage growth
SELECT
    database_name,
    (MAX(storage_bytes) - MIN(storage_bytes)) / 1024 / 1024 / 1024 AS storage_growth_gb,
    (MAX(storage_bytes) - MIN(storage_bytes)) / NULLIF(MIN(storage_bytes), 0) * 100 AS storage_growth_percent
FROM
    SNOWFLAKE.ACCOUNT_USAGE.DATABASE_STORAGE_USAGE
WHERE
    usage_date > DATEADD('day', -30, CURRENT_DATE())
GROUP BY
    database_name
HAVING
    (MAX(storage_bytes) - MIN(storage_bytes)) / 1024 / 1024 / 1024 > 10  -- >10 GB growth
ORDER BY
    storage_growth_gb DESC;
```

**Best Practices for DATABASE_STORAGE_USAGE**:
1. Use this view to monitor storage usage by database.
2. Identify databases with high storage growth for further investigation.
3. Correlate with TABLE_STORAGE_METRICS to identify which tables are driving storage growth.
4. Use the data for capacity planning and cost allocation.
5. Set up alerts for databases that are growing rapidly.

### 14. SCHEMA_STORAGE_USAGE

**Purpose**: Provides information about storage usage by schema.

**Retention**: 365 days

**Key Columns**:
- DATABASE_NAME: Name of the database
- SCHEMA_NAME: Name of the schema
- USAGE_DATE: Date of the storage usage
- STORAGE_BYTES: Total storage bytes used by the schema on that date

**Example Queries**:

```sql
-- Get daily storage usage by schema for the last 30 days
SELECT
    database_name,
    schema_name,
    usage_date,
    storage_bytes / 1024 / 1024 / 1024 AS storage_gb
FROM
    SNOWFLAKE.ACCOUNT_USAGE.SCHEMA_STORAGE_USAGE
WHERE
    usage_date > DATEADD('day', -30, CURRENT_DATE())
ORDER BY
    usage_date DESC, storage_gb DESC;

-- Get average storage by schema for the last 30 days
SELECT
    database_name,
    schema_name,
    AVG(storage_bytes / 1024 / 1024 / 1024) AS avg_storage_gb,
    MAX(storage_bytes / 1024 / 1024 / 1024) AS max_storage_gb,
    MIN(storage_bytes / 1024 / 1024 / 1024) AS min_storage_gb
FROM
    SNOWFLAKE.ACCOUNT_USAGE.SCHEMA_STORAGE_USAGE
WHERE
    usage_date > DATEADD('day', -30, CURRENT_DATE())
GROUP BY
    database_name, schema_name
ORDER BY
    avg_storage_gb DESC;

-- Get schemas with high storage growth
SELECT
    database_name,
    schema_name,
    (MAX(storage_bytes) - MIN(storage_bytes)) / 1024 / 1024 / 1024 AS storage_growth_gb,
    (MAX(storage_bytes) - MIN(storage_bytes)) / NULLIF(MIN(storage_bytes), 0) * 100 AS storage_growth_percent
FROM
    SNOWFLAKE.ACCOUNT_USAGE.SCHEMA_STORAGE_USAGE
WHERE
    usage_date > DATEADD('day', -30, CURRENT_DATE())
GROUP BY
    database_name, schema_name
HAVING
    (MAX(storage_bytes) - MIN(storage_bytes)) / 1024 / 1024 / 1024 > 5  -- >5 GB growth
ORDER BY
    storage_growth_gb DESC;
```

**Best Practices for SCHEMA_STORAGE_USAGE**:
1. Use this view to monitor storage usage by schema.
2. Identify schemas with high storage growth for further investigation.
3. Correlate with TABLE_STORAGE_METRICS to identify which tables are driving storage growth.
4. Use the data for capacity planning and cost allocation.
5. Set up alerts for schemas that are growing rapidly.

### 15. ACCOUNT_USAGE

**Purpose**: Provides a summary of account-level usage, including storage and compute.

**Retention**: 365 days

**Key Columns**:
- USAGE_DATE: Date of the usage
- COMPUTE_CREDITS_USED: Total compute credits used on that date
- STORAGE_BYTES: Total storage bytes used on that date
- CLOUD_SERVICES_CREDITS_USED: Total cloud services credits used on that date

**Example Queries**:

```sql
-- Get daily usage summary for the last 30 days
SELECT
    usage_date,
    compute_credits_used,
    storage_bytes / 1024 / 1024 / 1024 AS storage_gb,
    cloud_services_credits_used,
    (compute_credits_used + cloud_services_credits_used) * 0.028 AS estimated_daily_cost_usd  -- Assuming $0.028 per credit
FROM
    SNOWFLAKE.ACCOUNT_USAGE.ACCOUNT_USAGE
WHERE
    usage_date > DATEADD('day', -30, CURRENT_DATE())
ORDER BY
    usage_date DESC;

-- Get usage trend for the last 30 days
SELECT
    usage_date,
    compute_credits_used,
    storage_bytes / 1024 / 1024 / 1024 AS storage_gb,
    cloud_services_credits_used,
    compute_credits_used - LAG(compute_credits_used, 1) OVER (ORDER BY usage_date) AS day_over_day_compute_change,
    (compute_credits_used - LAG(compute_credits_used, 1) OVER (ORDER BY usage_date)) /
        NULLIF(LAG(compute_credits_used, 1) OVER (ORDER BY usage_date), 0) * 100 AS day_over_day_compute_change_percent,
    (storage_bytes - LAG(storage_bytes, 1) OVER (ORDER BY usage_date)) / 1024 / 1024 / 1024 AS day_over_day_storage_change_gb
FROM
    SNOWFLAKE.ACCOUNT_USAGE.ACCOUNT_USAGE
WHERE
    usage_date > DATEADD('day', -30, CURRENT_DATE())
ORDER BY
    usage_date DESC;

-- Get monthly usage summary
SELECT
    DATE_TRUNC('MONTH', usage_date) AS month,
    SUM(compute_credits_used) AS monthly_compute_credits,
    SUM(storage_bytes) / 1024 / 1024 / 1024 AS monthly_storage_gb,
    SUM(cloud_services_credits_used) AS monthly_cloud_services_credits,
    (SUM(compute_credits_used) + SUM(cloud_services_credits_used)) * 0.028 AS estimated_monthly_cost_usd
FROM
    SNOWFLAKE.ACCOUNT_USAGE.ACCOUNT_USAGE
WHERE
    usage_date > DATEADD('month', -12, CURRENT_DATE())
GROUP BY
    DATE_TRUNC('MONTH', usage_date)
ORDER BY
    month DESC;

-- Get usage by day of week
SELECT
    DAYNAME(usage_date) AS day_of_week,
    AVG(compute_credits_used) AS avg_compute_credits,
    AVG(storage_bytes / 1024 / 1024 / 1024) AS avg_storage_gb,
    AVG(cloud_services_credits_used) AS avg_cloud_services_credits
FROM
    SNOWFLAKE.ACCOUNT_USAGE.ACCOUNT_USAGE
WHERE
    usage_date > DATEADD('day', -90, CURRENT_DATE())
GROUP BY
    DAYNAME(usage_date), DAYOFWEEK(usage_date)
ORDER BY
    DAYOFWEEK(usage_date);
```

**Best Practices for ACCOUNT_USAGE**:
1. Use this view to get a high-level overview of account usage.
2. Monitor compute, storage, and cloud services credits separately.
3. Use the data for budget forecasting and cost management.
4. Identify usage patterns (e.g., higher usage on certain days of the week).
5. Correlate with other views (e.g., WAREHOUSE_METERING_HISTORY, STORAGE_USAGE) for detailed analysis.

## Performance Considerations for Account Usage Views

When querying Account Usage views, consider the following performance best practices:

1. **Use Time Range Filters**: Always filter by date/time columns (e.g., START_TIME, USAGE_DATE) to limit the amount of data scanned.

   ```sql
   -- Good: Filtered by time range
   SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE start_time > DATEADD('day', -7, CURRENT_TIMESTAMP());

   -- Bad: No time filter (scans all 365 days of data)
   SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY;
   ```

2. **Use Specific Column Filters**: Filter by specific columns (e.g., WAREHOUSE_NAME, USER_NAME) to further reduce the result set size.

   ```sql
   -- Good: Filtered by warehouse and user
   SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE warehouse_name = 'MY_WH'
     AND user_name = 'MY_USER'
     AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP());
   ```

3. **Use Aggregation**: When analyzing large time ranges, use aggregation to summarize the data.

   ```sql
   -- Good: Aggregated by day
   SELECT
       DATE_TRUNC('DAY', start_time) AS day,
       COUNT(*) AS query_count,
       SUM(credits_used) AS total_credits_used
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       start_time > DATEADD('month', -1, CURRENT_TIMESTAMP())
   GROUP BY
       DATE_TRUNC('DAY', start_time)
   ORDER BY
       day DESC;

   -- Bad: No aggregation (returns all rows)
   SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE start_time > DATEADD('month', -1, CURRENT_TIMESTAMP());
   ```

4. **Avoid SELECT ***: Only select the columns you need to reduce the amount of data transferred.

   ```sql
   -- Good: Specific columns
   SELECT
       query_id,
       warehouse_name,
       start_time,
       credits_used
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       start_time > DATEADD('day', -1, CURRENT_TIMESTAMP());

   -- Bad: All columns
   SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE start_time > DATEADD('day', -1, CURRENT_TIMESTAMP());
   ```

5. **Use LIMIT for Testing**: When testing queries, use LIMIT to restrict the number of rows returned.

   ```sql
   -- Good: Limited to 100 rows for testing
   SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   LIMIT 100;

   -- Bad: No limit (may return millions of rows)
   SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE start_time > DATEADD('day', -7, CURRENT_TIMESTAMP());
   ```

6. **Use WHERE Instead of HAVING**: Filter data in the WHERE clause rather than in the HAVING clause when possible.

   ```sql
   -- Good: Filter in WHERE
   SELECT
       user_name,
       COUNT(*) AS query_count
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
       AND credits_used > 100
   GROUP BY
       user_name;

   -- Bad: Filter in HAVING
   SELECT
       user_name,
       COUNT(*) AS query_count
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   GROUP BY
       user_name
   HAVING
       COUNT(*) > 0 AND AVG(credits_used) > 100;
   ```

7. **Use Indexed Columns**: Some Account Usage views may have implicit clustering on certain columns (e.g., START_TIME). Use these columns in your WHERE clauses for better performance.

8. **Avoid Complex Joins**: Account Usage views can be large. Avoid complex joins that may result in large intermediate results.

   ```sql
   -- Good: Simple query
   SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE start_time > DATEADD('day', -1, CURRENT_TIMESTAMP());

   -- Bad: Complex join with large intermediate results
   SELECT
       qh.*,
       wh.credits_used AS warehouse_credits
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY qh
   JOIN
       SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY wh
       ON qh.warehouse_name = wh.warehouse_name
       AND qh.start_time = wh.start_time
   WHERE
       qh.start_time > DATEADD('day', -1, CURRENT_TIMESTAMP());
   ```

9. **Use Materialized Views**: For frequently run queries against Account Usage views, consider creating materialized views to improve performance.

   ```sql
   CREATE MATERIALIZED VIEW daily_query_stats AS
   SELECT
       DATE_TRUNC('DAY', start_time) AS day,
       warehouse_name,
       user_name,
       COUNT(*) AS query_count,
       SUM(credits_used) AS total_credits_used,
       AVG(execution_time) AS avg_execution_time
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       start_time > DATEADD('month', -1, CURRENT_TIMESTAMP())
   GROUP BY
       DATE_TRUNC('DAY', start_time), warehouse_name, user_name;
   ```

10. **Use Approximate Functions**: For large datasets, use approximate functions (e.g., APPROX_COUNT_DISTINCT) instead of exact functions to improve performance.

    ```sql
    -- Good: Approximate count
    SELECT
        warehouse_name,
        APPROX_COUNT_DISTINCT(query_id) AS approx_query_count
    FROM
        SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE
        start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
    GROUP BY
        warehouse_name;

    -- Bad: Exact count (slower for large datasets)
    SELECT
        warehouse_name,
        COUNT(DISTINCT query_id) AS query_count
    FROM
        SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE
        start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
    GROUP BY
        warehouse_name;
    ```

## Security Considerations for Account Usage Views

1. **Access Control**: Account Usage views contain sensitive information about queries, users, and resource usage. Restrict access to these views to authorized users only.

   ```sql
   -- Grant access to a role
   GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE my_analyst_role;

   -- Do not grant to all users
   -- GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE PUBLIC;
   ```

2. **Data Masking**: Consider masking sensitive information in query text (e.g., passwords, tokens) when sharing data from Account Usage views.

   ```sql
   -- Create a view that masks sensitive information
   CREATE VIEW masked_query_history AS
   SELECT
       query_id,
       REGEXP_REPLACE(query_text, '(password|token|secret|key)[=:]?[^ ]+', '***MASKED***') AS masked_query_text,
       warehouse_name,
       user_name,
       start_time,
       credits_used
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       start_time > DATEADD('day', -7, CURRENT_TIMESTAMP());
   ```

3. **Audit Logging**: Monitor access to Account Usage views to detect and investigate suspicious activity.

   ```sql
   -- Check for queries against Account Usage views
   SELECT
       query_id,
       query_text,
       user_name,
       start_time,
       warehouse_name
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       query_text LIKE '%SNOWFLAKE.ACCOUNT_USAGE.%'
       AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY
       start_time DESC;
   ```

4. **Row-Level Security**: Use row-level security to restrict access to specific rows in Account Usage views based on user attributes.

   ```sql
   -- Create a row access policy to restrict access to QUERY_HISTORY
   CREATE ROW ACCESS POLICY query_history_policy AS (user_name STRING) RETURNS BOOLEAN ->
     user_name = CURRENT_USER() OR CURRENT_ROLE() = 'ACCOUNTADMIN';

   -- Apply the policy to a view
   CREATE VIEW restricted_query_history AS
   SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WITH ROW ACCESS POLICY query_history_policy;
   ```

5. **Data Retention**: Be aware that Account Usage views retain data for 365 days. For compliance requirements that mandate shorter retention periods, you may need to implement additional data management processes.

## Common Use Cases for Account Usage Views

### 1. Cost Analysis and Optimization

**Goal**: Identify cost drivers and optimize Snowflake spending.

**Queries**:
```sql
-- Daily cost by warehouse
SELECT
    warehouse_name,
    DATE_TRUNC('DAY', start_time) AS day,
    SUM(credits_used) AS daily_credits,
    SUM(credits_used) * 0.028 AS estimated_daily_cost_usd
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('month', -1, CURRENT_TIMESTAMP())
GROUP BY
    warehouse_name, DATE_TRUNC('DAY', start_time)
ORDER BY
    day DESC, warehouse_name;

-- Top cost drivers (queries)
SELECT
    query_text,
    COUNT(*) AS query_count,
    SUM(credits_used) AS total_credits_used,
    SUM(credits_used) * 0.028 AS estimated_cost_usd
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('month', -1, CURRENT_TIMESTAMP())
GROUP BY
    query_text
ORDER BY
    estimated_cost_usd DESC
LIMIT 20;

-- Cost by user
SELECT
    user_name,
    SUM(credits_used) AS total_credits_used,
    SUM(credits_used) * 0.028 AS estimated_cost_usd,
    COUNT(*) AS query_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('month', -1, CURRENT_TIMESTAMP())
GROUP BY
    user_name
ORDER BY
    estimated_cost_usd DESC;

-- Cost by query tag
SELECT
    query_tag,
    SUM(credits_used) AS total_credits_used,
    SUM(credits_used) * 0.028 AS estimated_cost_usd,
    COUNT(*) AS query_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    query_tag IS NOT NULL
    AND start_time > DATEADD('month', -1, CURRENT_TIMESTAMP())
GROUP BY
    query_tag
ORDER BY
    estimated_cost_usd DESC;
```

**Actions**:
1. Identify and optimize expensive queries.
2. Right-size warehouses based on usage patterns.
3. Implement resource monitors to control costs.
4. Use query tagging to allocate costs to specific teams/projects.
5. Set up alerts for unusual cost spikes.


### 2. Performance Analysis and Optimization

**Goal**: Identify performance bottlenecks and optimize query performance.

**Queries**:
```sql
-- Slow queries (>10 seconds)
SELECT
    query_id,
    query_text,
    warehouse_name,
    user_name,
    start_time,
    execution_time / 1000 AS execution_time_seconds,
    bytes_scanned / 1024 / 1024 AS bytes_scanned_mb,
    credits_used
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    execution_time > 10000  -- 10 seconds in milliseconds
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    execution_time DESC;

-- Queries with high bytes scanned (>1GB)
SELECT
    query_id,
    query_text,
    warehouse_name,
    user_name,
    start_time,
    bytes_scanned / 1024 / 1024 / 1024 AS bytes_scanned_gb,
    partitions_scanned,
    credits_used
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    bytes_scanned > 1024 * 1024 * 1024  -- 1GB
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    bytes_scanned DESC;

-- Queries with high queue time (>1 minute)
SELECT
    query_id,
    query_text,
    warehouse_name,
    user_name,
    start_time,
    queue_time / 1000 AS queue_time_seconds,
    execution_time / 1000 AS execution_time_seconds
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    queue_time > 60000  -- 1 minute in milliseconds
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    queue_time DESC;

-- Warehouse performance metrics
SELECT
    warehouse_name,
    AVG(execution_time) / 1000 AS avg_execution_time_seconds,
    AVG(queue_time) / 1000 AS avg_queue_time_seconds,
    AVG(bytes_scanned) / 1024 / 1024 AS avg_bytes_scanned_mb,
    COUNT(*) AS query_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    warehouse_name
ORDER BY
    avg_execution_time_seconds DESC;

-- Query profile for a specific query
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id_from_QUERY_HISTORY'));
```

**Actions**:
1. Optimize slow queries (add filters, use clustering, rewrite queries).
2. Right-size warehouses based on performance requirements.
3. Use multi-cluster warehouses for high concurrency workloads.
4. Implement query prioritization for mixed workloads.
5. Set up alerts for performance issues.

### 3. Workload Analysis and Capacity Planning

**Goal**: Understand workload patterns and plan for future capacity needs.

**Queries**:
```sql
-- Query volume by hour
SELECT
    HOUR(start_time) AS hour_of_day,
    COUNT(*) AS query_count,
    SUM(credits_used) AS total_credits_used,
    AVG(execution_time) / 1000 AS avg_execution_time_seconds
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    HOUR(start_time)
ORDER BY
    hour_of_day;

-- Query volume by day of week
SELECT
    DAYNAME(start_time) AS day_of_week,
    COUNT(*) AS query_count,
    SUM(credits_used) AS total_credits_used,
    AVG(execution_time) / 1000 AS avg_execution_time_seconds
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY
    DAYNAME(start_time), DAYOFWEEK(start_time)
ORDER BY
    DAYOFWEEK(start_time);

-- Warehouse utilization by hour
SELECT
    warehouse_name,
    HOUR(start_time) AS hour_of_day,
    AVG(running_queries) AS avg_running_queries,
    AVG(queued_queries) AS avg_queued_queries,
    AVG(credit_usage) AS avg_credit_usage
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    warehouse_name, HOUR(start_time)
ORDER BY
    warehouse_name, hour_of_day;

-- Peak warehouse utilization
SELECT
    warehouse_name,
    MAX(running_queries) AS peak_running_queries,
    MAX(queued_queries) AS peak_queued_queries,
    MAX(credit_usage) AS peak_credit_usage,
    start_time AS peak_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    warehouse_name, start_time
ORDER BY
    peak_credit_usage DESC;

-- Storage growth trend
SELECT
    usage_date,
    storage_bytes / 1024 / 1024 / 1024 AS storage_gb,
    storage_bytes / 1024 / 1024 / 1024 - LAG(storage_bytes / 1024 / 1024 / 1024, 1) OVER (ORDER BY usage_date) AS day_over_day_growth_gb
FROM
    SNOWFLAKE.ACCOUNT_USAGE.STORAGE_USAGE
WHERE
    usage_date > DATEADD('day', -30, CURRENT_DATE())
ORDER BY
    usage_date DESC;
```

**Actions**:
1. Identify peak usage periods and plan for capacity.
2. Right-size warehouses based on peak and average usage.
3. Implement auto-suspend and auto-resume to optimize costs.
4. Use multi-cluster warehouses for variable workloads.
5. Set up alerts for unusual usage patterns.

### 4. Security and Compliance Monitoring

**Goal**: Monitor for security issues and ensure compliance with organizational policies.

**Queries**:
```sql
-- Failed login attempts
SELECT
    event_time,
    user_name,
    client_ip,
    client_type,
    error_message,
    error_number
FROM
    SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE
    status = 'FAILED'
    AND event_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    event_time DESC;

-- Suspicious login attempts (multiple failures from same IP)
SELECT
    client_ip,
    COUNT(*) AS failed_attempts,
    COUNT(DISTINCT user_name) AS unique_users,
    MIN(event_time) AS first_attempt,
    MAX(event_time) AS last_attempt
FROM
    SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE
    status = 'FAILED'
    AND event_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
GROUP BY
    client_ip
HAVING
    COUNT(*) > 5  -- More than 5 failed attempts
ORDER BY
    failed_attempts DESC;

-- Successful logins from unusual locations
SELECT
    user_name,
    client_ip,
    client_type,
    event_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE
    status = 'SUCCESS'
    AND client_ip NOT IN ('192.0.2.0', '203.0.113.0')  -- Whitelisted IPs
    AND event_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    event_time DESC;

-- Queries from unusual locations
SELECT
    query_id,
    query_text,
    user_name,
    client_ip,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    client_ip NOT IN ('192.0.2.0', '203.0.113.0')  -- Whitelisted IPs
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;

-- MFA usage statistics
SELECT
    mfa_used,
    second_factor_type,
    COUNT(*) AS login_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE
    event_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY
    mfa_used, second_factor_type
ORDER BY
    login_count DESC;

-- User activity (logins per user)
SELECT
    user_name,
    COUNT(*) AS login_count,
    MIN(event_time) AS first_login,
    MAX(event_time) AS last_login
FROM
    SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE
    status = 'SUCCESS'
    AND event_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY
    user_name
ORDER BY
    login_count DESC;
```

**Actions**:
1. Investigate and block suspicious IP addresses.
2. Enforce MFA for all users.
3. Implement network policies to restrict access to known IP ranges.
4. Monitor user activity for unusual patterns.
5. Set up alerts for security issues.

### 5. Resource Monitor Analysis

**Goal**: Monitor resource monitor usage and ensure quotas are appropriate.

**Queries**:
```sql
-- Resource monitor usage
SELECT
    monitor_name,
    monitor_type,
    credit_quota,
    used_credits,
    remaining_credits,
    used_credits * 100.0 / NULLIF(credit_quota, 0) AS percent_used,
    start_time,
    end_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS
ORDER BY
    percent_used DESC;

-- Resource monitor notifications
SELECT
    monitor_name,
    notification_time,
    notification_type,
    threshold_reached,
    current_usage,
    credit_quota
FROM
    SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITOR_HISTORY
WHERE
    notification_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    notification_time DESC;

-- Resource monitors near or at their limit
SELECT
    monitor_name,
    monitor_type,
    credit_quota,
    used_credits,
    remaining_credits,
    used_credits * 100.0 / credit_quota AS percent_used
FROM
    SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS
WHERE
    used_credits * 100.0 / credit_quota > 90  -- >90% used
ORDER BY
    percent_used DESC;

-- Credit usage by resource monitor
SELECT
    rm.monitor_name,
    rm.monitor_type,
    SUM(qh.credits_used) AS total_credits_used,
    COUNT(*) AS query_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS rm
JOIN
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY qh
    ON rm.monitor_name = qh.resource_monitor
WHERE
    qh.start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    rm.monitor_name, rm.monitor_type
ORDER BY
    total_credits_used DESC;

-- Resource monitor usage trend
SELECT
    monitor_name,
    DATE_TRUNC('DAY', start_time) AS day,
    used_credits,
    credit_quota,
    used_credits * 100.0 / credit_quota AS percent_used
FROM
    SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS
WHERE
    start_time > DATEADD('day', -30, CURRENT_TIMESTAMP())
ORDER BY
    monitor_name, day;
```

**Actions**:
1. Adjust credit quotas based on actual usage.
2. Investigate monitors that frequently reach their limits.
3. Set up alerts for resource monitor warnings and suspensions.
4. Use the data to enforce budget controls.
5. Document resource monitor configurations and policies.

## Troubleshooting with Account Usage Views

### 1. High Credit Usage

**Symptoms**: Unexpectedly high credit usage.

**Troubleshooting Steps**:
1. Identify the time period with high credit usage:
   ```sql
   SELECT
       DATE_TRUNC('DAY', start_time) AS day,
       SUM(credits_used) AS daily_credits
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   GROUP BY
       DATE_TRUNC('DAY', start_time)
   ORDER BY
       daily_credits DESC;
   ```

2. Identify the warehouses with high credit usage:
   ```sql
   SELECT
       warehouse_name,
       SUM(credits_used) AS total_credits_used,
       COUNT(*) AS query_count
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
   GROUP BY
       warehouse_name
   ORDER BY
       total_credits_used DESC;
   ```

3. Identify the queries with high credit usage:
   ```sql
   SELECT
       query_id,
       query_text,
       warehouse_name,
       user_name,
       start_time,
       credits_used,
       execution_time,
       bytes_scanned
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       credits_used > 100  -- >100 credits
       AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
   ORDER BY
       credits_used DESC;
   ```

4. Analyze the query profile for expensive queries:
   ```sql
   SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'));
   ```

5. Check for resource monitor suspensions:
   ```sql
   SELECT
       monitor_name,
       notification_time,
       notification_type,
       threshold_reached,
       current_usage,
       credit_quota
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITOR_HISTORY
   WHERE
       notification_type = 'SUSPENSION'
       AND notification_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
   ORDER BY
       notification_time DESC;
   ```

**Solutions**:
1. Optimize expensive queries (add filters, use clustering, rewrite queries).
2. Right-size warehouses (use smaller warehouses if possible).
3. Implement resource monitors to limit credit usage.
4. Set up alerts for high credit usage.
5. Use query tagging to identify and track expensive queries.

### 2. Slow Query Performance

**Symptoms**: Queries are running slower than expected.

**Troubleshooting Steps**:
1. Identify slow queries:
   ```sql
   SELECT
       query_id,
       query_text,
       warehouse_name,
       user_name,
       start_time,
       execution_time,
       bytes_scanned,
       partitions_scanned
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       execution_time > 10000  -- >10 seconds
       AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY
       execution_time DESC;
   ```

2. Analyze the query profile:
   ```sql
   SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'));
   ```

3. Check for high bytes scanned:
   ```sql
   SELECT
       query_id,
       query_text,
       bytes_scanned / 1024 / 1024 AS bytes_scanned_mb,
       partitions_scanned
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       bytes_scanned > 1024 * 1024 * 1024  -- >1GB
       AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
   ORDER BY
       bytes_scanned DESC;
   ```

4. Check for high queue time:
   ```sql
   SELECT
       query_id,
       query_text,
       warehouse_name,
       queue_time / 1000 AS queue_time_seconds,
       execution_time / 1000 AS execution_time_seconds
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       queue_time > 60000  -- >1 minute
       AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
   ORDER BY
       queue_time DESC;
   ```

5. Check warehouse load:
   ```sql
   SELECT
       warehouse_name,
       start_time,
       running_queries,
       queued_queries,
       credit_usage
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
   WHERE
       start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
       AND warehouse_name = 'problem_warehouse'
   ORDER BY
       start_time DESC;
   ```

**Solutions**:
1. Optimize queries (add filters, use clustering, rewrite queries).
2. Right-size warehouses or use multi-cluster warehouses.
3. Implement query prioritization for mixed workloads.
4. Use result caching for repetitive queries.
5. Set up alerts for slow queries.

### 3. Warehouse Overloaded

**Symptoms**: Queries are queued or failing due to warehouse overloading.

**Troubleshooting Steps**:
1. Check warehouse load:
   ```sql
   SELECT
       warehouse_name,
       start_time,
       running_queries,
       queued_queries,
       credit_usage
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
   WHERE
       start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
   ORDER BY
       queued_queries DESC, start_time DESC;
   ```

2. Check current warehouse status:
   ```sql
   SELECT
       warehouse_name,
       size,
       state,
       running_queries,
       queued_queries,
       total_queries,
       cluster_number,
       total_clusters
   FROM
       SNOWFLAKE.INFORMATION_SCHEMA.WAREHOUSE_MONITOR
   ORDER BY
       queued_queries DESC;
   ```

3. Identify queries causing high load:
   ```sql
   SELECT
       query_id,
       query_text,
       warehouse_name,
       user_name,
       start_time,
       execution_time,
       queue_time,
       credits_used
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       warehouse_name = 'overloaded_warehouse'
       AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
   ORDER BY
       credits_used DESC, execution_time DESC;
   ```

4. Check for resource monitor suspensions:
   ```sql
   SELECT
       monitor_name,
       warehouse_name,
       notification_time,
       notification_type,
       threshold_reached
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITOR_HISTORY rmh
   JOIN
       SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSES w
       ON rmh.monitor_name = w.resource_monitor
   WHERE
       w.warehouse_name = 'overloaded_warehouse'
       AND notification_type = 'SUSPENSION'
       AND notification_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
   ORDER BY
       notification_time DESC;
   ```

**Solutions**:
1. Increase warehouse size or use multi-cluster warehouses.
2. Implement query prioritization for critical queries.
3. Set up resource monitors to limit credit usage.
4. Use separate warehouses for different workload types.
5. Set up alerts for warehouse overloading.

### 4. Resource Monitor Suspensions

**Symptoms**: Warehouses are suspended due to resource monitor limits being reached.

**Troubleshooting Steps**:
1. Check resource monitor status:
   ```sql
   SELECT
       monitor_name,
       monitor_type,
       credit_quota,
       used_credits,
       remaining_credits,
       used_credits * 100.0 / NULLIF(credit_quota, 0) AS percent_used,
       start_time,
       end_time
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS
   WHERE
       used_credits >= credit_quota
   ORDER BY
       percent_used DESC;
   ```

2. Check suspension notifications:
   ```sql
   SELECT
       monitor_name,
       notification_time,
       notification_type,
       threshold_reached,
       current_usage,
       credit_quota
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITOR_HISTORY
   WHERE
       notification_type = 'SUSPENSION'
       AND notification_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
   ORDER BY
       notification_time DESC;
   ```

3. Identify queries that consumed the most credits:
   ```sql
   SELECT
       query_id,
       query_text,
       warehouse_name,
       user_name,
       start_time,
       credits_used,
       execution_time
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       resource_monitor = 'suspended_monitor'
       AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
   ORDER BY
       credits_used DESC;
   ```

4. Check warehouse status:
   ```sql
   SELECT
       warehouse_name,
       resource_monitor,
       state
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSES
   WHERE
       resource_monitor = 'suspended_monitor';
   ```

**Solutions**:
1. Resume suspended warehouses (temporary fix):
   ```sql
   ALTER WAREHOUSE suspended_wh RESUME;
   ```

2. Increase credit quota for the resource monitor (permanent fix):
   ```sql
   ALTER RESOURCE MONITOR suspended_monitor SET CREDIT_QUOTA = 20000;
   ```

3. Adjust thresholds for warnings and suspensions:
   ```sql
   ALTER RESOURCE MONITOR suspended_monitor SET NOTIFY_THRESHOLD = 90;
   ALTER RESOURCE MONITOR suspended_monitor SET SUSPEND_THRESHOLD = 100;
   ```

4. Optimize queries to reduce credit usage.
5. Implement query timeouts to prevent runaway queries.
6. Set up alerts for resource monitor warnings.

### 5. Failed Queries

**Symptoms**: Queries are failing with errors.

**Troubleshooting Steps**:
1. Identify failed queries:
   ```sql
   SELECT
       query_id,
       query_text,
       warehouse_name,
       user_name,
       start_time,
       error_message,
       error_number
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       execution_status = 'FAILED'
       AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   ORDER BY
       start_time DESC;
   ```

2. Group failed queries by error type:
   ```sql
   SELECT
       error_number,
       error_message,
       COUNT(*) AS failure_count,
       COUNT(DISTINCT user_name) AS affected_users,
       COUNT(DISTINCT warehouse_name) AS affected_warehouses
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       execution_status = 'FAILED'
       AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
   GROUP BY
       error_number, error_message
   ORDER BY
       failure_count DESC;
   ```

3. Check for specific error patterns:
   ```sql
   -- Compilation errors
   SELECT
       query_id,
       query_text,
       user_name,
       start_time,
       error_message
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       execution_status = 'FAILED'
       AND error_number IN (1000, 1001, 1002, 1003)  -- Common compilation errors
       AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
   ORDER BY
       start_time DESC;

   -- Runtime errors
   SELECT
       query_id,
       query_text,
       user_name,
       start_time,
       error_message
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       execution_status = 'FAILED'
       AND error_number NOT IN (1000, 1001, 1002, 1003)  -- Exclude compilation errors
       AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
   ORDER BY
       start_time DESC;

   -- Permission errors
   SELECT
       query_id,
       query_text,
       user_name,
       role_name,
       start_time,
       error_message
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       execution_status = 'FAILED'
       AND error_message LIKE '%permission%'
       AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
   ORDER BY
       start_time DESC;
   ```

4. Check for resource issues:
   ```sql
   -- Out of memory errors
   SELECT
       query_id,
       query_text,
       warehouse_name,
       user_name,
       start_time,
       error_message
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       execution_status = 'FAILED'
       AND error_message LIKE '%memory%'
       AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
   ORDER BY
       start_time DESC;

   -- Timeout errors
   SELECT
       query_id,
       query_text,
       warehouse_name,
       user_name,
       start_time,
       execution_time,
       error_message
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       execution_status = 'FAILED'
       AND error_message LIKE '%timeout%'
       AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
   ORDER BY
       start_time DESC;
   ```

**Solutions**:
1. Fix syntax errors in query text.
2. Grant necessary permissions to users/roles.
3. Increase warehouse size for out-of-memory errors.
4. Set appropriate timeouts for long-running queries.
5. Optimize queries to reduce resource usage.
6. Check for and resolve deadlocks or resource contention.
7. Set up alerts for failed queries.

## Best Practices for Using Account Usage Views

### 1. Query Performance Best Practices

1. **Always Filter by Time Range**: Use START_TIME, END_TIME, or USAGE_DATE to limit the time range of your queries.

2. **Use Specific Column Filters**: Filter by specific columns (e.g., WAREHOUSE_NAME, USER_NAME, QUERY_TAG) to reduce the result set size.

3. **Avoid SELECT ***: Only select the columns you need to reduce the amount of data transferred.

4. **Use Aggregation**: When analyzing large time ranges, use aggregation to summarize the data.

5. **Use LIMIT for Testing**: When testing queries, use LIMIT to restrict the number of rows returned.

6. **Use Approximate Functions**: For large datasets, use approximate functions (e.g., APPROX_COUNT_DISTINCT) instead of exact functions.

7. **Create Materialized Views**: For frequently run queries, consider creating materialized views to improve performance.

8. **Avoid Complex Joins**: Avoid complex joins that may result in large intermediate results.

9. **Use WHERE Instead of HAVING**: Filter data in the WHERE clause rather than in the HAVING clause when possible.

10. **Monitor Query Performance**: Use QUERY_HISTORY to monitor the performance of your queries against Account Usage views.

### 2. Data Management Best Practices

1. **Understand Retention Periods**: Account Usage views retain data for 365 days. Plan your data analysis accordingly.

2. **Export Historical Data**: For long-term analysis or compliance requirements, export historical data from Account Usage views to your own storage.

   ```sql
   -- Export query history to a table
   CREATE TABLE query_history_export AS
   SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE start_time > DATEADD('year', -1, CURRENT_TIMESTAMP());

   -- Export to a stage
   COPY INTO @my_stage/query_history_
   FROM (
       SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
       WHERE start_time > DATEADD('month', -1, CURRENT_TIMESTAMP())
   )
   FILE_FORMAT = (TYPE = 'PARQUET');
   ```

3. **Use External Tables**: For frequent analysis of historical data, create external tables pointing to your exported data.

   ```sql
   CREATE EXTERNAL TABLE query_history_external (
       query_id STRING,
       query_text STRING,
       warehouse_name STRING,
       -- other columns
       start_time TIMESTAMP_LTZ
   )
   WITH LOCATION = @my_stage/query_history_
   FILE_FORMAT = (TYPE = 'PARQUET');
   ```

4. **Implement Data Lifecycle Policies**: Define how long to retain exported data based on your organization's requirements.

5. **Monitor Storage Usage**: Use STORAGE_USAGE and TABLE_STORAGE_METRICS to monitor storage usage for exported data.

### 3. Security Best Practices

1. **Restrict Access**: Only grant access to Account Usage views to users who need it for their roles.

2. **Use Row-Level Security**: Implement row-level security to restrict access to specific rows based on user attributes.

3. **Mask Sensitive Data**: Mask sensitive information (e.g., passwords, tokens) in query text when sharing data.

4. **Audit Access**: Monitor access to Account Usage views to detect and investigate suspicious activity.

5. **Use Secure Views**: Create secure views that only expose necessary data to users.

   ```sql
   CREATE SECURE VIEW query_history_secure AS
   SELECT
       query_id,
       REGEXP_REPLACE(query_text, '(password|token|secret|key)[=:]?[^ ]+', '***MASKED***') AS masked_query_text,
       warehouse_name,
       user_name,
       start_time,
       credits_used,
       execution_time
   FROM
       SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE
       start_time > DATEADD('day', -7, CURRENT_TIMESTAMP());
   ```

### 4. Monitoring Best Practices

1. **Set Up Regular Monitoring**: Implement regular monitoring of Account Usage views to track performance, cost, and usage trends.

2. **Create Dashboards**: Build dashboards to visualize key metrics from Account Usage views.

3. **Set Up Alerts**: Configure alerts for important events (e.g., high credit usage, slow queries, warehouse overloading).

4. **Monitor Alert History**: Review alert history to identify recurring issues and trends.

5. **Integrate with Third-Party Tools**: Integrate Account Usage data with third-party monitoring and visualization tools.

6. **Document Monitoring Processes**: Document your monitoring processes, including what is monitored, how often, and who is responsible.

### 5. Cost Management Best Practices

1. **Monitor Credit Usage**: Regularly monitor credit usage to identify cost drivers and optimize spending.

2. **Set Up Resource Monitors**: Implement resource monitors to limit credit usage and prevent bill shocks.

3. **Right-Size Warehouses**: Use the smallest warehouse size that meets your performance requirements.

4. **Use Auto-Suspend**: Configure auto-suspend to suspend idle warehouses and save credits.

5. **Optimize Queries**: Optimize queries to reduce bytes scanned and execution time, which reduces credit usage.

6. **Use Result Caching**: Enable result caching for repetitive queries to reduce credit usage.

7. **Use Clustering**: Cluster tables on frequently filtered columns to reduce bytes scanned.

8. **Use Materialized Views**: Create materialized views for expensive, repetitive queries to reduce credit usage.

9. **Monitor Storage Usage**: Regularly monitor storage usage and implement data lifecycle policies.

10. **Set Up Cost Alerts**: Configure alerts for unusual credit usage patterns or cost spikes.

## Advanced Techniques for Account Usage Views

### 1. Time Series Analysis

Account Usage views are excellent for time series analysis to identify trends and patterns in your Snowflake usage.

**Example: Daily Credit Usage Trend**
```sql
SELECT
    DATE_TRUNC('DAY', start_time) AS day,
    SUM(credits_used) AS daily_credits,
    SUM(credits_used) - LAG(SUM(credits_used), 1) OVER (ORDER BY DATE_TRUNC('DAY', start_time)) AS day_over_day_change,
    (SUM(credits_used) - LAG(SUM(credits_used), 1) OVER (ORDER BY DATE_TRUNC('DAY', start_time))) /
        NULLIF(LAG(SUM(credits_used), 1) OVER (ORDER BY DATE_TRUNC('DAY', start_time)), 0) * 100 AS day_over_day_change_percent,
    AVG(SUM(credits_used)) OVER (ORDER BY DATE_TRUNC('DAY', start_time) ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS weekly_moving_avg
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('month', -3, CURRENT_TIMESTAMP())
GROUP BY
    DATE_TRUNC('DAY', start_time)
ORDER BY
    day;
```

**Example: Hourly Warehouse Utilization**
```sql
SELECT
    warehouse_name,
    DATE_TRUNC('HOUR', start_time) AS hour,
    AVG(running_queries) AS avg_running_queries,
    AVG(queued_queries) AS avg_queued_queries,
    AVG(credit_usage) AS avg_credit_usage,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY running_queries) AS p95_running_queries
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY
    warehouse_name, DATE_TRUNC('HOUR', start_time)
ORDER BY
    warehouse_name, hour;
```

### 2. Anomaly Detection

Use Account Usage views to detect anomalies in your Snowflake usage, such as unusual credit spikes, slow queries, or failed logins.

**Example: Credit Usage Anomalies**
```sql
WITH daily_credits AS (
    SELECT
        DATE_TRUNC('DAY', start_time) AS day,
        SUM(credits_used) AS daily_credits,
        AVG(SUM(credits_used)) OVER (ORDER BY DATE_TRUNC('DAY', start_time) ROWS BETWEEN 6 PRECEDING AND 1 PRECEDING) AS weekly_avg,
        STDDEV(SUM(credits_used)) OVER (ORDER BY DATE_TRUNC('DAY', start_time) ROWS BETWEEN 6 PRECEDING AND 1 PRECEDING) AS weekly_stddev
    FROM
        SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE
        start_time > DATEADD('month', -1, CURRENT_TIMESTAMP())
    GROUP BY
        DATE_TRUNC('DAY', start_time)
)
SELECT
    day,
    daily_credits,
    weekly_avg,
    weekly_stddev,
    (daily_credits - weekly_avg) / NULLIF(weekly_stddev, 0) AS z_score,
    CASE
        WHEN (daily_credits - weekly_avg) / NULLIF(weekly_stddev, 0) > 3 THEN 'High Anomaly'
        WHEN (daily_credits - weekly_avg) / NULLIF(weekly_stddev, 0) > 2 THEN 'Medium Anomaly'
        ELSE 'Normal'
    END AS anomaly_level
FROM
    daily_credits
WHERE
    weekly_stddev > 0
ORDER BY
    z_score DESC;
```

**Example: Query Performance Anomalies**
```sql
WITH query_stats AS (
    SELECT
        query_id,
        warehouse_name,
        execution_time,
        AVG(execution_time) OVER (PARTITION BY warehouse_name ROWS BETWEEN 100 PRECEDING AND 1 PRECEDING) AS warehouse_avg,
        STDDEV(execution_time) OVER (PARTITION BY warehouse_name ROWS BETWEEN 100 PRECEDING AND 1 PRECEDING) AS warehouse_stddev
    FROM
        SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE
        start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
)
SELECT
    query_id,
    warehouse_name,
    execution_time,
    warehouse_avg,
    warehouse_stddev,
    (execution_time - warehouse_avg) / NULLIF(warehouse_stddev, 0) AS z_score,
    CASE
        WHEN (execution_time - warehouse_avg) / NULLIF(warehouse_stddev, 0) > 3 THEN 'High Anomaly'
        WHEN (execution_time - warehouse_avg) / NULLIF(warehouse_stddev, 0) > 2 THEN 'Medium Anomaly'
        ELSE 'Normal'
    END AS anomaly_level
FROM
    query_stats
WHERE
    warehouse_stddev > 0
    AND (execution_time - warehouse_avg) / NULLIF(warehouse_stddev, 0) > 2
ORDER BY
    z_score DESC;
```

### 3. Forecasting

Use historical data from Account Usage views to forecast future usage and costs.

**Example: Credit Usage Forecast**
```sql
WITH daily_credits AS (
    SELECT
        DATE_TRUNC('DAY', start_time) AS day,
        SUM(credits_used) AS daily_credits
    FROM
        SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE
        start_time > DATEADD('month', -6, CURRENT_TIMESTAMP())
    GROUP BY
        DATE_TRUNC('DAY', start_time)
),
stats AS (
    SELECT
        AVG(daily_credits) AS avg_daily_credits,
        STDDEV(daily_credits) AS stddev_daily_credits,
        COUNT(*) AS days
    FROM
        daily_credits
)
SELECT
    'Next 7 Days' AS period,
    AVG(daily_credits) * 7 AS forecast_credits,
    AVG(daily_credits) * 7 * 0.028 AS forecast_cost_usd,
    AVG(daily_credits) * 7 + 1.96 * (SELECT stddev_daily_credits FROM stats) * SQRT(7) AS upper_bound,
    AVG(daily_credits) * 7 - 1.96 * (SELECT stddev_daily_credits FROM stats) * SQRT(7) AS lower_bound
FROM
    daily_credits;
```

**Example: Storage Growth Forecast**
```sql
WITH daily_storage AS (
    SELECT
        usage_date AS day,
        storage_bytes / 1024 / 1024 / 1024 AS storage_gb
    FROM
        SNOWFLAKE.ACCOUNT_USAGE.STORAGE_USAGE
    WHERE
        usage_date > DATEADD('month', -6, CURRENT_DATE())
),
growth_rate AS (
    SELECT
        (MAX(storage_gb) - MIN(storage_gb)) / NULLIF(DATEDIFF('day', MIN(day), MAX(day)), 0) AS avg_daily_growth_gb
    FROM
        daily_storage
)
SELECT
    'Next 30 Days' AS period,
    MAX(storage_gb) AS current_storage_gb,
    MAX(storage_gb) + (SELECT avg_daily_growth_gb FROM growth_rate) * 30 AS forecast_storage_gb,
    (MAX(storage_gb) + (SELECT avg_daily_growth_gb FROM growth_rate) * 30) - MAX(storage_gb) AS forecast_growth_gb
FROM
    daily_storage;
```

### 4. Correlation Analysis

Identify correlations between different metrics using Account Usage views.

**Example: Correlation Between Query Complexity and Credit Usage**
```sql
SELECT
    CORR(execution_time, credits_used) AS corr_execution_credits,
    CORR(bytes_scanned, credits_used) AS corr_bytes_credits,
    CORR(partitions_scanned, credits_used) AS corr_partitions_credits
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
    AND warehouse_name = 'MY_WH';
```

**Example: Correlation Between Warehouse Size and Performance**
```sql
SELECT
    warehouse_size,
    AVG(execution_time) AS avg_execution_time,
    AVG(credits_used) AS avg_credits_used,
    COUNT(*) AS query_count,
    CORR(warehouse_size_multiplier, execution_time) AS corr_size_execution,
    CORR(warehouse_size_multiplier, credits_used) AS corr_size_credits
FROM (
    SELECT
        qh.*,
        CASE
            WHEN w.warehouse_size = 'XSMALL' THEN 1
            WHEN w.warehouse_size = 'SMALL' THEN 2
            WHEN w.warehouse_size = 'MEDIUM' THEN 4
            WHEN w.warehouse_size = 'LARGE' THEN 8
            WHEN w.warehouse_size = 'XLARGE' THEN 16
            WHEN w.warehouse_size = 'XXLARGE' THEN 32
            WHEN w.warehouse_size = 'XXXLARGE' THEN 64
            WHEN w.warehouse_size = 'XXXXLARGE' THEN 128
            ELSE 1
        END AS warehouse_size_multiplier
    FROM
        SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY qh
    JOIN
        SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSES w
        ON qh.warehouse_name = w.warehouse_name
    WHERE
        qh.start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
)
GROUP BY
    warehouse_size;
```

### 5. Custom Metrics and KPIs

Create custom metrics and KPIs using data from Account Usage views.

**Example: Query Performance Score**
```sql
WITH query_metrics AS (
    SELECT
        query_id,
        warehouse_name,
        execution_time,
        queue_time,
        bytes_scanned,
        partitions_scanned,
        credits_used,
        -- Normalize metrics (lower is better)
        CASE
            WHEN execution_time = 0 THEN 0
            ELSE 1 / execution_time
        END AS execution_score,
        CASE
            WHEN queue_time = 0 THEN 1
            ELSE 1 / (1 + queue_time)
        END AS queue_score,
        CASE
            WHEN bytes_scanned = 0 THEN 1
            ELSE 1 / (1 + LOG(bytes_scanned))
        END AS scan_score,
        CASE
            WHEN partitions_scanned = 0 THEN 1
            ELSE 1 / (1 + LOG(partitions_scanned))
        END AS partition_score,
        CASE
            WHEN credits_used = 0 THEN 1
            ELSE 1 / (1 + LOG(credits_used))
        END AS credit_score
    FROM
        SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE
        start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
)
SELECT
    query_id,
    warehouse_name,
    execution_time,
    queue_time,
    bytes_scanned,
    partitions_scanned,
    credits_used,
    -- Weighted score (adjust weights as needed)
    (0.4 * execution_score +
     0.2 * queue_score +
     0.1 * scan_score +
     0.1 * partition_score +
     0.2 * credit_score) * 100 AS performance_score
FROM
    query_metrics
ORDER BY
    performance_score DESC
LIMIT 20;
```

**Example: Warehouse Efficiency Score**
```sql
WITH warehouse_metrics AS (
    SELECT
        warehouse_name,
        warehouse_size,
        -- Performance metrics
        AVG(execution_time) AS avg_execution_time,
        AVG(queue_time) AS avg_queue_time,
        -- Cost metrics
        SUM(credits_used) AS total_credits_used,
        COUNT(*) AS query_count,
        -- Utilization metrics
        AVG(running_queries) AS avg_running_queries,
        AVG(queued_queries) AS avg_queued_queries
    FROM
        SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY qh
    JOIN
        SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSES w
        ON qh.warehouse_name = w.warehouse_name
    WHERE
        qh.start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
    GROUP BY
        warehouse_name, warehouse_size
)
SELECT
    warehouse_name,
    warehouse_size,
    avg_execution_time,
    avg_queue_time,
    total_credits_used,
    query_count,
    avg_running_queries,
    avg_queued_queries,
    -- Normalize metrics
    CASE
        WHEN avg_execution_time = 0 THEN 1
        ELSE 1 / (1 + LOG(avg_execution_time))
    END AS execution_normalized,
    CASE
        WHEN avg_queue_time = 0 THEN 1
        ELSE 1 / (1 + LOG(avg_queue_time))
    END AS queue_normalized,
    CASE
        WHEN total_credits_used = 0 THEN 0
        ELSE query_count / total_credits_used
    END AS efficiency_normalized,
    CASE
        WHEN avg_running_queries = 0 THEN 0
        ELSE avg_running_queries / NULLIF(
            CASE warehouse_size
                WHEN 'XSMALL' THEN 1
                WHEN 'SMALL' THEN 2
                WHEN 'MEDIUM' THEN 4
                WHEN 'LARGE' THEN 8
                WHEN 'XLARGE' THEN 16
                WHEN 'XXLARGE' THEN 32
                WHEN 'XXXLARGE' THEN 64
                WHEN 'XXXXLARGE' THEN 128
                ELSE 1
            END, 0)
    END AS utilization_normalized,
    -- Weighted score
    (0.3 * execution_normalized +
     0.2 * queue_normalized +
     0.3 * efficiency_normalized +
     0.2 * utilization_normalized) * 100 AS efficiency_score
FROM
    warehouse_metrics
ORDER BY
    efficiency_score DESC;
```

## Integrating Account Usage Views with Third-Party Tools

### 1. Exporting Data to BI Tools

Export data from Account Usage views to BI tools like Tableau, Power BI, or Looker for visualization and analysis.

**Example: Export to Tableau**
1. Set up a Snowflake connector in Tableau.
2. Connect to the SNOWFLAKE.ACCOUNT_USAGE schema.
3. Create extracts or live connections to the views you want to visualize.

**Example: Export to Power BI**
1. In Power BI, select "Get Data" and choose Snowflake.
2. Enter your Snowflake connection details.
3. Select the Account Usage views you want to import.
4. Transform and load the data into Power BI.

### 2. Using with Monitoring Tools

Integrate Account Usage data with monitoring tools like Datadog, New Relic, or Grafana.

**Example: Snowflake Integration with Datadog**
1. Set up the Snowflake integration in Datadog.
2. Configure the integration to collect data from Account Usage views.
3. Create dashboards and alerts based on the collected data.

**Example: Custom Metrics in Datadog**
```sql
-- Create a view for Datadog to collect
CREATE VIEW datadog_metrics AS
SELECT
    DATE_TRUNC('MINUTE', start_time) AS timestamp,
    warehouse_name,
    'snowflake.credits.used' AS metric_name,
    SUM(credits_used) AS metric_value,
    'warehouse:' || warehouse_name AS tags
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
GROUP BY
    DATE_TRUNC('MINUTE', start_time), warehouse_name

UNION ALL

SELECT
    DATE_TRUNC('MINUTE', start_time) AS timestamp,
    warehouse_name,
    'snowflake.queries.running' AS metric_name,
    AVG(running_queries) AS metric_value,
    'warehouse:' || warehouse_name AS tags
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE
    start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
GROUP BY
    DATE_TRUNC('MINUTE', start_time), warehouse_name;
```

### 3. Using with Log Management Tools

Export Account Usage data to log management tools like Splunk or ELK for centralized logging and analysis.

**Example: Export to Splunk**
1. Set up a Snowflake add-on in Splunk.
2. Configure the add-on to collect data from Account Usage views.
3. Create Splunk dashboards and alerts based on the data.

**Example: Query for Splunk**
```sql
-- Create a view for Splunk to collect
CREATE VIEW splunk_logs AS
SELECT
    query_id AS snowflake_query_id,
    query_text AS snowflake_query,
    warehouse_name AS snowflake_warehouse,
    user_name AS snowflake_user,
    start_time AS snowflake_start_time,
    end_time AS snowflake_end_time,
    execution_status AS snowflake_status,
    error_message AS snowflake_error,
    credits_used AS snowflake_credits,
    execution_time AS snowflake_execution_time_ms,
    bytes_scanned AS snowflake_bytes_scanned,
    CASE execution_status
        WHEN 'SUCCESS' THEN 'INFO'
        WHEN 'FAILED' THEN 'ERROR'
        ELSE 'WARN'
    END AS splunk_level,
    'snowflake' AS splunk_source,
    'query' AS splunk_sourcetype
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP());
```

### 4. Using with Data Pipelines

Integrate Account Usage data into your data pipelines for further processing and analysis.

**Example: Snowpipe for Account Usage Data**
```sql
-- Create a stage for Account Usage data
CREATE STAGE account_usage_stage URL = 's3://my-bucket/account-usage/';

-- Create a file format
CREATE FILE FORMAT account_usage_format TYPE = 'JSON';

-- Create a Snowpipe
CREATE PIPE account_usage_pipe
  AUTO_INGEST = TRUE
  AS COPY INTO account_usage_raw
     FROM @account_usage_stage
     FILE_FORMAT = (TYPE = 'JSON');

-- Create a table for raw data
CREATE TABLE account_usage_raw (
    event_time TIMESTAMP_LTZ,
    event_type STRING,
    data VARIANT
);

-- Create a stored procedure to export data
CREATE OR REPLACE PROCEDURE export_account_usage_data()
RETURNS STRING
LANGUAGE JAVASCRIPT
AS
$$
  // Export QUERY_HISTORY
  const query_history = `
    SELECT
        OBJECT_CONSTRUCT(
            'query_id', query_id,
            'query_text', query_text,
            'warehouse_name', warehouse_name,
            'user_name', user_name,
            'start_time', start_time,
            'end_time', end_time,
            'execution_status', execution_status,
            'error_message', error_message,
            'credits_used', credits_used,
            'execution_time', execution_time,
            'bytes_scanned', bytes_scanned
        ) AS data
    FROM
        SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE
        start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
  `;

  const result = snowflake.execute({sqlText: query_history});
  const statement = snowflake.createStatement({sqlText: `PUT file:///tmp/query_history.json @account_usage_stage AUTO_COMPRESS = TRUE`});
  statement.execute();

  return "Data exported successfully";
$$;

-- Schedule the procedure to run hourly
CREATE TASK export_account_usage_task
  WAREHOUSE = admin_wh
  SCHEDULE = 'USING CRON 0 * * * * America/Los_Angeles'
AS
  CALL export_account_usage_data();
```

## Limitations of Account Usage Views

While Account Usage views are powerful for monitoring and analysis, they have some limitations to be aware of:

1. **Data Latency**: Account Usage views may have a latency of up to 3 hours. For real-time monitoring, use the corresponding Information Schema views.

2. **Retention Period**: Data in Account Usage views is retained for 365 days. For long-term analysis, you need to export the data to your own storage.

3. **No Real-Time Data**: Account Usage views do not provide real-time data. Use Information Schema views for real-time monitoring.

4. **No Row-Level Security**: Account Usage views do not support row-level security. All users with access can see all data in the views.

5. **No Data Masking**: Account Usage views do not support data masking. Sensitive information (e.g., query text with passwords) is visible to all users with access.

6. **Performance**: Queries against Account Usage views can be slow if not properly filtered, especially for large time ranges.

7. **No Custom Metrics**: Account Usage views provide predefined metrics. For custom metrics, you need to create your own queries or views.

8. **No Alerting**: Account Usage views do not support native alerting. You need to create your own alerts using Snowflake alerts or external tools.

9. **No Data Modification**: Account Usage views are read-only. You cannot modify the data in these views.

10. **Limited to Account-Level Data**: Account Usage views only provide account-level data. For more granular data (e.g., database-level, schema-level), you may need to use other views or query the metadata directly.

## Conclusion

Account Usage views in Snowflake are a powerful tool for monitoring, analyzing, and optimizing your Snowflake workloads. They provide historical data about queries, warehouses, storage, and other account-level activities, enabling you to:

- Monitor performance and identify bottlenecks
- Track and optimize costs
- Analyze usage patterns and trends
- Troubleshoot issues
- Ensure security and compliance
- Plan for future capacity needs

By understanding the available views, their purposes, and how to query them effectively, you can gain deep insights into your Snowflake environment and make data-driven decisions to improve performance, reduce costs, and enhance the overall efficiency of your data platform.

Remember to:
1. Use appropriate filters to limit the time range and result set size
2. Follow security best practices to protect sensitive data
3. Combine Account Usage views with Information Schema views for comprehensive monitoring
4. Export historical data for long-term analysis
5. Integrate with third-party tools for advanced visualization and alerting

With these best practices and techniques, you can leverage Account Usage views to build a robust monitoring and optimization framework for your Snowflake environment.
