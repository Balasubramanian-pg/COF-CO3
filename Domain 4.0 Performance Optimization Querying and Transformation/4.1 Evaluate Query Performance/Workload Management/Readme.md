# **Snowflake Workload Management: Production-Grade Technical Deep Dive**


## **1. Workload Management Overview**

### **Mermaid: Snowflake Workload Management Architecture**
```mermaid
%% Snowflake Workload Management Architecture
flowchart TD
    subgraph Users["Users & Applications"]
        A[("User 1\n(Analyst)")] -->|Query| B[("Workload Group 1\n(Ad-Hoc)")]
        C[("User 2\n(Data Engineer)")] -->|Query| D[("Workload Group 2\n(ETL)")]
        E[("App 1\n(Dashboard)")] -->|Query| B
        F[("App 2\n(Batch Job)")] -->|Query| D
    end

    subgraph WorkloadManagement["Workload Management"]
        B --> G[("Resource Monitor 1\n(100 Credits/Day)")]
        D --> H[("Resource Monitor 2\n(1000 Credits/Day)")]
        G --> I[("Query Queue 1\n(Priority: MEDIUM)")]
        H --> J[("Query Queue 2\n(Priority: HIGH)")]
        I --> K[("Warehouse 1\n(SMALL)")]
        J --> L[("Warehouse 2\n(X-LARGE, Multi-Cluster)")]
    end

    subgraph Snowflake["Snowflake Services"]
        K --> M[("Query Execution\n(Ad-Hoc)")]
        L --> N[("Query Execution\n(ETL)")]
        M --> O[("Metadata Service")]
        N --> O
    end

    subgraph Monitoring["Monitoring & Observability"]
        P[("ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY")]
        Q[("ACCOUNT_USAGE.QUERY_HISTORY")]
        R[("ACCOUNT_USAGE.RESOURCE_MONITORS")]
    end
    K --> P
    L --> P
    M --> Q
    N --> Q
    G --> R
    H --> R

    subgraph Alerts["Alerts & Notifications"]
        S[("Credit Limit Reached")]
        T[("Query Timeout")]
        U[("Warehouse Overloaded")]
    end
    R --> S
    P --> T
    P --> U

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef users fill:#4285f4,stroke:#1976d2;
    classDef workload fill:#ff9800,stroke:#f57c00;
    classDef snowflake fill:#29abe2,stroke:#1a8fb8;
    classDef monitoring fill:#9c27b0,stroke:#7b1fa2;
    classDef alerts fill:#f44336,stroke:#d32f2f;
    class A,C,E,F users;
    class B,D,G,H,I,J,K,L workload;
    class M,N,O,P,Q,R snowflake;
    class S,T,U alerts;
```

### **Why Workload Management Matters**
Workload management in Snowflake ensures:
1. **Performance Consistency**: Prevents **noisy neighbor** problems where one workload impacts others.
2. **Cost Control**: Limits **credit usage** to stay within budget.
3. **Resource Allocation**: Prioritizes **critical workloads** (e.g., production ETL over ad-hoc queries).
4. **Scalability**: Handles **concurrent workloads** without manual intervention.
5. **SLA Compliance**: Meets **performance SLAs** for different user groups.

### **Key Workload Management Components**

| **Component** | **Purpose** | **Scope** | **Configuration** | **Monitoring** |
|---------------|-------------|-----------|------------------|----------------|
| **Resource Monitors** | Limit credit usage for warehouses/users/roles | Account, Warehouse, User, Role | `CREATE RESOURCE MONITOR` | `ACCOUNT_USAGE.RESOURCE_MONITORS` |
| **Warehouses** | Provide compute resources for queries | Account | `CREATE WAREHOUSE` | `ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY` |
| **Multi-Cluster Warehouses** | Scale out to handle concurrent queries | Warehouse | `CREATE WAREHOUSE ... MAX_CLUSTER_COUNT` | `ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY` |
| **Query Queues** | Manage concurrent queries in a warehouse | Warehouse | `ALTER WAREHOUSE ... QUERY_PRIORITY` | `ACCOUNT_USAGE.QUERY_HISTORY` |
| **Query Prioritization** | Prioritize queries within a warehouse | Session/Query | `ALTER WAREHOUSE ... QUERY_PRIORITY` | `QUERY_HISTORY` |
| **Query Timeouts** | Prevent long-running queries | Warehouse | `ALTER WAREHOUSE ... STATEMENT_TIMEOUT` | `QUERY_HISTORY` |
| **Workload Groups** | Group similar workloads (e.g., ETL, Reporting) | Account | N/A (use Resource Monitors + Warehouses) | Custom monitoring |

## **2. Resource Monitors Deep Dive**

### **A. Definition and Architecture**
**Resource Monitors** are Snowflake objects that **track and limit credit usage** for **warehouses, users, or roles**. They enable **cost control** and **budget enforcement** by:
- **Tracking credit consumption** in real-time.
- **Sending notifications** when thresholds are reached.
- **Suspending warehouses** when credit limits are exceeded.

```mermaid
%% Resource Monitor Architecture
flowchart TD
    subgraph ResourceMonitor["Resource Monitor"]
        A[("Credit Quota\n(1000 Credits/Day)")] --> B[("Tracking")]
        B --> C[("Current Usage: 800 Credits")]
        C --> D{Usage > Warning Threshold?}
        D -->|Yes| E[("Send Warning Notification")]
        D -->|No| F{Usage > Limit?}
        F -->|Yes| G[("Send Limit Notification\nSuspend Warehouses")]
        F -->|No| B
    end

    subgraph Warehouses["Warehouses"]
        H[("Warehouse 1\n(SMALL)")] -->|Uses Credits| B
        I[("Warehouse 2\n(MEDIUM)")] -->|Uses Credits| B
    end

    subgraph Notifications["Notifications"]
        E --> J[("Email\n(Admin)")]
        G --> J
        E --> K[("Slack/Webhook")]
        G --> K
    end

    subgraph Monitoring["Monitoring"]
        L[("ACCOUNT_USAGE.RESOURCE_MONITORS")]
        M[("ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY")]
    end
    B --> L
    H --> M
    I --> M

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef monitor fill:#ff9800,stroke:#f57c00;
    classDef warehouses fill:#29abe2,stroke:#1a8fb8;
    classDef notifications fill:#4caf50,stroke:#2e7d32;
    classDef monitoring fill:#9c27b0,stroke:#7b1fa2;
    class A,B,C,D,E,F,G monitor;
    class H,I warehouses;
    class J,K notifications;
    class L,M monitoring;
```

### **B. How Resource Monitors Work**
1. **Credit Tracking**:
   - Resource monitors **track credit usage** for assigned **warehouses, users, or roles**.
   - Credits are **consumed** based on **warehouse usage** (1 credit = 1 second of X-Small warehouse).

2. **Thresholds**:
   - **Warning Threshold**: Percentage of credit quota at which a **warning notification** is sent (default: 80%).
   - **Limit Threshold**: Percentage of credit quota at which **warehouses are suspended** (default: 100%).

3. **Notifications**:
   - **Email Notifications**: Sent to **account admins** and **specified users**.
   - **Webhook Notifications**: Sent to a **custom endpoint** (e.g., Slack, PagerDuty).

4. **Suspension**:
   - When the **limit threshold** is reached, all **assigned warehouses are suspended**.
   - Suspended warehouses **cannot execute queries** until the monitor is reset or the quota period resets.

5. **Quota Periods**:
   - **Daily**: Resets at **midnight UTC**.
   - **Weekly**: Resets at **midnight UTC on Sunday**.
   - **Monthly**: Resets at **midnight UTC on the 1st of the month**.
   - **Custom**: User-defined start/end timestamps.

### **C. When to Use Resource Monitors**
✅ **Cost Control**: Limit credit usage to stay within **budget**.
✅ **Departmental Chargeback**: Allocate credits to **teams/departments**.
✅ **Production Protection**: Prevent **runaway queries** from consuming all credits.
✅ **Development Environments**: Limit credits for **dev/test** workloads.
✅ **Compliance**: Enforce **credit limits** for regulatory compliance.

### **D. When NOT to Use Resource Monitors**
❌ **Unlimited Budgets**: If credit usage is not a concern.
❌ **Shared Warehouses**: If all users share the same warehouse (use **query prioritization** instead).
❌ **Serverless Workloads**: Resource monitors do not apply to **serverless** operations (e.g., Snowpipe, Ingestion Service).

### **E. Resource Monitor Types**

| **Type** | **Description** | **Use Case** | **Example** |
|----------|-----------------|--------------|-------------|
| **Account-Level** | Monitors credit usage for the **entire account** | Global credit limits | `CREATE RESOURCE MONITOR account_monitor WITH CREDIT_QUOTA = 10000` |
| **Warehouse-Level** | Monitors credit usage for **specific warehouses** | Warehouse-specific limits | `CREATE RESOURCE MONITOR etl_monitor WITH CREDIT_QUOTA = 5000` |
| **User-Level** | Monitors credit usage for **specific users** | User-specific limits | `CREATE RESOURCE MONITOR user_monitor WITH CREDIT_QUOTA = 100` |
| **Role-Level** | Monitors credit usage for **specific roles** | Role-specific limits | `CREATE RESOURCE MONITOR role_monitor WITH CREDIT_QUOTA = 1000` |

### **F. Resource Monitor Configuration Parameters**

| **Parameter** | **Description** | **Default** | **Valid Values** | **Performance Impact** |
|---------------|-----------------|-------------|------------------|------------------------|
| `NAME` | Name of the resource monitor | None (required) | String | None |
| `CREDIT_QUOTA` | Credit limit for the monitor | None (required) | 1-1000000 | Higher = more credits allowed |
| `FREQUENCY` | Quota period frequency | `DAILY` | `DAILY`, `WEEKLY`, `MONTHLY`, `YEARLY`, `CUSTOM` | Affects quota reset timing |
| `START_TIMESTAMP` | Start of the quota period | `DATEADD('day', -1, CURRENT_TIMESTAMP())` | Timestamp | None |
| `END_TIMESTAMP` | End of the quota period | `DATEADD('day', 1, CURRENT_TIMESTAMP())` | Timestamp | None |
| `NOTIFY_USERS` | Users to notify when thresholds are reached | `(SELECT user_name FROM SNOWFLAKE.ACCOUNT_USAGE.USERS WHERE role_name = 'ACCOUNTADMIN')` | List of users | None |
| `NOTIFY_THRESHOLD` | Percentage of quota at which to send warnings | `80` | 1-100 | Lower = earlier warnings |
| `SUSPEND_THRESHOLD` | Percentage of quota at which to suspend warehouses | `100` | 1-100 | Lower = earlier suspension |
| `SUSPEND_IMMEDIATELY` | Suspend warehouses immediately when limit is reached | `FALSE` | `TRUE`, `FALSE` | `TRUE` = stricter enforcement |

### **G. Production-Ready Setup**

#### **1. Account-Level Resource Monitor**
```sql
-- Create an account-level resource monitor with daily quota
CREATE RESOURCE MONITOR account_daily_monitor
  WITH CREDIT_QUOTA = 10000  -- 10,000 credits per day
  FREQUENCY = DAILY
  START_TIMESTAMP = DATEADD('day', -1, CURRENT_TIMESTAMP())
  NOTIFY_USERS = (SELECT user_name FROM SNOWFLAKE.ACCOUNT_USAGE.USERS WHERE role_name = 'ACCOUNTADMIN')
  NOTIFY_THRESHOLD = 80  -- Warn at 80% of quota
  SUSPEND_THRESHOLD = 100  -- Suspend at 100% of quota
  SUSPEND_IMMEDIATELY = FALSE;

-- Assign the monitor to the account
ALTER ACCOUNT SET RESOURCE_MONITOR = account_daily_monitor;
```

#### **2. Warehouse-Level Resource Monitors**
```sql
-- Create a resource monitor for ETL workloads
CREATE RESOURCE MONITOR etl_monitor
  WITH CREDIT_QUOTA = 5000  -- 5,000 credits per day
  FREQUENCY = DAILY
  NOTIFY_USERS = ('etl_admin@company.com', 'data_team@company.com')
  NOTIFY_THRESHOLD = 70
  SUSPEND_THRESHOLD = 90;

-- Assign the monitor to ETL warehouses
ALTER WAREHOUSE ETL_WH SET RESOURCE_MONITOR = etl_monitor;
ALTER WAREHOUSE ETL_LARGE_WH SET RESOURCE_MONITOR = etl_monitor;

-- Create a resource monitor for reporting workloads
CREATE RESOURCE MONITOR reporting_monitor
  WITH CREDIT_QUOTA = 2000  -- 2,000 credits per day
  FREQUENCY = DAILY
  NOTIFY_USERS = ('reporting_admin@company.com')
  NOTIFY_THRESHOLD = 80
  SUSPEND_THRESHOLD = 100;

-- Assign the monitor to reporting warehouses
ALTER WAREHOUSE REPORTING_WH SET RESOURCE_MONITOR = reporting_monitor;
```

