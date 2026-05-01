# Snowflake Storage Fundamentals

```mermaid
graph TD
  Storage[Storage in Snowflake] --> MP[Micro Partitions]
  Storage --> Comp[Compression]
  Storage --> Clust[Clustering]
  Storage --> TT[Time Travel]
  Storage --> FS[Fail Safe]
  Storage --> Stages[Internal and External Stages]
```

| Concept | What It Is | Why It Matters |
|---------|-----------|----------------|
| Micro partitions | Small immutable blocks of data 50 to 500 MB uncompressed | Enables fast pruning, efficient updates, and time travel |
| Columnar storage | Data stored by column not by row | Speeds up analytical queries that read few columns |
| Automatic compression | Snowflake compresses data on ingest using best algorithm per column | Reduces storage cost and I/O without user action |
| Clustering keys | Metadata that tracks value ranges per micro partition | Helps Snowflake skip irrelevant data during queries |
| Time Travel | Access to historical data for a configurable window | Recover from mistakes, audit changes, reproduce results |
| Fail safe | Protected recovery period after Time Travel ends | Emergency only recovery for compliance and disaster scenarios |
| Internal stage | Temporary storage area inside your Snowflake account | Hold files during load or unload operations |
| External stage | Reference to cloud storage like S3 Azure Blob or GCS | Query data where it lives without moving it |

```mermaid
flowchart LR
  A[Data loaded into Snowflake] --> B[Converted to columnar format]
  B --> C[Compressed with best algorithm per column]
  C --> D[Split into micro partitions]
  D --> E[Metadata indexed for pruning]
  E --> F[Ready for fast querying]
```

## Micro Partitions Deep Dive

| Property | Detail |
|----------|--------|
| Size range | 50 to 500 MB uncompressed per micro partition |
| Immutability | Once written micro partitions never change |
| Update behavior | Updates create new micro partitions old ones marked for cleanup |
| Pruning | Snowflake skips micro partitions that cannot match query filters |
| Metadata tracked | Min max values null count distinct count per column per micro partition |

```mermaid
graph TD
  Table[Logical Table] --> MP1[Micro Partition 1]
  Table --> MP2[Micro Partition 2]
  Table --> MP3[Micro Partition 3]
  Table --> MPn[Micro Partition N]
  
  MP1 --> Col1[Column A values]
  MP1 --> Col2[Column B values]
  MP1 --> Meta1[Metadata min max nulls]
```

- Micro partitions are created automatically during data load or DML
- You do not manage them directly. Snowflake handles creation merging and cleanup
- Queries use metadata to skip micro partitions that cannot contain matching rows
- This pruning is why Snowflake can scan petabytes but read only gigabytes
- Clustering keys improve pruning efficiency for filtered queries

## Compression and Columnar Storage

| Feature | How It Works | Benefit |
|---------|-------------|---------|
| Columnar format | Each column stored separately | Read only columns you need skip the rest |
| Per column compression | Snowflake picks best algorithm per column type | Higher compression ratios than one size fits all |
| Dictionary encoding | Repeated values stored as references | Saves space for low cardinality columns like status or region |
| Run length encoding | Consecutive identical values stored once | Efficient for sorted or time series data |
| Automatic application | No user config needed | Works out of the box with no tuning |

```mermaid
flowchart LR
  Raw[Raw Row Data] --> Col[Split into Columns]
  Col --> Comp[Compress Each Column]
  Comp --> Store[Store in Micro Partitions]
  Store --> Query[Query Reads Only Needed Columns]
```

- Compression ratios typically 3x to 10x depending on data type and patterns
- You are billed for compressed storage size not raw size
- Columnar format enables vectorized execution for faster analytics
- No need to create indexes. Micro partition metadata serves that purpose

## Clustering and Pruning

```mermaid
graph TD
  Q[Query with filter] --> M[Check micro partition metadata]
  M --> P[Prune partitions outside filter range]
  P --> R[Read only relevant partitions]
  R --> Res[Return results faster]
```

