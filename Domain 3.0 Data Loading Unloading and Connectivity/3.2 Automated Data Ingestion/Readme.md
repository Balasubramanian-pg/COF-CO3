# Snowflake Automated Data Ingestion: Production-Grade Technical Deep Dive

## Architecture & Execution Flow

### Mermaid: Automated Ingestion Paths
```mermaid
flowchart TD
    subgraph DataSources["Data Sources"]
        A[("Cloud Storage\n(S3/Azure/GCS)")] -->|Event Notification| B[("Snowpipe")]
        C[("Kafka\n(Confluent/Self-Managed)")] -->|Consumer Group| D[("Kafka Connector")]
        E[("REST APIs\n(HTTP/HTTPS)")] -->|Webhook| F[("Snowflake Ingestion Service")]
        G[("External Databases\n(Postgres/MySQL)")] -->|CDC| H[("Snowflake Connector")]
        I[("Data Lakes\n(Delta/Iceberg)")] -->|External Table| J[("Snowflake External Tables")]
    end

    subgraph SnowflakeControlPlane["Snowflake Control Plane"]
        B --> K[("Pipe Metadata")]
        D --> L[("Kafka Offset Manager")]
        F --> M[("Ingestion Service Metadata")]
        H --> N[("Replication Service")]
        J --> O[("External Table Metadata")]
    end

    subgraph IngestionLayer["Ingestion Layer"]
        K --> P[("COPY INTO\n(Micro-Batches)")]
        L --> Q[("Kafka Consumer\n(Parallel)")]
        M --> R[("Streaming Ingest\n(Real-Time)")]
        N --> S[("Replication Task")]
        O --> T[("Query External Data")]
    end

    subgraph StorageLayer["Storage Layer"]
        P --> U[("Target Table")]
        Q --> U
        R --> U
        S --> U
        T --> V[("External Stage")]
    end

    subgraph Monitoring["Monitoring"]
        W[("ACCOUNT_USAGE.PIPE_USAGE_HISTORY")]
        X[("ACCOUNT_USAGE.KAFKA_CONSUMER_GROUPS")]
        Y[("ACCOUNT_USAGE.INGESTION_HISTORY")]
        Z[("INFORMATION_SCHEMA.TASK_HISTORY")]
    end
    P --> W
    Q --> X
    R --> Y
    S --> Z

    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef cloud fill:#e3f2fd,stroke:#90caf9;
    classDef kafka fill:#fff3e0,stroke:#ef6c00;
    classDef rest fill:#f3e5f5,stroke:#7b1fa2;
    classDef db fill:#e8f5e9,stroke:#2e7d32;
    classDef lake fill:#fff8e1,stroke:#f57f17;
    class A cloud;
    class C kafka;
    class E rest;
    class G db;
    class I lake;
```

### Automated Ingestion Methods Comparison

| Ingestion Method | Trigger Mechanism | Latency | Throughput | Data Freshness | Transactional Guarantees | Cost Model | Best For |
|-----------------|-------------------|---------|------------|----------------|--------------------------|------------|----------|
| Snowpipe | Cloud storage event | 1-10 min | 100-1000 MB/min | Near real-time | Atomic per file | Compute + Storage | Batch loading from cloud storage |
| Kafka Connector | Kafka consumer | <1 sec | 50-500 MB/sec | Real-time | At-least-once | Compute + Kafka | Streaming from Kafka topics |
| Ingestion Service | REST API | <1 sec | 1-10 MB/sec | Real-time | At-least-once | Compute + API calls | REST API sources |
| External Tables | Query-time | N/A | N/A | Real-time | N/A | Compute only | Query external data lakes |
| Snowflake Connector | CDC polling | 1-5 min | 10-100 MB/min | Near real-time | Atomic per batch | Compute | Database replication |
| Tasks + Stored Procedures | Schedule | 1-60 min | 10-1000 MB/min | Scheduled | Atomic per run | Compute | Scheduled batch ingestion |


## Execution Internals & Transactional Boundaries

### Snowpipe Internals
#### Micro-Batching Architecture
Snowpipe processes files in micro-batches with the following execution flow:

1. **Event Notification**:
   - Cloud storage (S3/Azure/GCS) sends event to Snowflake via:
     - **S3**: SQS/SNS notifications
     - **Azure**: Event Grid notifications
     - **GCS**: Pub/Sub notifications
   - **Notification Latency**: 1-5 minutes (configurable via `NOTIFY_CHANNEL`)

2. **Event Buffering**:
   - Events buffered in Snowflake's **ingestion queue** (max 10,000 events)
   - **Buffer Retention**: 14 days (configurable via `ERROR_INTEGRATION`)

3. **Pipe Execution**:
   - Snowflake **polls the queue** every 1-5 minutes (configurable via `PIPE_EXECUTION_INTERVAL`)
   - **Worker Allocation**:
     - 1 worker per pipe (scales with warehouse size)
     - **Max Concurrent Pipes**: 100 per warehouse (Enterprise Edition: 200)
   - **COPY INTO Execution**:
     - Uses **serverless compute** (no warehouse required)
     - **Batch Size**: 1 file per COPY (default), configurable to 10 files via `COPY_OPTIONS`

4. **Atomicity**:
   - **File-Level**: Each file is loaded atomically (all rows succeed or fail together)
   - **Transaction Boundary**: Per-file (no multi-file transactions)
   - **Rollback**: On failure, file is moved to **failed state** and retried (max 3 retries)

5. **Error Handling**:
   - **Retry Logic**: Exponential backoff (1s, 2s, 4s)
   - **DLQ**: Files with persistent errors moved to `@pipe_name_DLQ` stage
   - **Error Classification**:
     | Error Type | Retryable | DLQ | Notification |
     |------------|-----------|-----|--------------|
     | `STAGE_FILE_NOT_FOUND` | No | Yes | Email |
     | `STAGE_CONNECTION_ERROR` | Yes | No | None |
     | `FILE_FORMAT_MISMATCH` | No | Yes | Email |
     | `WAREHOUSE_SIZE_TOO_SMALL` | No | No | Email |

#### Thread Allocation Model
- **Serverless Workers**:
  - **1 worker per pipe** (scales with number of pipes)
  - **Memory**: 256MB per worker (fixed)
  - **CPU**: 1 vCPU per worker (fixed)
- **Warehouse Workers** (if using warehouse):
  - **X-Small**: 1 worker
  - **Small**: 2 workers
  - **Medium**: 4 workers
  - **Large**: 8 workers
  - **X-Large**: 16 workers

#### Credit Calculation
- **Serverless**:
  - **Compute**: 0.0000002 credits per MB processed
  - **Storage**: 0.1 credits per GB/month (for staged files)
- **Warehouse**:
  - **Compute**: Standard warehouse rates (0.00028 credits per second for X-Small)
  - **Storage**: Same as serverless

#### Performance Characteristics
| Warehouse Size | Max Throughput | Latency (File to Table) | Concurrent Files | Credit Cost (Per GB) |
|----------------|----------------|-------------------------|------------------|----------------------|
| Serverless | 1000 MB/min | 1-10 min | 100 | 0.0002 |
| X-Small | 200 MB/min | 5-15 min | 10 | 0.0028 |
| Small | 400 MB/min | 5-15 min | 20 | 0.0014 |
| Medium | 800 MB/min | 5-15 min | 40 | 0.0007 |
| Large | 1600 MB/min | 5-15 min | 80 | 0.00035 |
| X-Large | 3200 MB/min | 5-15 min | 160 | 0.000175 |


### Kafka Connector Internals
#### Consumer Group Architecture
1. **Connector Setup**:
   - **Consumer Group**: Dedicated group per Snowflake account (format: `snowflake-{account_id}`)
   - **Topic Subscription**: Subscribes to specified Kafka topics
   - **Offset Management**: Stores offsets in Snowflake metadata (not Kafka)

2. **Message Consumption**:
   - **Polling Interval**: 100ms (configurable via `KAFKA_POLL_INTERVAL_MS`)
   - **Batch Size**: 10,000 messages per batch (configurable via `KAFKA_BATCH_SIZE`)
   - **Parallelism**:
     - **Partitions**: 1 consumer thread per Kafka partition
     - **Max Threads**: 100 (Enterprise Edition: 200)

3. **Message Processing**:
   - **Deserialization**:
     - **Avro**: Uses Confluent Schema Registry
     - **JSON**: Native parsing
     - **Protobuf**: Custom deserializer
   - **Transformation**:
     - **JSON Path**: Extract fields via `TRANSFORMATION` parameter
     - **Flattening**: Nested structures flattened to columns
   - **Validation**:
     - Schema validation (if `ENABLE_SCHEMA_VALIDATION = TRUE`)
     - Size validation (max 16MB per message)

4. **Snowflake Ingestion**:
   - **COPY INTO**: Uses internal COPY command
   - **Buffering**:
     - In-memory buffer: 100MB per thread
     - Spill threshold: 200MB (to SSD)
   - **Atomicity**:
     - **Message-Level**: Each Kafka message loaded atomically
     - **Batch-Level**: Entire batch succeeds or fails together

5. **Offset Commit**:
   - **At-Least-Once**: Offsets committed after successful COPY
   - **Commit Frequency**: Every 10,000 messages or 10 seconds (whichever comes first)
   - **Manual Override**: `ALTER KAFKA CONNECTOR ... SET START_OFFSET = 'earliest'`

6. **Error Handling**:
   - **Retry Logic**: Infinite retries with exponential backoff (1s, 2s, 4s, max 60s)
   - **DLQ**: Messages with persistent errors routed to Kafka DLQ topic
   - **Error Classification**:
     | Error Type | Retryable | DLQ | Notification |
     |------------|-----------|-----|--------------|
     | `KAFKA_CONNECTION_ERROR` | Yes | No | None |
     | `DESERIALIZATION_ERROR` | No | Yes | Email |
     | `SCHEMA_VALIDATION_ERROR` | No | Yes | Email |
     | `SIZE_LIMIT_EXCEEDED` | No | Yes | Email |

#### Thread Allocation Model
- **Consumer Threads**: 1 per Kafka partition (max 100)
- **Processing Threads**: 1 per consumer thread
- **Memory per Thread**: 256MB
- **Spill Behavior**: To SSD at 200MB threshold

#### Credit Calculation
- **Compute**: 0.0000005 credits per message
- **Storage**: 0.1 credits per GB/month (for staged messages)
- **Kafka Costs**: Separate (customer-managed)

#### Performance Characteristics
| Kafka Partitions | Max Throughput | Latency | Consumer Threads | Credit Cost (Per MB) |
|------------------|----------------|---------|------------------|----------------------|
| 1 | 50 MB/sec | <1 sec | 1 | 0.0005 |
| 10 | 500 MB/sec | <1 sec | 10 | 0.0005 |
| 50 | 2.5 GB/sec | <1 sec | 50 | 0.0005 |
| 100 | 5 GB/sec | <1 sec | 100 | 0.0005 |