#### **3. User/Role-Level Resource Monitors**
```sql
-- Create a resource monitor for a specific user
CREATE RESOURCE MONITOR user_monitor
  WITH CREDIT_QUOTA = 100  -- 100 credits per day
  FREQUENCY = DAILY
  NOTIFY_USERS = ('user@company.com', 'manager@company.com')
  NOTIFY_THRESHOLD = 50
  SUSPEND_THRESHOLD = 100;

-- Assign the monitor to the user
ALTER USER my_user SET RESOURCE_MONITOR = user_monitor;

-- Create a resource monitor for a specific role
CREATE RESOURCE MONITOR role_monitor
  WITH CREDIT_QUOTA = 500  -- 500 credits per day
  FREQUENCY = DAILY
  NOTIFY_USERS = ('role_admin@company.com')
  NOTIFY_THRESHOLD = 75
  SUSPEND_THRESHOLD = 100;

-- Assign the monitor to the role
ALTER ROLE my_role SET RESOURCE_MONITOR = role_monitor;
```

#### **4. Custom Quota Period**
```sql
-- Create a resource monitor with a custom quota period (e.g., fiscal quarter)
CREATE RESOURCE MONITOR fiscal_quarter_monitor
  WITH CREDIT_QUOTA = 50000  -- 50,000 credits per quarter
  FREQUENCY = CUSTOM
  START_TIMESTAMP = '2026-04-01 00:00:00'  -- Start of Q1 FY2026
  END_TIMESTAMP = '2026-06-30 23:59:59'    -- End of Q1 FY2026
  NOTIFY_USERS = ('finance@company.com')
  NOTIFY_THRESHOLD = 90
  SUSPEND_THRESHOLD = 100;
```

#### **5. Webhook Notifications**
```sql
-- Create a resource monitor with webhook notifications
CREATE RESOURCE MONITOR webhook_monitor
  WITH CREDIT_QUOTA = 1000
  FREQUENCY = DAILY
  NOTIFY_USERS = ('admin@company.com')
  NOTIFY_THRESHOLD = 80
  SUSPEND_THRESHOLD = 100;

-- Configure webhook (via Snowflake UI or API)
-- 1. Navigate to Admin > Resource Monitors in Snowsight
-- 2. Select the monitor and click "Edit"
-- 3. Under "Notifications", add a webhook URL (e.g., https://hooks.slack.com/services/...)
-- 4. Save changes

-- Example webhook payload (sent to Slack)
{
  "text": "⚠️ Snowflake Resource Monitor Alert",
  "attachments": [
    {
      "color": "#ffcc00",
      "title": "Resource Monitor Warning: webhook_monitor",
      "fields": [
        {
          "title": "Current Usage",
          "value": "800 credits (80% of quota)",
          "short": true
        },
        {
          "title": "Quota",
          "value": "1000 credits/day",
          "short": true
        },
        {
          "title": "Monitor Type",
          "value": "Account-Level",
          "short": true
        }
      ],
      "footer": "Snowflake",
      "ts": 1651234567
    }
  ]
}
```

### **H. Monitoring Resource Monitors**
```sql
-- Check resource monitor usage
SELECT
    monitor_name,
    monitor_type,
    credit_quota,
    used_credits,
    remaining_credits,
    used_credits * 100.0 / credit_quota AS percent_used,
    start_time,
    end_time,
    frequency
FROM
    SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS
WHERE
    monitor_name = 'account_daily_monitor'
ORDER BY
    start_time DESC;

-- Check resource monitor history
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
    monitor_name = 'etl_monitor'
    AND notification_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    notification_time DESC;

-- Check which warehouses are assigned to a resource monitor
SELECT
    warehouse_name,
    resource_monitor
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSES
WHERE
    resource_monitor = 'etl_monitor';
```

### **I. Resource Monitor Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Start with Account-Level Monitor** | Create an account-level monitor to **track overall usage** | `CREATE RESOURCE MONITOR account_monitor WITH CREDIT_QUOTA = 10000` |
| **Use Warehouse-Level Monitors for Isolation** | Assign **separate monitors** to different workloads (ETL, Reporting, Ad-Hoc) | `CREATE RESOURCE MONITOR etl_monitor`, `CREATE RESOURCE MONITOR reporting_monitor` |
| **Set Warning Thresholds** | Set **warning thresholds** (e.g., 80%) to get **early alerts** | `NOTIFY_THRESHOLD = 80` |
| **Set Suspend Thresholds** | Set **suspend thresholds** (e.g., 100%) to **prevent overages** | `SUSPEND_THRESHOLD = 100` |
| **Notify the Right People** | Send notifications to **relevant users** (e.g., ETL team for ETL monitor) | `NOTIFY_USERS = ('etl_admin@company.com')` |
| **Use Custom Quota Periods for Fiscal Years** | Align quota periods with **fiscal years/quarters** | `FREQUENCY = CUSTOM, START_TIMESTAMP = '2026-04-01'` |
| **Monitor Usage Regularly** | Check `RESOURCE_MONITORS` and `RESOURCE_MONITOR_HISTORY` for usage trends | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS` |
| **Adjust Quotas Based on Usage** | Increase or decrease quotas based on **actual usage** | `ALTER RESOURCE MONITOR my_monitor SET CREDIT_QUOTA = 20000` |
| **Use Webhooks for Integration** | Integrate with **Slack, PagerDuty, or custom systems** | Configure webhook in Snowsight |
| **Test Suspension Behavior** | Test **suspension** in a non-production environment | `ALTER RESOURCE MONITOR test_monitor SET SUSPEND_THRESHOLD = 1` |
| **Document Quota Policies** | Document **quota policies** and **escalation paths** | Internal wiki or Confluence page |

## **3. Warehouse Management Deep Dive**

### **A. Warehouse Types**

| **Type** | **Description** | **Use Case** | **Scaling** | **Cost** | **Concurrency** |
|----------|-----------------|--------------|-------------|----------|-----------------|
| **Standard** | Single-cluster warehouse | General-purpose queries, small workloads | Fixed size | Low-Medium | Low (1-8 queries) |
| **Multi-Cluster** | Multi-cluster warehouse for high concurrency | High concurrency workloads (ETL, reporting) | Auto-scaling (1-10 clusters) | Medium-High | High (10-100+ queries) |
| **Serverless** | Serverless compute for specific operations | Snowpipe, Ingestion Service, Replication | Auto-scaling | Pay-per-use | Very High (1000+ operations) |

### **B. Warehouse Sizing**

| **Size** | **Compute (Credits/Hour)** | **Memory (GB)** | **Max Threads** | **Best For** | **Credit Cost/Hour** | **Concurrent Queries** |
|----------|----------------------------|-----------------|-----------------|--------------|----------------------|------------------------|
| **X-Small** | 1 | 16 | 8 | Development, testing, small queries | 0.28 | 1 |
| **Small** | 2 | 32 | 16 | Small workloads, medium queries | 0.56 | 2 |
| **Medium** | 4 | 64 | 32 | Medium workloads, large queries | 1.12 | 4 |
| **Large** | 8 | 128 | 64 | Large workloads, complex queries | 2.24 | 8 |
| **X-Large** | 16 | 256 | 128 | Very large workloads, high concurrency | 4.48 | 16 |
| **2X-Large** | 32 | 512 | 256 | Extremely large workloads | 8.96 | 32 |
| **3X-Large** | 64 | 1024 | 512 | Massive workloads | 17.92 | 64 |
| **4X-Large** | 128 | 2048 | 1024 | Largest workloads | 35.84 | 128 |

### **C. Warehouse Configuration Parameters**

| **Parameter** | **Description** | **Default** | **Valid Values** | **Performance Impact** |
|---------------|-----------------|-------------|------------------|------------------------|
| `WAREHOUSE_SIZE` | Size of the warehouse | `MEDIUM` | `XSMALL`, `SMALL`, `MEDIUM`, `LARGE`, `XLARGE`, `XXLARGE`, `XXXLARGE`, `XXXXLARGE` | Larger = more compute/memory |
| `MAX_CLUSTER_COUNT` | Max number of clusters (multi-cluster only) | `1` | 1-10 | Higher = more concurrency |
| `MIN_CLUSTER_COUNT` | Min number of clusters (multi-cluster only) | `1` | 1-10 | Higher = more idle clusters |
| `SCALING_POLICY` | Scaling policy for multi-cluster warehouses | `STANDARD` | `STANDARD`, `ECONOMY` | `STANDARD` = aggressive scaling, `ECONOMY` = conservative scaling |
| `AUTO_SUSPEND` | Time (in seconds) before warehouse suspends | `600` (10 min) | 0-86400 | Higher = longer idle time, higher cost |
| `AUTO_RESUME` | Whether warehouse resumes automatically | `TRUE` | `TRUE`, `FALSE` | `TRUE` = better user experience |
| `QUERY_PRIORITY` | Default priority for queries | `MEDIUM` | `HIGH`, `MEDIUM`, `LOW` | Higher = better performance for critical queries |
| `STATEMENT_TIMEOUT_IN_SECONDS` | Timeout (in seconds) for queries | `86400` (24 hr) | 0-86400 | Lower = prevents long-running queries |
| `QUERY_QUEUE_TIMEOUT_IN_SECONDS` | Timeout (in seconds) for queued queries | `300` (5 min) | 0-86400 | Lower = prevents queue backlogs |
| `RESOURCE_MONITOR` | Resource monitor for the warehouse | `NULL` | Resource monitor name | Limits credit usage |

### **D. Standard Warehouses**

#### **1. Definition**
- **Single-cluster warehouses** with **fixed compute and memory**.
- **Best for**: Small to medium workloads, development, testing, ad-hoc queries.

#### **2. When to Use**
✅ **Small workloads** (<10 concurrent queries).
✅ **Development and testing**.
✅ **Ad-hoc queries**.
✅ **Cost-sensitive environments**.

#### **3. When NOT to Use**
❌ **High concurrency workloads** (>10 concurrent queries).
❌ **Large, complex queries** (use larger warehouses).
❌ **Production ETL workloads** (use multi-cluster warehouses).

#### **4. Configuration Example**
```sql
-- Create a standard warehouse
CREATE WAREHOUSE dev_wh
  WAREHOUSE_SIZE = 'SMALL'
  AUTO_SUSPEND = 300  -- 5 minutes
  AUTO_RESUME = TRUE
  QUERY_PRIORITY = 'LOW'
  STATEMENT_TIMEOUT_IN_SECONDS = 300;  -- 5 minutes

-- Alter a standard warehouse
ALTER WAREHOUSE dev_wh SET WAREHOUSE_SIZE = 'MEDIUM';
```

### **E. Multi-Cluster Warehouses**

#### **1. Definition**
- **Multi-cluster warehouses** can **scale out** to handle **high concurrency** by adding **additional clusters**.
- Each **cluster** is a **separate warehouse** of the specified size.
- **Best for**: High concurrency workloads (ETL, reporting, dashboards).

#### **2. How Multi-Cluster Warehouses Work**
1. **Initial Cluster**:
   - The warehouse starts with **1 cluster** (default).
2. **Scaling Out**:
   - When **query demand increases**, Snowflake **adds clusters** (up to `MAX_CLUSTER_COUNT`).
   - Each cluster can **execute queries in parallel**.
3. **Scaling In**:
   - When **query demand decreases**, Snowflake **removes clusters** (down to `MIN_CLUSTER_COUNT`).
4. **Scaling Policies**:
   - **STANDARD**: Aggressively adds clusters to meet demand (default).
   - **ECONOMY**: Conservatively adds clusters to save costs.

```mermaid
%% Multi-Cluster Warehouse Scaling
flowchart TD
    A[("Initial State\n(1 Cluster)")] -->|Query Demand Increases| B[("Add Cluster\n(2 Clusters)")]
    B -->|Query Demand Increases| C[("Add Cluster\n(3 Clusters)")]
    C -->|Query Demand Decreases| B
    B -->|Query Demand Decreases| A

    subgraph ScalingPolicy["Scaling Policy"]
        D[("STANDARD\n(Aggressive)")]
        E[("ECONOMY\n(Conservative)")]
    end

    A --> D
    B --> D
    C --> D
    A --> E
    B --> E
    C --> E

    subgraph MaxClusters["Max Clusters"]
        F[("MAX_CLUSTER_COUNT = 4")]
    end
    C --> F

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef state fill:#4285f4,stroke:#1976d2;
    classDef policy fill:#ff9800,stroke:#f57c00;
    classDef max fill:#e91e63,stroke:#c2185b;
    class A,B,C state;
    class D,E policy;
    class F max;