| Term | Meaning |
|------|---------|
| Clustering key | One or more columns used to co locate related data in micro partitions |
| Clustering depth | Measure of how well data is organized by clustering key lower is better |
| Automatic clustering | Background process that reorganizes data to maintain clustering quality |
| Pruning | Skipping micro partitions that cannot match query filters based on metadata |

| When clustering helps | When it does not help |
|----------------------|----------------------|
| Frequent filters on same column | Queries always scan full table |
| Large tables with selective filters | Small tables under 10 GB |
| Time series data filtered by date | Random access patterns with no filter pattern |
| Join keys used in frequent joins | Columns with high cardinality and no filter selectivity |

```mermaid
flowchart TD
  Q1[Start: Do you filter on same columns often]
  Q1 -->|Yes| Q2[Is table larger than 10 GB]
  Q1 -->|No| A[No clustering needed]
  
  Q2 -->|Yes| B[Define clustering key on filter columns]
  Q2 -->|No| A
  
  B --> C[Monitor clustering depth]
  C --> D[Enable automatic clustering if depth grows]
  D --> E[Review cost vs performance monthly]
```

## Time Travel and Fail Safe

| Feature | Standard Edition | Enterprise and Above |
|---------|-----------------|---------------------|
| Time Travel window | 1 day fixed | 1 to 90 days configurable |
| Fail safe period | 7 days after Time Travel ends | 7 days after Time Travel ends |
| Access method | SELECT with AT or BEFORE clause | Same plus programmatic access via API |
| Use cases | Undo accidental deletes or updates | Compliance auditing point in time analysis |
| Billing | Storage for historical micro partitions | Storage for historical micro partitions |

```mermaid
graph TD
  Now[Current Data] --> TT[Time Travel Window]
  TT --> FS[Fail Safe Window]
  FS --> Perm[Permanently Removed]
  
  Now --> Q1[Query current state]
  TT --> Q2[Query historical state with AT BEFORE]
  FS --> Q3[Emergency recovery only via Snowflake Support]
```

| Action | Time Travel Syntax Example |
|--------|---------------------------|
| Query table as of timestamp | SELECT * FROM my_table AT TIMESTAMP TO_TIMESTAMP_TZ 2024 01 15 10 00 00 UTC |
| Query table as of offset | SELECT * FROM my_table BEFORE OFFSET 60 MINUTE |
| Query table as of statement | SELECT * FROM my_table AT STATEMENT 4f2a1b3c 0000 1234 abcd 5678ef901234 |
| Undelete table | UNDELETE TABLE my_table WITH TIMESTAMP TO_TIMESTAMP_TZ 2024 01 15 10 00 00 UTC |

- Time Travel stores historical micro partitions separately from current data
- You pay storage costs for historical data during the Time Travel window
- Fail safe data is not queryable. It is for emergency recovery only via Snowflake Support
- Setting Time Travel to 0 days disables historical access but does not reduce storage for current data
- Use shorter Time Travel windows for dev and test environments to control cost

## Internal and External Stages

| Stage Type | Location | Best For | Access Pattern |
|------------|----------|----------|---------------|
| Internal user stage | Inside your Snowflake account tied to your user | Quick ad hoc file loads and unloads | PUT and GET commands from client |
| Internal table stage | Inside your Snowflake account tied to a specific table | Loading data into one table repeatedly | COPY INTO table FROM stage |
| Internal named stage | Inside your Snowflake account with custom name | Shared file storage across tables or users | GRANT usage to roles for collaboration |
| External stage | Reference to cloud storage S3 Azure Blob GCS | Querying data where it lives or loading from external pipelines | Define with URL and credentials or IAM role |

```mermaid
flowchart LR
  Ext[External Cloud Storage] --> Stage[External Stage Definition]
  Stage --> Copy[COPY INTO Table]
  Copy --> Snowflake[Snowflake Table]
  
  User[User Machine] --> Put[PUT Command]
  Put --> IntStage[Internal Stage]
  IntStage --> Copy2[COPY INTO Table]
  Copy2 --> Snowflake
```