### Snowflake Ingestion Service Internals
#### Streaming Architecture
1. **REST API Endpoint**:
   - **Ingest Endpoint**: `https://{account}.snowflakecomputing.com/api/v2/ingest`
   - **Authentication**: JWT tokens (24-hour expiry)
   - **Rate Limits**: 10,000 requests/sec per account

2. **Request Processing**:
   - **Batch Size**: 100-10,000 rows per request (configurable via `BATCH_SIZE`)
   - **Payload Size**: Max 16MB per request (hard limit)
   - **Compression**: Gzip supported (reduces payload size by ~70%)

3. **Stream Processing**:
   - **Buffering**:
     - In-memory: 100MB per stream
     - Spill: To SSD at 200MB threshold
   - **Micro-Batching**:
     - Flush interval: 1 second (configurable via `FLUSH_INTERVAL`)
     - Max batch size: 10,000 rows

4. **Table Writing**:
   - **COPY INTO**: Uses internal COPY command
   - **Atomicity**:
     - **Request-Level**: All rows in a request succeed or fail together
     - **Row-Level**: Individual rows within a request may fail
   - **Ordering**: Not guaranteed (use `SEQUENCE` column for ordering)

5. **Error Handling**:
   - **Retry Logic**: Automatic retry with exponential backoff (max 3 attempts)
   - **DLQ**: Failed rows routed to `@ingest_DLQ` stage
   - **Error Classification**:
     | Error Type | Retryable | DLQ | HTTP Status |
     |------------|-----------|-----|--------------|
     | `PAYLOAD_TOO_LARGE` | No | No | 413 |
     | `INVALID_TOKEN` | No | No | 401 |
     | `RATE_LIMIT_EXCEEDED` | Yes | No | 429 |
     | `SCHEMA_MISMATCH` | No | Yes | 400 |

#### Thread Allocation Model
- **Stream Workers**: 1 per ingest endpoint
- **Processing Threads**: 1 per 100 requests
- **Memory per Worker**: 512MB

#### Credit Calculation
- **Compute**: 0.000001 credits per row
- **Storage**: 0.1 credits per GB/month (for buffered data)

#### Performance Characteristics
| Batch Size | Throughput | Latency | Credit Cost (Per 1M Rows) |
|------------|------------|---------|---------------------------|
| 100 | 1000 rows/sec | <1 sec | 1 |
| 1000 | 10,000 rows/sec | <1 sec | 1 |
| 5000 | 50,000 rows/sec | <1 sec | 1 |
| 10000 | 100,000 rows/sec | <1 sec | 1 |


### External Tables Internals
#### Query-Time Processing
1. **Metadata Resolution**:
   - **External Stage**: Resolves to cloud storage location
   - **File Format**: Resolves parsing rules
   - **Partitioning**: Resolves partition columns (if any)

2. **Query Planning**:
   - **Predicate Pushdown**: Filters pushed to cloud storage
   - **Column Pruning**: Only reads required columns
   - **Partition Pruning**: Only scans relevant partitions

3. **Execution**:
   - **Parallel Reads**: 1 thread per file (max 100 concurrent)
   - **Buffering**:
     - In-memory: 100MB per thread
     - Spill: To SSD at 200MB threshold
   - **Caching**: Frequently accessed files cached in local SSD (TTL: 1 hour)

4. **Atomicity**:
   - **Query-Level**: Entire query succeeds or fails
   - **Row-Level**: No atomicity guarantees (streaming reads)

5. **Error Handling**:
   - **File-Level Errors**: Skipped files logged in `QUERY_HISTORY`
   - **Row-Level Errors**: Returned as NULL or error (depending on `ON_ERROR`)

#### Thread Allocation Model
- **Query Threads**: 1 per file (max 100)
- **Memory per Thread**: 100MB
- **Spill Behavior**: To SSD at 200MB threshold

#### Credit Calculation
- **Compute**: Standard query pricing (based on warehouse size and duration)
- **Storage**: 0 (external storage costs not billed by Snowflake)

#### Performance Characteristics
| Warehouse Size | Max Concurrent Files | Throughput | Latency | Credit Cost (Per GB) |
|----------------|-----------------------|------------|---------|----------------------|
| X-Small | 10 | 200 MB/min | 100-500ms | 0.0028 |
| Small | 20 | 400 MB/min | 100-500ms | 0.0014 |
| Medium | 40 | 800 MB/min | 100-500ms | 0.0007 |
| Large | 80 | 1600 MB/min | 100-500ms | 0.00035 |
| X-Large | 100 | 2000 MB/min | 100-500ms | 0.000175 |


### Snowflake Connector (CDC) Internals
#### Change Data Capture Architecture
1. **Source Connection**:
   - **Database Types**: PostgreSQL, MySQL, SQL Server, Oracle
   - **Connection Method**: JDBC (encrypted)
   - **Polling Interval**: 1-5 minutes (configurable via `POLLING_INTERVAL`)

2. **Change Detection**:
   - **Log-Based**: Reads from:
     - PostgreSQL: WAL (Write-Ahead Log)
     - MySQL: Binary Log
     - SQL Server: CDC tables
     - Oracle: Redo Logs
   - **Timestamp-Based**: Polls `LAST_MODIFIED` columns (fallback)

3. **Change Buffering**:
   - **In-Memory**: 100MB per table
   - **Spill**: To SSD at 200MB threshold
   - **Batch Size**: 10,000 rows per batch (configurable via `BATCH_SIZE`)

4. **Replication**:
   - **COPY INTO**: Uses internal COPY command
   - **Atomicity**:
     - **Batch-Level**: Entire batch succeeds or fails together
   - **Ordering**: Preserved via `CDC_TIMESTAMP` column

5. **Error Handling**:
   - **Retry Logic**: Exponential backoff (1s, 2s, 4s, max 60s)
   - **DLQ**: Failed batches routed to `@cdc_DLQ` stage
   - **Error Classification**:
     | Error Type | Retryable | DLQ | Notification |
     |------------|-----------|-----|--------------|
     | `CONNECTION_ERROR` | Yes | No | Email |
     | `SCHEMA_CHANGE` | No | Yes | Email |
     | `PERMISSION_ERROR` | No | No | Email |

#### Thread Allocation Model
- **Polling Threads**: 1 per source database
- **Replication Threads**: 1 per table
- **Memory per Thread**: 256MB

#### Credit Calculation
- **Compute**: 0.0000003 credits per row
- **Storage**: 0.1 credits per GB/month (for buffered changes)

#### Performance Characteristics
| Source DB | Polling Interval | Max Throughput | Latency | Credit Cost (Per 1M Rows) |
|-----------|-------------------|----------------|---------|---------------------------|
| PostgreSQL | 1 min | 500 MB/min | 1-5 min | 0.3 |
| MySQL | 1 min | 400 MB/min | 1-5 min | 0.3 |
| SQL Server | 5 min | 300 MB/min | 5-10 min | 0.3 |
| Oracle | 5 min | 200 MB/min | 5-10 min | 0.3 |


### Tasks + Stored Procedures Internals
#### Scheduled Execution
1. **Task Definition**:
   - **Schedule**: CRON or interval (1 minute minimum)
   - **Warehouse**: Dedicated or shared
   - **Timeout**: Max 8 days (configurable via `SESSION_TIMEOUT`)

2. **Execution Flow**:
   - **Trigger**: Based on schedule
   - **Warehouse Allocation**: Reserved for task duration
   - **Stored Procedure Execution**:
     - **Language**: SQL, JavaScript, Java, Python, Scala
     - **Isolation**: Runs in separate session

3. **Error Handling**:
   - **Retry Logic**: Configurable via `RETRY_COUNT` (default: 0)
   - **Backoff**: Exponential (1s, 2s, 4s)
   - **DLQ**: Errors logged in `TASK_HISTORY`
   - **Notification**: Email on failure (configurable)

4. **Atomicity**:
   - **Task-Level**: Entire task succeeds or fails
   - **Transaction-Level**: Depends on stored procedure implementation

5. **Resource Management**:
   - **Concurrency**: Max 100 concurrent tasks per account
   - **Warehouse Scaling**: Uses assigned warehouse size

#### Thread Allocation Model
- **Task Threads**: 1 per task
- **Warehouse Threads**: Based on warehouse size
- **Memory**: Based on warehouse size

#### Credit Calculation
- **Compute**: Standard warehouse rates (based on runtime)
- **Storage**: Standard storage rates (if data is staged)

#### Performance Characteristics
| Warehouse Size | Max Concurrent Tasks | Throughput | Latency | Credit Cost (Per Hour) |
|----------------|-----------------------|------------|---------|------------------------|
| X-Small | 1 | 200 MB/min | 1-60 min | 0.28 |
| Small | 2 | 400 MB/min | 1-60 min | 0.56 |
| Medium | 4 | 800 MB/min | 1-60 min | 1.12 |
| Large | 8 | 1600 MB/min | 1-60 min | 2.24 |
| X-Large | 16 | 3200 MB/min | 1-60 min | 4.48 |


## Parameter/Configuration Deep Dive

### Snowpipe Parameters
| Parameter | Internal Behavior | Performance Impact | Compliance/Edge Cases | Production Default |
|-----------|-------------------|--------------------|-----------------------|---------------------|
| `AUTO_INGEST` | Enables automatic file detection | Reduces latency to 1-5 min | Requires cloud notifications | FALSE |
| `NOTIFY_CHANNEL` | Cloud notification channel (SQS/SNS/Event Grid/Pub/Sub) | No impact on throughput | Must be configured for AUTO_INGEST | None |
| `PIPE_EXECUTION_INTERVAL` | Polling interval for non-auto-ingest pipes (minutes) | Lower = higher throughput, higher cost | Min: 1, Max: 1440 | 5 |
| `ERROR_INTEGRATION` | Error notification channel | No impact on performance | Must be configured for error alerts | None |
| `COPY_OPTIONS` | COPY INTO options (ON_ERROR, VALIDATION_MODE, etc.) | Affects error handling | Must match target table schema | ON_ERROR = 'CONTINUE' |
| `FILE_FORMAT` | File format name | Affects parsing speed | Must exist in database | None (required) |
| `STAGE_NAME` | Stage name | Affects storage location | Must exist in database | None (required) |
| `TARGET_TABLE` | Target table name | Affects atomicity | Must exist in database | None (required) |
| `ENABLE_DUPLICATE_DETECTION` | Detects duplicate files | Adds 5% overhead | Requires file checksums | FALSE |
| `DUPLICATE_HANDLING` | How to handle duplicates (SKIP/FAIL) | SKIP = faster, FAIL = safer | SKIP may miss data | SKIP |