```

#### **3. When to Use Multi-Cluster Warehouses**
✅ **High concurrency workloads** (>10 concurrent queries).
✅ **ETL pipelines** with many parallel tasks.
✅ **Reporting dashboards** with many simultaneous users.
✅ **Production environments** with variable workloads.

#### **4. When NOT to Use Multi-Cluster Warehouses**
❌ **Low concurrency workloads** (<10 concurrent queries; use standard warehouse).
❌ **Cost-sensitive environments** (multi-cluster warehouses are more expensive).
❌ **Small workloads** (use smaller warehouses).

#### **5. Multi-Cluster Warehouse Configuration**
```sql
-- Create a multi-cluster warehouse with STANDARD scaling
CREATE WAREHOUSE etl_mc_wh
  WAREHOUSE_SIZE = 'LARGE'
  MAX_CLUSTER_COUNT = 4
  MIN_CLUSTER_COUNT = 1
  SCALING_POLICY = 'STANDARD'
  AUTO_SUSPEND = 600
  AUTO_RESUME = TRUE
  QUERY_PRIORITY = 'HIGH'
  STATEMENT_TIMEOUT_IN_SECONDS = 3600;  -- 1 hour

-- Create a multi-cluster warehouse with ECONOMY scaling
CREATE WAREHOUSE reporting_mc_wh
  WAREHOUSE_SIZE = 'MEDIUM'
  MAX_CLUSTER_COUNT = 8
  MIN_CLUSTER_COUNT = 2
  SCALING_POLICY = 'ECONOMY'
  AUTO_SUSPEND = 1800  -- 30 minutes
  AUTO_RESUME = TRUE
  QUERY_PRIORITY = 'MEDIUM';

-- Alter a multi-cluster warehouse
ALTER WAREHOUSE etl_mc_wh SET MAX_CLUSTER_COUNT = 8;
ALTER WAREHOUSE etl_mc_wh SET SCALING_POLICY = 'ECONOMY';
```

#### **6. Multi-Cluster Warehouse Monitoring**
```sql
-- Check multi-cluster warehouse load
SELECT
    warehouse_name,
    cluster_number,
    start_time,
    end_time,
    running_queries,
    queued_queries,
    credit_usage
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE
    warehouse_name = 'etl_mc_wh'
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;

-- Check multi-cluster warehouse metering
SELECT
    warehouse_name,
    start_time,
    end_time,
    credits_used,
    query_type
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE
    warehouse_name = 'etl_mc_wh'
    AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;

-- Check current cluster count
SELECT
    warehouse_name,
    size,
    running_queries,
    queued_queries,
    cluster_number,
    total_clusters
FROM
    SNOWFLAKE.INFORMATION_SCHEMA.WAREHOUSE_MONITOR
WHERE
    warehouse_name = 'etl_mc_wh';
```

#### **7. Multi-Cluster Warehouse Best Practices**
| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Start with MAX_CLUSTER_COUNT = 2** | Test with 2 clusters before scaling up | `CREATE WAREHOUSE my_wh MAX_CLUSTER_COUNT = 2` |
| **Use STANDARD Scaling for Critical Workloads** | Use `STANDARD` scaling for performance-critical workloads | `ALTER WAREHOUSE my_wh SET SCALING_POLICY = 'STANDARD'` |
| **Use ECONOMY Scaling for Cost-Sensitive Workloads** | Use `ECONOMY` scaling to save costs | `ALTER WAREHOUSE my_wh SET SCALING_POLICY = 'ECONOMY'` |
| **Set MIN_CLUSTER_COUNT = 1** | Avoid idle clusters to save costs | `ALTER WAREHOUSE my_wh SET MIN_CLUSTER_COUNT = 1` |
| **Monitor Cluster Usage** | Check `WAREHOUSE_LOAD_HISTORY` for cluster usage | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY` |
| **Set AUTO_SUSPEND for Idle Warehouses** | Suspend warehouses when idle to save costs | `ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = 600` |
| **Use Separate Warehouses for Different Workloads** | Avoid resource contention | `CREATE WAREHOUSE etl_wh`, `CREATE WAREHOUSE reporting_wh` |
| **Set QUERY_PRIORITY for Critical Queries** | Prioritize critical queries | `ALTER WAREHOUSE my_wh SET QUERY_PRIORITY = 'HIGH'` |
| **Set STATEMENT_TIMEOUT for Long-Running Queries** | Prevent runaway queries | `ALTER WAREHOUSE my_wh SET STATEMENT_TIMEOUT_IN_SECONDS = 300` |

### **F. Serverless Warehouses**
- **Serverless warehouses** are used for **specific Snowflake operations** that do not require a traditional warehouse:
  - **Snowpipe**: File ingestion from cloud storage.
  - **Ingestion Service**: Row-based ingestion via REST API.
  - **Replication**: Database replication.
  - **Search Optimization Service**: Search index maintenance.
- **Pricing**: Pay-per-use (credits consumed based on operation).
- **No Configuration**: Serverless warehouses are **managed by Snowflake**.

## **4. Query Queues and Prioritization Deep Dive**

### **A. Query Queue Architecture**

```mermaid
%% Query Queue Architecture
flowchart TD
    subgraph Warehouse["Warehouse"]
        A[("Query Queue")] --> B[("Query 1\n(Priority: HIGH)")]
        A --> C[("Query 2\n(Priority: MEDIUM)")]
        A --> D[("Query 3\n(Priority: LOW)")]
        B --> E[("Running\n(Cluster 1)")]
        C --> F[("Queued")]
        D --> F
    end

    subgraph MultiCluster["Multi-Cluster Warehouse"]
        G[("Query Queue")] --> H[("Query 1\n(Priority: HIGH)")]
        G --> I[("Query 2\n(Priority: HIGH)")]
        G --> J[("Query 3\n(Priority: MEDIUM)")]
        H --> K[("Running\n(Cluster 1)")]
        I --> L[("Running\n(Cluster 2)")]
        J --> M[("Queued")]
    end

    subgraph Prioritization["Prioritization Rules"]
        N[("1. Priority (HIGH > MEDIUM > LOW)")]
        O[("2. Submission Time (FIFO within priority)")]
        P[("3. Query Size (Smaller queries first)")]
    end
    A --> N
    A --> O
    A --> P
    G --> N
    G --> O
    G --> P

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef warehouse fill:#29abe2,stroke:#1a8fb8;
    classDef multi fill:#ff9800,stroke:#f57c00;
    classDef prioritization fill:#4caf50,stroke:#2e7d32;
    class A,B,C,D warehouse;
    class G,H,I,J,K,L,M multi;
    class N,O,P prioritization;
```

### **B. How Query Queues Work**
1. **Query Submission**:
   - Queries are submitted to the **warehouse's query queue**.
2. **Prioritization**:
   - Queries are **prioritized** based on:
     - **Priority Level**: `HIGH` > `MEDIUM` > `LOW`.
     - **Submission Time**: First-in, first-out (FIFO) within the same priority.
     - **Query Size**: Smaller queries may be prioritized over larger ones.
3. **Execution**:
   - Queries are **executed** in order of priority.
   - In **multi-cluster warehouses**, queries are **distributed across clusters**.
4. **Queue Limits**:
   - **Max Queued Queries**: Default **50** (configurable via `QUERY_QUEUE_TIMEOUT_IN_SECONDS`).
   - **Queue Timeout**: Queries **time out** if they wait too long in the queue (default: **5 minutes**).

### **C. Priority Levels**

| **Priority** | **Description** | **Use Case** | **Example** |
|--------------|-----------------|--------------|-------------|
| **HIGH** | Highest priority; runs before MEDIUM and LOW | Critical production queries, SLAs | `ALTER WAREHOUSE my_wh SET QUERY_PRIORITY = 'HIGH'` |
| **MEDIUM** | Default priority; runs after HIGH, before LOW | General queries, reporting | Default |
| **LOW** | Lowest priority; runs after HIGH and MEDIUM | Ad-hoc queries, development | `ALTER SESSION SET QUERY_PRIORITY = 'LOW'` |

### **D. When to Use Query Prioritization**
✅ **Critical Production Queries**: Prioritize **SLA-bound queries** (e.g., customer-facing dashboards).
✅ **ETL Workloads**: Prioritize **ETL jobs** to ensure they complete on time.
✅ **Mixed Workloads**: Separate **production** (HIGH) from **development** (LOW) queries.
✅ **Resource Contention**: Manage **concurrent queries** in a shared warehouse.

### **E. When NOT to Use Query Prioritization**
❌ **Dedicated Warehouses**: If each workload has its own warehouse, prioritization is unnecessary.
❌ **Low Concurrency**: If the warehouse rarely has queued queries, prioritization adds complexity.
❌ **Serverless Workloads**: Prioritization does not apply to serverless operations.

### **F. Query Prioritization Configuration**

#### **1. Warehouse-Level Priority**
```sql
-- Set default priority for a warehouse
ALTER WAREHOUSE my_wh SET QUERY_PRIORITY = 'HIGH';

-- Set priority for a specific query (using query tag)
ALTER SESSION SET QUERY_TAG = 'priority=high';
```

#### **2. Session-Level Priority**
```sql
-- Set priority for the current session
ALTER SESSION SET QUERY_PRIORITY = 'HIGH';

-- Set priority for a specific query (using hint)
SELECT * FROM my_table /*+ PRIORITY(HIGH) */;
```

#### **3. User/Role-Level Priority**
```sql
-- Set priority for a specific user (via role)
ALTER ROLE high_priority_role SET QUERY_PRIORITY = 'HIGH';
GRANT ROLE high_priority_role TO USER my_user;

-- Set priority for a specific role
ALTER ROLE etl_role SET QUERY_PRIORITY = 'HIGH';
```

### **G. Query Queue Configuration**

| **Parameter** | **Description** | **Default** | **Valid Values** | **Performance Impact** |
|---------------|-----------------|-------------|------------------|------------------------|
| `QUERY_PRIORITY` | Default priority for queries | `MEDIUM` | `HIGH`, `MEDIUM`, `LOW` | Higher = better performance for critical queries |
| `STATEMENT_QUEUE_TIMEOUT_IN_SECONDS` | Timeout (in seconds) for queued queries | `300` (5 min) | 0-86400 | Lower = prevents queue backlogs |
| `STATEMENT_TIMEOUT_IN_SECONDS` | Timeout (in seconds) for running queries | `86400` (24 hr) | 0-86400 | Lower = prevents long-running queries |

```sql
-- Set query queue timeout
ALTER WAREHOUSE my_wh SET STATEMENT_QUEUE_TIMEOUT_IN_SECONDS = 60;  -- 1 minute

-- Set statement timeout
ALTER WAREHOUSE my_wh SET STATEMENT_TIMEOUT_IN_SECONDS = 300;  -- 5 minutes
```

### **H. Query Queue Monitoring**
```sql
-- Check queued queries
SELECT
    query_id,
    query_text,
    warehouse_name,
    priority,
    queue_time,
    execution_status
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    execution_status = 'QUEUED'
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY
    queue_time DESC;

-- Check warehouse queue status
SELECT
    warehouse_name,
    running_queries,
    queued_queries,
    total_queries
FROM
    SNOWFLAKE.INFORMATION_SCHEMA.WAREHOUSE_MONITOR
WHERE
    warehouse_name = 'my_wh';
```

### **I. Query Prioritization Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use HIGH Priority for Critical Queries** | Prioritize **SLA-bound queries** (e.g., customer dashboards) | `ALTER WAREHOUSE prod_wh SET QUERY_PRIORITY = 'HIGH'` |
| **Use MEDIUM Priority for General Queries** | Default priority for most queries | Default |
| **Use LOW Priority for Ad-Hoc Queries** | Deprioritize **development and ad-hoc queries** | `ALTER SESSION SET QUERY_PRIORITY = 'LOW'` |
| **Set Query Timeouts** | Prevent long-running queries from blocking others | `ALTER WAREHOUSE my_wh SET STATEMENT_TIMEOUT_IN_SECONDS = 300` |
| **Set Queue Timeouts** | Prevent queries from waiting too long in the queue | `ALTER WAREHOUSE my_wh SET STATEMENT_QUEUE_TIMEOUT_IN_SECONDS = 60` |
| **Monitor Queue Status** | Check `WAREHOUSE_MONITOR` for queued queries | `SELECT * FROM SNOWFLAKE.INFORMATION_SCHEMA.WAREHOUSE_MONITOR` |
| **Use Separate Warehouses for Different Priorities** | Avoid priority conflicts | `CREATE WAREHOUSE high_priority_wh`, `CREATE WAREHOUSE low_priority_wh` |
| **Use Query Tags for Prioritization** | Tag queries for prioritization | `ALTER SESSION SET QUERY_TAG = 'priority=high'` |
| **Test Priority Changes** | Test priority changes in a **non-production environment** | `ALTER WAREHOUSE test_wh SET QUERY_PRIORITY = 'HIGH'` |

