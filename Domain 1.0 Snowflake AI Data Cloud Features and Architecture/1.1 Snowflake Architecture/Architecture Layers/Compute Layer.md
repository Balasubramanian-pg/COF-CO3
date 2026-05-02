# Compute Layer

<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/395ffe55-795d-4ed2-a548-8bd13fa1c88f" />

## Definition and Purpose
  - The Compute Layer consists of one or more Virtual Warehouses that provide the computational resources to execute queries, data loading, and DML operations
  - Virtual Warehouses are essentially clusters of compute nodes assigned to an individual Snowflake account isolated from all other accounts
  - Abstracts underlying cloud compute instances (e.g., AWS EC2, Azure VMs, GCP VMs) into a managed, elastic MPP (Massively Parallel Processing) engine
  - Users never directly manage servers, operating systems, or software updates; Snowflake handles all infrastructure provisioning and patching

## Core Components and Concepts
  - Virtual Warehouse: A named, provisioned cluster of identical compute nodes, each with a portion of CPU, memory, and local SSD storage
  - Warehouse Size: Dictates the number of nodes per cluster; available in t-shirt sizes from X-Small to 6X-Large
  - Node: Each node is an independent worker that processes a subset of data in parallel; nodes in a warehouse operate on shared-nothing principles
  - Warehouse State: Can be Started (running), Suspended (stopped, not consuming credits), or Resuming (transitioning from suspended to running)
  - Multi-cluster Warehouse: Allows a single warehouse name to manage multiple independent clusters of the same size, providing scale-out concurrency

## Virtual Warehouse Sizes and Credit Consumption
<img width="1664" height="928" alt="image" src="https://github.com/user-attachments/assets/9e195c55-6823-478d-b17c-33f6b88972b3" />

  - Size determines compute resources and credits consumed per full hour of runtime (per-second billing with 60-second minimum after initial start)
  - Standard sizes and relative node counts:
    - X-Small: 1 node, 1 credit per hour (base)
    - Small: 2 nodes, 2 credits per hour
    - Medium: 4 nodes, 4 credits per hour
    - Large: 8 nodes, 8 credits per hour
    - X-Large: 16 nodes, 16 credits per hour
    - 2X-Large: 32 nodes, 32 credits per hour
    - 3X-Large: 64 nodes, 64 credits per hour
    - 4X-Large: 128 nodes, 128 credits per hour
    - 5X-Large: 256 nodes, 256 credits per hour
    - 6X-Large: 512 nodes, 512 credits per hour
  - Doubling the warehouse size halves query execution time for perfectly parallelizable workloads, but cost remains roughly the same because credits are consumed at double the rate for half the time
<img width="1664" height="928" alt="image" src="https://github.com/user-attachments/assets/0091e08e-c2ac-47b6-9cb8-6007f366b39b" />

## Multi-cluster Warehouses and Concurrency
<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/7157da24-b28c-4131-ad9c-a3f6bda01704" />

  - Configured in MAX_CLUSTERS mode,
      - A warehouse can **automatically spawn** additional clusters (each of the defined size) when the number of queued queries _exceeds thresholds_
  - MIN_CLUSTERS and MAX_CLUSTERS define the scaling range; when demand drops, clusters are automatically shut down

>[!tip]
>Each cluster is an **independent**, **identical set of nodes** providing computational isolation; multiple queries run concurrently on different clusters without resource contention
  - Ideal for unpredictable, concurrent user-facing analytics where hundreds of simultaneous queries may arrive
  - Multi-cluster warehouses use the same warehouse name and are billed as separate clusters for the time each runs

## Auto-suspend and Auto-resume

>[!Note] 
>**Auto-suspend**: Automatically shuts down a virtual warehouse after a user-defined inactivity period (default 10 minutes, can be as low as 1 minute or disabled)
  - When suspended, all compute resources are terminated and no credits are consumed, though cache data on local SSDs is discarded

>[!Note]
>**Auto-resume**: Automatically restarts the warehouse when a new query is submitted; takes a few seconds for the cluster to provision and warm up
  - Enables cost optimization by eliminating idle compute time, a core tenet of the platform’s separation of compute and storage

## Scaling Strategies
<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/854487c3-4bcc-4947-8f9e-7bb111ec5ba5" />

  - Scale-Up (Sizing): Increase the warehouse size to add more nodes to a single cluster, giving each query more compute power and reducing execution time for large, complex queries
  - Scale-Out (Multi-cluster): Increase MAX_CLUSTERS to handle more concurrent queries without impacting individual query performance; clusters are independent so no inter-query interference
  - Not mutually exclusive: can have a Large multi-cluster warehouse with up to, say, 3 clusters to speed up heavy ETL jobs and still absorb concurrency spikes
  - Resource monitors can be set to track credit usage and automatically suspend or notify on warehouse scaling events

## Caching in the Compute Layer
  - Local Disk Cache (Warehouse Cache): Each warehouse node maintains a local SSD cache of micro-partition data from cloud storage
  - When a query scans a table, nodes fetch micro-partitions from cloud storage; once cached, subsequent queries hitting the same warehouse can read from very fast local SSD
  - Cache is invalidated when the underlying storage data changes (due to table updates) within a few seconds; warehouse suspension clears the entire local cache
  - Result Cache (actually in Cloud Services, but interacts with compute): Warehouse results may be cached globally and served without re-execution; compute layer only invoked on cache miss
  - Understanding cache behavior is crucial for performance tuning and cost control; often one would run a "warm-up" query on a warehouse before benchmarking