| Command | Purpose | Example |
|---------|---------|---------|
| PUT | Upload file from local machine to internal stage | PUT file data.csv @my_stage |
| GET | Download file from internal stage to local machine | GET @my_stage data.csv file tmp |
| LIST | Show files in a stage | LIST @my_stage |
| REMOVE | Delete files from internal stage | REMOVE @my_stage old_file.csv |
| COPY INTO | Load data from stage into table | COPY INTO my_table FROM @my_stage FILE_FORMAT CSV |

- Internal stages are managed by Snowflake. You do not see the underlying cloud storage
- External stages require network connectivity and proper IAM or credential configuration
- Files in internal stages are encrypted and compressed automatically
- External stages support file format inference and schema detection for semi structured data
- Use named stages for reusable file storage across multiple loads or teams

## Storage Billing and Cost Control

| Billing Component | How It Is Measured | Typical Rate US East AWS |
|------------------|-------------------|-------------------------|
| Compressed storage | Average bytes stored per day including micro partitions and historical data | 23 USD per TB per month |
| Time Travel storage | Historical micro partitions retained within configured window | Same rate as compressed storage |
| Fail safe storage | Protected recovery data after Time Travel ends | Same rate as compressed storage |
| Data transfer | Bytes moved out of Snowflake to external destinations | Varies by cloud provider and region |

```mermaid
graph LR
  Total[Total Storage Cost] --> Current[Current Data]
  Total --> TTStore[Time Travel Data]
  Total --> FSStore[Fail Safe Data]
  
  Current --> Opt1[Drop unused tables]
  TTStore --> Opt2[Shorten Time Travel for dev]
  FSStore --> Opt3[Cannot reduce fixed 7 days]
```

| Practice | How To Implement | Expected Impact |
|----------|-----------------|-----------------|
| Drop or truncate unused tables | Identify with ACCOUNT_USAGE views | Immediate storage savings |
| Shorten Time Travel for dev test | ALTER DATABASE SET DATA_RETENTION_TIME_IN_DAYS 1 | Reduce historical storage by up to 90 percent |
| Use transient tables for temporary data | CREATE TRANSIENT TABLE instead of TABLE | No Time Travel or Fail safe storage costs |
| Archive old data to external stage | COPY to S3 then DROP from Snowflake | Move cold data to cheaper cloud storage |
| Monitor storage growth weekly | Query STORAGE_METERING_HISTORY views | Catch surprises before they impact budget |

```mermaid
flowchart TD
  Q1[Start: Review storage usage]
  Q1 --> Q2[Which databases are largest]
  Q1 --> Q3[Which tables have longest Time Travel]
  Q1 --> Q4[Are there unused or duplicate tables]
  
  Q2 --> A[Archive or drop if not needed]
  Q3 --> B[Shorten retention for non prod]
  Q4 --> C[Consolidate or remove duplicates]
  
  A --> D[Re measure after 1 week]
  B --> D
  C --> D
```

## Data Organization Best Practices

| Practice | Why It Matters | How To Apply |
|----------|---------------|--------------|
| Use databases to separate business domains | Clear ownership and access control | Create FINANCE_DB MARKETING_DB OPERATIONS_DB |
| Use schemas to separate data lifecycle stages | Organize raw cleaned and curated data | Use RAW CLEANED REPORTING schemas inside each database |
| Name tables with purpose and freshness | Avoid confusion about data source and recency | Use naming like events_raw_daily user_facts_hourly |
| Use transient tables for intermediate results | Avoid paying for Time Travel on temporary data | CREATE TRANSIENT TABLE for ETL staging |
| Document table purpose and retention | Helps teammates understand when to archive or drop | Add table comment with owner and expected lifespan |