## **5. Workload Isolation Strategies**

### **A. Why Workload Isolation Matters**
Workload isolation ensures:
1. **Performance Consistency**: Prevents **noisy neighbor** problems.
2. **Cost Control**: Limits **credit usage** per workload.
3. **Resource Allocation**: Dedicated resources for **critical workloads**.
4. **SLA Compliance**: Meets **performance SLAs** for different user groups.
5. **Security**: Isolates **sensitive workloads** (e.g., production vs. development).

### **B. Workload Isolation Strategies**

| **Strategy** | **Description** | **Implementation** | **Use Case** | **Pros** | **Cons** |
|--------------|-----------------|--------------------|--------------|----------|----------|
| **Separate Warehouses** | Use separate warehouses for different workloads | `CREATE WAREHOUSE etl_wh`, `CREATE WAREHOUSE reporting_wh` | Production vs. Development, ETL vs. Reporting | ✅ Full isolation, ✅ Independent scaling | ❌ Higher cost, ❌ More management |
| **Multi-Cluster Warehouses** | Use multi-cluster warehouses for high concurrency | `CREATE WAREHOUSE mc_wh MAX_CLUSTER_COUNT = 4` | High concurrency workloads | ✅ Auto-scaling, ✅ Cost-effective | ❌ Complex configuration, ❌ Shared resources |
| **Resource Monitors** | Limit credit usage per workload | `CREATE RESOURCE MONITOR etl_monitor WITH CREDIT_QUOTA = 5000` | Cost control, departmental chargeback | ✅ Cost control, ✅ Budget enforcement | ❌ No performance isolation, ❌ Suspension risk |
| **Query Prioritization** | Prioritize queries within a warehouse | `ALTER WAREHOUSE my_wh SET QUERY_PRIORITY = 'HIGH'` | Mixed workloads | ✅ Simple to implement, ✅ No additional cost | ❌ Limited isolation, ❌ Priority conflicts |
| **Query Tagging** | Tag queries for monitoring and prioritization | `ALTER SESSION SET QUERY_TAG = 'workload=etl'` | Cost allocation, monitoring | ✅ Fine-grained control, ✅ No additional cost | ❌ Manual tagging, ❌ No enforcement |
| **Serverless Operations** | Use serverless for specific workloads | Snowpipe, Ingestion Service | File ingestion, row ingestion | ✅ Auto-scaling, ✅ Pay-per-use | ❌ Limited to specific operations |

### **C. Workload Isolation Architecture**

```mermaid
%% Workload Isolation Architecture
flowchart TD
    subgraph Production["Production Workloads"]
        A[("ETL Warehouse\n(X-Large, Multi-Cluster)")] -->|Resource Monitor| B[("ETL Monitor\n(1000 Credits/Day)")]
        C[("Reporting Warehouse\n(Large, Multi-Cluster)")] -->|Resource Monitor| D[("Reporting Monitor\n(500 Credits/Day)")]
        E[("Ad-Hoc Warehouse\n(Medium)")] -->|Resource Monitor| F[("Ad-Hoc Monitor\n(100 Credits/Day)")]
    end

    subgraph Development["Development Workloads"]
        G[("Dev Warehouse\n(Small)")] -->|Resource Monitor| H[("Dev Monitor\n(50 Credits/Day)")]
        I[("Test Warehouse\n(Small)")] -->|Resource Monitor| J[("Test Monitor\n(50 Credits/Day)")]
    end

    subgraph Users["Users"]
        K[("ETL Team")] --> A
        L[("Reporting Team")] --> C
        M[("Analysts")] --> E
        N[("Developers")] --> G
        O[("Testers")] --> I
    end

    subgraph Serverless["Serverless Workloads"]
        P[("Snowpipe")] --> Q[("Serverless Compute")]
        R[("Ingestion Service")] --> Q
    end

    subgraph Monitoring["Monitoring"]
        S[("ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY")]
        T[("ACCOUNT_USAGE.RESOURCE_MONITORS")]
    end
    A --> S
    C --> S
    E --> S
    G --> S
    I --> S
    B --> T
    D --> T
    F --> T
    H --> T
    J --> T

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef production fill:#4285f4,stroke:#1976d2;
    classDef development fill:#ff9800,stroke:#f57c00;
    classDef users fill:#4caf50,stroke:#2e7d32;
    classDef serverless fill:#e91e63,stroke:#c2185b;
    classDef monitoring fill:#9c27b0,stroke:#7b1fa2;
    class A,C,E production;
    class G,I development;
    class K,L,M,N,O users;
    class P,R serverless;
    class B,D,F,H,J monitoring;
    class Q serverless;
    class S,T monitoring;
```

### **D. Workload Isolation Best Practices**

| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use Separate Warehouses for Production and Development** | Isolate **production workloads** from **development workloads** | `CREATE WAREHOUSE prod_wh`, `CREATE WAREHOUSE dev_wh` |
| **Use Multi-Cluster Warehouses for High Concurrency** | Use multi-cluster warehouses for **ETL and reporting** | `CREATE WAREHOUSE etl_wh MAX_CLUSTER_COUNT = 4` |
| **Use Resource Monitors for Cost Control** | Limit **credit usage** per workload | `CREATE RESOURCE MONITOR etl_monitor WITH CREDIT_QUOTA = 5000` |
| **Use Query Prioritization for Mixed Workloads** | Prioritize **critical queries** in shared warehouses | `ALTER WAREHOUSE shared_wh SET QUERY_PRIORITY = 'HIGH'` |
| **Use Query Tagging for Monitoring** | Tag queries for **cost allocation** and **monitoring** | `ALTER SESSION SET QUERY_TAG = 'workload=etl'` |
| **Use Serverless for Specific Workloads** | Use **Snowpipe** and **Ingestion Service** for file/row ingestion | `CREATE PIPE my_pipe AUTO_INGEST = TRUE` |
| **Monitor Workload Performance** | Check `WAREHOUSE_LOAD_HISTORY` and `QUERY_HISTORY` | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY` |
| **Right-Size Warehouses** | Use the **smallest warehouse** that meets performance requirements | `CREATE WAREHOUSE my_wh WAREHOUSE_SIZE = 'SMALL'` |
| **Set Auto-Suspend for Idle Warehouses** | Suspend warehouses when idle to **save costs** | `ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = 600` |
| **Document Workload Policies** | Document **warehouse assignments**, **resource monitors**, and **priority levels** | Internal wiki or Confluence page |

### **E. Workload Isolation Example: Production vs. Development**

#### **1. Production Workloads**
```sql
-- Production ETL Warehouse (Multi-Cluster)
CREATE WAREHOUSE prod_etl_wh
  WAREHOUSE_SIZE = 'X-LARGE'
  MAX_CLUSTER_COUNT = 4
  MIN_CLUSTER_COUNT = 1
  SCALING_POLICY = 'STANDARD'
  AUTO_SUSPEND = 1800  -- 30 minutes
  AUTO_RESUME = TRUE
  QUERY_PRIORITY = 'HIGH'
  RESOURCE_MONITOR = prod_etl_monitor;

-- Production Reporting Warehouse (Multi-Cluster)
CREATE WAREHOUSE prod_reporting_wh
  WAREHOUSE_SIZE = 'LARGE'
  MAX_CLUSTER_COUNT = 8
  MIN_CLUSTER_COUNT = 2
  SCALING_POLICY = 'ECONOMY'
  AUTO_SUSPEND = 1800
  AUTO_RESUME = TRUE
  QUERY_PRIORITY = 'MEDIUM'
  RESOURCE_MONITOR = prod_reporting_monitor;

-- Production Ad-Hoc Warehouse (Standard)
CREATE WAREHOUSE prod_adhoc_wh
  WAREHOUSE_SIZE = 'MEDIUM'
  AUTO_SUSPEND = 600  -- 10 minutes
  AUTO_RESUME = TRUE
  QUERY_PRIORITY = 'LOW'
  RESOURCE_MONITOR = prod_adhoc_monitor;

-- Production Resource Monitors
CREATE RESOURCE MONITOR prod_etl_monitor
  WITH CREDIT_QUOTA = 20000  -- 20,000 credits/day
  FREQUENCY = DAILY
  NOTIFY_USERS = ('etl_admin@company.com', 'data_team@company.com')
  NOTIFY_THRESHOLD = 80
  SUSPEND_THRESHOLD = 100;

CREATE RESOURCE MONITOR prod_reporting_monitor
  WITH CREDIT_QUOTA = 10000  -- 10,000 credits/day
  FREQUENCY = DAILY
  NOTIFY_USERS = ('reporting_admin@company.com')
  NOTIFY_THRESHOLD = 80
  SUSPEND_THRESHOLD = 100;

CREATE RESOURCE MONITOR prod_adhoc_monitor
  WITH CREDIT_QUOTA = 2000  -- 2,000 credits/day
  FREQUENCY = DAILY
  NOTIFY_USERS = ('analysts@company.com')
  NOTIFY_THRESHOLD = 80
  SUSPEND_THRESHOLD = 100;
```

#### **2. Development Workloads**
```sql
-- Development ETL Warehouse (Standard)
CREATE WAREHOUSE dev_etl_wh
  WAREHOUSE_SIZE = 'MEDIUM'
  AUTO_SUSPEND = 300  -- 5 minutes
  AUTO_RESUME = TRUE
  QUERY_PRIORITY = 'MEDIUM'
  RESOURCE_MONITOR = dev_etl_monitor;

-- Development Reporting Warehouse (Standard)
CREATE WAREHOUSE dev_reporting_wh
  WAREHOUSE_SIZE = 'SMALL'
  AUTO_SUSPEND = 300
  AUTO_RESUME = TRUE
  QUERY_PRIORITY = 'LOW'
  RESOURCE_MONITOR = dev_reporting_monitor;

-- Development Resource Monitors
CREATE RESOURCE MONITOR dev_etl_monitor
  WITH CREDIT_QUOTA = 1000  -- 1,000 credits/day
  FREQUENCY = DAILY
  NOTIFY_USERS = ('dev_team@company.com')
  NOTIFY_THRESHOLD = 80
  SUSPEND_THRESHOLD = 100;

CREATE RESOURCE MONITOR dev_reporting_monitor
  WITH CREDIT_QUOTA = 500  -- 500 credits/day
  FREQUENCY = DAILY
  NOTIFY_USERS = ('dev_team@company.com')
  NOTIFY_THRESHOLD = 80
  SUSPEND_THRESHOLD = 100;
```

#### **3. Serverless Workloads**
```sql
-- Snowpipe (Serverless)
CREATE STAGE my_s3_stage URL = 's3://my-bucket/path/';
CREATE PIPE my_pipe
  AUTO_INGEST = TRUE
  AS COPY INTO my_table FROM @my_s3_stage;

-- Ingestion Service (Serverless)
-- (No warehouse configuration needed; uses serverless compute)
```

#### **4. User Assignments**
```sql
-- Production Users
GRANT USAGE ON WAREHOUSE prod_etl_wh TO ROLE etl_role;
GRANT USAGE ON WAREHOUSE prod_reporting_wh TO ROLE reporting_role;
GRANT USAGE ON WAREHOUSE prod_adhoc_wh TO ROLE analyst_role;

-- Development Users
GRANT USAGE ON WAREHOUSE dev_etl_wh TO ROLE dev_etl_role;
GRANT USAGE ON WAREHOUSE dev_reporting_wh TO ROLE dev_reporting_role;