### Kafka Connector Parameters
| Parameter | Internal Behavior | Performance Impact | Compliance/Edge Cases | Production Default |
|-----------|-------------------|--------------------|-----------------------|---------------------|
| `KAFKA_BROKER` | Kafka broker URL | No impact | Must be reachable from Snowflake | None (required) |
| `KAFKA_TOPIC` | Kafka topic name | No impact | Must exist in Kafka | None (required) |
| `KAFKA_PARTITIONS` | Number of partitions to consume | More partitions = higher throughput | Max: 100 | None (all) |
| `KAFKA_CONSUMER_GROUP` | Consumer group ID | No impact | Must be unique per connector | snowflake-{account_id} |
| `KAFKA_POLL_INTERVAL_MS` | Polling interval (ms) | Lower = lower latency, higher CPU | Min: 10, Max: 10000 | 100 |
| `KAFKA_BATCH_SIZE` | Messages per batch | Larger = higher throughput, higher latency | Min: 1, Max: 10000 | 10000 |
| `KAFKA_START_OFFSET` | Starting offset (earliest/latest) | No impact | earliest = full replay | latest |
| `TRANSFORMATION` | JSON path to extract data | Affects parsing speed | Must match message structure | None |
| `ENABLE_SCHEMA_VALIDATION` | Validates against schema | Adds 10% overhead | Requires schema definition | FALSE |
| `SCHEMA_REGISTRY_URL` | Confluent Schema Registry URL | No impact | Required for Avro | None |
| `KAFKA_SASL_MECHANISM` | SASL mechanism (PLAIN/SCRAM/SASL_SSL) | No impact | Must match Kafka config | PLAIN |
| `KAFKA_SECURITY_PROTOCOL` | Security protocol (PLAINTEXT/SSL/SASL_PLAINTEXT/SASL_SSL) | SSL = 5% overhead | PLAINTEXT not recommended | SSL |

### Ingestion Service Parameters
| Parameter | Internal Behavior | Performance Impact | Compliance/Edge Cases | Production Default |
|-----------|-------------------|--------------------|-----------------------|---------------------|
| `BATCH_SIZE` | Rows per request | Larger = higher throughput, higher latency | Min: 100, Max: 10000 | 1000 |
| `FLUSH_INTERVAL` | Flush interval (seconds) | Lower = lower latency, higher cost | Min: 1, Max: 60 | 1 |
| `COMPRESSION` | Request compression (NONE/GZIP) | GZIP = 30% smaller payload | GZIP adds 5% CPU overhead | GZIP |
| `ON_ERROR` | Error handling (ABORT/CONTINUE) | CONTINUE = higher resilience | ABORT = safer | CONTINUE |
| `TRANSFORMATION` | JSON path to extract data | Affects parsing speed | Must match payload structure | None |
| `SEQUENCE_COLUMN` | Column to preserve ordering | Adds 5% overhead | Must be INTEGER or TIMESTAMP | None |

### External Table Parameters
| Parameter | Internal Behavior | Performance Impact | Compliance/Edge Cases | Production Default |
|-----------|-------------------|--------------------|-----------------------|---------------------|
| `LOCATION` | External stage path | Affects storage location | Must exist in database | None (required) |
| `FILE_FORMAT` | File format name | Affects parsing speed | Must exist in database | None (required) |
| `PATTERN` | File pattern (regex) | Affects file filtering | Must match stage files | .* |
| `AUTO_REFRESH` | Auto-refresh metadata | Adds 1% overhead | Requires cloud notifications | FALSE |
| `REFRESH_MODE` | Refresh mode (AUTO/MANUAL) | AUTO = lower latency | MANUAL = more control | AUTO |
| `PARTITION_BY` | Partition columns | Enables partition pruning | Must match file structure | None |
| `INFER_SCHEMA` | Infer schema from files | Adds 10% overhead | May fail for complex schemas | TRUE |

### Snowflake Connector (CDC) Parameters
| Parameter | Internal Behavior | Performance Impact | Compliance/Edge Cases | Production Default |
|-----------|-------------------|--------------------|-----------------------|---------------------|
| `SOURCE_DATABASE` | Source database connection | No impact | Must be reachable from Snowflake | None (required) |
| `SOURCE_SCHEMA` | Source schema name | No impact | Must exist in source | None (required) |
| `SOURCE_TABLE` | Source table name | No impact | Must exist in source | None (required) |
| `POLLING_INTERVAL` | Polling interval (minutes) | Lower = lower latency, higher cost | Min: 1, Max: 1440 | 5 |
| `BATCH_SIZE` | Rows per batch | Larger = higher throughput, higher latency | Min: 100, Max: 10000 | 1000 |
| `CDC_MODE` | CDC mode (LOG_BASED/TIMESTAMP_BASED) | LOG_BASED = lower latency | TIMESTAMP_BASED = fallback | LOG_BASED |
| `INCLUDE_COLUMNS` | Columns to replicate | Affects storage size | Must exist in source | All |
| `EXCLUDE_COLUMNS` | Columns to exclude | Affects storage size | Must exist in source | None |
| `TRANSFORMATION` | Column transformations | Affects parsing speed | Must match target schema | None |

### Task Parameters
| Parameter | Internal Behavior | Performance Impact | Compliance/Edge Cases | Production Default |
|-----------|-------------------|--------------------|-----------------------|---------------------|
| `WAREHOUSE` | Warehouse for task execution | Affects throughput and cost | Must exist in account | None (required) |
| `SCHEDULE` | Task schedule (CRON/interval) | Affects frequency | Min interval: 1 minute | None (required) |
| `WHEN` | Conditional execution | Affects execution count | Must be valid SQL | TRUE |
| `RETRY_COUNT` | Number of retries | Higher = higher resilience | Max: 10 | 0 |
| `RETRY_DELAY` | Delay between retries (seconds) | Higher = lower resource usage | Min: 1, Max: 3600 | 1 |
| `ALLOW_OVERLAPPING_EXECUTION` | Allow overlapping runs | TRUE = higher throughput | May cause resource contention | FALSE |
| `USER_TASK_TIMEOUT_MS` | Task timeout (ms) | Lower = faster failure | Min: 1000, Max: 691200000 | 86400000 (24 hours) |
| `SESSION_TIMEOUT` | Session timeout (minutes) | Lower = faster failure | Min: 1, Max: 11520 | 60 |


## Performance & Resource Implications

### Memory Usage by Ingestion Method
| Ingestion Method | Per-Thread Memory | Spill Threshold | Max Threads | Memory Notes |
|------------------|-------------------|-----------------|-------------|---------------|
| Snowpipe | 256MB | 512MB | 100 | Serverless workers |
| Kafka Connector | 256MB | 512MB | 100 | 1 thread per partition |
| Ingestion Service | 512MB | 1GB | 100 | 1 worker per endpoint |
| External Tables | 100MB | 200MB | 100 | 1 thread per file |
| Snowflake Connector | 256MB | 512MB | 10 | 1 thread per table |
| Tasks | Warehouse-dependent | Warehouse-dependent | 100 | Uses warehouse memory |

### Throughput by Ingestion Method
| Ingestion Method | X-Small | Small | Medium | Large | X-Large | Bottleneck |
|------------------|---------|-------|--------|-------|---------|------------|
| Snowpipe | 200 MB/min | 400 MB/min | 800 MB/min | 1.6 GB/min | 3.2 GB/min | Cloud storage event rate |
| Kafka Connector | 50 MB/sec | 100 MB/sec | 200 MB/sec | 400 MB/sec | 500 MB/sec | Kafka partition count |
| Ingestion Service | 1 MB/sec | 2 MB/sec | 4 MB/sec | 8 MB/sec | 10 MB/sec | API rate limits |
| External Tables | 200 MB/min | 400 MB/min | 800 MB/min | 1.6 GB/min | 2 GB/min | Warehouse size |
| Snowflake Connector | 50 MB/min | 100 MB/min | 200 MB/min | 400 MB/min | 500 MB/min | Source DB polling rate |
| Tasks | 200 MB/min | 400 MB/min | 800 MB/min | 1.6 GB/min | 3.2 GB/min | Warehouse size |

### Latency by Ingestion Method
| Ingestion Method | Min Latency | Max Latency | Latency Notes |
|------------------|-------------|-------------|---------------|
| Snowpipe | 1 min | 10 min | Depends on cloud notifications |
| Kafka Connector | <1 sec | 5 sec | Depends on Kafka lag |
| Ingestion Service | <1 sec | 2 sec | Depends on network latency |
| External Tables | 100 ms | 500 ms | Query-time only |
| Snowflake Connector | 1 min | 10 min | Depends on polling interval |
| Tasks | 1 min | 60 min | Depends on schedule |

### Credit Costs by Ingestion Method
| Ingestion Method | Compute Cost | Storage Cost | Notes |
|------------------|--------------|--------------|-------|
| Snowpipe | 0.0000002 credits/MB | 0.1 credits/GB/month | Serverless compute |
| Kafka Connector | 0.0000005 credits/message | 0.1 credits/GB/month | + Kafka costs |
| Ingestion Service | 0.000001 credits/row | 0 | + API costs |
| External Tables | Warehouse-dependent | 0 | No storage costs |
| Snowflake Connector | 0.0000003 credits/row | 0.1 credits/GB/month | + Source DB costs |
| Tasks | Warehouse-dependent | Warehouse-dependent | Standard rates |

### Concurrency Limits
| Ingestion Method | Max Concurrent Operations | Scaling Notes |
|------------------|---------------------------|---------------|
| Snowpipe | 100 pipes | 1 worker per pipe |
| Kafka Connector | 100 consumers | 1 thread per partition |
| Ingestion Service | 100 streams | 1 worker per endpoint |
| External Tables | 100 files | 1 thread per file |
| Snowflake Connector | 10 connectors | 1 thread per table |
| Tasks | 100 tasks | Uses warehouse concurrency |


## Monitoring, Observability & Troubleshooting

