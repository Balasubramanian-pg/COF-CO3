# Database Storage Layer
We Will narrow down on Database storage layer in this section
<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/bf04354c-6d55-4788-8c38-191702d13dc3" />

## Definition and Purpose
  - The Database Storage Layer is the persistent, durable repository of all table data and metadata for user databases, schemas, and tables
  - Built on cloud object storage services such as Amazon S3, Azure Blob Storage, and Google Cloud Storage
  - Data is stored in a highly compressed, encrypted, columnar format optimized for fast analytic queries
  - Completely decoupled from compute;
      - Storage scales independently, automatically, and transparently,
      - With no capacity management required by users
  - Snowflake handles all aspects of data durability, availability, replication, and disaster recovery within this layer

## Core Storage Unit: Micro-Partitions
  - All table data is physically stored as **immutable micro-partitions** in cloud object storage
>[!Note]
>A micro-partition is a compressed, columnar file that typically contains 50–500 MB of uncompressed data (compressed size much smaller)
  - Each micro-partition stores a **_subset of a table’s rows across all columns_** (no row/column chasm; it’s a **hybrid columnar structure**)
  - Micro-partitions are automatically created when data is inserted or loaded; they are never modified in-place
  - Contains **self-contained metadata** like:
      - MIN/MAX value ranges for each column,
      - Number of distinct values,
      - Null counts, which is stored in the Cloud Services metadata store and used for **very fast partition pruning**
  - Immutability ensures data consistency, enables time travel, fail-safe, cloning, and sharing without data copying
  - Columnar storage within each micro-partition allows scanning only necessary columns, reducing I/O

## Data Storage Characteristics
  - Encryption: All data is automatically encrypted using AES-256 strong encryption for data at rest; keys are managed by Snowflake, with optional customer-managed keys (Tri-Secret Secure). Data in transit is encrypted via TLS
  - Compression: Automatically selects optimal compression algorithm per column based on data type and statistics; extremely high compression ratios significantly reduce storage footprint and cost
  - Resiliency: Data is replicated across multiple availability zones (and optionally across regions with database replication) within the cloud provider to ensure 99.999999999% durability (11 nines)
  - Storage is billed based on compressed size per month (average monthly storage used); users do not pre-provision capacity

## Separation of Storage and Compute
<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/25ebcb1f-af30-42b7-9895-977fa34aa5d8" />

>[!Tip]
>Compute (virtual warehouses) and storage are independent and communicate only via network; compute nodes fetch micro-partitions directly from cloud object storage
  - Multiple virtual warehouses can operate on the **same data simultaneously without contention**, each with its own cache
  - Storage **_persists indefinitely_** even when all warehouses are suspended;
>[!Important]
>Dropping a table or database is the only way to remove data (subject to time travel and fail-safe)
  - Because storage is decoupled, **workloads can be isolated** while accessing the same underlying data, enabling use cases like production, development, and QA environments sharing data via zero-copy cloning

## Time Travel and Data Retention
<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/63e993e3-5351-4d55-b555-cd7dea523d41" />

>[!Note]
>Time Travel allows querying and recovering data as it existed at any point within a configurable retention period (up to 90 days for Snowflake Enterprise Edition and higher)
  - Data changes (INSERTS, UPDATES, DELETES, TRUNCATES) result in new micro-partitions; old micro-partitions are **retained for the time travel window** before being purged
  - Retention period can be set
      - Per database,
      - Per schema, or
      - Per table; **a minimum of 1 day (0 for transient databases)**
  - Enables easy data recovery from accidental modifications, analysis of historical data, and **point-in-time cloning**

>[!Note]
>Fail-safe is an additional 7-day period of recoverability after Time Travel expires (not user-queryable, but Snowflake can recover data via support); provides a final safety net
  - The diagram below shows the data lifecycle with time travel and fail-safe

### Diagram: Data Lifecycle with Time Travel and Fail-safe
  ```mermaid
  gantt
  title Data Retention Lifecycle
  dateFormat YYYY-MM-DD
  axisFormat %d
  
  section Time Travel
  Point-in-time queries, clones, undrop :active, tt, 2026-01-01, 11d
  
  section Fail-safe
  Snowflake-only recovery, no user query :fs, after tt, 7d
  
  section Permanent Purge
  Data permanently deleted :done, purge, after fs, 1d
  ```
  Time Travel is user-accessible for the configured retention (e.g., 10 days). Fail-safe extends 7 more days, inaccessible to users. Past that, data is gone.