## Resource Isolation and Concurrency
<img width="1664" height="928" alt="image" src="https://github.com/user-attachments/assets/5328f6c2-6f93-4e87-bf33-c4b180907157" />

  - Each virtual warehouse is an isolated compute pool; queries in one warehouse never impact the performance of queries in another
  - Within a single warehouse, queries are queued if all available execution slots are occupied; maximum queries per warehouse depends on size and complexity, but no strict concurrency limit is enforced—queuing happens based on resource availability
  - For mixed workloads, best practice is to separate ETL/ELT and BI/reporting into different warehouses to prevent resource contention
  - Snowflake leverages the concept of “workload isolation” as a first-class design principle, not an afterthought

## Serverless Compute
<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/5301fb75-4b4d-48e0-8f1e-fb603786260f" />

  - Some features provision compute transparently without a user-defined virtual warehouse: serverless tasks, Snowpipe, automatic clustering, materialized view maintenance, search optimization service, etc.
  - These use Cloud Services compute resources that are billed as Serverless Credits separate from virtual warehouse credits
  - Serverless compute scales elastically and is fully managed; users do not need to configure warehouse size or concurrency
  - Allows running background maintenance tasks without dedicating a virtual warehouse, simplifying operations

## Interaction with Other Layers
<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/f4f32a7b-6b1c-417a-a172-7532f25cb6f7" />

  - Cloud Services Layer: Sends execution plans to the virtual warehouse; warehouse nodes execute the plan and return results; all metadata lookups are handled by Cloud Services, never by the warehouse
  - Database Storage Layer: Warehouse nodes read micro-partitions directly from cloud object storage; they never write permanent data to local disk—all persistent data is stored remotely; locally cached data is volatile and complementary
  - Query Lifecycle within Compute:
    1. Cloud Services dispatches a compiled query plan to the warehouse
    2. Each node scans assigned micro-partitions (pruned by metadata), retrieves data from cache or remote storage, and executes filters, joins, aggregations in parallel
    3. Intermediate results are shuffled between nodes when necessary (e.g., large joins)
    4. Final results are sent back to Cloud Services for aggregation and return to the client
  - The compute layer is purely execution; it retains no permanent metadata or persistent table state

### Diagram: Virtual Warehouse Execution Architecture
  ```mermaid
  graph TD
    CS[Cloud Services Layer]

    subgraph VW[Virtual Warehouse Cluster]
        Node1[Node 1 <br/>CPU / Mem / SSD Cache]
        Node2[Node 2 <br/>CPU / Mem / SSD Cache]
        Node3[Node N <br/>CPU / Mem / SSD Cache]
    end

    Storage[(Cloud Object Storage)]

    CS -- Dispatch Query Plans --> VW
    Node1 <-- Data I/O --> Storage
    Node2 <-- Data I/O --> Storage
    Node3 <-- Data I/O --> Storage
    Node1 <-- Data Shuffle --> Node2
    Node2 <-- Data Shuffle --> Node3
    Node1 <-- Data Shuffle --> Node3
    VW -- Final Results --> CS
  ```
  Each warehouse node reads directly from cloud storage into local SSD cache. Intermediate data shuffling happens over high-bandwidth internal network.

### Diagram: Multi-cluster Warehouse Scaling
  ```mermaid
  graph LR
  Client[Client Queries] --> LB[Logical Warehouse Name]
  
  subgraph Cluster1["Cluster 1 (active)"]
    N1[Node1] --- N2[Node2] --- N3[...]
  end
  
  subgraph Cluster2["Cluster 2 (auto-scaled)"]
    C1[Node1] --- C2[Node2] --- C3[...]
  end
  
  LB --> Cluster1
  LB --> Cluster2
  Cluster1 --> Storage[(Cloud Storage)]
  Cluster2 --> Storage
  ```
  When queuing exceeds threshold, additional clusters are provisioned automatically. Each cluster operates independently, reading from the same shared storage.

## Metering and Considerations
<img width="1664" height="928" alt="image" src="https://github.com/user-attachments/assets/5f328b76-ad0c-4058-922c-b25fe189ee49" />

  - Virtual warehouses consume Snowflake credits based on size and uptime;
  - Billing is per-second with a 60-second minimum upon start
  - Suspended warehouses incur **zero compute costs**
  - Storage costs are separate and constant
  - Multi-cluster warehouses bill for each running cluster independently
  - Query complexity, data volume, and cache hit ratio directly influence credit consumption
  - Careless warehouse sizing leads to cost overruns
  - Resource monitors can enforce credit limits per warehouse or account, suspending them to stop further charges

## Summary
  - The Compute Layer is the elastic, scalable execution engine that processes user queries and data operations, embodied by Virtual Warehouses
  - It provides complete workload isolation, on-demand scaling (up and out), and transparent caching for high performance
  - By separating compute from storage and making compute ephemeral, Snowflake achieves cost efficiency and near-infinite elasticity without manual infrastructure management