### Key Monitoring Views
| View | Purpose | Example Query | Retention |
|------|---------|---------------|-----------|
| `ACCOUNT_USAGE.PIPE_USAGE_HISTORY` | Snowpipe execution history | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.PIPE_USAGE_HISTORY WHERE PIPE_NAME = 'MY_PIPE' AND START_TIME > DATEADD('day', -7, CURRENT_TIMESTAMP());` | 365 days |
| `ACCOUNT_USAGE.KAFKA_CONSUMER_GROUPS` | Kafka connector status | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.KAFKA_CONSUMER_GROUPS;` | 365 days |
| `ACCOUNT_USAGE.KAFKA_TOPICS` | Kafka topic metadata | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.KAFKA_TOPICS WHERE TOPIC_NAME = 'MY_TOPIC';` | 365 days |
| `ACCOUNT_USAGE.INGESTION_HISTORY` | Ingestion Service history | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.INGESTION_HISTORY WHERE STREAM_NAME = 'MY_STREAM' AND START_TIME > DATEADD('hour', -24, CURRENT_TIMESTAMP());` | 365 days |
| `INFORMATION_SCHEMA.TASK_HISTORY` | Task execution history | `SELECT * FROM INFORMATION_SCHEMA.TASK_HISTORY WHERE NAME = 'MY_TASK' AND SCHEDULED_TIME > DATEADD('day', -7, CURRENT_TIMESTAMP());` | 365 days |
| `INFORMATION_SCHEMA.COPY_HISTORY` | COPY INTO history | `SELECT * FROM INFORMATION_SCHEMA.COPY_HISTORY WHERE PIPE_NAME = 'MY_PIPE';` | Session lifetime |
| `INFORMATION_SCHEMA.EXTERNAL_TABLE_FILES` | External table file metadata | `SELECT * FROM INFORMATION_SCHEMA.EXTERNAL_TABLE_FILES('MY_EXTERNAL_TABLE');` | Session lifetime |
| `SNOWFLAKE.ACCOUNT_USAGE.REPLICATION_GROUPS` | CDC replication status | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.REPLICATION_GROUPS;` | 365 days |

### Error Categorization & Runbooks

#### Snowpipe Errors
| Error Code | Root Cause | Impact | Severity | Runbook | Monitoring View |
|------------|------------|--------|----------|---------|-----------------|
| `STAGE_FILE_NOT_FOUND` | File deleted before processing | File not loaded | High | 1. Check stage: `LIST @MY_STAGE;` 2. Re-upload file. 3. Check cloud notifications. | `ACCOUNT_USAGE.PIPE_USAGE_HISTORY` |
| `STAGE_CONNECTION_ERROR` | Cloud storage unreachable | All files fail | Critical | 1. Check `SYSTEM$PIPE_DIAGNOSTIC('MY_PIPE');` 2. Validate cloud permissions. 3. Retry. | `ACCOUNT_USAGE.PIPE_USAGE_HISTORY` |
| `FILE_FORMAT_MISMATCH` | File format doesn't match | File not loaded | High | 1. Verify file format: `SELECT TYPE FROM INFORMATION_SCHEMA.FILE_FORMATS WHERE NAME = 'MY_FORMAT';` 2. Re-upload with correct format. | `ACCOUNT_USAGE.PIPE_USAGE_HISTORY` |
| `PIPE_PAUSED` | Pipe manually paused | All files queued | Medium | 1. Resume pipe: `ALTER PIPE MY_PIPE SET PAUSED = FALSE;` 2. Check for errors. | `ACCOUNT_USAGE.PIPE_USAGE_HISTORY` |
| `WAREHOUSE_SIZE_TOO_SMALL` | Warehouse too small for file | File not loaded | Medium | 1. Use larger warehouse. 2. Split file into smaller chunks. | `ACCOUNT_USAGE.PIPE_USAGE_HISTORY` |
| `DUPLICATE_FILE` | Duplicate file detected | File skipped | Low | 1. Enable `ENABLE_DUPLICATE_DETECTION = TRUE`. 2. Use `DUPLICATE_HANDLING = 'SKIP'`. | `ACCOUNT_USAGE.PIPE_USAGE_HISTORY` |

#### Kafka Connector Errors
| Error Code | Root Cause | Impact | Severity | Runbook | Monitoring View |
|------------|------------|--------|----------|---------|-----------------|
| `KAFKA_CONNECTION_ERROR` | Kafka broker unreachable | All messages fail | Critical | 1. Check `SYSTEM$KAFKA_CONNECTOR_DIAGNOSTIC('MY_CONNECTOR');` 2. Validate Kafka broker URL. 3. Retry. | `ACCOUNT_USAGE.KAFKA_CONSUMER_GROUPS` |
| `KAFKA_TOPIC_NOT_FOUND` | Topic doesn't exist | All messages fail | Critical | 1. Verify topic: `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.KAFKA_TOPICS WHERE TOPIC_NAME = 'MY_TOPIC';` 2. Create topic in Kafka. | `ACCOUNT_USAGE.KAFKA_CONSUMER_GROUPS` |
| `DESERIALIZATION_ERROR` | Message deserialization failed | Message skipped | High | 1. Check message format. 2. Use `TRANSFORMATION` to extract correct fields. | `ACCOUNT_USAGE.KAFKA_CONSUMER_GROUPS` |
| `SCHEMA_VALIDATION_ERROR` | Message doesn't match schema | Message skipped | High | 1. Disable `ENABLE_SCHEMA_VALIDATION`. 2. Fix schema. | `ACCOUNT_USAGE.KAFKA_CONSUMER_GROUPS` |
| `OFFSET_OUT_OF_RANGE` | Offset no longer exists | Messages skipped | Medium | 1. Reset offset: `ALTER KAFKA CONNECTOR MY_CONNECTOR SET START_OFFSET = 'earliest';` | `ACCOUNT_USAGE.KAFKA_CONSUMER_GROUPS` |

#### Ingestion Service Errors
| Error Code | Root Cause | Impact | Severity | Runbook | Monitoring View |
|------------|------------|--------|----------|---------|-----------------|
| `INVALID_TOKEN` | JWT token expired/invalid | All requests fail | Critical | 1. Refresh token. 2. Check token expiry. | `ACCOUNT_USAGE.INGESTION_HISTORY` |
| `PAYLOAD_TOO_LARGE` | Payload >16MB | Request rejected | Medium | 1. Split payload into smaller batches. 2. Use compression. | `ACCOUNT_USAGE.INGESTION_HISTORY` |
| `RATE_LIMIT_EXCEEDED` | >10,000 requests/sec | Requests throttled | Medium | 1. Reduce request rate. 2. Use larger batches. | `ACCOUNT_USAGE.INGESTION_HISTORY` |
| `SCHEMA_MISMATCH` | Payload doesn't match table schema | Rows skipped | High | 1. Fix payload schema. 2. Use `TRANSFORMATION`. | `ACCOUNT_USAGE.INGESTION_HISTORY` |
| `INVALID_JSON` | Malformed JSON | Rows skipped | High | 1. Validate JSON syntax. 2. Use `ON_ERROR = 'CONTINUE'`. | `ACCOUNT_USAGE.INGESTION_HISTORY` |

#### External Table Errors
| Error Code | Root Cause | Impact | Severity | Runbook | Monitoring View |
|------------|------------|--------|----------|---------|-----------------|
| `STAGE_FILE_NOT_FOUND` | File deleted from stage | Query fails | High | 1. Check stage: `LIST @MY_STAGE;` 2. Re-upload file. | `INFORMATION_SCHEMA.EXTERNAL_TABLE_FILES` |
| `FILE_FORMAT_MISMATCH` | File format doesn't match | Query fails | High | 1. Verify file format. 2. Re-upload with correct format. | `INFORMATION_SCHEMA.EXTERNAL_TABLE_FILES` |
| `PERMISSION_DENIED` | Insufficient permissions | Query fails | Critical | 1. Grant `USAGE` on stage: `GRANT USAGE ON STAGE MY_STAGE TO ROLE MY_ROLE;` | `INFORMATION_SCHEMA.EXTERNAL_TABLE_FILES` |
| `EXTERNAL_STAGE_CONNECTION_ERROR` | Cloud storage unreachable | Query fails | Critical | 1. Check `SYSTEM$STAGE_DIAGNOSTIC('MY_STAGE');` 2. Validate cloud permissions. | `INFORMATION_SCHEMA.EXTERNAL_TABLE_FILES` |
| `PARTITION_NOT_FOUND` | Partition doesn't exist | Query fails | Medium | 1. Check partition columns. 2. Add missing partitions. | `INFORMATION_SCHEMA.EXTERNAL_TABLE_FILES` |

#### Snowflake Connector (CDC) Errors
| Error Code | Root Cause | Impact | Severity | Runbook | Monitoring View |
|------------|------------|--------|----------|---------|-----------------|
| `CONNECTION_ERROR` | Source DB unreachable | Replication paused | Critical | 1. Check `SYSTEM$REPLICATION_DIAGNOSTIC('MY_CONNECTOR');` 2. Validate DB connection. | `SNOWFLAKE.ACCOUNT_USAGE.REPLICATION_GROUPS` |
| `SCHEMA_CHANGE` | Source schema changed | Replication paused | High | 1. Update target schema. 2. Resume replication. | `SNOWFLAKE.ACCOUNT_USAGE.REPLICATION_GROUPS` |
| `PERMISSION_ERROR` | Insufficient source permissions | Replication paused | Critical | 1. Grant permissions on source DB. 2. Validate credentials. | `SNOWFLAKE.ACCOUNT_USAGE.REPLICATION_GROUPS` |
| `LOG_NOT_ACCESSIBLE` | WAL/binlog not accessible | Replication paused | Critical | 1. Enable WAL/binlog on source. 2. Validate permissions. | `SNOWFLAKE.ACCOUNT_USAGE.REPLICATION_GROUPS` |

#### Task Errors
| Error Code | Root Cause | Impact | Severity | Runbook | Monitoring View |
|------------|------------|--------|----------|---------|-----------------|
| `WAREHOUSE_UNAVAILABLE` | Warehouse suspended/overloaded | Task fails | Medium | 1. Resume warehouse: `ALTER WAREHOUSE MY_WH RESUME;` 2. Use larger warehouse. | `INFORMATION_SCHEMA.TASK_HISTORY` |
| `STORED_PROCEDURE_ERROR` | Procedure failed | Task fails | High | 1. Check procedure logs. 2. Fix procedure. | `INFORMATION_SCHEMA.TASK_HISTORY` |
| `TIMEOUT` | Task exceeded timeout | Task fails | Medium | 1. Increase `USER_TASK_TIMEOUT_MS`. 2. Optimize procedure. | `INFORMATION_SCHEMA.TASK_HISTORY` |
| `CONDITION_NOT_MET` | WHEN condition false | Task skipped | Low | 1. Check WHEN condition. 2. Update condition. | `INFORMATION_SCHEMA.TASK_HISTORY` |
| `RETRY_LIMIT_EXCEEDED` | Max retries reached | Task fails | Medium | 1. Increase `RETRY_COUNT`. 2. Fix root cause. | `INFORMATION_SCHEMA.TASK_HISTORY` |

### Incident Runbooks

#### Runbook: Snowpipe Stuck Files
```sql
-- Step 1: Identify stuck files
SELECT
    pipe_name,
    file_name,
    state,
    last_load_time,
    error_message
FROM
    SNOWFLAKE.ACCOUNT_USAGE.PIPE_USAGE_HISTORY
WHERE
    pipe_name = 'MY_PIPE'
    AND state = 'PENDING'
    AND last_load_time < DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY
    last_load_time;

-- Step 2: Check pipe status
SELECT
    SYSTEM$PIPE_STATUS('MY_PIPE');

-- Step 3: Check cloud notifications
-- For S3:
SELECT
    SYSTEM$PIPE_DIAGNOSTIC('MY_PIPE');