## Zero-Copy Cloning

>[!Note]
> Zero-Copy Cloning creates a new database, schema, or table that shares the **same underlying micro-partitions** as the source

  - Clone operations are purely metadata-only; no data is physically copied at creation time
  - New data (inserts/updates) to the cloned or source object results in new micro-partitions, while the shared micro-partitions remain intact for the unchanged portion
  - Enables instant, storage-efficient environment creation for development, testing, and data science
  - Works in conjunction with Time Travel: can clone an object as it existed at a past point in time

## Data Sharing
  - Secure data sharing allows live, read-only access to a shared database from another Snowflake account without physically copying data
  - The underlying micro-partitions remain in the provider’s storage; the consumer’s compute directly accesses them via the services layer authorization
  - No storage duplication means shares are always up-to-date and low cost

## Automatic Clustering and Optimization
  - Clustering Keys can be defined on a table to co-locate related rows in the same micro-partitions, improving partition pruning
  - Without a clustering key, data is naturally ordered by ingestion time; Snowflake may still perform well due to automatic micro-partition metadata pruning, but large tables with heavy filter-based queries benefit from clustering
  - Automatic Clustering service (serverless) re-organizes micro-partitions in the background to maintain clustering ratio and order, ensuring consistent query performance even with frequent DML
  - Background compaction coalesces small micro-partitions into optimal larger ones, reducing the number of files scanned and metadata overhead
  - Both are fully managed and operate transparently

## Storage Lifecycle and Operations
  - Ingestion: Data loaded via COPY INTO, Snowpipe, streams, etc., is automatically converted into micro-partitions and stored in cloud storage, with metadata registered in Cloud Services
  - Deletion: DML operations (DELETE, UPDATE, TRUNCATE, DROP) retain old micro-partitions for the time travel period, then they are automatically garbage collected
  - Storage metrics: Total and average compressed bytes are tracked; visible via ACCOUNT_USAGE views and billing
  - Storage access: Data is never directly accessed by users via file system; all reads and writes go through Snowflake SQL or Snowpipe REST API

## Interaction with Other Layers
  - Cloud Services Layer: Maintains the metadata catalog that maps tables and partitions to physical micro-partition files in cloud storage; holds clustering metadata, statistical information, and version history that enables partition pruning
  - Compute Layer (Virtual Warehouses): Fetches micro-partitions directly from object storage using pre-signed URLs; caches them on local SSDs; never writes permanent data to the storage layer except via DML operations coordinated by Cloud Services
  - Snowflake’s architecture ensures that the storage layer is passive: it just hosts files; the intelligence for data layout, optimization, and access control resides in the services layer, while execution lives in compute

### Diagram: Micro-Partition Structure and Columnar Access
  ```mermaid
  graph TD
  subgraph MP[Micro-Partition: Sales Table]
    direction TB
    col1[Column: Date<br/>min: 2025-01-01 <br/>max: 2025-01-31]
    col2[Column: Region<br/>values: US, EU, APAC]
    col3[Column: Amount<br/>range: 10 - 5000]
    colN[Column: N <br/> ...]
  end

  Query["SELECT SUM(Amount)<br/>WHERE Date = '2025-01-15' AND Region = 'EU'"]

  Client --> Query
  Query --> Metadata[(Metadata: Micro-partition ranges)]
  Metadata -- Prune irrelevant partitions --> MP
  MP -- Read only Amount and Region columns<br/>from relevant partitions --> Warehouse[Virtual Warehouse]
  ```
  Metadata-driven pruning eliminates entire micro-partitions before data is read. Columnar layout further limits I/O to columns referenced in the query.

## Storage Durability and Disaster Recovery
  - Data is synchronously replicated within a cloud region to multiple fault domains; durability exceeds 99.999999999%
  - Database Replication and Failover/Failback allow replicating databases across different regions and cloud providers for disaster recovery and business continuity
  - Replication copies the underlying micro-partitions and metadata continuously; the standby copy can be promoted to serve queries when the primary region fails
  - Storage in Snowflake is an integral part of the cross-region, cross-cloud architecture

## Summary
  - The Database Storage Layer provides secure, durable, and highly elastic data persistence using cloud object storage and a proprietary micro-partition format
  - Its immutable, columnar nature supports high-performance analytics, time travel, zero-copy cloning, and data sharing without physical data movement
  - Decoupling storage from compute allows independent scaling, cost control, and workload isolation while maintaining a single source of truth for all data in the Snowflake ecosystem