```mermaid
graph TD
  Org[Data Organization] --> DB[Database by Business Domain]
  DB --> SC[Schema by Data Stage]
  SC --> TBL[Table by Entity and Freshness]
  TBL --> TAG[Tags for Ownership and Retention]
```

## Common Storage Mistakes and Fixes

| Mistake | What Happens | How To Fix |
|---------|--------------|------------|
| Leaving 90 day Time Travel on dev databases | Paying for 90 days of history on data that changes daily | Set DATA_RETENTION_TIME_IN_DAYS to 1 for non prod |
| Using permanent tables for temporary ETL steps | Accumulating Time Travel storage for data you never query | Use transient tables or drop immediately after use |
| Never reviewing table sizes | Large unused tables keep consuming storage | Query ACCOUNT_USAGE.TABLE_STORAGE_METRICS monthly |
| Loading duplicate files to internal stages | Wasting storage on redundant raw files | Implement deduplication logic before load or use external stage |
| Ignoring clustering depth on large filtered tables | Queries scan more micro partitions than needed | Define clustering key and enable automatic clustering |
| Assuming compression is free | Compressed storage still costs money | Archive cold data to external cloud storage for lower cost |

```mermaid
flowchart TD
  Prob[Storage issue] --> Q1[Cost higher than expected]
  Prob --> Q2[Queries slower than expected]
  Prob --> Q3[Running out of quota]
  
  Q1 --> A[Review Time Travel settings and unused tables]
  Q2 --> B[Check clustering and pruning efficiency]
  Q3 --> C[Archive cold data or request quota increase]
  
  A --> Fix1[Shorten retention drop unused consolidate duplicates]
  B --> Fix2[Add clustering key enable auto clustering]
  C --> Fix3[Move historical data to external stage]
```

## Quick Reference Decision Flow

```mermaid
flowchart TD
  Q1[Start: What are you storing]
  Q1 --> Q2[Raw ingested files]
  Q1 --> Q3[Cleaned analytical tables]
  Q1 --> Q4[Temporary ETL results]
  Q1 --> Q5[Historical audit data]
  
  Q2 --> A[Load to RAW schema use internal stage for staging]
  Q3 --> B[Store in CLEANED schema set appropriate Time Travel]
  Q4 --> C[Use transient tables drop after downstream load]
  Q5 --> D[Store in AUDIT schema with extended Time Travel]
  
  A --> E[Monitor size and archive old raw files]
  B --> F[Add clustering keys if filtered often]
  C --> G[Ensure no Time Travel costs apply]
  D --> H[Review retention policy quarterly]
```

## Key Principles

- Storage is billed for compressed data not raw size. Compression is automatic and free
- Time Travel and Fail safe add storage costs. Configure retention based on actual need
- Micro partitions and pruning make large tables fast. You do not need to manage indexes
- Clustering improves query performance for filtered workloads. Monitor depth to justify cost
- Separate workloads by database and schema. Makes access control and cost attribution clearer
- Use transient tables for temporary data. Avoid paying for history you never query
- Review storage usage monthly. Workloads change and your storage strategy should too
- Archive cold data to external storage. Keep hot data in Snowflake for performance

## Bottom Line

- Snowflake storage is automatic columnar compressed and partitioned
- You control cost through retention settings table lifecycle and data organization
- Micro partitions and metadata enable fast queries without manual indexing
- Time Travel is powerful but costs money. Set it intentionally per environment
- Stages are your bridge to files. Use internal for quick loads external for integration
- Measure before you optimize. One week of storage metrics beats guessing
- Document your data organization. Future you and your teammates need context

Think of Snowflake storage like a smart warehouse:
- Items arrive and are automatically sorted labeled and compressed
- You can ask for any item from any point in the recent past
- The warehouse skips aisles it knows do not contain what you need
- You pay for the space your items actually occupy not the raw size they arrived in
- Temporary items can be marked for quick disposal to save space

Use the warehouse wisely. Keep what you need. Archive what you do not. Organize so you can find things fast. Save space without losing what matters.