-- Assign roles to users
GRANT ROLE etl_role TO USER etl_user1, etl_user2;
GRANT ROLE reporting_role TO USER reporting_user1, reporting_user2;
GRANT ROLE analyst_role TO USER analyst1, analyst2;
GRANT ROLE dev_etl_role TO USER dev1, dev2;
GRANT ROLE dev_reporting_role TO USER dev1, dev2;
```

## **6. Performance and Cost Optimization**

### **A. Right-Sizing Warehouses**

#### **1. Warehouse Sizing Guidelines**
| **Workload Type** | **Recommended Size** | **Max Concurrent Queries** | **Credit Cost/Hour** | **Notes** |
|-------------------|----------------------|-----------------------------|----------------------|-----------|
| **Development/Testing** | X-Small | 1 | 0.28 | Small queries, low concurrency |
| **Ad-Hoc Queries** | Small | 2 | 0.56 | Medium queries, low concurrency |
| **Small ETL Jobs** | Medium | 4 | 1.12 | Small to medium workloads |
| **Medium ETL Jobs** | Large | 8 | 2.24 | Medium workloads, moderate concurrency |
| **Large ETL Jobs** | X-Large | 16 | 4.48 | Large workloads, high concurrency |
| **Production ETL** | 2X-Large | 32 | 8.96 | Very large workloads, very high concurrency |
| **Production Reporting** | 3X-Large | 64 | 17.92 | Massive workloads, highest concurrency |
| **Critical Production** | 4X-Large | 128 | 35.84 | Largest workloads, highest SLAs |

#### **2. Warehouse Sizing Workflow**
1. **Identify Workload Requirements**:
   - **Query Complexity**: Simple vs. complex queries.
   - **Concurrency**: Number of concurrent queries.
   - **Data Volume**: Amount of data scanned per query.
   - **SLA**: Performance requirements (e.g., <1 second, <10 seconds).

2. **Start Small**:
   - Begin with a **small warehouse** (e.g., Small or Medium).
   - Monitor **performance** and **credit usage**.

3. **Monitor Performance**:
   - Check `WAREHOUSE_LOAD_HISTORY` for **queue times** and **execution times**.
   - Check `QUERY_HISTORY` for **credit usage** and **bytes scanned**.

4. **Scale Up if Needed**:
   - If queries are **queued** or **slow**, increase the **warehouse size** or **max cluster count**.
   - If **credit usage is high**, optimize queries or use **result caching**.

5. **Scale Down if Underutilized**:
   - If the warehouse is **idle** or **underutilized**, decrease the **warehouse size** or **auto-suspend time**.

#### **3. Warehouse Sizing Example**
```sql
-- Step 1: Identify slow queries
SELECT
    warehouse_name,
    query_id,
    execution_time,
    queue_time,
    bytes_scanned,
    credits_used
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    warehouse_name = 'my_wh'
    AND execution_time > 10000  -- >10 seconds
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    execution_time DESC;

-- Step 2: Check warehouse utilization
SELECT
    warehouse_name,
    start_time,
    running_queries,
    queued_queries,
    credit_usage
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE
    warehouse_name = 'my_wh'
    AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;

-- Step 3: Increase warehouse size if needed
ALTER WAREHOUSE my_wh SET WAREHOUSE_SIZE = 'LARGE';

-- Step 4: Monitor after changes
SELECT
    warehouse_name,
    execution_time,
    queue_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    warehouse_name = 'my_wh'
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;
```

### **B. Auto-Suspend and Auto-Resume**

#### **1. Auto-Suspend**
- **Auto-Suspend**: Warehouse **suspends** after a period of **inactivity** (default: **10 minutes**).
- **Suspended Warehouses**:
  - **No credit usage** while suspended.
  - **Queries are queued** until the warehouse resumes.
- **Auto-Suspend Best Practices**:
  - Set **shorter auto-suspend times** for **development warehouses** (e.g., 5 minutes).
  - Set **longer auto-suspend times** for **production warehouses** (e.g., 30 minutes) to avoid frequent suspend/resume cycles.
  - **Disable auto-suspend** for **always-on warehouses** (e.g., 24/7 production workloads).

```sql
-- Set auto-suspend time (in seconds)
ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = 300;  -- 5 minutes
ALTER WAREHOUSE prod_wh SET AUTO_SUSPEND = 1800;  -- 30 minutes

-- Disable auto-suspend
ALTER WAREHOUSE always_on_wh SET AUTO_SUSPEND = NULL;
```

#### **2. Auto-Resume**
- **Auto-Resume**: Warehouse **automatically resumes** when a new query is submitted.
- **Resume Time**: Typically **1-10 seconds** (depends on warehouse size).
- **Auto-Resume Best Practices**:
  - Enable **auto-resume** for **interactive workloads** (e.g., BI tools, ad-hoc queries).
  - Disable **auto-resume** for **batch workloads** (e.g., ETL jobs) to avoid unnecessary resumes.

```sql
-- Enable auto-resume
ALTER WAREHOUSE my_wh SET AUTO_RESUME = TRUE;

-- Disable auto-resume
ALTER WAREHOUSE batch_wh SET AUTO_RESUME = FALSE;
```

### **C. Multi-Cluster Warehouse Scaling**

#### **1. Scaling Policies**
| **Policy** | **Description** | **When to Use** | **Example** |
|------------|-----------------|-----------------|-------------|
| **STANDARD** | Aggressively adds clusters to meet demand | Performance-critical workloads | `ALTER WAREHOUSE my_wh SET SCALING_POLICY = 'STANDARD'` |
| **ECONOMY** | Conservatively adds clusters to save costs | Cost-sensitive workloads | `ALTER WAREHOUSE my_wh SET SCALING_POLICY = 'ECONOMY'` |

#### **2. Scaling Best Practices**
| **Best Practice** | **Description** | **Example** |
|-------------------|-----------------|-------------|
| **Use STANDARD for Critical Workloads** | Ensure **performance SLAs** are met | `ALTER WAREHOUSE prod_wh SET SCALING_POLICY = 'STANDARD'` |
| **Use ECONOMY for Cost-Sensitive Workloads** | Save costs for **non-critical workloads** | `ALTER WAREHOUSE dev_wh SET SCALING_POLICY = 'ECONOMY'` |
| **Set MIN_CLUSTER_COUNT = 1** | Avoid idle clusters to save costs | `ALTER WAREHOUSE my_wh SET MIN_CLUSTER_COUNT = 1` |
| **Set MAX_CLUSTER_COUNT Based on Concurrency** | Set `MAX_CLUSTER_COUNT` to **2-3x average concurrency** | `ALTER WAREHOUSE my_wh SET MAX_CLUSTER_COUNT = 4` |
| **Monitor Cluster Usage** | Check `WAREHOUSE_LOAD_HISTORY` for cluster usage | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY` |
| **Start with MAX_CLUSTER_COUNT = 2** | Test with 2 clusters before scaling up | `CREATE WAREHOUSE my_wh MAX_CLUSTER_COUNT = 2` |

#### **3. Scaling Example**
```sql
-- Step 1: Check current cluster usage
SELECT
    warehouse_name,
    cluster_number,
    running_queries,
    queued_queries
FROM
    SNOWFLAKE.INFORMATION_SCHEMA.WAREHOUSE_MONITOR
WHERE
    warehouse_name = 'my_wh';

-- Step 2: Check historical cluster usage
SELECT
    warehouse_name,
    start_time,
    running_queries,
    queued_queries,
    cluster_number
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE
    warehouse_name = 'my_wh'
    AND start_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;

-- Step 3: Increase MAX_CLUSTER_COUNT if needed
ALTER WAREHOUSE my_wh SET MAX_CLUSTER_COUNT = 4;

-- Step 4: Switch to ECONOMY scaling if costs are too high
ALTER WAREHOUSE my_wh SET SCALING_POLICY = 'ECONOMY';
```

### **D. Query Timeout Management**

#### **1. Statement Timeout**
- **Statement Timeout**: Maximum time (in seconds) a query can run before being **canceled**.
- **Default**: **24 hours** (86400 seconds).
- **Best Practices**:
  - Set **shorter timeouts** for **interactive queries** (e.g., 30-300 seconds).
  - Set **longer timeouts** for **batch queries** (e.g., 3600 seconds = 1 hour).
  - Set **no timeout** for **long-running ETL jobs** (use `0` or `NULL`).

```sql
-- Set statement timeout (in seconds)
ALTER WAREHOUSE my_wh SET STATEMENT_TIMEOUT_IN_SECONDS = 300;  -- 5 minutes
ALTER WAREHOUSE batch_wh SET STATEMENT_TIMEOUT_IN_SECONDS = 3600;  -- 1 hour
ALTER WAREHOUSE etl_wh SET STATEMENT_TIMEOUT_IN_SECONDS = 0;  -- No timeout
```

#### **2. Queue Timeout**
- **Queue Timeout**: Maximum time (in seconds) a query can wait in the **queue** before being **canceled**.
- **Default**: **5 minutes** (300 seconds).
- **Best Practices**:
  - Set **shorter queue timeouts** for **interactive queries** (e.g., 60 seconds).
  - Set **longer queue timeouts** for **batch queries** (e.g., 300-600 seconds).
  - Set **no queue timeout** for **critical queries** (use `0` or `NULL`).

```sql
-- Set queue timeout (in seconds)
ALTER WAREHOUSE my_wh SET STATEMENT_QUEUE_TIMEOUT_IN_SECONDS = 60;  -- 1 minute
ALTER WAREHOUSE batch_wh SET STATEMENT_QUEUE_TIMEOUT_IN_SECONDS = 600;  -- 10 minutes
```

### **E. Cost Optimization Strategies**

| **Strategy** | **Description** | **Implementation** | **Cost Savings** | **Performance Impact** |
|--------------|-----------------|--------------------|------------------|-------------------------|
| **Right-Size Warehouses** | Use the smallest warehouse that meets performance requirements | `CREATE WAREHOUSE my_wh WAREHOUSE_SIZE = 'SMALL'` | ⭐⭐⭐⭐ | ⬆️ Minimal |
| **Use Auto-Suspend** | Suspend warehouses when idle | `ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = 300` | ⭐⭐⭐⭐⭐ | ⬆️ Minimal |
| **Use Multi-Cluster Warehouses** | Scale out for high concurrency | `CREATE WAREHOUSE my_wh MAX_CLUSTER_COUNT = 4` | ⭐⭐⭐ | ⬆️ Improved concurrency |
| **Use ECONOMY Scaling** | Conservatively add clusters | `ALTER WAREHOUSE my_wh SET SCALING_POLICY = 'ECONOMY'` | ⭐⭐⭐⭐ | ⬇️ Slower scaling |
| **Use Resource Monitors** | Limit credit usage | `CREATE RESOURCE MONITOR my_monitor WITH CREDIT_QUOTA = 1000` | ⭐⭐⭐⭐⭐ | ⬆️ Prevents overages |
| **Use Query Timeouts** | Prevent long-running queries | `ALTER WAREHOUSE my_wh SET STATEMENT_TIMEOUT_IN_SECONDS = 300` | ⭐⭐⭐ | ⬆️ Prevents runaway queries |
| **Use Result Caching** | Cache query results | `ALTER SESSION SET USE_CACHED_RESULTS = TRUE` | ⭐⭐⭐⭐ | ⬆️ Faster queries |
| **Use Clustering** | Reduce bytes scanned | `ALTER TABLE my_table CLUSTER BY (date)` | ⭐⭐⭐⭐ | ⬆️ Faster queries |
| **Use Materialized Views** | Pre-compute expensive queries | `CREATE MATERIALIZED VIEW my_mv AS SELECT ...` | ⭐⭐⭐ | ⬆️ Faster queries |
| **Optimize Queries** | Reduce bytes scanned and execution time | Rewrite queries, add filters | ⭐⭐⭐⭐ | ⬆️ Faster queries |
| **Use Serverless for Specific Workloads** | Use Snowpipe, Ingestion Service | `CREATE PIPE my_pipe AUTO_INGEST = TRUE` | ⭐⭐⭐⭐⭐ | ⬆️ Pay-per-use |

## **7. Monitoring and Alerting**

### **A. Key Monitoring Views**

| **View** | **Purpose** | **Retention** | **Key Columns** | **Example Query** |
|----------|-------------|---------------|-----------------|-------------------|
| `WAREHOUSE_LOAD_HISTORY` | Warehouse utilization history | 365 days | `warehouse_name`, `start_time`, `end_time`, `running_queries`, `queued_queries`, `credit_usage` | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY WHERE warehouse_name = 'MY_WH' AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP());` |
| `WAREHOUSE_METERING_HISTORY` | Warehouse credit usage history | 365 days | `warehouse_name`, `start_time`, `end_time`, `credits_used`, `query_type` | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY WHERE warehouse_name = 'MY_WH' AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP());` |
| `QUERY_HISTORY` | Query execution history | 365 days | `query_id`, `query_text`, `warehouse_name`, `execution_time`, `queue_time`, `bytes_scanned`, `credits_used`, `execution_status` | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY WHERE warehouse_name = 'MY_WH' AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP());` |
| `RESOURCE_MONITORS` | Resource monitor usage | 365 days | `monitor_name`, `monitor_type`, `credit_quota`, `used_credits`, `remaining_credits`, `start_time`, `end_time` | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS WHERE monitor_name = 'MY_MONITOR';` |
| `RESOURCE_MONITOR_HISTORY` | Resource monitor notification history | 365 days | `monitor_name`, `notification_time`, `notification_type`, `threshold_reached`, `current_usage`, `credit_quota` | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITOR_HISTORY WHERE monitor_name = 'MY_MONITOR' AND notification_time > DATEADD('day', -7, CURRENT_TIMESTAMP());` |
| `WAREHOUSE_MONITOR` | Current warehouse status | Session | `warehouse_name`, `size`, `running_queries`, `queued_queries`, `cluster_number`, `total_clusters` | `SELECT * FROM SNOWFLAKE.INFORMATION_SCHEMA.WAREHOUSE_MONITOR;` |

### **B. Workload Monitoring Dashboard**

```sql
-- Workload Monitoring Dashboard
CREATE VIEW WORKLOAD_MONITORING_DASHBOARD AS
SELECT
    -- Warehouse Info
    w.name AS warehouse_name,
    w.size AS warehouse_size,
    w.max_cluster_count,
    w.scaling_policy,
    w.auto_suspend,
    w.auto_resume,

    -- Resource Monitor Info
    rm.name AS resource_monitor_name,
    rm.credit_quota,
    rm.used_credits,
    rm.remaining_credits,
    rm.used_credits * 100.0 / NULLIF(rm.credit_quota, 0) AS percent_used,

    -- Current Load
    wm.running_queries,
    wm.queued_queries,
    wm.total_queries,
    wm.cluster_number,
    wm.total_clusters,

    -- Recent Credit Usage
    wh.credits_used AS recent_credits_used,
    wh.start_time AS recent_usage_start,
    wh.end_time AS recent_usage_end,

    -- Recent Query Performance
    qh.avg_execution_time,
    qh.avg_queue_time,
    qh.avg_bytes_scanned,
    qh.query_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSES w