-- Step 4: Force refresh
ALTER PIPE MY_PIPE REFRESH;

-- Step 5: Check for duplicate files
SELECT
    file_name,
    COUNT(*) AS dup_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.PIPE_USAGE_HISTORY
WHERE
    pipe_name = 'MY_PIPE'
GROUP BY
    file_name
HAVING
    COUNT(*) > 1;

-- Step 6: Reprocess stuck files
ALTER PIPE MY_PIPE SET PAUSED = TRUE;
ALTER PIPE MY_PIPE SET PAUSED = FALSE;
```

#### Runbook: Kafka Connector Lag
```sql
-- Step 1: Check consumer lag
SELECT
    consumer_group,
    topic,
    partition,
    current_offset,
    end_offset,
    lag,
    last_poll_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.KAFKA_CONSUMER_GROUPS
WHERE
    consumer_group = 'snowflake-{account_id}'
    AND topic = 'MY_TOPIC'
    AND lag > 1000
ORDER BY
    lag DESC;

-- Step 2: Check connector status
SELECT
    SYSTEM$KAFKA_CONNECTOR_STATUS('MY_CONNECTOR');

-- Step 3: Check for errors
SELECT
    error_message,
    error_count,
    last_error_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.KAFKA_CONSUMER_GROUPS
WHERE
    consumer_group = 'snowflake-{account_id}'
    AND error_count > 0;

-- Step 4: Restart connector
ALTER KAFKA CONNECTOR MY_CONNECTOR SET ENABLED = FALSE;
ALTER KAFKA CONNECTOR MY_CONNECTOR SET ENABLED = TRUE;

-- Step 5: Reset offset (if needed)
ALTER KAFKA CONNECTOR MY_CONNECTOR SET START_OFFSET = 'earliest';
```

#### Runbook: Ingestion Service Failures
```sql
-- Step 1: Check ingestion history
SELECT
    stream_name,
    request_id,
    status,
    rows_processed,
    rows_failed,
    error_message,
    start_time,
    end_time
FROM
    SNOWFLAKE.ACCOUNT_USAGE.INGESTION_HISTORY
WHERE
    stream_name = 'MY_STREAM'
    AND status = 'FAILED'
    AND start_time > DATEADD('hour', -1, CURRENT_TIMESTAMP())
ORDER BY
    start_time DESC;

-- Step 2: Check for rate limits
SELECT
    stream_name,
    request_count,
    failed_count,
    rate_limited_count
FROM
    SNOWFLAKE.ACCOUNT_USAGE.INGESTION_HISTORY
WHERE
    stream_name = 'MY_STREAM'
    AND start_time > DATEADD('minute', -5, CURRENT_TIMESTAMP())
GROUP BY
    stream_name, request_count, failed_count, rate_limited_count;

-- Step 3: Test with sample payload
SELECT
    SYSTEM$INGESTION_DIAGNOSTIC('MY_STREAM', '{"test": "data"}');

-- Step 4: Check token validity
SELECT
    SYSTEM$INGESTION_TOKEN_VALIDITY('MY_STREAM');

-- Step 5: Rotate token
ALTER INGESTION STREAM MY_STREAM REGENERATE_TOKEN;
```

#### Runbook: External Table Query Failures
```sql
-- Step 1: Check file metadata
SELECT
    file_name,
    size,
    last_modified,
    error_message
FROM
    INFORMATION_SCHEMA.EXTERNAL_TABLE_FILES('MY_EXTERNAL_TABLE')
WHERE
    error_message IS NOT NULL;

-- Step 2: Check stage connectivity
SELECT
    SYSTEM$STAGE_DIAGNOSTIC('MY_STAGE');

-- Step 3: Refresh metadata
ALTER EXTERNAL TABLE MY_EXTERNAL_TABLE REFRESH;

-- Step 4: Check file format
SELECT
    name,
    type,
    field_delimiter,
    record_delimiter
FROM
    INFORMATION_SCHEMA.FILE_FORMATS
WHERE
    name = 'MY_FORMAT';

-- Step 5: Test with sample file
SELECT
    COUNT(*)
FROM
    @MY_STAGE/sample.parquet
FILE_FORMAT = (TYPE = 'PARQUET');
```

#### Runbook: CDC Replication Lag
```sql
-- Step 1: Check replication status
SELECT
    replication_group,
    source_database,
    source_schema,
    source_table,
    last_replicated_time,
    lag_seconds,
    status,
    error_message
FROM
    SNOWFLAKE.ACCOUNT_USAGE.REPLICATION_GROUPS
WHERE
    replication_group = 'MY_REPLICATION'
    AND lag_seconds > 300
ORDER BY
    lag_seconds DESC;

-- Step 2: Check connector status
SELECT
    SYSTEM$REPLICATION_DIAGNOSTIC('MY_CONNECTOR');

-- Step 3: Check for schema changes
SELECT
    change_time,
    change_type,
    changed_object
FROM
    SNOWFLAKE.ACCOUNT_USAGE.REPLICATION_CHANGES
WHERE
    replication_group = 'MY_REPLICATION'
    AND change_time > DATEADD('hour', -24, CURRENT_TIMESTAMP())
ORDER BY
    change_time DESC;

-- Step 4: Resume replication
ALTER REPLICATION GROUP MY_REPLICATION RESUME;

-- Step 5: Check source DB logs
-- (Source DB specific)
```

#### Runbook: Task Execution Failures
```sql
-- Step 1: Check task history
SELECT
    name,
    scheduled_time,
    start_time,
    end_time,
    state,
    return_code,
    error_message
FROM
    INFORMATION_SCHEMA.TASK_HISTORY
WHERE
    name = 'MY_TASK'
    AND state = 'FAILED'
    AND scheduled_time > DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY
    scheduled_time DESC;

-- Step 2: Check warehouse availability
SELECT
    name,
    state,
    size,
    running_queries,
    queued_queries
FROM
    SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSES
WHERE
    name = 'MY_WH';

-- Step 3: Check procedure logs
-- (Depends on procedure implementation)

-- Step 4: Test procedure manually
CALL MY_PROCEDURE();

-- Step 5: Resume task
ALTER TASK MY_TASK RESUME;
```

### Proactive Alerts

#### Alert: Snowpipe Stuck Files
```sql
CREATE OR REPLACE ALERT SNOWPIPE_STUCK_FILES_ALERT
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON 0 * * * * America/Los_Angeles'
AS
  SELECT
    pipe_name,
    file_name,
    state,
    last_load_time,
    DATEDIFF('minute', last_load_time, CURRENT_TIMESTAMP()) AS minutes_stuck,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.PIPE_USAGE_HISTORY
  WHERE
    state = 'PENDING'
    AND last_load_time < DATEADD('hour', -1, CURRENT_TIMESTAMP());
```

#### Alert: Kafka Connector Lag
```sql
CREATE OR REPLACE ALERT KAFKA_CONNECTOR_LAG_ALERT
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON */15 * * * * America/Los_Angeles'
AS
  SELECT
    consumer_group,
    topic,
    partition,
    lag,
    last_poll_time,
    DATEDIFF('minute', last_poll_time, CURRENT_TIMESTAMP()) AS minutes_behind,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.KAFKA_CONSUMER_GROUPS
  WHERE
    consumer_group = 'snowflake-{account_id}'
    AND lag > 1000;
```

#### Alert: Ingestion Service Failures
```sql
CREATE OR REPLACE ALERT INGESTION_SERVICE_FAILURES_ALERT
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'
AS
  SELECT
    stream_name,
    request_id,
    status,
    rows_failed,
    error_message,
    start_time,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.INGESTION_HISTORY
  WHERE
    status = 'FAILED'
    AND start_time > DATEADD('minute', -10, CURRENT_TIMESTAMP());
```

#### Alert: External Table Errors
```sql
CREATE OR REPLACE ALERT EXTERNAL_TABLE_ERRORS_ALERT
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON 0 * * * * America/Los_Angeles'
AS
  SELECT
    table_name,
    file_name,
    error_message,
    last_error_time,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    INFORMATION_SCHEMA.EXTERNAL_TABLE_FILES
  WHERE
    error_message IS NOT NULL;
```

#### Alert: CDC Replication Lag
```sql
CREATE OR REPLACE ALERT CDC_REPLICATION_LAG_ALERT
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON 0 * * * * America/Los_Angeles'
AS
  SELECT
    replication_group,
    source_database,
    source_schema,
    source_table,
    lag_seconds,
    last_replicated_time,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    SNOWFLAKE.ACCOUNT_USAGE.REPLICATION_GROUPS
  WHERE
    lag_seconds > 300;
```

#### Alert: Task Execution Failures
```sql
CREATE OR REPLACE ALERT TASK_EXECUTION_FAILURES_ALERT
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON */15 * * * * America/Los_Angeles'
AS
  SELECT
    name,
    scheduled_time,
    state,
    return_code,
    error_message,
    CURRENT_TIMESTAMP() AS alert_time
  FROM
    INFORMATION_SCHEMA.TASK_HISTORY
  WHERE
    state = 'FAILED'
    AND scheduled_time > DATEADD('hour', -1, CURRENT_TIMESTAMP());