LEFT JOIN
    SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS rm
    ON w.resource_monitor = rm.name
LEFT JOIN
    SNOWFLAKE.INFORMATION_SCHEMA.WAREHOUSE_MONITOR wm
    ON w.name = wm.warehouse_name
LEFT JOIN (
    SELECT
        warehouse_name,
        SUM(credits_used) AS credits_used,
        MIN(start_time) AS start_time,
        MAX(end_time) AS end_time
    FROM
        SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
    WHERE
        start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
    GROUP BY
        warehouse_name
) wh ON w.name = wh.warehouse_name
LEFT JOIN (
    SELECT
        warehouse_name,
        AVG(execution_time) AS avg_execution_time,
        AVG(queue_time) AS avg_queue_time,
        AVG(bytes_scanned) AS avg_bytes_scanned,
        COUNT(*) AS query_count
    FROM
        SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE
        start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
    GROUP BY
        warehouse_name
) qh ON w.name = qh.warehouse_name;
```

### **C. Proactive Alerts**

#### **1. Warehouse Overloaded Alert**
```sql
CREATE OR REPLACE ALERT WAREHOUSE_OVERLOADED_ALERT
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'
AS
  SELECT
    warehouse_name,
    running_queries,
    queued_queries,
    cluster_number,
    total_clusters,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.INFORMATION_SCHEMA.WAREHOUSE_MONITOR
  WHERE
    queued_queries > 0  -- Queries are queued
    OR running_queries = total_clusters * 10  -- All clusters are busy
  AND
    warehouse_name IN ('prod_etl_wh', 'prod_reporting_wh');  -- Monitor critical warehouses
```

#### **2. Resource Monitor Warning Alert**
```sql
CREATE OR REPLACE ALERT RESOURCE_MONITOR_WARNING_ALERT
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'
AS
  SELECT
    monitor_name,
    credit_quota,
    used_credits,
    remaining_credits,
    used_credits * 100.0 / credit_quota AS percent_used,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS
  WHERE
    used_credits * 100.0 / credit_quota > 80  -- >80% used
    AND monitor_name IN ('prod_etl_monitor', 'prod_reporting_monitor');
```

#### **3. Resource Monitor Limit Alert**
```sql
CREATE OR REPLACE ALERT RESOURCE_MONITOR_LIMIT_ALERT
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'
AS
  SELECT
    monitor_name,
    credit_quota,
    used_credits,
    remaining_credits,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS
  WHERE
    used_credits >= credit_quota  -- Limit reached
    AND monitor_name IN ('prod_etl_monitor', 'prod_reporting_monitor');
```

#### **4. Long-Running Query Alert**
```sql
CREATE OR REPLACE ALERT LONG_RUNNING_QUERY_ALERT
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'
AS
  SELECT
    query_id,
    query_text,
    warehouse_name,
    execution_time,
    start_time,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE
    execution_time > 300  -- >5 minutes
    AND execution_status = 'RUNNING'
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
    AND warehouse_name IN ('prod_etl_wh', 'prod_reporting_wh');
```

#### **5. High Queue Time Alert**
```sql
CREATE OR REPLACE ALERT HIGH_QUEUE_TIME_ALERT
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'
AS
  SELECT
    query_id,
    query_text,
    warehouse_name,
    queue_time,
    start_time,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE
    queue_time > 60  -- >1 minute
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
    AND warehouse_name IN ('prod_etl_wh', 'prod_reporting_wh');
```

### **D. Workload Troubleshooting Runbooks**

#### **1. Warehouse Overloaded**
**Symptoms**:
- High `queued_queries` in `WAREHOUSE_MONITOR`.
- High `queue_time` in `QUERY_HISTORY`.
- Slow query performance.

**Runbook**:
```sql
-- Step 1: Identify overloaded warehouses
SELECT
    warehouse_name,
    running_queries,
    queued_queries,
    cluster_number,
    total_clusters
FROM
    SNOWFLAKE.INFORMATION_SCHEMA.WAREHOUSE_MONITOR
WHERE
    queued_queries > 0
ORDER BY
    queued_queries DESC;

-- Step 2: Check recent query history
SELECT
    query_id,
    query_text,
    execution_time,
    queue_time,
    warehouse_name
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    warehouse_name = 'overloaded_wh'
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY
    queue_time DESC;

-- Step 3: Increase warehouse size or max cluster count
ALTER WAREHOUSE overloaded_wh SET WAREHOUSE_SIZE = 'X-LARGE';
ALTER WAREHOUSE overloaded_wh SET MAX_CLUSTER_COUNT = 8;

-- Step 4: Check if issue is resolved
SELECT
    warehouse_name,
    running_queries,
    queued_queries
FROM
    SNOWFLAKE.INFORMATION_SCHEMA.WAREHOUSE_MONITOR
WHERE
    warehouse_name = 'overloaded_wh';
```

#### **2. Resource Monitor Limit Reached**
**Symptoms**:
- Warehouses are **suspended**.
- `SUSPEND_THRESHOLD` reached in `RESOURCE_MONITORS`.

**Runbook**:
```sql
-- Step 1: Check resource monitor status
SELECT
    monitor_name,
    credit_quota,
    used_credits,
    remaining_credits,
    suspend_threshold
FROM
    SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS
WHERE
    used_credits >= credit_quota
ORDER BY
    used_credits DESC;

-- Step 2: Check suspended warehouses
SELECT
    warehouse_name,
    resource_monitor,
    state
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSES
WHERE
    state = 'SUSPENDED'
ORDER BY
    warehouse_name;

-- Step 3: Resume suspended warehouses (temporary fix)
ALTER WAREHOUSE suspended_wh RESUME;

-- Step 4: Increase credit quota (permanent fix)
ALTER RESOURCE MONITOR my_monitor SET CREDIT_QUOTA = 20000;

-- Step 5: Investigate high credit usage
SELECT
    query_id,
    query_text,
    warehouse_name,
    credits_used,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    warehouse_name IN (SELECT warehouse_name FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSES WHERE resource_monitor = 'my_monitor')
    AND start_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY
    credits_used DESC;
```

#### **3. Long-Running Queries**
**Symptoms**:
- High `execution_time` in `QUERY_HISTORY`.
- Queries **time out** or **consume excessive credits**.

**Runbook**:
```sql
-- Step 1: Identify long-running queries
SELECT
    query_id,
    query_text,
    warehouse_name,
    execution_time,
    bytes_scanned,
    credits_used,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    execution_time > 300  -- >5 minutes
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY
    execution_time DESC;

-- Step 2: Check query profile for bottlenecks
SELECT * FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.QUERY_PROFILE('query_id'));

-- Step 3: Optimize the query (add filters, use clustering, etc.)
-- Example: Add a filter to reduce bytes scanned
ALTER TABLE my_table CLUSTER BY (date);
SELECT * FROM my_table WHERE date > '2023-01-01';  -- Filter on clustered column

-- Step 4: Set a statement timeout for the warehouse
ALTER WAREHOUSE my_wh SET STATEMENT_TIMEOUT_IN_SECONDS = 300;  -- 5 minutes

-- Step 5: Set a statement timeout for the specific query
SELECT * FROM my_table /*+ STATEMENT_TIMEOUT(300) */;
```

#### **4. High Queue Time**
**Symptoms**:
- High `queue_time` in `QUERY_HISTORY`.
- Queries **waiting in queue** for a long time.

**Runbook**:
```sql
-- Step 1: Identify queries with high queue time
SELECT
    query_id,
    query_text,
    warehouse_name,
    queue_time,
    execution_time,
    start_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE
    queue_time > 60  -- >1 minute
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY
    queue_time DESC;

-- Step 2: Check warehouse load
SELECT
    warehouse_name,
    running_queries,
    queued_queries,
    cluster_number,
    total_clusters
FROM
    SNOWFLAKE.INFORMATION_SCHEMA.WAREHOUSE_MONITOR
WHERE
    warehouse_name = 'my_wh';

-- Step 3: Increase warehouse size or max cluster count
ALTER WAREHOUSE my_wh SET WAREHOUSE_SIZE = 'X-LARGE';
ALTER WAREHOUSE my_wh SET MAX_CLUSTER_COUNT = 8;

-- Step 4: Set query priority for critical queries
ALTER WAREHOUSE my_wh SET QUERY_PRIORITY = 'HIGH';

-- Step 5: Set queue timeout for the warehouse
ALTER WAREHOUSE my_wh SET STATEMENT_QUEUE_TIMEOUT_IN_SECONDS = 60;  -- 1 minute
```

## **8. Decision Matrix: Workload Management Strategies**

### **Mermaid: Workload Management Strategy Selection**
```mermaid
%% Workload Management Strategy Selection
flowchart TD
    A[("Workload Management\nRequirement")] --> B{Workload Type?}
    B -->|ETL| C[("Use Multi-Cluster Warehouse\n(High Concurrency)")]
    B -->|Reporting| D[("Use Multi-Cluster Warehouse\n(High Concurrency)")]
    B -->|Ad-Hoc| E[("Use Standard Warehouse\n(Low Concurrency)")]
    B -->|Development| F[("Use Small Warehouse\n(Cost-Effective)")]

    A --> G{Performance Requirements?}
    G -->|High (SLA < 1s)| H[("Use Multi-Cluster + STANDARD Scaling")]
    G -->|Medium (SLA < 10s)| I[("Use Multi-Cluster + ECONOMY Scaling")]
    G -->|Low (SLA > 10s)| J[("Use Standard Warehouse")]

    A --> K{Cost Constraints?}
    K -->|Strict Budget| L[("Use Resource Monitors + Small Warehouses")]
    K -->|Flexible Budget| M[("Use Larger Warehouses + Multi-Cluster")]

    A --> N{Concurrency Requirements?}
    N -->|High (>10 queries)| O[("Use Multi-Cluster Warehouse")]
    N -->|Medium (5-10 queries)| P[("Use Standard Warehouse + Query Prioritization")]
    N -->|Low (<5 queries)| Q[("Use Standard Warehouse")]

    A --> R{Isolation Requirements?}
    R -->|Full Isolation| S[("Use Separate Warehouses")]
    R -->|Partial Isolation| T[("Use Multi-Cluster + Query Prioritization")]
    R -->|No Isolation| U[("Use Shared Warehouse + Resource Monitors")]

    C --> V[("MAX_CLUSTER_COUNT = 4-8\nSCALING_POLICY = STANDARD")]
    D --> V
    E --> W[("WAREHOUSE_SIZE = MEDIUM\nAUTO_SUSPEND = 600")]
    F --> X[("WAREHOUSE_SIZE = SMALL\nAUTO_SUSPEND = 300")]

    H --> Y[("MAX_CLUSTER_COUNT = 8\nSCALING_POLICY = STANDARD")]
    I --> Z[("MAX_CLUSTER_COUNT = 4\nSCALING_POLICY = ECONOMY")]
    J --> W

    L --> AA[("CREDIT_QUOTA = 1000-5000\nNOTIFY_THRESHOLD = 80")]
    M --> AB[("WAREHOUSE_SIZE = X-LARGE\nMAX_CLUSTER_COUNT = 8")]

    O --> AC[("MAX_CLUSTER_COUNT = 4-8")]
    P --> AD[("QUERY_PRIORITY = HIGH/MEDIUM/LOW")]
    Q --> W

    S --> AE[("Separate Warehouses for ETL/Reporting/Ad-Hoc")]
    T --> AF[("Multi-Cluster + QUERY_PRIORITY")]
    U --> AG[("Resource Monitors + Query Prioritization")]

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80,81,82,83,84,85,86,87,88,89,90,91,92,93,94,95,96,97,98,99,100,101,102,103,104,105,106,107,108,109,110 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef etl fill:#4285f4,stroke:#1976d2;
    classDef reporting fill:#ff9800,stroke:#f57c00;
    classDef adhoc fill:#4caf50,stroke:#2e7d32;
    classDef dev fill:#e91e63,stroke:#c2185b;
    classDef performance fill:#9c27b0,stroke:#7b1fa2;
    classDef cost fill:#f44336,stroke:#d32f2f;
    classDef concurrency fill:#009688,stroke:#00796b;
    classDef isolation fill:#795548,stroke:#5d4037;
    classDef config fill:#2196f3,stroke:#03a9f4;
    class A default;
    class B etl;
    class C,D etl;
    class E adhoc;
    class F dev;
    class G performance;
    class H,I performance;
    class J adhoc;
    class K cost;
    class L,M cost;
    class N concurrency;
    class O,P,Q concurrency;
    class R isolation;
    class S,T,U isolation;
    class V,W,X,Y,Z,AA,AB,AC,AD,AE,AF,AG config;