```

## Advanced Production Patterns

### Idempotency Strategies

#### Snowpipe Idempotency
- **File-Level Deduplication**:
  - Enable `ENABLE_DUPLICATE_DETECTION` to detect duplicate files via checksum
  - Use `DUPLICATE_HANDLING = 'SKIP'` to skip duplicates
  - Example:
    ```sql
    CREATE PIPE MY_PIPE
      AUTO_INGEST = TRUE
      ENABLE_DUPLICATE_DETECTION = TRUE
      DUPLICATE_HANDLING = 'SKIP'
      AS COPY INTO MY_TABLE FROM @MY_STAGE;
    ```

- **Checksum-Based Tracking**:
  - Maintain a table of loaded file checksums
  - Example:
    ```sql
    CREATE TABLE LOADED_FILES (
        file_name STRING PRIMARY KEY,
        md5 STRING,
        load_time TIMESTAMP_LTZ
    );

    -- Check if file already loaded
    CREATE OR REPLACE PROCEDURE CHECK_FILE_LOADED(FILE_NAME STRING)
    RETURNS BOOLEAN
    LANGUAGE SQL
    AS
    $$
    DECLARE
        count INT;
    BEGIN
        SELECT COUNT(*) INTO count
        FROM LOADED_FILES
        WHERE file_name = FILE_NAME;

        RETURN count > 0;
    END;
    $$;

    -- Load and record
    CREATE OR REPLACE PROCEDURE LOAD_FILE(FILE_NAME STRING)
    RETURNS STRING
    LANGUAGE SQL
    AS
    $$
    DECLARE
        already_loaded BOOLEAN;
        md5 STRING;
    BEGIN
        already_loaded := CHECK_FILE_LOADED(FILE_NAME);

        IF (NOT already_loaded) THEN
            -- Get MD5 checksum
            SELECT MD5(CONTENT) INTO md5 FROM @MY_STAGE/{FILE_NAME};

            -- Load file
            COPY INTO MY_TABLE FROM @MY_STAGE/{FILE_NAME};

            -- Record loaded file
            INSERT INTO LOADED_FILES VALUES (FILE_NAME, md5, CURRENT_TIMESTAMP());

            RETURN 'Loaded ' || FILE_NAME;
        ELSE
            RETURN FILE_NAME || ' already loaded';
        END IF;
    END;
    $$;
    ```

#### Kafka Connector Idempotency
- **At-Least-Once Processing**:
  - Kafka offsets are committed **after** successful COPY INTO
  - Example:
    ```sql
    CREATE KAFKA CONNECTOR MY_CONNECTOR
      KAFKA_BROKER = 'my-broker:9092'
      KAFKA_TOPIC = 'my-topic'
      START_OFFSET = 'earliest'
      ENABLED = TRUE;
    ```

- **Deduplication Table**:
  - Track processed messages by key
  - Example:
    ```sql
    CREATE TABLE PROCESSED_MESSAGES (
        message_key STRING PRIMARY KEY,
        message_offset BIGINT,
        processed_time TIMESTAMP_LTZ
    );

    -- Check if message already processed
    CREATE OR REPLACE PROCEDURE PROCESS_MESSAGE(MESSAGE_KEY STRING, MESSAGE_OFFSET BIGINT)
    RETURNS BOOLEAN
    LANGUAGE SQL
    AS
    $$
    DECLARE
        count INT;
    BEGIN
        SELECT COUNT(*) INTO count
        FROM PROCESSED_MESSAGES
        WHERE message_key = MESSAGE_KEY;

        RETURN count > 0;
    END;
    $$;
    ```

#### Ingestion Service Idempotency
- **Request-Level Deduplication**:
  - Use `request_id` to track processed requests
  - Example:
    ```sql
    CREATE TABLE PROCESSED_REQUESTS (
        request_id STRING PRIMARY KEY,
        processed_time TIMESTAMP_LTZ
    );

    -- Check if request already processed
    CREATE OR REPLACE PROCEDURE PROCESS_REQUEST(REQUEST_ID STRING, PAYLOAD VARIANT)
    RETURNS STRING
    LANGUAGE SQL
    AS
    $$
    DECLARE
        already_processed BOOLEAN;
    BEGIN
        already_processed := (SELECT COUNT(*) FROM PROCESSED_REQUESTS WHERE request_id = REQUEST_ID) > 0;

        IF (NOT already_processed) THEN
            -- Insert payload into target table
            INSERT INTO MY_TABLE SELECT PAYLOAD:*::STRING;

            -- Record processed request
            INSERT INTO PROCESSED_REQUESTS VALUES (REQUEST_ID, CURRENT_TIMESTAMP());

            RETURN 'Processed ' || REQUEST_ID;
        ELSE
            RETURN REQUEST_ID || ' already processed';
        END IF;
    END;
    $$;
    ```

### DLQ Routing & Recovery

#### Snowpipe DLQ
- **DLQ Stage**:
  - Files with persistent errors are moved to `@pipe_name_DLQ` stage
  - Example:
    ```sql
    -- List DLQ files
    LIST @MY_PIPE_DLQ;

    -- Reprocess DLQ files
    COPY INTO MY_TABLE
    FROM @MY_PIPE_DLQ
    FILE_FORMAT = (TYPE = 'CSV')
    ON_ERROR = 'CONTINUE';
    ```

- **DLQ Table**:
  - Create a table to track DLQ files
  - Example:
    ```sql
    CREATE TABLE PIPE_DLQ (
        pipe_name STRING,
        file_name STRING,
        error_message STRING,
        error_time TIMESTAMP_LTZ,
        retry_count INT
    );

    -- Insert into DLQ table on error
    CREATE OR REPLACE PROCEDURE LOG_DLQ(PIPE_NAME STRING, FILE_NAME STRING, ERROR_MESSAGE STRING)
    RETURNS STRING
    LANGUAGE SQL
    AS
    $$
    BEGIN
        INSERT INTO PIPE_DLQ
        SELECT PIPE_NAME, FILE_NAME, ERROR_MESSAGE, CURRENT_TIMESTAMP(), 0;

        RETURN 'Logged DLQ for ' || FILE_NAME;
    END;
    $$;
    ```

#### Kafka Connector DLQ
- **DLQ Topic**:
  - Messages with persistent errors are sent to a Kafka DLQ topic
  - Example:
    ```sql
    CREATE KAFKA CONNECTOR MY_CONNECTOR
      KAFKA_BROKER = 'my-broker:9092'
      KAFKA_TOPIC = 'my-topic'
      DLQ_TOPIC = 'my-dlq-topic'
      ENABLED = TRUE;
    ```

- **DLQ Consumer**:
  - Create a consumer to process DLQ messages
  - Example:
    ```sql
    CREATE KAFKA CONNECTOR DLQ_CONNECTOR
      KAFKA_BROKER = 'my-broker:9092'
      KAFKA_TOPIC = 'my-dlq-topic'
      TARGET_TABLE = 'DLQ_TABLE'
      ENABLED = TRUE;
    ```

#### Ingestion Service DLQ
- **DLQ Stage**:
  - Failed rows are written to `@ingest_DLQ` stage
  - Example:
    ```sql
    -- List DLQ files
    LIST @ingest_DLQ;

    -- Reprocess DLQ files
    COPY INTO MY_TABLE
    FROM @ingest_DLQ
    FILE_FORMAT = (TYPE = 'JSON')
    ON_ERROR = 'CONTINUE';
    ```

### CI/CD Validation

#### File Format Validation
- **Validate File Format**:
  ```sql
  SELECT
      SYSTEM$VALIDATE_FILE_FORMAT('MY_FORMAT', 'CSV', 'field1,field2');
  ```

- **Check for Breaking Changes**:
  ```sql
  SELECT
      name,
      type,
      compression,
      field_delimiter
  FROM
      INFORMATION_SCHEMA.FILE_FORMATS
  WHERE
      name = 'MY_FORMAT'
      AND (type != 'CSV' OR compression != 'AUTO');
  ```

#### Pipe Validation
- **Check Pipe Definition**:
  ```sql
  SELECT
      name,
      definition,
      state,
      auto_ingest,
      error_integration
  FROM
      INFORMATION_SCHEMA.PIPES
  WHERE
      name = 'MY_PIPE';
  ```

- **Test Pipe with Sample File**:
  ```sql
  -- Upload sample file
  PUT file:///sample.csv @MY_STAGE;

  -- Test pipe
  ALTER PIPE MY_PIPE REFRESH;

  -- Check history
  SELECT * FROM INFORMATION_SCHEMA.COPY_HISTORY WHERE PIPE_NAME = 'MY_PIPE';
  ```

#### Kafka Connector Validation
- **Check Connector Definition**:
  ```sql
  SELECT
      name,
      kafka_broker,
      kafka_topic,
      enabled,
      last_error_message
  FROM
      INFORMATION_SCHEMA.KAFKA_CONNECTORS
  WHERE
      name = 'MY_CONNECTOR';
  ```

- **Test Connector with Sample Messages**:
  ```sql
  -- Produce sample messages to Kafka topic
  -- (Kafka CLI or custom script)

  -- Check consumer lag
  SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.KAFKA_CONSUMER_GROUPS
  WHERE consumer_group = 'snowflake-{account_id}'
  AND topic = 'MY_TOPIC';
  ```

### Retry & Backpressure Logic

#### Exponential Backoff (Snowpipe)
```sql
-- Configure pipe with retry logic
CREATE PIPE MY_PIPE
  AUTO_INGEST = TRUE
  ERROR_INTEGRATION = 'MY_NOTIFICATION_CHANNEL'
  AS COPY INTO MY_TABLE FROM @MY_STAGE
  ON_ERROR = 'CONTINUE';

-- Note: Retry logic is built-in (3 retries with exponential backoff)
```

#### Circuit Breaker (Kafka Connector)
```sql
-- Create a procedure to monitor and disable connector on failures
CREATE OR REPLACE PROCEDURE MONITOR_KAFKA_CONNECTOR()
RETURNS STRING
LANGUAGE SQL
AS
$$
DECLARE
    error_count INT;
    max_errors INT := 10;
BEGIN
    -- Check for errors in last hour
    SELECT COUNT(*) INTO error_count
    FROM SNOWFLAKE.ACCOUNT_USAGE.KAFKA_CONSUMER_GROUPS
    WHERE consumer_group = 'snowflake-{account_id}'
    AND error_count > 0
    AND last_error_time > DATEADD('hour', -1, CURRENT_TIMESTAMP());

    -- Disable connector if error threshold exceeded
    IF (error_count > max_errors) THEN
        ALTER KAFKA CONNECTOR MY_CONNECTOR SET ENABLED = FALSE;

        -- Send alert
        CALL SYSTEM$SEND_EMAIL(
            'admin@example.com',
            'Kafka Connector Disabled',
            'Kafka connector MY_CONNECTOR disabled due to errors'
        );

        RETURN 'Connector disabled';
    ELSE
        RETURN 'Connector healthy';
    END IF;
END;
$$;

-- Schedule monitoring
CREATE TASK MONITOR_KAFKA_CONNECTOR_TASK
  WAREHOUSE = MONITORING_WH
  SCHEDULE = 'USING CRON */5 * * * * America/Los_Angeles'
AS
  CALL MONITOR_KAFKA_CONNECTOR();
```

#### Rate Limiting (Ingestion Service)
```sql
-- Create a procedure to implement rate limiting
CREATE OR REPLACE PROCEDURE RATE_LIMITED_INGEST(PAYLOAD VARIANT, MAX_REQUESTS_PER_MINUTE INT)
RETURNS STRING
LANGUAGE JAVASCRIPT
AS
$$
    // Check request count in last minute
    const request_count = snowflake.execute({
        sql: `SELECT COUNT(*) FROM INGESTION_REQUESTS
              WHERE request_time > DATEADD('minute', -1, CURRENT_TIMESTAMP())`,
        binds: {}
    }).next().getColumn(1);

    if (request_count >= MAX_REQUESTS_PER_MINUTE) {
        return 'Rate limit exceeded';
    } else {
        // Insert into target table
        snowflake.execute({
            sql: `INSERT INTO MY_TABLE SELECT PAYLOAD:*::STRING`,
            binds: {PAYLOAD: PAYLOAD}
        });

        // Record request
        snowflake.execute({
            sql: `INSERT INTO INGESTION_REQUESTS VALUES (CURRENT_TIMESTAMP())`,
            binds: {}
        });

        return 'Ingested successfully';
    }