```

### **Quick Reference Table: Workload Management Strategies**

| **Workload Type** | **Warehouse Type** | **Warehouse Size** | **Max Cluster Count** | **Scaling Policy** | **Auto-Suspend** | **Resource Monitor** | **Query Priority** | **Best For** |
|-------------------|--------------------|--------------------|-----------------------|-------------------|----------------|----------------------|----------------|--------------|
| **ETL (High Concurrency)** | Multi-Cluster | Large-X-Large | 4-8 | STANDARD | 1800 | ✅ Yes | HIGH | Production ETL, high concurrency |
| **Reporting (High Concurrency)** | Multi-Cluster | Medium-X-Large | 4-8 | ECONOMY | 1800 | ✅ Yes | MEDIUM | Production reporting, dashboards |
| **Ad-Hoc (Low Concurrency)** | Standard | Small-Medium | 1 | N/A | 600 | ✅ Yes | LOW | Ad-hoc queries, development |
| **Development** | Standard | X-Small-Small | 1 | N/A | 300 | ✅ Yes | LOW | Development, testing |
| **Production (SLA < 1s)** | Multi-Cluster | X-Large-2X-Large | 8 | STANDARD | 1800 | ✅ Yes | HIGH | Critical production workloads |
| **Production (SLA < 10s)** | Multi-Cluster | Large-X-Large | 4-8 | ECONOMY | 1800 | ✅ Yes | MEDIUM | Production workloads with moderate SLAs |
| **Cost-Sensitive** | Standard/Multi-Cluster | Small-Medium | 1-2 | ECONOMY | 300-600 | ✅ Yes | LOW | Cost-sensitive environments |
| **High Isolation** | Separate Warehouses | Varies | Varies | Varies | Varies | ✅ Yes | Varies | Full workload isolation |
| **Partial Isolation** | Multi-Cluster | Varies | 4-8 | STANDARD/ECONOMY | 1800 | ✅ Yes | HIGH/MEDIUM/LOW | Partial workload isolation |

## **9. Key Engineering Principles for Workload Management**

### **A. Core Principles**

1. **Isolate Critical Workloads**:
   - Use **separate warehouses** for **production vs. development**, **ETL vs. reporting**.
   - Use **resource monitors** to **limit credit usage** per workload.

2. **Right-Size Warehouses**:
   - Use the **smallest warehouse** that meets **performance SLAs**.
   - **Monitor usage** and **adjust sizes** as needed.

3. **Auto-Suspend Idle Warehouses**:
   - **Suspend warehouses** when idle to **save credits**.
   - Use **shorter auto-suspend times** for **development** and **longer times** for **production**.

4. **Use Multi-Cluster for High Concurrency**:
   - Use **multi-cluster warehouses** for **high concurrency workloads** (ETL, reporting).
   - Use **STANDARD scaling** for **performance-critical workloads** and **ECONOMY scaling** for **cost-sensitive workloads**.

5. **Prioritize Critical Queries**:
   - Use **query prioritization** (`HIGH`, `MEDIUM`, `LOW`) to **prioritize critical queries**.
   - Use **separate warehouses** for **different priority levels**.

6. **Monitor and Alert**:
   - **Monitor warehouse load**, **query performance**, and **credit usage**.
   - **Set up alerts** for **overloaded warehouses**, **long-running queries**, and **resource monitor limits**.

7. **Optimize Queries**:
   - **Reduce bytes scanned** (use filters, clustering, partition pruning).
   - **Use result caching** for repetitive queries.
   - **Use materialized views** for expensive aggregations.

8. **Use Serverless for Specific Workloads**:
   - Use **Snowpipe** and **Ingestion Service** for **file/row ingestion**.
   - Use **serverless compute** for **replication** and **search optimization**.

9. **Document Workload Policies**:
   - Document **warehouse assignments**, **resource monitors**, **priority levels**, and **SLA requirements**.
   - **Train users** on workload management best practices.

10. **Test Changes in Non-Production**:
    - **Test warehouse changes** (sizing, scaling, prioritization) in **non-production environments** before applying to production.
    - **Monitor impact** on performance and costs.

### **B. Production Checklist for Workload Management**

#### **Resource Monitors**
- [ ] **Create Resource Monitors** for all **production workloads** (ETL, reporting, ad-hoc).
- [ ] **Set Credit Quotas** based on **budget** and **usage patterns**.
- [ ] **Set Warning Thresholds** (e.g., 80%) for **early alerts**.
- [ ] **Set Suspend Thresholds** (e.g., 100%) to **prevent overages**.
- [ ] **Notify the Right People** (e.g., ETL team for ETL monitor).
- [ ] **Use Webhooks** for **Slack/PagerDuty integration**.
- [ ] **Monitor Resource Monitor Usage** regularly.
- [ ] **Adjust Quotas** based on **actual usage**.
- [ ] **Test Suspension Behavior** in a non-production environment.

#### **Warehouses**
- [ ] **Create Separate Warehouses** for **production vs. development**, **ETL vs. reporting**.
- [ ] **Right-Size Warehouses** based on **workload requirements**.
- [ ] **Use Multi-Cluster Warehouses** for **high concurrency workloads**.
- [ ] **Set Scaling Policy** (`STANDARD` for performance, `ECONOMY` for cost).
- [ ] **Set Auto-Suspend** to **suspend idle warehouses**.
- [ ] **Set Auto-Resume** for **interactive workloads**.
- [ ] **Set Query Priority** for **critical queries**.
- [ ] **Set Statement Timeout** to **prevent long-running queries**.
- [ ] **Set Queue Timeout** to **prevent queue backlogs**.
- [ ] **Assign Resource Monitors** to **limit credit usage**.
- [ ] **Monitor Warehouse Load** regularly.
- [ ] **Adjust Warehouse Sizes** based on **usage patterns**.

#### **Query Queues and Prioritization**
- [ ] **Use Query Prioritization** for **mixed workloads**.
- [ ] **Set HIGH Priority** for **critical production queries**.
- [ ] **Set MEDIUM Priority** for **general queries**.
- [ ] **Set LOW Priority** for **ad-hoc and development queries**.
- [ ] **Set Statement Timeout** to **prevent long-running queries**.
- [ ] **Set Queue Timeout** to **prevent queue backlogs**.
- [ ] **Monitor Queue Status** regularly.
- [ ] **Adjust Priorities** based on **workload requirements**.

#### **Workload Isolation**
- [ ] **Use Separate Warehouses** for **different workloads** (ETL, reporting, ad-hoc).
- [ ] **Use Resource Monitors** to **limit credit usage** per workload.
- [ ] **Use Query Prioritization** for **mixed workloads**.
- [ ] **Use Query Tagging** for **cost allocation** and **monitoring**.
- [ ] **Use Serverless** for **specific workloads** (Snowpipe, Ingestion Service).
- [ ] **Monitor Workload Performance** regularly.
- [ ] **Document Workload Policies** (warehouse assignments, resource monitors, priority levels).

#### **Monitoring and Alerting**
- [ ] **Set Up Monitoring** for **warehouse load**, **query performance**, and **credit usage**.
- [ ] **Set Up Alerts** for:
  - **Warehouse overloaded** (high `queued_queries`).
  - **Resource monitor warning** (80% of quota used).
  - **Resource monitor limit reached** (100% of quota used).
  - **Long-running queries** (>5 minutes).
  - **High queue time** (>1 minute).
- [ ] **Monitor Alerts** and **take action** as needed.
- [ ] **Review Monitoring Data** regularly to **identify trends** and **optimize workloads**.

### **C. Bottom Line: Workload Management in Snowflake**

| **Metric** | **Standard Warehouse** | **Multi-Cluster Warehouse** | **Resource Monitors** | **Query Prioritization** | **Best Choice** |
|------------|-------------------------|-----------------------------|------------------------|--------------------------|-----------------|
| **Concurrency** | Low (1-8 queries) | High (10-100+ queries) | N/A | Medium | Multi-Cluster for high concurrency |
| **Cost** | Low-Medium | Medium-High | Low (no additional cost) | Low (no additional cost) | Standard for cost-sensitive workloads |
| **Performance** | Medium | High | N/A | Medium | Multi-Cluster for performance-critical workloads |
| **Scalability** | Fixed | Auto-scaling | N/A | N/A | Multi-Cluster for scalable workloads |
| **Isolation** | Full (separate warehouse) | Partial (shared clusters) | N/A | Partial (shared warehouse) | Separate warehouses for full isolation |
| **Cost Control** | Limited | Limited | ✅ High | Limited | Resource Monitors for cost control |
| **SLA Compliance** | Medium | High | N/A | Medium | Multi-Cluster + STANDARD scaling for SLAs |
| **Best For** | Small workloads, ad-hoc queries | High concurrency workloads (ETL, reporting) | Cost control, departmental chargeback | Mixed workloads | Depends on requirements |

**Final Recommendations**:
- Use **Multi-Cluster Warehouses** for **high concurrency workloads** (ETL, reporting).
- Use **Standard Warehouses** for **small workloads** (ad-hoc queries, development).
- Use **Resource Monitors** for **cost control** and **budget enforcement**.
- Use **Query Prioritization** for **mixed workloads** in shared warehouses.
- Use **Separate Warehouses** for **full isolation** of critical workloads.
- **Monitor and Alert** on warehouse load, query performance, and credit usage.
- **Right-Size Warehouses** and **adjust configurations** based on usage patterns.

## **10. Production-Ready Snippets**

### **A. Resource Monitor Setup**

#### **1. Account-Level Resource Monitor**
```sql
-- Create an account-level resource monitor with daily quota
CREATE RESOURCE MONITOR account_daily_monitor
  WITH CREDIT_QUOTA = 10000  -- 10,000 credits per day
  FREQUENCY = DAILY
  START_TIMESTAMP = DATEADD('day', -1, CURRENT_TIMESTAMP())
  NOTIFY_USERS = (SELECT user_name FROM SNOWFLAKE.ACCOUNT_USAGE.USERS WHERE role_name = 'ACCOUNTADMIN')
  NOTIFY_THRESHOLD = 80  -- Warn at 80% of quota
  SUSPEND_THRESHOLD = 100  -- Suspend at 100% of quota
  SUSPEND_IMMEDIATELY = FALSE;

-- Assign the monitor to the account
ALTER ACCOUNT SET RESOURCE_MONITOR = account_daily_monitor;
```

#### **2. Warehouse-Level Resource Monitors**
```sql
-- Create resource monitors for different workloads
CREATE RESOURCE MONITOR etl_monitor
  WITH CREDIT_QUOTA = 5000  -- 5,000 credits per day
  FREQUENCY = DAILY
  NOTIFY_USERS = ('etl_admin@company.com', 'data_team@company.com')
  NOTIFY_THRESHOLD = 70
  SUSPEND_THRESHOLD = 90;

CREATE RESOURCE MONITOR reporting_monitor
  WITH CREDIT_QUOTA = 2000  -- 2,000 credits per day
  FREQUENCY = DAILY
  NOTIFY_USERS = ('reporting_admin@company.com')
  NOTIFY_THRESHOLD = 80
  SUSPEND_THRESHOLD = 100;

CREATE RESOURCE MONITOR adhoc_monitor
  WITH CREDIT_QUOTA = 1000  -- 1,000 credits per day
  FREQUENCY = DAILY
  NOTIFY_USERS = ('analysts@company.com')
  NOTIFY_THRESHOLD = 80
  SUSPEND_THRESHOLD = 100;

-- Assign monitors to warehouses
ALTER WAREHOUSE etl_wh SET RESOURCE_MONITOR = etl_monitor;
ALTER WAREHOUSE reporting_wh SET RESOURCE_MONITOR = reporting_monitor;
ALTER WAREHOUSE adhoc_wh SET RESOURCE_MONITOR = adhoc_monitor;
```

#### **3. User/Role-Level Resource Monitors**
```sql
-- Create resource monitors for specific users/roles
CREATE RESOURCE MONITOR user_monitor
  WITH CREDIT_QUOTA = 100  -- 100 credits per day
  FREQUENCY = DAILY
  NOTIFY_USERS = ('user@company.com', 'manager@company.com')
  NOTIFY_THRESHOLD = 50
  SUSPEND_THRESHOLD = 100;

CREATE RESOURCE MONITOR role_monitor
  WITH CREDIT_QUOTA = 500  -- 500 credits per day
  FREQUENCY = DAILY
  NOTIFY_USERS = ('role_admin@company.com')
  NOTIFY_THRESHOLD = 75
  SUSPEND_THRESHOLD = 100;

-- Assign monitors to users/roles
ALTER USER my_user SET RESOURCE_MONITOR = user_monitor;
ALTER ROLE my_role SET RESOURCE_MONITOR = role_monitor;
```

#### **4. Webhook Notifications**
```sql
-- Create a resource monitor with webhook notifications
CREATE RESOURCE MONITOR webhook_monitor
  WITH CREDIT_QUOTA = 1000
  FREQUENCY = DAILY
  NOTIFY_USERS = ('admin@company.com')
  NOTIFY_THRESHOLD = 80
  SUSPEND_THRESHOLD = 100;

-- Configure webhook in Snowsight:
-- 1. Navigate to Admin > Resource Monitors
-- 2. Select the monitor and click "Edit"
-- 3. Under "Notifications", add a webhook URL (e.g., https://hooks.slack.com/services/...)
-- 4. Save changes
```

### **B. Warehouse Setup**

#### **1. Standard Warehouses**
```sql
-- Development warehouse
CREATE WAREHOUSE dev_wh
  WAREHOUSE_SIZE = 'SMALL'
  AUTO_SUSPEND = 300  -- 5 minutes
  AUTO_RESUME = TRUE
  QUERY_PRIORITY = 'LOW'
  STATEMENT_TIMEOUT_IN_SECONDS = 300;  -- 5 minutes

-- Ad-hoc warehouse
CREATE WAREHOUSE adhoc_wh
  WAREHOUSE_SIZE = 'MEDIUM'
  AUTO_SUSPEND = 600  -- 10 minutes
  AUTO_RESUME = TRUE
  QUERY_PRIORITY = 'LOW'
  STATEMENT_TIMEOUT_IN_SECONDS = 600;  -- 10 minutes
```

#### **2. Multi-Cluster Warehouses**
```sql
-- ETL warehouse (STANDARD scaling)
CREATE WAREHOUSE etl_wh
  WAREHOUSE_SIZE = 'X-LARGE'
  MAX_CLUSTER_COUNT = 4
  MIN_CLUSTER_COUNT = 1
  SCALING_POLICY = 'STANDARD'
  AUTO_SUSPEND = 1800  -- 30 minutes
  AUTO_RESUME = TRUE
  QUERY_PRIORITY = 'HIGH'
  STATEMENT_TIMEOUT_IN_SECONDS = 3600;  -- 1 hour

-- Reporting warehouse (ECONOMY scaling)
CREATE WAREHOUSE reporting_wh
  WAREHOUSE_SIZE = 'LARGE'
  MAX_CLUSTER_COUNT = 8
  MIN_CLUSTER_COUNT = 2
  SCALING_POLICY = 'ECONOMY'
  AUTO_SUSPEND = 1800
  AUTO_RESUME = TRUE
  QUERY_PRIORITY = 'MEDIUM'
  STATEMENT_TIMEOUT_IN_SECONDS = 1800;  -- 30 minutes
```

#### **3. Warehouse Alterations**
```sql
-- Increase warehouse size
ALTER WAREHOUSE my_wh SET WAREHOUSE_SIZE = 'LARGE';

-- Increase max cluster count
ALTER WAREHOUSE my_wh SET MAX_CLUSTER_COUNT = 8;

-- Change scaling policy
ALTER WAREHOUSE my_wh SET SCALING_POLICY = 'ECONOMY';

-- Set auto-suspend
ALTER WAREHOUSE my_wh SET AUTO_SUSPEND = 1800;  -- 30 minutes

-- Set query priority
ALTER WAREHOUSE my_wh SET QUERY_PRIORITY = 'HIGH';

-- Set statement timeout
ALTER WAREHOUSE my_wh SET STATEMENT_TIMEOUT_IN_SECONDS = 300;  -- 5 minutes

-- Set queue timeout
ALTER WAREHOUSE my_wh SET STATEMENT_QUEUE_TIMEOUT_IN_SECONDS = 60;  -- 1 minute

-- Assign resource monitor
ALTER WAREHOUSE my_wh SET RESOURCE_MONITOR = my_monitor;
```

### **C. Query Prioritization Setup**

#### **1. Warehouse-Level Priority**
```sql
-- Set default priority for a warehouse
ALTER WAREHOUSE my_wh SET QUERY_PRIORITY = 'HIGH';
```

#### **2. Session-Level Priority**
```sql
-- Set priority for the current session
ALTER SESSION SET QUERY_PRIORITY = 'HIGH';
```

#### **3. Query-Level Priority (Hint)**
```sql
-- Set priority for a specific query
SELECT * FROM my_table /*+ PRIORITY(HIGH) */;
```

#### **4. User/Role-Level Priority**
```sql
-- Set priority for a role
ALTER ROLE high_priority_role SET QUERY_PRIORITY = 'HIGH';
GRANT ROLE high_priority_role TO USER my_user;
```

### **D. Workload Isolation Setup**

#### **1. Separate Warehouses for Different Workloads**
```sql
-- Production warehouses
CREATE WAREHOUSE prod_etl_wh WAREHOUSE_SIZE = 'X-LARGE' MAX_CLUSTER_COUNT = 4;
CREATE WAREHOUSE prod_reporting_wh WAREHOUSE_SIZE = 'LARGE' MAX_CLUSTER_COUNT = 8;
CREATE WAREHOUSE prod_adhoc_wh WAREHOUSE_SIZE = 'MEDIUM';

-- Development warehouses
CREATE WAREHOUSE dev_etl_wh WAREHOUSE_SIZE = 'MEDIUM';
CREATE WAREHOUSE dev_reporting_wh WAREHOUSE_SIZE = 'SMALL';

-- Assign resource monitors
ALTER WAREHOUSE prod_etl_wh SET RESOURCE_MONITOR = prod_etl_monitor;
ALTER WAREHOUSE prod_reporting_wh SET RESOURCE_MONITOR = prod_reporting_monitor;
ALTER WAREHOUSE dev_etl_wh SET RESOURCE_MONITOR = dev_etl_monitor;
```

#### **2. Query Tagging for Cost Allocation**
```sql
-- Set query tag for the session
ALTER SESSION SET QUERY_TAG = 'workload=etl,team=data_engineering';

-- Set query tag for a specific query
SELECT * FROM my_table /*+ QUERY_TAG('workload=reporting,team=analytics') */;
```

### **E. Monitoring and Alerting Setup**

#### **1. Workload Monitoring Dashboard**
```sql
CREATE VIEW WORKLOAD_MONITORING_DASHBOARD AS
SELECT
    w.name AS warehouse_name,
    w.size AS warehouse_size,
    w.max_cluster_count,
    w.scaling_policy,
    w.auto_suspend,
    w.auto_resume,
    rm.name AS resource_monitor_name,
    rm.credit_quota,
    rm.used_credits,
    rm.remaining_credits,
    rm.used_credits * 100.0 / NULLIF(rm.credit_quota, 0) AS percent_used,
    wm.running_queries,
    wm.queued_queries,
    wm.total_queries,
    wm.cluster_number,
    wm.total_clusters,
    wh.credits_used AS recent_credits_used,
    qh.avg_execution_time,
    qh.avg_queue_time,
    qh.avg_bytes_scanned,
    qh.query_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSES w
LEFT JOIN
    SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS rm
    ON w.resource_monitor = rm.name
LEFT JOIN
    SNOWFLAKE.INFORMATION_SCHEMA.WAREHOUSE_MONITOR wm
    ON w.name = wm.warehouse_name
LEFT JOIN (
    SELECT
        warehouse_name,
        SUM(credits_used) AS credits_used,
        MIN(start_time) AS start_time,
        MAX(end_time) AS end_time
    FROM
        SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
    WHERE
        start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
    GROUP BY
        warehouse_name
) wh ON w.name = wh.warehouse_name
LEFT JOIN (
    SELECT
        warehouse_name,
        AVG(execution_time) AS avg_execution_time,
        AVG(queue_time) AS avg_queue_time,
        AVG(bytes_scanned) AS avg_bytes_scanned,
        COUNT(*) AS query_count
    FROM
        SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE
        start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
    GROUP BY
        warehouse_name
) qh ON w.name = qh.warehouse_name;
```

#### **2. Proactive Alerts**
```sql
-- Warehouse overloaded alert
CREATE OR REPLACE ALERT WAREHOUSE_OVERLOADED_ALERT
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'
AS
  SELECT
    warehouse_name,
    running_queries,
    queued_queries,
    cluster_number,
    total_clusters,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.INFORMATION_SCHEMA.WAREHOUSE_MONITOR
  WHERE
    queued_queries > 0
    OR running_queries = total_clusters * 10
  AND
    warehouse_name IN ('prod_etl_wh', 'prod_reporting_wh');

-- Resource monitor warning alert
CREATE OR REPLACE ALERT RESOURCE_MONITOR_WARNING_ALERT
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'
AS
  SELECT
    monitor_name,
    credit_quota,
    used_credits,
    remaining_credits,
    used_credits * 100.0 / credit_quota AS percent_used,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS
  WHERE
    used_credits * 100.0 / credit_quota > 80
    AND monitor_name IN ('prod_etl_monitor', 'prod_reporting_monitor');

-- Long-running query alert
CREATE OR REPLACE ALERT LONG_RUNNING_QUERY_ALERT
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'
AS
  SELECT
    query_id,
    query_text,
    warehouse_name,
    execution_time,
    start_time,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE
    execution_time > 300
    AND execution_status = 'RUNNING'
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
    AND warehouse_name IN ('prod_etl_wh', 'prod_reporting_wh');
```

## **11. Final Notes**

### **For Further Reading**
- [Snowflake Workload Management Documentation](https://docs.snowflake.com/en/user-guide/workload-management)
- [Resource Monitors](https://docs.snowflake.com/en/user-guide/resource-monitors)
- [Warehouses](https://docs.snowflake.com/en/user-guide/warehouses)
- [Multi-Cluster Warehouses](https://docs.snowflake.com/en/user-guide/multi-cluster-warehouses)
- [Query Prioritization](https://docs.snowflake.com/en/user-guide/query-prioritization)
- [Query Queues](https://docs.snowflake.com/en/user-guide/query-queues)
- [Monitoring Warehouses](https://docs.snowflake.com/en/user-guide/monitoring-warehouses)
- [Monitoring Resource Monitors](https://docs.snowflake.com/en/user-guide/monitoring-resource-monitors)

### **Open Questions for Your Environment**
1. **Workload Types**:
   - What **types of workloads** do you have (ETL, reporting, ad-hoc, development)?
   - What are the **performance SLAs** for each workload (e.g., <1s, <10s, <1min)?

2. **Concurrency Requirements**:
   - What is the **maximum concurrency** for each workload (e.g., 10 queries, 100 queries)?
   - Do you have **peak usage periods** (e.g., morning batch jobs, end-of-day reporting)?

3. **Cost Constraints**:
   - What is your **monthly budget** for Snowflake credits?
   - Do you need to **allocate costs** to specific teams/departments?

4. **Isolation Requirements**:
   - Do you need **full isolation** between workloads (e.g., production vs. development)?
   - Do you need **partial isolation** (e.g., ETL vs. reporting in the same warehouse)?

5. **Warehouse Configuration**:
   - What **warehouse sizes** are you currently using?
   - Are you using **multi-cluster warehouses**? If so, what is your `MAX_CLUSTER_COUNT` and `SCALING_POLICY`?
   - Are you using **auto-suspend** and **auto-resume**? What are your settings?

6. **Resource Monitors**:
   - Are you using **resource monitors**? If so, what are your **credit quotas** and **thresholds**?
   - Are you using **webhook notifications** for alerts?

7. **Query Prioritization**:
   - Are you using **query prioritization**? If so, what are your **priority levels** for different workloads?
   - Are you using **query tagging** for monitoring and cost allocation?

8. **Monitoring and Alerting**:
   - Are you **monitoring warehouse load**, **query performance**, and **credit usage**?
   - Are you **alerting** on workload issues (e.g., overloaded warehouses, long-running queries)?

9. **Performance Issues**:
   - Are you experiencing **performance issues** (e.g., slow queries, queued queries)?
   - Are you experiencing **cost overruns** (e.g., exceeding credit budgets)?

10. **Future Requirements**:
    - Do you expect **growth** in workload volume or complexity?
    - Do you plan to **migrate** any workloads to Snowflake in the future?