$$;
```

### Security & Compliance Controls

#### Encryption
- **Snowpipe**:
  - Use **CMK** for internal stages
  - Example:
    ```sql
    CREATE STAGE MY_STAGE
      URL = 's3://my-bucket'
      ENCRYPTION = (TYPE = 'CUSTOMER_MANAGED' KEY = 'arn:aws:kms:us-west-2:123456789012:key/abcd1234');
    ```

- **Kafka Connector**:
  - Use **SSL/SASL** for secure communication
  - Example:
    ```sql
    CREATE KAFKA CONNECTOR MY_CONNECTOR
      KAFKA_BROKER = 'my-broker:9092'
      KAFKA_SECURITY_PROTOCOL = 'SASL_SSL'
      KAFKA_SASL_MECHANISM = 'SCRAM_SHA_256'
      KAFKA_SASL_USERNAME = 'my-user'
      KAFKA_SASL_PASSWORD = 'my-password';
    ```

- **Ingestion Service**:
  - Use **JWT tokens** with short expiry
  - Example:
    ```sql
    CREATE INGESTION STREAM MY_STREAM
      TOKEN_EXPIRY = 1;  -- 1 hour
    ```

#### Network Policies
- **IP Whitelisting**:
  - Restrict ingestion to specific IP ranges
  - Example:
    ```sql
    ALTER ACCOUNT SET NETWORK_POLICY = (
        IP_RANGES = ('192.168.1.0/24', '10.0.0.0/16'),
        BLOCKED_IP_RANGES = ('0.0.0.0/0')
    );
    ```

- **PrivateLink**:
  - Use **AWS PrivateLink** or **Azure Private Link** for Snowpipe
  - Example:
    ```sql
    CREATE STAGE MY_STAGE
      URL = 's3://my-bucket'
      STORAGE_INTEGRATION = 'MY_PRIVATELINK_INTEGRATION';
    ```

#### Audit Logging
- **Enable Ingestion Logging**:
  ```sql
  ALTER ACCOUNT SET INGESTION_LOGGING = TRUE;
  ```

- **Query Ingestion Logs**:
  ```sql
  SELECT
      stream_name,
      request_id,
      status,
      rows_processed,
      start_time,
      end_time
  FROM
      SNOWFLAKE.ACCOUNT_USAGE.INGESTION_HISTORY
  WHERE
      start_time > DATEADD('day', -7, CURRENT_TIMESTAMP());
  ```

#### RBAC
- **Pipe Permissions**:
  ```sql
  GRANT MONITOR ON PIPE MY_PIPE TO ROLE MONITOR_ROLE;
  GRANT OPERATE ON PIPE MY_PIPE TO ROLE OPERATOR_ROLE;
  ```

- **Kafka Connector Permissions**:
  ```sql
  GRANT MONITOR ON KAFKA CONNECTOR MY_CONNECTOR TO ROLE MONITOR_ROLE;
  GRANT OPERATE ON KAFKA CONNECTOR MY_CONNECTOR TO ROLE OPERATOR_ROLE;
  ```

- **Ingestion Stream Permissions**:
  ```sql
  GRANT USAGE ON INGESTION STREAM MY_STREAM TO ROLE INGEST_ROLE;
  ```

## Decision Matrix

### Ingestion Method Selection Flowchart
```
Data Source Type
├── Cloud Storage (S3/Azure/GCS)
│   ├── Real-Time Requirements?
│   │   ├── Yes → Snowpipe (AUTO_INGEST=TRUE)
│   │   └── No → Snowpipe (AUTO_INGEST=FALSE) or Tasks
│   └── File Size?
│       ├── < 100MB → Snowpipe
│       └── > 100MB → Snowpipe (partitioned)
├── Kafka
│   ├── Throughput Requirements?
│   │   ├── < 100MB/sec → Kafka Connector
│   │   └── > 100MB/sec → Kafka Connector (multiple connectors)
│   └── Latency Requirements?
│       ├── < 1 sec → Kafka Connector
│       └── > 1 sec → Kafka Connector or Snowpipe
├── REST APIs
│   ├── Real-Time Requirements?
│   │   ├── Yes → Ingestion Service
│   │   └── No → Tasks + Stored Procedures
│   └── Payload Size?
│       ├── < 16MB → Ingestion Service
│       └── > 16MB → Tasks + Stored Procedures
├── External Databases
│   ├── Change Volume?
│   │   ├── High → Snowflake Connector (CDC)
│   │   └── Low → Tasks + Stored Procedures
│   └── Latency Requirements?
│       ├── < 5 min → Snowflake Connector
│       └── > 5 min → Tasks + Stored Procedures
└── Data Lakes (Delta/Iceberg)
    └── Query Patterns?
        ├── Ad-Hoc → External Tables
        └── Scheduled → Tasks + COPY INTO
```

### Quick Reference Table
| Use Case | Ingestion Method | Latency | Throughput | Cost | Complexity | Best For |
|----------|------------------|---------|------------|------|------------|----------|
| Real-time cloud storage | Snowpipe (AUTO_INGEST) | 1-10 min | 100-1000 MB/min | Low | Low | Batch loading from cloud storage |
| High-throughput streaming | Kafka Connector | <1 sec | 50-500 MB/sec | Medium | Medium | Real-time Kafka ingestion |
| REST API ingestion | Ingestion Service | <1 sec | 1-10 MB/sec | Medium | Low | REST API sources |
| Query external data | External Tables | 100-500 ms | N/A | Low | Low | Query external data lakes |
| Database replication | Snowflake Connector | 1-5 min | 10-100 MB/min | Medium | High | Database CDC |
| Scheduled batch | Tasks + Stored Procedures | 1-60 min | 10-1000 MB/min | Low | Medium | Scheduled batch ingestion |
| Low-latency cloud storage | Snowpipe (AUTO_INGEST) + Serverless | 1-5 min | 100-1000 MB/min | Medium | Low | Near real-time cloud storage |
| High-volume Kafka | Kafka Connector (multiple) | <1 sec | 500+ MB/sec | High | High | Enterprise Kafka ingestion |
| Cost-sensitive | Snowpipe (Serverless) | 1-10 min | 100-1000 MB/min | Low | Low | Cost-optimized cloud storage |
| Complex transformations | Tasks + Stored Procedures | 1-60 min | 10-1000 MB/min | Medium | High | Custom transformation logic |

## Key Engineering Principles

### Core Principles
1. **At-Least-Once > Exactly-Once**:
   - Snowflake guarantees **at-least-once** processing for all ingestion methods
   - Implement **idempotency** in target tables to handle duplicates

2. **Serverless > Warehouse for Ingestion**:
   - **Snowpipe/Ingestion Service**: Use serverless compute to reduce costs
   - **Kafka Connector**: Serverless scaling with Kafka partitions
   - **Warehouse**: Only use for Tasks or complex transformations

3. **Micro-Batching > Streaming for Cost**:
   - Snowpipe processes files in **micro-batches** (1 file per COPY)
   - Kafka Connector processes messages in **batches** (10,000 messages per COPY)
   - Ingestion Service processes rows in **micro-batches** (100-10,000 rows per request)

4. **Partitioning > Single Thread**:
   - **Kafka**: 1 consumer thread per partition (scale with partitions)
   - **Snowpipe**: 1 worker per pipe (scale with pipes)
   - **External Tables**: 1 thread per file (scale with files)

5. **Error Handling is Critical**:
   - Always configure **DLQ** for persistent errors
   - Use **exponential backoff** for transient errors
   - Implement **circuit breakers** for repeated failures

6. **Monitoring is Non-Negotiable**:
   - **Snowpipe**: Monitor `PIPE_USAGE_HISTORY` for stuck files
   - **Kafka**: Monitor `KAFKA_CONSUMER_GROUPS` for lag
   - **Ingestion Service**: Monitor `INGESTION_HISTORY` for failures
   - **External Tables**: Monitor `EXTERNAL_TABLE_FILES` for errors

7. **Security by Default**:
   - **Encryption**: Always use CMK or cloud-native encryption
   - **Network**: Use PrivateLink or IP whitelisting
   - **RBAC**: Least-privilege access for all ingestion components

### Production Checklist
#### Ingestion Design
- [ ] **Snowpipe**:
  - Use `AUTO_INGEST = TRUE` for real-time requirements
  - Configure cloud notifications (SQS/SNS/Event Grid/Pub/Sub)
  - Set `ENABLE_DUPLICATE_DETECTION = TRUE` for idempotency
  - Use serverless compute for cost efficiency

- [ ] **Kafka Connector**:
  - Use 1 consumer thread per Kafka partition
  - Configure `KAFKA_POLL_INTERVAL_MS` and `KAFKA_BATCH_SIZE` for performance
  - Use SSL/SASL for secure communication
  - Set up DLQ topic for error handling

- [ ] **Ingestion Service**:
  - Use `BATCH_SIZE` and `FLUSH_INTERVAL` to balance latency and throughput
  - Enable `COMPRESSION = 'GZIP'` to reduce payload size
  - Implement token rotation for security

- [ ] **External Tables**:
  - Use `AUTO_REFRESH = TRUE` for real-time queries
  - Configure `PARTITION_BY` for partition pruning
  - Use columnar formats (Parquet/ORC) for performance

- [ ] **Snowflake Connector**:
  - Use `CDC_MODE = 'LOG_BASED'` for lowest latency
  - Configure `POLLING_INTERVAL` based on SLA requirements
  - Set up DLQ stage for error handling

- [ ] **Tasks**:
  - Use dedicated warehouse for consistent performance
  - Configure `RETRY_COUNT` and `RETRY_DELAY` for resilience
  - Set `ALLOW_OVERLAPPING_EXECUTION = FALSE` to avoid resource contention

#### Performance
- [ ] **Warehouse Sizing**:
  - **Snowpipe**: Serverless (no warehouse needed)
  - **Kafka Connector**: Serverless (scales with partitions)
  - **Ingestion Service**: Serverless (scales with requests)
  - **External Tables**: Size based on query concurrency
  - **Snowflake Connector**: Serverless (scales with tables)
  - **Tasks**: Size based on workload

- [ ] **Parallelism**:
  - **Snowpipe**: Scale with number of pipes
  - **Kafka Connector**: Scale with number of partitions
  - **Ingestion Service**: Scale with number of streams
  - **External Tables**: Scale with number of files
  - **Tasks**: Scale with warehouse size

- [ ] **Caching**:
  - **External Tables**: Cache frequently accessed files
  - **Tasks**: Cache intermediate results if possible

#### Error Handling
- [ ] **DLQ**:
  - Configure DLQ for all ingestion methods
  - Implement DLQ processing procedures
  - Monitor DLQ for persistent errors

- [ ] **Retry Logic**:
  - Use exponential backoff for transient errors
  - Configure max retries based on SLA
  - Implement circuit breakers for repeated failures

- [ ] **Alerting**:
  - Set up alerts for stuck files (Snowpipe)
  - Set up alerts for consumer lag (Kafka)
  - Set up alerts for ingestion failures (Ingestion Service)
  - Set up alerts for query errors (External Tables)
  - Set up alerts for replication lag (CDC)
  - Set up alerts for task failures (Tasks)

#### Security
- [ ] **Encryption**:
  - Use CMK for sensitive data
  - Use cloud-native encryption for external sources
  - Rotate keys regularly

- [ ] **Network**:
  - Use PrivateLink for cloud storage
  - Use IP whitelisting for REST APIs
  - Use SSL/SASL for Kafka

- [ ] **RBAC**:
  - Least-privilege access for all components
  - Separate roles for monitoring and operations
  - Regular access reviews

#### Cost Controls
- [ ] **Serverless**:
  - Use serverless compute for Snowpipe, Kafka Connector, Ingestion Service
  - Monitor serverless credit usage

- [ ] **Warehouse**:
  - Use dedicated warehouse for Tasks
  - Size warehouse based on workload
  - Monitor warehouse credit usage

- [ ] **Storage**:
  - Use external stages for cost-sensitive data
  - Configure lifecycle policies for cloud storage
  - Monitor storage costs

## Production-Ready Snippets

### Snowpipe Setup
#### Basic Snowpipe
```sql
-- Create stage
CREATE STAGE MY_STAGE
  URL = 's3://my-bucket'
  CREDENTIALS = (AWS_KEY_ID = 'my-key' AWS_SECRET_KEY = 'my-secret')
  FILE_FORMAT = (TYPE = 'CSV');

-- Create file format
CREATE FILE FORMAT MY_FORMAT
  TYPE = 'CSV'
  SKIP_HEADER = 1
  NULL_IF = ('NULL', 'null');

-- Create target table
CREATE TABLE MY_TABLE (
    id INTEGER,
    name STRING,
    created_at TIMESTAMP_LTZ
);

-- Create pipe
CREATE PIPE MY_PIPE
  AUTO_INGEST = TRUE
  AS COPY INTO MY_TABLE FROM @MY_STAGE;
```

#### Snowpipe with Error Handling
```sql
-- Create DLQ stage
CREATE STAGE MY_PIPE_DLQ;

-- Create pipe with error handling
CREATE PIPE MY_PIPE
  AUTO_INGEST = TRUE
  ENABLE_DUPLICATE_DETECTION = TRUE
  DUPLICATE_HANDLING = 'SKIP'
  ERROR_INTEGRATION = 'MY_NOTIFICATION_CHANNEL'
  AS COPY INTO MY_TABLE FROM @MY_STAGE
  FILE_FORMAT = (TYPE = 'CSV', SKIP_HEADER = 1, NULL_IF = ('NULL', 'null'))
  ON_ERROR = 'CONTINUE';
```

#### Snowpipe with Cloud Notifications
```sql
-- For S3:
CREATE NOTIFICATION INTEGRATION MY_SNS_INTEGRATION
  TYPE = 'SNS'
  ENABLED = TRUE
  SNS_TOPIC_ARN = 'arn:aws:sns:us-west-2:123456789012:my-topic';

-- Create pipe with notification
CREATE PIPE MY_PIPE
  AUTO_INGEST = TRUE
  NOTIFY_CHANNEL = MY_SNS_INTEGRATION
  AS COPY INTO MY_TABLE FROM @MY_STAGE;
```

### Kafka Connector Setup
#### Basic Kafka Connector
```sql
-- Create target table
CREATE TABLE MY_TABLE (
    id INTEGER,
    name STRING,
    event_time TIMESTAMP_LTZ
);

-- Create Kafka connector
CREATE KAFKA CONNECTOR MY_CONNECTOR
  KAFKA_BROKER = 'my-broker:9092'
  KAFKA_TOPIC = 'my-topic'
  TARGET_TABLE = 'MY_TABLE'
  ENABLED = TRUE;
```

#### Kafka Connector with Avro
```sql
-- Create Kafka connector with Avro
CREATE KAFKA CONNECTOR MY_AVRO_CONNECTOR
  KAFKA_BROKER = 'my-broker:9092'
  KAFKA_TOPIC = 'my-avro-topic'
  TARGET_TABLE = 'MY_TABLE'
  TRANSFORMATION = 'avro'
  SCHEMA_REGISTRY_URL = 'https://my-schema-registry'
  ENABLE_SCHEMA_VALIDATION = TRUE
  ENABLED = TRUE;
```

#### Kafka Connector with Multiple Partitions
```sql
-- Create Kafka connector with multiple partitions
CREATE KAFKA CONNECTOR MY_PARTITIONED_CONNECTOR
  KAFKA_BROKER = 'my-broker:9092'
  KAFKA_TOPIC = 'my-topic'
  KAFKA_PARTITIONS = (0, 1, 2, 3)
  TARGET_TABLE = 'MY_TABLE'
  KAFKA_POLL_INTERVAL_MS = 50
  KAFKA_BATCH_SIZE = 5000
  ENABLED = TRUE;
```

### Ingestion Service Setup
#### Basic Ingestion Stream
```sql
-- Create target table
CREATE TABLE MY_TABLE (
    id INTEGER,
    name STRING,
    event_time TIMESTAMP_LTZ
);

-- Create ingestion stream
CREATE INGESTION STREAM MY_STREAM
  TARGET_TABLE = 'MY_TABLE'
  BATCH_SIZE = 1000
  FLUSH_INTERVAL = 1;
```

#### Ingestion Stream with Transformation
```sql
-- Create ingestion stream with transformation
CREATE INGESTION STREAM MY_STREAM
  TARGET_TABLE = 'MY_TABLE'
  TRANSFORMATION = 'json'
  BATCH_SIZE = 5000
  COMPRESSION = 'GZIP'
  ON_ERROR = 'CONTINUE';
```

### External Table Setup
#### Basic External Table
```sql
-- Create external stage
CREATE STAGE MY_EXTERNAL_STAGE
  URL = 's3://my-bucket'
  CREDENTIALS = (AWS_KEY_ID = 'my-key' AWS_SECRET_KEY = 'my-secret');

-- Create file format
CREATE FILE FORMAT MY_PARQUET_FORMAT
  TYPE = 'PARQUET';

-- Create external table
CREATE EXTERNAL TABLE MY_EXTERNAL_TABLE (
    id INTEGER,
    name STRING,
    created_at TIMESTAMP_LTZ
)
WITH LOCATION = @MY_EXTERNAL_STAGE
FILE_FORMAT = (TYPE = 'PARQUET');
```

#### Partitioned External Table
```sql
-- Create partitioned external table
CREATE EXTERNAL TABLE MY_PARTITIONED_TABLE (
    id INTEGER,
    name STRING,
    created_at TIMESTAMP_LTZ,
    partition_date DATE
)
WITH LOCATION = @MY_EXTERNAL_STAGE
FILE_FORMAT = (TYPE = 'PARQUET')
PARTITION BY = (partition_date);
```

### Snowflake Connector (CDC) Setup
#### Basic CDC Connector
```sql
-- Create replication group
CREATE REPLICATION GROUP MY_REPLICATION
  SOURCE_DATABASE = 'my-postgres-db'
  SOURCE_SCHEMA = 'public'
  SOURCE_TABLE = 'my_table'
  TARGET_DATABASE = 'MY_DB'
  TARGET_SCHEMA = 'PUBLIC'
  TARGET_TABLE = 'MY_TABLE'
  POLLING_INTERVAL = 1
  ENABLED = TRUE;
```

#### CDC Connector with Multiple Tables
```sql
-- Create replication group with multiple tables
CREATE REPLICATION GROUP MY_MULTI_TABLE_REPLICATION
  SOURCE_DATABASE = 'my-postgres-db'
  SOURCE_SCHEMA = 'public'
  SOURCE_TABLES = ('table1', 'table2', 'table3')
  TARGET_DATABASE = 'MY_DB'
  TARGET_SCHEMA = 'PUBLIC'
  CDC_MODE = 'LOG_BASED'
  ENABLED = TRUE;
```

### Task Setup
#### Basic Task
```sql
-- Create stored procedure
CREATE PROCEDURE MY_PROCEDURE()
RETURNS STRING
LANGUAGE SQL
AS
$$
BEGIN
    COPY INTO MY_TABLE FROM @MY_STAGE;
    RETURN 'Loaded successfully';
END;
$$;

-- Create task
CREATE TASK MY_TASK
  WAREHOUSE = MY_WH
  SCHEDULE = 'USING CRON 0 * * * * America/Los_Angeles'
AS
  CALL MY_PROCEDURE();
```

#### Task with Retry Logic
```sql
-- Create task with retry logic
CREATE TASK MY_TASK
  WAREHOUSE = MY_WH
  SCHEDULE = 'USING CRON 0 * * * * America/Los_Angeles'
  RETRY_COUNT = 3
  RETRY_DELAY = 60
  ALLOW_OVERLAPPING_EXECUTION = FALSE
AS
  CALL MY_PROCEDURE();
```

#### Task with Dependency
```sql
-- Create dependent tasks
CREATE TASK LOAD_DATA_TASK
  WAREHOUSE = MY_WH
  SCHEDULE = 'USING CRON 0 2 * * * America/Los_Angeles'
AS
  COPY INTO MY_TABLE FROM @MY_STAGE;

CREATE TASK PROCESS_DATA_TASK
  WAREHOUSE = MY_WH
  SCHEDULE = 'USING CRON 0 3 * * * America/Los_Angeles'
  DEPENDENCY = LOAD_DATA_TASK
AS
  CALL PROCESS_DATA();
```

## Final Notes

For Further Reading:
- [Snowflake Snowpipe Documentation](https://docs.snowflake.com/en/user-guide/data-load-snowpipe)
- [Snowflake Kafka Connector Documentation](https://docs.snowflake.com/en/user-guide/kafka-connector)
- [Snowflake Ingestion Service Documentation](https://docs.snowflake.com/en/developer-guide/ingestion-service)
- [Snowflake External Tables Documentation](https://docs.snowflake.com/en/user-guide/external-tables)
- [Snowflake Connector Documentation](https://docs.snowflake.com/en/user-guide/connector)
- [Snowflake Tasks Documentation](https://docs.snowflake.com/en/user-guide/tasks)

Open Questions for Your Environment:
1. What are your **latency requirements** for data ingestion (real-time, near real-time, batch)?
2. What **data sources** are you ingesting from (cloud storage, Kafka, REST APIs, databases)?
3. What is your **expected data volume** (MB/sec, GB/day)?
4. Do you have **idempotency requirements** for your ingestion pipelines?
5. What are your **cost constraints** for ingestion (compute, storage, network)?
6. How do you **monitor and alert** on ingestion failures today?
7. What **security and compliance** requirements do you have (encryption, network, RBAC)?
