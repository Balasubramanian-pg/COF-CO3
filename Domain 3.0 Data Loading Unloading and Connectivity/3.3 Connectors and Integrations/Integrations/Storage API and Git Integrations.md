# **Snowflake Storage API and Git Integrations: Production-Grade Technical Deep Dive**

---

## **1. Overview of Storage API and Git Integrations**

Snowflake's **Storage API** and **Git integrations** enable **programmatic access to cloud storage** and **version control for Snowflake objects**, respectively. These integrations are critical for **automating data pipelines**, **managing code and configurations**, and **enabling CI/CD workflows** in modern data environments.

---

### **Mermaid: Storage API and Git Integrations Ecosystem**
```mermaid
%% Storage API and Git Integrations Ecosystem
flowchart TD
    subgraph StorageAPI["Storage API Integrations"]
        A[("Snowflake Storage API\n(Preview)")] -->|Programmatic Access| B[("Cloud Storage\n(S3/Azure/GCS)")]
        C[("PUT/GET Commands")] -->|File Transfer| B
        D[("External Stage API")] -->|Stage Management| B
        E[("Cloud Storage SDKs\n(AWS/Azure/GCP)")] -->|Native Access| B
    end

    subgraph GitIntegrations["Git Integrations"]
        F[("Snowsight Git\n(Native)")] -->|Version Control| G[("GitHub/GitLab/Bitbucket")]
        H[("CI/CD Pipelines\n(GitHub Actions/GitLab CI)")] -->|Automation| G
        I[("Snowflake CLI + Git")] -->|Scripting| G
        J[("Terraform + Git")] -->|IaC| G
    end

    subgraph Snowflake["Snowflake"]
        A --> K[("Storage Service")]
        C --> K
        D --> K
        F --> L[("Metadata Service")]
        H --> L
    end

    subgraph Workflows["Common Workflows"]
        M[("Data Pipeline Automation")]
        N[("Infrastructure as Code")]
        O[("Code Versioning")]
        P[("CI/CD Deployment")]
    end
    A --> M
    C --> M
    E --> M
    F --> O
    H --> N
    H --> P
    I --> N
    J --> N

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef storage fill:#4285f4,stroke:#1976d2;
    classDef git fill:#f4511e,stroke:#d84315;
    classDef snowflake fill:#29abe2,stroke:#1a8fb8;
    classDef workflows fill:#009688,stroke:#00796b;
    class A,C,D,E storage;
    class F,G,H,I,J git;
    class K,L snowflake;
    class M,N,O,P workflows;
```


### **Storage API vs. Git Integrations Comparison Table**

| **Feature**               | **Storage API**                          | **Git Integrations**                     |
|---------------------------|------------------------------------------|------------------------------------------|
| **Purpose**               | Programmatic access to cloud storage    | Version control and CI/CD for Snowflake objects |
| **Primary Use Case**      | Data loading/unloading, file management | Code versioning, collaboration, automation |
| **Data Flow**             | Bidirectional (read/write)              | Unidirectional (push/pull)              |
| **Latency**               | 100-500ms (depends on operation)         | 1-10 sec (depends on Git provider)       |
| **Throughput**            | 10-1000 MB/min (depends on cloud storage) | N/A (file-based)                        |
| **Serverless**            | ✅ Yes (Storage API)                     | ❌ No (Git operations)                   |
| **Managed By**            | Snowflake/Cloud Provider                 | Git Provider (GitHub, GitLab, etc.)      |
| **Cost Model**            | Cloud storage costs + API calls          | Git provider costs (free for public repos) |
| **Best For**              | Automated data pipelines, programmatic file access | Code management, CI/CD, collaboration |
| **Security**              | IAM, CMK, PrivateLink                    | SSH, HTTPS, OAuth, PATs                  |
| **Authentication**        | JWT, OAuth, Cloud Credentials             | SSH keys, HTTPS, OAuth tokens            |



## **2. Storage API Integrations Deep Dive**

Storage API integrations enable **programmatic access** to Snowflake's **cloud storage** (stages) and **external cloud storage** (S3, Azure Blob, GCS). These APIs allow you to **automate data loading/unloading**, **manage files**, and **integrate Snowflake with external systems**.


### **A. Snowflake Storage API (Preview)**

#### **1. Definition and Architecture**
The **Snowflake Storage API** (currently in **private preview**) provides a **RESTful API** for **programmatic access** to Snowflake's **internal and external stages**. It allows you to **list, upload, download, and delete files** directly via HTTP requests, without using SQL commands like `PUT` or `GET`.

```mermaid
%% Snowflake Storage API Architecture
flowchart TD
    subgraph Client["Client Application"]
        A[("HTTP Client\n(Python, cURL, etc.)")] -->|REST API| B[("Storage API Gateway")]
    end

    subgraph Snowflake["Snowflake"]
        B -->|AuthN/AuthZ| C[("Authentication Service")]
        C -->|Validate| D[("Metadata Service")]
        D -->|Route| E[("Storage Service")]
        E -->|Internal Stages| F[("Snowflake Blob Storage")]
        E -->|External Stages| G[("Cloud Storage\n(S3/Azure/GCS)")]
    end

    subgraph CloudStorage["Cloud Storage"]
        H[("S3")] --> G
        I[("Azure Blob")] --> G
        J[("GCS")] --> G
    end

    subgraph Operations["Supported Operations"]
        K[("List Files")]
        L[("Upload File")]
        M[("Download File")]
        N[("Delete File")]
        O[("Get File Metadata")]
        P[("Generate Presigned URL")]
    end
    B --> K
    B --> L
    B --> M
    B --> N
    B --> O
    B --> P

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef client fill:#4285f4,stroke:#1976d2;
    classDef snowflake fill:#29abe2,stroke:#1a8fb8;
    classDef cloud fill:#009688,stroke:#00796b;
    classDef operations fill:#ff9800,stroke:#f57c00;
    class A client;
    class B,C,D,E,F,G snowflake;
    class H,I,J cloud;
    class K,L,M,N,O,P operations;
```

#### **2. How It Works**
1. **Authentication**:
   - Clients authenticate using **JWT tokens** (for service accounts) or **OAuth 2.0** (for users).
   - Tokens are obtained via Snowflake's **authentication endpoints** or **key pair authentication**.

2. **Request Routing**:
   - Requests are sent to the **Storage API Gateway** (e.g., `https://{account}.snowflakecomputing.com/api/v2/storage`).
   - The gateway **validates the token** and **checks permissions** (RBAC).

3. **Operation Execution**:
   - The gateway **routes requests** to the appropriate **storage service**:
     - **Internal Stages**: Snowflake's **Blob Storage**.
     - **External Stages**: **Cloud storage** (S3, Azure Blob, GCS).
   - The storage service **executes the operation** (list, upload, download, delete, etc.).

4. **Response**:
   - The API returns **JSON responses** with operation status and metadata.
   - For file operations, it may return **presigned URLs** for direct cloud storage access.

#### **3. Key Endpoints**
| **Endpoint** | **Method** | **Description** | **Parameters** | **Response** |
|--------------|------------|-----------------|----------------|--------------|
| `/api/v2/storage/stages` | GET | List all stages | `type` (INTERNAL/EXTERNAL) | Array of stage objects |
| `/api/v2/storage/stages/{stage_name}` | GET | Get stage details | None | Stage object |
| `/api/v2/storage/stages/{stage_name}/files` | GET | List files in stage | `prefix`, `limit`, `token` (pagination) | Array of file objects |
| `/api/v2/storage/stages/{stage_name}/files/{file_path}` | GET | Get file metadata | None | File metadata |
| `/api/v2/storage/stages/{stage_name}/files/{file_path}` | PUT | Upload file | `overwrite` (boolean) | File metadata |
| `/api/v2/storage/stages/{stage_name}/files/{file_path}` | DELETE | Delete file | None | Success/failure |
| `/api/v2/storage/stages/{stage_name}/files/{file_path}/download` | GET | Generate presigned URL | `expiry` (seconds) | Presigned URL |
| `/api/v2/storage/stages/{stage_name}/files/{file_path}/upload` | POST | Generate presigned URL for upload | `expiry` (seconds) | Presigned URL |

#### **4. When to Use**
✅ **Automated data pipelines** (upload/download files programmatically)
✅ **Integration with external systems** (e.g., ETL tools, custom apps)
✅ **High-volume file operations** (avoid SQL overhead)
✅ **Serverless architectures** (no need for Snowflake CLI or drivers)
✅ **Multi-cloud workflows** (unified API for S3/Azure Blob/GCS)

#### **5. When NOT to Use**
❌ **Ad-hoc file operations** (use `PUT`/`GET` commands or Snowsight)
❌ **Small-scale operations** (<100 files/day; use SQL commands)
❌ **Production workloads** (Storage API is in **private preview**)
❌ **Transactional workloads** (use Snowflake tables for ACID compliance)

#### **6. Performance Characteristics**
| **Operation** | **Latency** | **Throughput** | **Parallelism** | **Credit Cost** |
|---------------|-------------|----------------|----------------|-----------------|
| List Files | 100-300ms | 100-1000 files/sec | 10-100 | 0.0001 credits/request |
| Upload File | 100-500ms | 10-100 MB/sec | 1-10 | 0.0001 credits/request + cloud storage costs |
| Download File | 100-500ms | 10-100 MB/sec | 1-10 | 0.0001 credits/request + cloud storage costs |
| Delete File | 100-200ms | 100-1000 files/sec | 10-100 | 0.0001 credits/request |
| Get Metadata | 50-100ms | 1000-10000 files/sec | 10-100 | 0.0001 credits/request |

#### **7. Security Considerations**
| **Security Feature** | **Supported** | **Notes** |
|----------------------|---------------|-----------|
| **JWT Authentication** | ✅ Yes | Recommended for service accounts |
| **OAuth 2.0** | ✅ Yes | For user authentication |
| **Key Pair Auth** | ✅ Yes | For long-lived tokens |
| **RBAC** | ✅ Yes | Fine-grained access control |
| **Network Policies** | ✅ Yes | IP whitelisting |
| **PrivateLink/PSC** | ✅ Yes | Private connectivity to cloud storage |
| **Encryption** | ✅ Yes | TLS 1.2+ for all requests |
| **Presigned URLs** | ✅ Yes | Temporary, time-limited access to cloud storage |

#### **8. Limitations**
- **Private Preview**: Not yet generally available (contact Snowflake for access).
- **Rate Limits**: 100 requests/second per account (may vary).
- **File Size Limits**: 5GB per file (for direct upload/download; larger files require presigned URLs).
- **No Transaction Support**: Operations are **not atomic** (no rollback on failure).
- **No Resumable Uploads**: Large file uploads **cannot be resumed** if interrupted.

#### **9. Configuration and Usage**

##### **Authentication Setup**
```python
import requests
import jwt
import time
from cryptography.hazmat.primitives import serialization

# Load private key
with open('rsa_private_key.p8', 'rb') as key:
    private_key = serialization.load_pem_private_key(
        key.read(),
        password=None  # Or provide password if encrypted
    )

# Generate JWT token
def generate_jwt_token(user, account, private_key):
    payload = {
        'iss': f'{user}@{account}',
        'sub': f'{user}@{account}',
        'iat': int(time.time()),
        'exp': int(time.time()) + 3600,  # 1 hour expiry
        'scope': 'session:role:MY_ROLE'
    }
    token = jwt.encode(payload, private_key, algorithm='RS256')
    return token

# Get session token (for Storage API)
def get_session_token(user, account, private_key):
    jwt_token = generate_jwt_token(user, account, private_key)
    url = f'https://{account}.snowflakecomputing.com/api/v2/authentication/request'
    headers = {
        'X-Snowflake-Authorization-Token-Type': 'KEYPAIR_JWT',
        'Content-Type': 'application/json'
    }
    data = {
        'token': jwt_token,
        'warehouse': 'MY_WH',
        'database': 'MY_DB',
        'schema': 'MY_SCHEMA',
        'role': 'MY_ROLE'
    }
    response = requests.post(url, headers=headers, json=data)
    response.raise_for_status()
    return response.json()['data']['sessionToken']
```

##### **List Stages**
```python
def list_stages(account, session_token):
    url = f'https://{account}.snowflakecomputing.com/api/v2/storage/stages'
    headers = {
        'Authorization': f'Bearer {session_token}',
        'Content-Type': 'application/json'
    }
    params = {
        'type': 'INTERNAL'  # or 'EXTERNAL'
    }
    response = requests.get(url, headers=headers, params=params)
    response.raise_for_status()
    return response.json()['data']
```

##### **List Files in Stage**
```python
def list_files(account, session_token, stage_name, prefix=''):
    url = f'https://{account}.snowflakecomputing.com/api/v2/storage/stages/{stage_name}/files'
    headers = {
        'Authorization': f'Bearer {session_token}',
        'Content-Type': 'application/json'
    }
    params = {
        'prefix': prefix,
        'limit': 1000
    }
    response = requests.get(url, headers=headers, params=params)
    response.raise_for_status()
    return response.json()['data']
```

##### **Upload File**
```python
def upload_file(account, session_token, stage_name, file_path, file_content):
    url = f'https://{account}.snowflakecomputing.com/api/v2/storage/stages/{stage_name}/files/{file_path}'
    headers = {
        'Authorization': f'Bearer {session_token}',
        'Content-Type': 'application/octet-stream'
    }
    params = {
        'overwrite': 'true'
    }
    response = requests.put(url, headers=headers, params=params, data=file_content)
    response.raise_for_status()
    return response.json()['data']
```

##### **Download File**
```python
def download_file(account, session_token, stage_name, file_path):
    url = f'https://{account}.snowflakecomputing.com/api/v2/storage/stages/{stage_name}/files/{file_path}'
    headers = {
        'Authorization': f'Bearer {session_token}'
    }
    response = requests.get(url, headers=headers)
    response.raise_for_status()
    return response.content
```

##### **Generate Presigned URL for Download**
```python
def generate_presigned_url(account, session_token, stage_name, file_path, expiry=3600):
    url = f'https://{account}.snowflakecomputing.com/api/v2/storage/stages/{stage_name}/files/{file_path}/download'
    headers = {
        'Authorization': f'Bearer {session_token}',
        'Content-Type': 'application/json'
    }
    params = {
        'expiry': expiry
    }
    response = requests.get(url, headers=headers, params=params)
    response.raise_for_status()
    return response.json()['data']['presignedUrl']
```

### **B. PUT/GET Commands**

#### **1. Definition and Architecture**
Snowflake's **`PUT`** and **`GET`** commands allow you to **upload files to** and **download files from** Snowflake **stages** (internal or external) using **SQL**. These commands are the **primary method** for file transfer in Snowflake and are **widely used** in data pipelines.

```mermaid
%% PUT/GET Commands Architecture
flowchart TD
    subgraph Client["Client"]
        A[("SnowSQL/Snowflake CLI")] -->|PUT/GET| B[("Snowflake Session")]
        C[("JDBC/ODBC/Python")] -->|PUT/GET| B
    end

    subgraph Snowflake["Snowflake"]
        B -->|AuthN/AuthZ| D[("Authentication Service")]
        D -->|Validate| E[("Metadata Service")]
        E -->|Route| F[("Storage Service")]
        F -->|Internal Stages| G[("Snowflake Blob Storage")]
        F -->|External Stages| H[("Cloud Storage\n(S3/Azure/GCS)")]
    end

    subgraph CloudStorage["Cloud Storage"]
        I[("S3")] --> H
        J[("Azure Blob")] --> H
        K[("GCS")] --> H
    end

    subgraph Operations["Operations"]
        L[("PUT: Upload to Stage")]
        M[("GET: Download from Stage")]
        N[("LIST: List Files in Stage")]
        O[("REMOVE: Delete from Stage")]
    end
    B --> L
    B --> M
    B --> N
    B --> O

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef client fill:#4285f4,stroke:#1976d2;
    classDef snowflake fill:#29abe2,stroke:#1a8fb8;
    classDef cloud fill:#009688,stroke:#00796b;
    classDef operations fill:#ff9800,stroke:#f57c00;
    class A,C client;
    class B,D,E,F,G,H snowflake;
    class I,J,K cloud;
    class L,M,N,O operations;
```

#### **2. How It Works**
1. **PUT Command**:
   - Uploads **local files** to a Snowflake **stage** (internal or external).
   - **Internal Stages**: Files are stored in **Snowflake's Blob Storage**.
   - **External Stages**: Files are stored in **cloud storage** (S3, Azure Blob, GCS).
   - **Compression**: Files can be **automatically compressed** (Gzip, Snappy, etc.).
   - **Parallelism**: Multiple files can be uploaded in **parallel**.

2. **GET Command**:
   - Downloads files from a Snowflake **stage** to a **local directory**.
   - **Internal Stages**: Files are downloaded from **Snowflake's Blob Storage**.
   - **External Stages**: Files are downloaded from **cloud storage**.
   - **Decompression**: Compressed files are **automatically decompressed**.

3. **LIST Command**:
   - Lists files in a **stage** with metadata (size, last modified, etc.).
   - Supports **pattern matching** (e.g., `LIST @stage '*.csv'`).

4. **REMOVE Command**:
   - Deletes files from a **stage**.
   - **Internal Stages**: Files are deleted from **Snowflake's Blob Storage**.
   - **External Stages**: Files are deleted from **cloud storage**.

#### **3. When to Use**
✅ **Ad-hoc file transfers** (manual uploads/downloads)
✅ **Small to medium-scale data pipelines** (<1TB/day)
✅ **Development and testing** (simple to use)
✅ **Integration with scripts** (SnowSQL, Python, etc.)
✅ **Batch data loading** (combine with `COPY INTO`)

#### **4. When NOT to Use**
❌ **High-volume file transfers** (>1TB/day; use Storage API or cloud storage SDKs)
❌ **Real-time streaming** (use Kafka Connector or Snowpipe Streaming)
❌ **Large files** (>5GB; use presigned URLs or cloud storage SDKs)
❌ **Serverless architectures** (use Storage API)

#### **5. Performance Characteristics**
| **Operation** | **Latency** | **Throughput** | **Parallelism** | **Credit Cost** |
|---------------|-------------|----------------|----------------|-----------------|
| PUT (small files) | 100-500ms | 10-100 MB/sec | 1-10 files | 0.0001 credits/GB |
| PUT (large files) | 1-10 sec | 100-1000 MB/min | 1 | 0.0001 credits/GB |
| GET (small files) | 100-500ms | 10-100 MB/sec | 1-10 files | 0.0001 credits/GB |
| GET (large files) | 1-10 sec | 100-1000 MB/min | 1 | 0.0001 credits/GB |
| LIST | 100-300ms | 100-1000 files/sec | 1 | 0.0001 credits/request |
| REMOVE | 100-200ms | 100-1000 files/sec | 1-10 | 0.0001 credits/request |

#### **6. Security Considerations**
| **Security Feature** | **Supported** | **Notes** |
|----------------------|---------------|-----------|
| **TLS Encryption** | ✅ Yes | For all PUT/GET operations |
| **RBAC** | ✅ Yes | Fine-grained access control for stages |
| **Network Policies** | ✅ Yes | IP whitelisting for PUT/GET |
| **PrivateLink/PSC** | ✅ Yes | Private connectivity to cloud storage |
| **Encryption at Rest** | ✅ Yes | For internal stages (Snowflake-managed) |
| **CMK Encryption** | ✅ Yes | For external stages (customer-managed) |
| **Audit Logging** | ✅ Yes | Logs all PUT/GET operations |

#### **7. Limitations**
- **File Size Limits**:
  - **Internal Stages**: 50GB per file (Snowflake limit).
  - **External Stages**: 5TB per file (cloud storage limit).
- **No Resumable Uploads**: Large file uploads **cannot be resumed** if interrupted.
- **No Transaction Support**: PUT/GET operations are **not atomic** (no rollback on failure).
- **No Partial Downloads**: GET downloads the **entire file** (no range requests).

#### **8. Configuration Parameters**

| **Parameter** | **Description** | **Default** | **Valid Values** | **Performance Impact** |
|---------------|-----------------|-------------|------------------|------------------------|
| `AUTO_COMPRESS` | Automatically compress files on PUT | `TRUE` | `TRUE`, `FALSE` | `TRUE` = smaller files, faster uploads |
| `SOURCE_COMPRESSION` | Compression type for source files | `AUTO_DETECT` | `AUTO_DETECT`, `GZIP`, `BZ2`, `BROTLI`, `ZSTD`, `DEFLATE`, `RAW_DEFLATE`, `NONE` | Affects compression ratio and speed |
| `OVERWRITE` | Overwrite existing files | `FALSE` | `TRUE`, `FALSE` | `TRUE` = replaces existing files |
| `PARALLEL` | Number of parallel threads | 4 | 1-100 | Higher = faster uploads/downloads |
| `PRESERVE_FILE_METADATA` | Preserve file metadata (e.g., timestamps) | `FALSE` | `TRUE`, `FALSE` | `TRUE` = retains original metadata |

#### **9. Production-Ready Setup**

##### **Basic PUT/GET Commands**
```sql
-- Create an internal stage
CREATE STAGE MY_INTERNAL_STAGE;

-- Upload a file to the stage (from local machine)
PUT file:///local/path/data.csv @MY_INTERNAL_STAGE;

-- List files in the stage
LIST @MY_INTERNAL_STAGE;

-- Download a file from the stage (to local machine)
GET @MY_INTERNAL_STAGE/data.csv file:///local/path/;

-- Remove a file from the stage
REMOVE @MY_INTERNAL_STAGE/data.csv;

-- Create an external stage (S3)
CREATE STAGE MY_S3_STAGE
  URL = 's3://my-bucket/path/'
  CREDENTIALS = (AWS_KEY_ID = '...' AWS_SECRET_KEY = '...');

-- Upload a file to the external stage
PUT file:///local/path/data.csv @MY_S3_STAGE;

-- Download a file from the external stage
GET @MY_S3_STAGE/data.csv file:///local/path/;
```

##### **PUT with Compression**
```sql
-- Upload and compress a file
PUT file:///local/path/data.csv @MY_INTERNAL_STAGE
  AUTO_COMPRESS = TRUE;

-- Upload with specific compression
PUT file:///local/path/data.csv @MY_INTERNAL_STAGE
  SOURCE_COMPRESSION = 'GZIP';
```

##### **PUT with Parallelism**
```sql
-- Upload multiple files in parallel
PUT file:///local/path/*.csv @MY_INTERNAL_STAGE
  PARALLEL = 10;
```

##### **GET with Pattern Matching**
```sql
-- Download all CSV files
GET @MY_INTERNAL_STAGE '*.csv' file:///local/path/;
```

##### **LIST with Pattern Matching**
```sql
-- List all CSV files
LIST @MY_INTERNAL_STAGE PATTERN = '.*\\.csv';
```

##### **PUT/GET with External Stages (S3)**
```sql
-- Upload to S3 via external stage
PUT file:///local/path/data.parquet @MY_S3_STAGE
  SOURCE_COMPRESSION = 'SNAPPY';

-- Download from S3 via external stage
GET @MY_S3_STAGE/data.parquet file:///local/path/;
```

### **C. Cloud Storage SDK Integrations**

While Snowflake's **PUT/GET** commands and **Storage API** provide direct file transfer capabilities, you can also use **cloud storage SDKs** (AWS S3, Azure Blob, GCS) to **directly interact with external stages**. This approach is useful for **high-volume file operations**, **resumable uploads**, and **advanced cloud storage features**.

#### **1. AWS S3 SDK Integration**
##### **When to Use**
✅ **High-volume file transfers** (>1TB/day)
✅ **Resumable uploads** (for large files)
✅ **Advanced S3 features** (e.g., S3 Batch, S3 Event Notifications)
✅ **Multi-part uploads** (for files >5GB)
✅ **Serverless architectures** (e.g., AWS Lambda)

##### **Configuration Example (Python)**
```python
import boto3
from botocore.exceptions import ClientError

# Initialize S3 client
s3 = boto3.client(
    's3',
    aws_access_key_id='...',
    aws_secret_access_key='...',
    region_name='us-east-1'
)

bucket_name = 'my-bucket'
stage_path = 'snowflake-stage/'  # Path in S3 for Snowflake stage

# Upload a file to S3 (for Snowflake external stage)
def upload_to_s3(file_path, s3_key):
    try:
        s3.upload_file(
            file_path,
            bucket_name,
            f'{stage_path}{s3_key}',
            ExtraArgs={
                'ServerSideEncryption': 'aws:kms',
                'SSEKMSKeyId': 'arn:aws:kms:us-east-1:123456789012:key/abcd1234'
            }
        )
        print(f"Uploaded {file_path} to {s3_key}")
    except ClientError as e:
        print(f"Error uploading {file_path}: {e}")

# Download a file from S3
def download_from_s3(s3_key, file_path):
    try:
        s3.download_file(
            bucket_name,
            f'{stage_path}{s3_key}',
            file_path
        )
        print(f"Downloaded {s3_key} to {file_path}")
    except ClientError as e:
        print(f"Error downloading {s3_key}: {e}")

# List files in S3 stage
def list_s3_files(prefix=''):
    paginator = s3.get_paginator('list_objects_v2')
    for page in paginator.paginate(Bucket=bucket_name, Prefix=f'{stage_path}{prefix}'):
        for obj in page.get('Contents', []):
            print(obj['Key'])

# Example usage
upload_to_s3('/local/path/data.csv', 'data.csv')
download_from_s3('data.csv', '/local/path/data_downloaded.csv')
list_s3_files()
```

##### **Resumable Upload Example (Multipart Upload)**
```python
import boto3
import os

# Initialize S3 client
s3 = boto3.client('s3')

bucket_name = 'my-bucket'
file_path = '/local/path/large_file.csv'  # >5GB
s3_key = 'snowflake-stage/large_file.csv'

# Multipart upload
def multipart_upload(file_path, bucket_name, s3_key):
    part_size = 8 * 1024 * 1024  # 8MB parts (minimum for multipart upload)
    file_size = os.path.getsize(file_path)
    parts = (file_size + part_size - 1) // part_size

    # Initiate multipart upload
    mpu = s3.create_multipart_upload(
        Bucket=bucket_name,
        Key=s3_key,
        ServerSideEncryption='aws:kms',
        SSEKMSKeyId='arn:aws:kms:us-east-1:123456789012:key/abcd1234'
    )
    mpu_id = mpu['UploadId']

    # Upload parts
    part_tags = []
    with open(file_path, 'rb') as f:
        for i in range(parts):
            offset = i * part_size
            f.seek(offset)
            data = f.read(part_size)
            part = s3.upload_part(
                Bucket=bucket_name,
                Key=s3_key,
                PartNumber=i + 1,
                UploadId=mpu_id,
                Body=data
            )
            part_tags.append({'PartNumber': i + 1, 'ETag': part['ETag']})

    # Complete multipart upload
    s3.complete_multipart_upload(
        Bucket=bucket_name,
        Key=s3_key,
        UploadId=mpu_id,
        MultipartUpload={'Parts': part_tags}
    )
    print(f"Uploaded {file_path} to {s3_key} using multipart upload")

# Example usage
multipart_upload(file_path, bucket_name, s3_key)
```

#### **2. Azure Blob Storage SDK Integration**
##### **When to Use**
✅ **High-volume file transfers** (>1TB/day)
✅ **Resumable uploads** (for large files)
✅ **Advanced Azure Blob features** (e.g., Blob Lease, Blob Metadata)
✅ **Multi-part uploads** (for files >5GB)
✅ **Serverless architectures** (e.g., Azure Functions)

##### **Configuration Example (Python)**
```python
from azure.storage.blob import BlobServiceClient, BlobClient, ContainerClient
import os

# Initialize Blob Service Client
connection_string = "DefaultEndpointsProtocol=https;AccountName=myaccount;AccountKey=...;EndpointSuffix=core.windows.net"
blob_service_client = BlobServiceClient.from_connection_string(connection_string)

container_name = 'my-container'
stage_path = 'snowflake-stage/'  # Path in Azure Blob for Snowflake stage

# Upload a file to Azure Blob
def upload_to_azure(file_path, blob_name):
    blob_client = blob_service_client.get_blob_client(
        container=container_name,
        blob=f'{stage_path}{blob_name}'
    )
    with open(file_path, 'rb') as data:
        blob_client.upload_blob(
            data,
            overwrite=True,
            content_settings=ContentSettings(content_type='text/csv')
        )
    print(f"Uploaded {file_path} to {blob_name}")

# Download a file from Azure Blob
def download_from_azure(blob_name, file_path):
    blob_client = blob_service_client.get_blob_client(
        container=container_name,
        blob=f'{stage_path}{blob_name}'
    )
    with open(file_path, 'wb') as download_file:
        download_file.write(blob_client.download_blob().readall())
    print(f"Downloaded {blob_name} to {file_path}")

# List files in Azure Blob stage
def list_azure_blobs(prefix=''):
    container_client = blob_service_client.get_container_client(container_name)
    blob_list = container_client.list_blobs(name_starts_with=f'{stage_path}{prefix}')
    for blob in blob_list:
        print(blob.name)

# Example usage
upload_to_azure('/local/path/data.csv', 'data.csv')
download_from_azure('data.csv', '/local/path/data_downloaded.csv')
list_azure_blobs()
```

##### **Resumable Upload Example (Block Blob Upload)**
```python
from azure.storage.blob import BlobServiceClient, BlobBlock

# Initialize Blob Service Client
blob_service_client = BlobServiceClient.from_connection_string(connection_string)

bucket_name = 'my-container'
file_path = '/local/path/large_file.csv'  # >5GB
blob_name = 'snowflake-stage/large_file.csv'

# Block blob upload (resumable)
def block_blob_upload(file_path, container_name, blob_name):
    blob_client = blob_service_client.get_blob_client(
        container=container_name,
        blob=blob_name
    )

    block_ids = []
    with open(file_path, 'rb') as f:
        chunk_size = 4 * 1024 * 1024  # 4MB chunks
        chunk_number = 0
        while True:
            chunk_data = f.read(chunk_size)
            if not chunk_data:
                break
            block_id = f"{chunk_number:08d}"
            blob_client.stage_block(
                block_id,
                chunk_data,
                length=len(chunk_data)
            )
            block_ids.append(BlobBlock(block_id))
            chunk_number += 1

    blob_client.commit_block_list(block_ids)
    print(f"Uploaded {file_path} to {blob_name} using block blob upload")

# Example usage
block_blob_upload(file_path, container_name, blob_name)
```

#### **3. Google Cloud Storage SDK Integration**
##### **When to Use**
✅ **High-volume file transfers** (>1TB/day)
✅ **Resumable uploads** (for large files)
✅ **Advanced GCS features** (e.g., Object Versioning, Signed URLs)
✅ **Multi-part uploads** (for files >5GB)
✅ **Serverless architectures** (e.g., Cloud Functions)

##### **Configuration Example (Python)**
```python
from google.cloud import storage
import os

# Initialize Storage Client
storage_client = storage.Client.from_service_account_json('service-account.json')
bucket_name = 'my-bucket'
stage_path = 'snowflake-stage/'  # Path in GCS for Snowflake stage

# Upload a file to GCS
def upload_to_gcs(file_path, blob_name):
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(f'{stage_path}{blob_name}')
    blob.upload_from_filename(file_path)
    print(f"Uploaded {file_path} to {blob_name}")

# Download a file from GCS
def download_from_gcs(blob_name, file_path):
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(f'{stage_path}{blob_name}')
    blob.download_to_filename(file_path)
    print(f"Downloaded {blob_name} to {file_path}")

# List files in GCS stage
def list_gcs_blobs(prefix=''):
    bucket = storage_client.bucket(bucket_name)
    blobs = bucket.list_blobs(prefix=f'{stage_path}{prefix}')
    for blob in blobs:
        print(blob.name)

# Example usage
upload_to_gcs('/local/path/data.csv', 'data.csv')
download_from_gcs('data.csv', '/local/path/data_downloaded.csv')
list_gcs_blobs()
```

##### **Resumable Upload Example**
```python
from google.cloud import storage
import os

# Initialize Storage Client
storage_client = storage.Client.from_service_account_json('service-account.json')
bucket_name = 'my-bucket'
file_path = '/local/path/large_file.csv'  # >5GB
blob_name = 'snowflake-stage/large_file.csv'

# Resumable upload
def resumable_upload(file_path, bucket_name, blob_name):
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(blob_name)

    # Chunk size for resumable upload (recommended: 5MB)
    chunk_size = 5 * 1024 * 1024
    blob.chunk_size = chunk_size

    blob.upload_from_filename(
        file_path,
        timeout=60 * 60  # 1 hour timeout
    )
    print(f"Uploaded {file_path} to {blob_name} using resumable upload")

# Example usage
resumable_upload(file_path, bucket_name, blob_name)
```

### **D. External Stage API**

The **External Stage API** allows you to **programmatically create, manage, and query** external stages in Snowflake. This is useful for **automating stage management** in CI/CD pipelines, **dynamic stage creation**, and **integration with external systems**.

#### **1. How It Works**
1. **Stage Creation**:
   - Use `CREATE STAGE` SQL or the **Snowflake REST API** to create an external stage.
   - Configure **cloud storage URL**, **credentials**, and **file format**.

2. **Stage Management**:
   - Use `ALTER STAGE` to **update stage properties** (e.g., credentials, file format).
   - Use `DROP STAGE` to **delete a stage**.

3. **Stage Querying**:
   - Use `SHOW STAGES` to **list all stages**.
   - Use `DESCRIBE STAGE` to **get stage details**.

4. **File Operations**:
   - Use `LIST @stage` to **list files in a stage**.
   - Use `PUT`/`GET` to **upload/download files**.

#### **2. When to Use**
✅ **Automated stage management** (CI/CD pipelines)
✅ **Dynamic stage creation** (e.g., per-user stages)
✅ **Integration with external systems** (e.g., ETL tools)
✅ **Programmatic stage configuration** (e.g., updating credentials)

#### **3. When NOT to Use**
❌ **Ad-hoc stage operations** (use SQL or Snowsight)
❌ **Small-scale environments** (manual stage management is sufficient)
❌ **Non-programmatic workflows** (use Snowsight or SnowSQL)

#### **4. Configuration Examples**

##### **Create External Stage (SQL)**
```sql
-- Create an external stage for S3
CREATE STAGE MY_S3_STAGE
  URL = 's3://my-bucket/path/'
  CREDENTIALS = (AWS_KEY_ID = '...' AWS_SECRET_KEY = '...')
  FILE_FORMAT = (TYPE = 'PARQUET' COMPRESSION = 'SNAPPY')
  ENCRYPTION = (TYPE = 'AWS_SSE_KMS' KMS_KEY = 'arn:aws:kms:us-east-1:123456789012:key/abcd1234');

-- Create an external stage for Azure Blob
CREATE STAGE MY_AZURE_STAGE
  URL = 'azure://myaccount.blob.core.windows.net/my-container/path/'
  CREDENTIALS = (AZURE_SAS_TOKEN = '...')
  FILE_FORMAT = (TYPE = 'PARQUET');

-- Create an external stage for GCS
CREATE STAGE MY_GCS_STAGE
  URL = 'gcs://my-bucket/path/'
  CREDENTIALS = (GCS_SERVICE_ACCOUNT = '...')
  FILE_FORMAT = (TYPE = 'PARQUET');
```

##### **List All Stages (SQL)**
```sql
-- List all stages
SHOW STAGES;

-- List stages in a specific database/schema
SHOW STAGES IN DATABASE MY_DB SCHEMA MY_SCHEMA;
```

##### **Describe Stage (SQL)**
```sql
-- Get stage details
DESCRIBE STAGE MY_S3_STAGE;
```

##### **Alter Stage (SQL)**
```sql
-- Update stage credentials
ALTER STAGE MY_S3_STAGE
  SET CREDENTIALS = (AWS_KEY_ID = 'new-key' AWS_SECRET_KEY = 'new-secret');

-- Update stage file format
ALTER STAGE MY_S3_STAGE
  SET FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1);

-- Update stage URL
ALTER STAGE MY_S3_STAGE
  SET URL = 's3://my-new-bucket/path/';
```

##### **Drop Stage (SQL)**
```sql
-- Drop a stage
DROP STAGE IF EXISTS MY_S3_STAGE;
```

##### **List Files in Stage (SQL)**
```sql
-- List all files in a stage
LIST @MY_S3_STAGE;

-- List files matching a pattern
LIST @MY_S3_STAGE PATTERN = '.*\\.parquet';
```

## **3. Git Integrations Deep Dive**

Git integrations enable **version control** and **collaboration** for Snowflake objects (e.g., **stored procedures, functions, views, SQL scripts**). These integrations are critical for **CI/CD pipelines**, **code reviews**, and **audit trails** in modern data environments.


### **A. Snowsight Git Integration (Native)**

#### **1. Definition and Architecture**
**Snowsight Git Integration** allows you to **connect Snowsight worksheets, stored procedures, and other SQL objects** to **Git repositories** (GitHub, GitLab, Bitbucket). This enables **version control**, **collaboration**, and **CI/CD workflows** directly from Snowsight.

```mermaid
%% Snowsight Git Integration Architecture
flowchart TD
    subgraph Snowsight["Snowsight"]
        A[("Worksheet")] -->|Git Integration| B[("Git Client")]
        C[("Stored Procedure")] --> B
        D[("Function")] --> B
        E[("View")] --> B
    end

    subgraph Git["Git Repository"]
        B -->|Push/Pull| F[("GitHub/GitLab/Bitbucket")]
    end

    subgraph Workflows["Workflows"]
        G[("Code Versioning")] --> F
        H[("Collaboration")] --> F
        I[("CI/CD")] --> F
        J[("Audit Trail")] --> F
    end

    subgraph Authentication["Authentication"]
        K[("OAuth App")] --> F
        L[("Personal Access Token")] --> F
        M[("SSH Key")] --> F
    end
    B --> K
    B --> L
    B --> M

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef snowflake fill:#29abe2,stroke:#1a8fb8;
    classDef git fill:#f4511e,stroke:#d84315;
    classDef workflows fill:#009688,stroke:#00796b;
    classDef auth fill:#9c27b0,stroke:#7b1fa2;
    class A,B,C,D,E snowflake;
    class F git;
    class G,H,I,J workflows;
    class K,L,M auth;
```

#### **2. How It Works**
1. **Git Repository Connection**:
   - Connect Snowsight to a **Git repository** (GitHub, GitLab, Bitbucket).
   - Configure **authentication** (OAuth, Personal Access Token, SSH Key).

2. **Object Versioning**:
   - **Worksheets**: Version-controlled as **`.sql` files** in the repository.
   - **Stored Procedures/Functions**: Version-controlled as **SQL scripts**.
   - **Views**: Version-controlled as **SQL DDL**.

3. **Push/Pull Workflow**:
   - **Push**: Save changes in Snowsight to the **Git repository**.
   - **Pull**: Load changes from the **Git repository** into Snowsight.
   - **Sync**: Automatically **sync changes** between Snowsight and Git.

4. **Branch Management**:
   - Work with **Git branches** (e.g., `main`, `dev`, `feature/*`).
   - **Merge requests/pull requests** for code reviews.

5. **Conflict Resolution**:
   - **Merge conflicts** are detected and must be **resolved manually**.
   - **Visual diff tool** in Snowsight for comparing changes.

#### **3. When to Use**
✅ **Team collaboration** on Snowflake SQL code
✅ **Version control** for stored procedures, functions, and views
✅ **CI/CD workflows** (automated testing, deployment)
✅ **Audit trails** for SQL changes
✅ **Code reviews** (pull requests, merge requests)

#### **4. When NOT to Use**
❌ **Individual development** (no need for version control)
❌ **Ad-hoc queries** (not suitable for version control)
❌ **Non-SQL objects** (e.g., tables, stages; use Terraform or other IaC tools)
❌ **High-frequency changes** (Git may not scale for >100 changes/day)

#### **5. Performance Characteristics**
| **Operation** | **Latency** | **Throughput** | **Parallelism** | **Notes** |
|---------------|-------------|----------------|----------------|-----------|
| Push | 1-10 sec | 1-10 files/min | 1 | Depends on Git provider |
| Pull | 1-10 sec | 1-10 files/min | 1 | Depends on Git provider |
| Sync | 1-10 sec | 1-10 files/min | 1 | Depends on Git provider |
| Conflict Detection | <1 sec | N/A | N/A | Real-time |

#### **6. Security Considerations**
| **Security Feature** | **Supported** | **Notes** |
|----------------------|---------------|-----------|
| **OAuth 2.0** | ✅ Yes | Recommended for GitHub/GitLab |
| **Personal Access Tokens (PAT)** | ✅ Yes | For GitHub/GitLab/Bitbucket |
| **SSH Keys** | ✅ Yes | For GitHub/GitLab/Bitbucket |
| **RBAC** | ✅ Yes | Fine-grained access control in Snowsight |
| **Repository Permissions** | ✅ Yes | Controlled by Git provider |
| **Audit Logging** | ✅ Yes | Logs all Git operations in Snowsight |

#### **7. Limitations**
- **Object Types**: Only supports **worksheets, stored procedures, functions, and views** (not tables, stages, etc.).
- **Git Provider Dependency**: Requires **GitHub, GitLab, or Bitbucket**.
- **Conflict Resolution**: **Manual resolution** required for merge conflicts.
- **No Offline Mode**: Requires **internet connectivity** to push/pull.
- **No Partial Commits**: **All changes** in a worksheet are committed together.

#### **8. Configuration Steps**

##### **Step 1: Set Up Git Provider**
1. **GitHub**:
   - Create a **new repository** (or use an existing one).
   - Generate a **Personal Access Token (PAT)** with `repo` permissions.
   - Or, set up an **OAuth App** for Snowsight.

2. **GitLab**:
   - Create a **new repository** (or use an existing one).
   - Generate a **Personal Access Token (PAT)** with `api` and `read_repository`/`write_repository` permissions.

3. **Bitbucket**:
   - Create a **new repository** (or use an existing one).
   - Generate an **App Password** with `read`/`write` permissions.

##### **Step 2: Connect Snowsight to Git**
1. Open **Snowsight** and navigate to **Worksheets**.
2. Click on the **Git icon** in the top-right corner.
3. Select **Connect to Git**.
4. Choose your **Git provider** (GitHub, GitLab, Bitbucket).
5. Enter the **repository URL** (e.g., `https://github.com/myorg/myrepo`).
6. Configure **authentication**:
   - For **GitHub/GitLab**: Use **OAuth** or **Personal Access Token**.
   - For **Bitbucket**: Use **App Password**.
7. Select the **branch** (e.g., `main`).
8. Click **Connect**.

##### **Step 3: Push Changes to Git**
1. Make changes to a **worksheet, stored procedure, function, or view**.
2. Click the **Git icon** in the top-right corner.
3. Enter a **commit message**.
4. Click **Push** to save changes to the Git repository.

##### **Step 4: Pull Changes from Git**
1. Click the **Git icon** in the top-right corner.
2. Click **Pull** to load the latest changes from the Git repository.
3. Resolve any **merge conflicts** if they exist.

##### **Step 5: Manage Branches**
1. Click the **Git icon** in the top-right corner.
2. Select **Switch Branch** to change branches.
3. Create a **new branch** for feature development.
4. Merge changes via **pull requests/merge requests** in your Git provider.

#### **9. Production-Ready Setup**

##### **Example: Snowsight + GitHub Workflow**
1. **Set Up GitHub Repository**:
   ```bash
   # Create a new repository
   gh repo create my-snowflake-repo --public --clone

   # Initialize repository
   cd my-snowflake-repo
   git init
   echo "# Snowflake SQL" > README.md
   git add README.md
   git commit -m "Initial commit"
   git push -u origin main
   ```

2. **Connect Snowsight to GitHub**:
   - In Snowsight, navigate to **Worksheets**.
   - Click **Git icon** > **Connect to Git**.
   - Select **GitHub** and authenticate with OAuth.
   - Enter repository URL: `https://github.com/myorg/my-snowflake-repo`.
   - Select branch: `main`.

3. **Create a Worksheet and Push to Git**:
   ```sql
   -- Example worksheet (my_worksheet.sql)
   SELECT
       user_id,
       COUNT(*) AS event_count,
       MAX(event_time) AS last_event_time
   FROM
       my_events_table
   WHERE
       event_time > CURRENT_DATE()
   GROUP BY
       user_id
   ORDER BY
       event_count DESC;
   ```
   - Save the worksheet.
   - Click **Git icon** > **Push** with commit message: "Add user events query".

4. **Create a Stored Procedure and Push to Git**:
   ```sql
   -- Example stored procedure (my_procedure.sql)
   CREATE OR REPLACE PROCEDURE my_procedure(user_id INT)
   RETURNS STRING
   LANGUAGE SQL
   AS
   $$
   DECLARE
       event_count INT;
   BEGIN
       SELECT COUNT(*) INTO event_count
       FROM my_events_table
       WHERE user_id = user_id;

       RETURN 'User ' || user_id || ' has ' || event_count || ' events';
   END;
   $$;
   ```
   - Save the stored procedure.
   - Click **Git icon** > **Push** with commit message: "Add my_procedure".

5. **Collaborate with Team**:
   - Team members **clone the repository** and connect Snowsight.
   - Make changes and **push to feature branches**.
   - Submit **pull requests** for code reviews.
   - Merge changes into `main` after approval.

### **B. CI/CD with Git**

CI/CD (Continuous Integration/Continuous Deployment) pipelines automate the **testing, validation, and deployment** of Snowflake code. By integrating **Git repositories** with **CI/CD tools** (GitHub Actions, GitLab CI, Azure DevOps, Jenkins), you can ensure **code quality**, **security**, and **reliability** in production.

#### **1. Architecture**
```mermaid
%% CI/CD with Git Architecture
flowchart TD
    subgraph Git["Git Repository"]
        A[("GitHub/GitLab")] -->|Webhook| B[("CI/CD Pipeline")]
    end

    subgraph CI["CI/CD Tools"]
        B -->|Checkout| C[("Source Code")]
        C -->|Lint/Test| D[("Code Quality")]
        D -->|Validate| E[("SQL Validation")]
        E -->|Deploy| F[("Snowflake Deployment")]
    end

    subgraph Snowflake["Snowflake"]
        F -->|Execute| G[("Snowflake Account")]
    end

    subgraph Workflows["Workflows"]
        H[("Pull Request Trigger")] --> B
        I[("Push to Main Trigger")] --> B
        J[("Scheduled Trigger")] --> B
    end

    subgraph Tools["CI/CD Tools"]
        K[("GitHub Actions")] --> B
        L[("GitLab CI")] --> B
        M[("Azure DevOps")] --> B
        N[("Jenkins")] --> B
    end

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef git fill:#f4511e,stroke:#d84315;
    classDef ci fill:#4285f4,stroke:#1976d2;
    classDef snowflake fill:#29abe2,stroke:#1a8fb8;
    classDef workflows fill:#009688,stroke:#00796b;
    classDef tools fill:#9c27b0,stroke:#7b1fa2;
    class A git;
    class B,C,D,E,F ci;
    class G snowflake;
    class H,I,J workflows;
    class K,L,M,N tools;
```

#### **2. CI/CD Workflows for Snowflake**
| **Workflow** | **Trigger** | **Steps** | **Tools** | **Use Case** |
|--------------|-------------|-----------|-----------|--------------|
| **Pull Request Validation** | Pull request opened/updated | Lint SQL, run tests, validate schema | GitHub Actions, GitLab CI | Code reviews, quality checks |
| **Main Branch Deployment** | Push to `main` | Deploy to production, run post-deployment tests | GitHub Actions, GitLab CI | Production deployments |
| **Scheduled Tests** | Cron schedule | Run regression tests, validate data | GitHub Actions, GitLab CI | Nightly tests, data validation |
| **Feature Branch Deployment** | Push to feature branch | Deploy to dev/test environment | GitHub Actions, GitLab CI | Feature testing, QA |
| **Infrastructure as Code (IaC)** | Push to `main` | Apply Terraform/Pulumi changes | GitHub Actions, GitLab CI, Azure DevOps | Infrastructure management |

#### **3. When to Use CI/CD with Git**
✅ **Team collaboration** on Snowflake code
✅ **Automated testing** (SQL linting, data validation)
✅ **Production deployments** (controlled, auditable)
✅ **Infrastructure as Code (IaC)** (Terraform, Pulumi)
✅ **Compliance requirements** (audit trails, approvals)

#### **4. When NOT to Use CI/CD with Git**
❌ **Individual development** (no need for automation)
❌ **Ad-hoc queries** (not suitable for CI/CD)
❌ **High-frequency changes** (>100 changes/day; may overwhelm CI/CD)
❌ **Non-code objects** (e.g., tables with data; use data pipelines instead)

#### **5. CI/CD Tools Comparison**

| **Tool** | **Hosted** | **Free Tier** | **Snowflake Integration** | **Best For** | **Limitations** |
|----------|------------|---------------|----------------------------|--------------|----------------|
| **GitHub Actions** | ✅ Yes | 2000 minutes/month | Native (GitHub Marketplace) | GitHub users, open-source | GitHub-only |
| **GitLab CI** | ✅ Yes (SaaS) / ❌ No (Self-Hosted) | 400 CI minutes/month | Native (GitLab templates) | GitLab users, self-hosted | Self-hosted requires setup |
| **Azure DevOps** | ✅ Yes | 1800 minutes/month | Native (Azure Marketplace) | Azure users, enterprise | Microsoft ecosystem |
| **Jenkins** | ❌ No | ✅ Yes (Self-Hosted) | Plugins (Snowflake Plugin) | Self-hosted, custom workflows | Requires maintenance |
| **CircleCI** | ✅ Yes | 6000 build minutes/month | Custom scripts | Custom workflows | Limited free tier |
| **Bitbucket Pipelines** | ✅ Yes | 50 build minutes/month | Custom scripts | Bitbucket users | Limited free tier |

#### **6. Production-Ready CI/CD Examples**

##### **Example 1: GitHub Actions for Snowflake SQL Linting**
```yaml
# .github/workflows/snowflake-lint.yml
name: Snowflake SQL Linting

on:
  pull_request:
    branches: [ main ]
    paths:
      - '**.sql'

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install sqlfluff snowflake-sqlalchemy

      - name: Lint SQL files
        run: |
          # Lint all SQL files with SQLFluff (Snowflake dialect)
          sqlfluff lint --dialect snowflake --rules L001,L003,L004,L009,L010,L014,L016,L021,L024,L025,L026,L027,L028,L029,L030,L031,L032,L033,L034,L035,L036,L037,L038,L039,L040,L042,L043,L044 .
          # Fail if linting errors are found
          sqlfluff lint --dialect snowflake --rules L001,L003,L004,L009,L010,L014,L016,L021,L024,L025,L026,L027,L028,L029,L030,L031,L032,L033,L034,L035,L036,L037,L038,L039,L040,L042,L043,L044 . || exit 1
```

##### **Example 2: GitHub Actions for Snowflake Deployment**
```yaml
# .github/workflows/snowflake-deploy.yml
name: Snowflake Deployment

on:
  push:
    branches: [ main ]
    paths:
      - 'stored_procedures/**'
      - 'functions/**'
      - 'views/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: Install Snowflake Connector
        run: |
          python -m pip install --upgrade pip
          pip install snowflake-connector-python

      - name: Deploy to Snowflake
        env:
          SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
          SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
          SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}
          SNOWFLAKE_WAREHOUSE: ${{ secrets.SNOWFLAKE_WAREHOUSE }}
          SNOWFLAKE_DATABASE: ${{ secrets.SNOWFLAKE_DATABASE }}
          SNOWFLAKE_SCHEMA: ${{ secrets.SNOWFLAKE_SCHEMA }}
          SNOWFLAKE_ROLE: ${{ secrets.SNOWFLAKE_ROLE }}
        run: |
          python deploy.py
```

```python
# deploy.py
import snowflake.connector
import os
import glob

# Connect to Snowflake
conn = snowflake.connector.connect(
    user=os.getenv('SNOWFLAKE_USER'),
    password=os.getenv('SNOWFLAKE_PASSWORD'),
    account=os.getenv('SNOWFLAKE_ACCOUNT'),
    warehouse=os.getenv('SNOWFLAKE_WAREHOUSE'),
    database=os.getenv('SNOWFLAKE_DATABASE'),
    schema=os.getenv('SNOWFLAKE_SCHEMA'),
    role=os.getenv('SNOWFLAKE_ROLE')
)

# Deploy stored procedures
for sp_file in glob.glob('stored_procedures/*.sql'):
    with open(sp_file, 'r') as f:
        sp_sql = f.read()
    sp_name = os.path.basename(sp_file).replace('.sql', '')
    cursor = conn.cursor()
    try:
        cursor.execute(f"CREATE OR REPLACE PROCEDURE {sp_name}() AS $$ {sp_sql} $$")
        print(f"Deployed stored procedure: {sp_name}")
    except Exception as e:
        print(f"Error deploying {sp_name}: {e}")
        raise

# Deploy functions
for func_file in glob.glob('functions/*.sql'):
    with open(func_file, 'r') as f:
        func_sql = f.read()
    func_name = os.path.basename(func_file).replace('.sql', '')
    cursor = conn.cursor()
    try:
        cursor.execute(f"CREATE OR REPLACE FUNCTION {func_name}() AS $$ {func_sql} $$")
        print(f"Deployed function: {func_name}")
    except Exception as e:
        print(f"Error deploying {func_name}: {e}")
        raise

# Deploy views
for view_file in glob.glob('views/*.sql'):
    with open(view_file, 'r') as f:
        view_sql = f.read()
    view_name = os.path.basename(view_file).replace('.sql', '')
    cursor = conn.cursor()
    try:
        cursor.execute(f"CREATE OR REPLACE VIEW {view_name} AS {view_sql}")
        print(f"Deployed view: {view_name}")
    except Exception as e:
        print(f"Error deploying {view_name}: {e}")
        raise

conn.close()
```

##### **Example 3: GitLab CI for Snowflake Testing**
```yaml
# .gitlab-ci.yml
stages:
  - test
  - deploy

snowflake-test:
  stage: test
  image: python:3.10
  before_script:
    - pip install snowflake-connector-python pytest
  script:
    - pytest tests/ --tb=short
  only:
    - merge_requests
    - main

snowflake-deploy:
  stage: deploy
  image: python:3.10
  before_script:
    - pip install snowflake-connector-python
  script:
    - python deploy.py
  only:
    - main
```

```python
# tests/test_views.py
import snowflake.connector
import pytest

@pytest.fixture
def snowflake_connection():
    conn = snowflake.connector.connect(
        user='CI_USER',
        password='CI_PASSWORD',
        account='CI_ACCOUNT',
        warehouse='CI_WAREHOUSE',
        database='CI_DATABASE',
        schema='CI_SCHEMA'
    )
    yield conn
    conn.close()

def test_view_exists(snowflake_connection):
    cursor = snowflake_connection.cursor()
    cursor.execute("SHOW VIEWS IN SCHEMA CI_SCHEMA")
    views = [row[0] for row in cursor.fetchall()]
    assert 'my_view' in views, "View 'my_view' does not exist"
```

##### **Example 4: Terraform + GitHub Actions for Infrastructure as Code (IaC)**
```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  push:
    branches: [ main ]
    paths:
      - 'terraform/**'
  pull_request:
    branches: [ main ]
    paths:
      - 'terraform/**'

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: terraform

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.3.0

      - name: Terraform Init
        id: init
        run: terraform init

      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color
        continue-on-error: true

      - name: Terraform Apply (Main Branch)
        if: github.ref == 'refs/heads/main' && steps.plan.outcome == 'success'
        run: terraform apply -auto-approve
        env:
          TF_VAR_snowflake_account: ${{ secrets.SNOWFLAKE_ACCOUNT }}
          TF_VAR_snowflake_user: ${{ secrets.SNOWFLAKE_USER }}
          TF_VAR_snowflake_password: ${{ secrets.SNOWFLAKE_PASSWORD }}
```

```hcl
# terraform/main.tf
terraform {
  required_providers {
    snowflake = {
      source  = "Snowflake-Labs/snowflake"
      version = "~> 0.68"
    }
  }
}

provider "snowflake" {
  account  = var.snowflake_account
  user     = var.snowflake_user
  password = var.snowflake_password
}

# Create a database
resource "snowflake_database" "my_db" {
  name = "MY_DB"
}

# Create a schema
resource "snowflake_schema" "my_schema" {
  name     = "MY_SCHEMA"
  database = snowflake_database.my_db.name
}

# Create a warehouse
resource "snowflake_warehouse" "my_wh" {
  name           = "MY_WH"
  warehouse_size = "XSMALL"
  auto_suspend   = 60
}

# Create a stage
resource "snowflake_stage" "my_stage" {
  name     = "MY_STAGE"
  database = snowflake_database.my_db.name
  schema   = snowflake_schema.my_schema.name
  url      = "s3://my-bucket/path/"
  credentials = {
    aws_key_id     = var.aws_key_id
    aws_secret_key = var.aws_secret_key
  }
}

# Create a table
resource "snowflake_table" "my_table" {
  name     = "MY_TABLE"
  database = snowflake_database.my_db.name
  schema   = snowflake_schema.my_schema.name
  column {
    name = "ID"
    type = "INTEGER"
  }
  column {
    name = "NAME"
    type = "STRING"
  }
}
```

### **C. Snowflake CLI + Git**

The **Snowflake CLI** (`snowflake` or `snowsql`) can be **integrated with Git** to enable **scripted workflows** for Snowflake operations. This is useful for **automating repetitive tasks**, **running scripts in CI/CD pipelines**, and **managing Snowflake objects as code**.

#### **1. How It Works**
1. **Install Snowflake CLI**:
   - Install the **Snowflake CLI** (`snowflake` for new CLI or `snowsql` for legacy).
   - Configure **connection profiles** for different environments (dev, test, prod).

2. **Script Snowflake Operations**:
   - Write **shell scripts** or **Python scripts** that use the CLI to:
     - Execute SQL queries.
     - Upload/download files.
     - Manage stages, tables, and other objects.

3. **Version Control Scripts**:
   - Store scripts in **Git repositories**.
   - Use **Git hooks** or **CI/CD pipelines** to run scripts.

4. **Automate Workflows**:
   - Run scripts **on a schedule** (e.g., cron jobs).
   - Trigger scripts **on Git events** (e.g., push, pull request).

#### **2. When to Use**
✅ **Scripted workflows** (e.g., data loading, backups)
✅ **CI/CD pipelines** (run Snowflake CLI commands in GitHub Actions/GitLab CI)
✅ **Local development** (run Snowflake commands from terminal)
✅ **Automated testing** (run SQL tests from scripts)
✅ **Infrastructure management** (create/drop objects as code)

#### **3. When NOT to Use**
❌ **Interactive queries** (use Snowsight or SnowSQL)
❌ **High-frequency operations** (>100 operations/sec; use Storage API)
❌ **Complex transformations** (use stored procedures or Spark)

#### **4. Production-Ready Setup**

##### **Install Snowflake CLI**
```bash
# Install Snowflake CLI (new)
curl -O https://downloads.snowflake.com/snowflake-cli/latest/snowflake_linux_x86_64
chmod +x snowflake_linux_x86_64
sudo mv snowflake_linux_x86_64 /usr/local/bin/snowflake

# Verify installation
snowflake --version

# Configure connection (interactive)
snowflake connection add my_conn
# Follow prompts to enter account, username, password, etc.
```

##### **Example: Automated Data Loading Script**
```bash
#!/bin/bash
# load_data.sh - Automated data loading script

# Set connection
CONNECTION="my_conn"

# Upload file to stage
snowflake stage upload @MY_STAGE /local/path/data.csv --connection $CONNECTION

# Load data into table
snowflake query "COPY INTO MY_TABLE FROM @MY_STAGE/data.csv FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1)" --connection $CONNECTION

# Verify load
snowflake query "SELECT COUNT(*) FROM MY_TABLE" --connection $CONNECTION
```

##### **Example: Git Hook for Pre-Commit SQL Linting**
```bash
#!/bin/sh
# .git/hooks/pre-commit - Pre-commit hook for SQL linting

# Install SQLFluff if not present
if ! command -v sqlfluff &> /dev/null; then
    pip install sqlfluff --user
    export PATH=$PATH:$HOME/.local/bin
fi

# Lint all staged SQL files
git diff --cached --name-only | grep '\.sql$' | xargs sqlfluff lint --dialect snowflake

# Exit with error if linting fails
if [ $? -ne 0 ]; then
    echo "SQL linting failed. Commit aborted."
    exit 1
fi
```

##### **Example: CI/CD Pipeline with Snowflake CLI**
```yaml
# .github/workflows/snowflake-cli.yml
name: Snowflake CLI

on:
  push:
    branches: [ main ]
    paths:
      - 'scripts/**'

jobs:
  run-script:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Install Snowflake CLI
        run: |
          curl -O https://downloads.snowflake.com/snowflake-cli/latest/snowflake_linux_x86_64
          chmod +x snowflake_linux_x86_64
          sudo mv snowflake_linux_x86_64 /usr/local/bin/snowflake

      - name: Configure connection
        run: |
          snowflake connection add ci_conn --account ${{ secrets.SNOWFLAKE_ACCOUNT }} --username ${{ secrets.SNOWFLAKE_USER }} --password ${{ secrets.SNOWFLAKE_PASSWORD }} --warehouse ${{ secrets.SNOWFLAKE_WAREHOUSE }} --database ${{ secrets.SNOWFLAKE_DATABASE }} --schema ${{ secrets.SNOWFLAKE_SCHEMA }} --role ${{ secrets.SNOWFLAKE_ROLE }}

      - name: Run script
        run: |
          chmod +x scripts/load_data.sh
          ./scripts/load_data.sh
```

### **D. Third-Party Git Tools Integration**

#### **1. GitHub Integration**
##### **Features**
- **Pull Request Reviews**: Review Snowflake SQL code before merging.
- **Issue Tracking**: Track bugs and feature requests for Snowflake objects.
- **Projects**: Manage Snowflake development workflows.
- **Actions**: Automate Snowflake deployments and testing.

##### **When to Use**
✅ **Team collaboration** on Snowflake code
✅ **Code reviews** (pull requests)
✅ **Issue tracking** (bugs, feature requests)
✅ **CI/CD pipelines** (GitHub Actions)

##### **Configuration Example**
1. **Set Up GitHub Repository**:
   ```bash
   gh repo create my-snowflake-project --public --clone
   cd my-snowflake-project
   git init
   echo "# Snowflake Project" > README.md
   git add README.md
   git commit -m "Initial commit"
   git push -u origin main
   ```

2. **Connect Snowsight to GitHub**:
   - In Snowsight, navigate to **Worksheets**.
   - Click **Git icon** > **Connect to Git**.
   - Select **GitHub** and authenticate with OAuth.
   - Enter repository URL: `https://github.com/myorg/my-snowflake-project`.
   - Select branch: `main`.

3. **Set Up GitHub Actions**:
   - Create `.github/workflows/snowflake.yml` (see examples above).

#### **2. GitLab Integration**
##### **Features**
- **Merge Requests**: Review Snowflake SQL code before merging.
- **Issues**: Track bugs and feature requests.
- **CI/CD Pipelines**: Automate Snowflake deployments and testing.
- **Self-Hosted**: Option to self-host GitLab for compliance.

##### **When to Use**
✅ **Team collaboration** on Snowflake code
✅ **Code reviews** (merge requests)
✅ **CI/CD pipelines** (GitLab CI)
✅ **Self-hosted environments** (compliance, security)

##### **Configuration Example**
1. **Set Up GitLab Repository**:
   - Create a new project in GitLab.
   - Clone the repository:
     ```bash
     git clone git@gitlab.com:myorg/my-snowflake-project.git
     cd my-snowflake-project
     echo "# Snowflake Project" > README.md
     git add README.md
     git commit -m "Initial commit"
     git push -u origin main
     ```

2. **Connect Snowsight to GitLab**:
   - In Snowsight, navigate to **Worksheets**.
   - Click **Git icon** > **Connect to Git**.
   - Select **GitLab** and authenticate with OAuth or PAT.
   - Enter repository URL: `https://gitlab.com/myorg/my-snowflake-project`.
   - Select branch: `main`.

3. **Set Up GitLab CI**:
   - Create `.gitlab-ci.yml` (see examples above).

#### **3. Bitbucket Integration**
##### **Features**
- **Pull Requests**: Review Snowflake SQL code before merging.
- **Issues**: Track bugs and feature requests.
- **Pipelines**: Automate Snowflake deployments and testing.
- **Atlassian Ecosystem**: Integrate with Jira, Confluence, etc.

##### **When to Use**
✅ **Team collaboration** on Snowflake code
✅ **Atlassian ecosystem** (Jira, Confluence)
✅ **CI/CD pipelines** (Bitbucket Pipelines)

##### **Configuration Example**
1. **Set Up Bitbucket Repository**:
   - Create a new repository in Bitbucket.
   - Clone the repository:
     ```bash
     git clone git@bitbucket.org:myorg/my-snowflake-project.git
     cd my-snowflake-project
     echo "# Snowflake Project" > README.md
     git add README.md
     git commit -m "Initial commit"
     git push -u origin main
     ```

2. **Connect Snowsight to Bitbucket**:
   - In Snowsight, navigate to **Worksheets**.
   - Click **Git icon** > **Connect to Git**.
   - Select **Bitbucket** and authenticate with App Password.
   - Enter repository URL: `https://bitbucket.org/myorg/my-snowflake-project`.
   - Select branch: `main`.

3. **Set Up Bitbucket Pipelines**:
   - Create `bitbucket-pipelines.yml`:
     ```yaml
     image: atlassian/default-image:3

     pipelines:
       default:
         - step:
             name: Lint SQL
             script:
               - pip install sqlfluff
               - sqlfluff lint --dialect snowflake .
     ```

## **4. Comparison Matrix**

### **Storage API vs. Git Integrations Comparison**

| **Feature** | **Snowflake Storage API** | **PUT/GET Commands** | **Cloud Storage SDKs** | **Snowsight Git** | **CI/CD with Git** | **Snowflake CLI + Git** |
|-------------|----------------------------|----------------------|-------------------------|------------------|--------------------|-------------------------|
| **Purpose** | Programmatic file access | File transfer via SQL | Native cloud storage access | Version control for SQL objects | Automated testing/deployment | Scripted Snowflake operations |
| **Data Flow** | Bidirectional | Bidirectional | Bidirectional | Unidirectional (Git) | Bidirectional | Bidirectional |
| **Latency** | 100-500ms | 100-500ms | 100-500ms | 1-10 sec | 1-10 sec | 100-500ms |
| **Throughput** | 10-1000 MB/min | 10-1000 MB/min | 10-1000 MB/min | N/A | N/A | 10-1000 MB/min |
| **Serverless** | ✅ Yes | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No |
| **Managed By** | Snowflake | Snowflake | Cloud Provider | Snowflake | Git Provider | Client |
| **Cost Model** | API calls + cloud storage | Compute | Cloud storage | Git provider | Git provider + compute | Compute |
| **Best For** | Automated data pipelines, programmatic access | Ad-hoc file transfers, SQL-based workflows | High-volume file operations, resumable uploads | Code versioning, collaboration | Automated testing/deployment | Scripted workflows, local development |
| **Authentication** | JWT, OAuth, Cloud Credentials | Snowflake credentials | Cloud credentials | OAuth, PAT, SSH | OAuth, PAT, SSH | Snowflake credentials |
| **Use Cases** | Programmatic file access, cloud storage integration | Manual file transfers, batch loading | High-volume file operations, advanced cloud features | Team collaboration, code reviews | CI/CD pipelines, automated testing | Scripted workflows, local development |

### **Storage API Operations Comparison**

| **Operation** | **Snowflake Storage API** | **PUT/GET Commands** | **Cloud Storage SDKs** |
|---------------|----------------------------|----------------------|-------------------------|
| **List Files** | ✅ Yes | ✅ Yes (`LIST`) | ✅ Yes |
| **Upload File** | ✅ Yes | ✅ Yes (`PUT`) | ✅ Yes |
| **Download File** | ✅ Yes | ✅ Yes (`GET`) | ✅ Yes |
| **Delete File** | ✅ Yes | ✅ Yes (`REMOVE`) | ✅ Yes |
| **Get Metadata** | ✅ Yes | ❌ No | ✅ Yes |
| **Presigned URLs** | ✅ Yes | ❌ No | ✅ Yes |
| **Resumable Uploads** | ❌ No | ❌ No | ✅ Yes |
| **Multi-Part Uploads** | ❌ No | ❌ No | ✅ Yes |
| **Parallel Uploads** | ✅ Yes (limited) | ✅ Yes (`PARALLEL`) | ✅ Yes |
| **Encryption** | ✅ Yes (TLS) | ✅ Yes (TLS) | ✅ Yes (TLS + CMK) |
| **Compression** | ✅ Yes | ✅ Yes (`AUTO_COMPRESS`) | ✅ Yes |

### **Git Integrations Comparison**

| **Feature** | **Snowsight Git** | **CI/CD with Git** | **Snowflake CLI + Git** | **Third-Party Git Tools** |
|-------------|------------------|--------------------|-------------------------|---------------------------|
| **Version Control** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| **Code Reviews** | ✅ Yes (Pull Requests) | ✅ Yes (Pull Requests) | ❌ No | ✅ Yes (Pull Requests) |
| **CI/CD Pipelines** | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes |
| **Automated Testing** | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes |
| **Automated Deployment** | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes |
| **Infrastructure as Code** | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes |
| **Collaboration** | ✅ Yes | ✅ Yes | ❌ No | ✅ Yes |
| **Audit Trail** | ✅ Yes | ✅ Yes | ❌ No | ✅ Yes |
| **Supported Objects** | Worksheets, Stored Procedures, Functions, Views | All SQL objects | All SQL objects | All SQL objects |
| **Authentication** | OAuth, PAT, SSH | OAuth, PAT, SSH | Snowflake credentials | OAuth, PAT, SSH |
| **Hosted** | ✅ Yes (Snowflake) | ✅ Yes (Git Provider) | ❌ No (Client) | ✅ Yes (Git Provider) |
| **Best For** | Code versioning, collaboration | Automated testing/deployment | Scripted workflows, local development | Team collaboration, CI/CD |

## **5. Decision Flowchart**

### **Mermaid: Storage API and Git Integrations Decision Tree**
```mermaid
%% Storage API and Git Integrations Decision Tree
flowchart TD
    A[("Integration\nRequirement")] --> B{Data or Code?}
    B -->|Data| C[("Storage API?")]
    B -->|Code| D[("Git Integration?")]

    C -->|Programmatic Access| E[("Use Storage API\n(Preview)")]
    C -->|SQL-Based| F[("Use PUT/GET\n(Simple)")]
    C -->|High Volume| G[("Use Cloud Storage SDKs\n(Resumable Uploads)")]
    C -->|External Stages| H[("Use External Stage API\n(Stage Management)")]

    D -->|Version Control| I[("Use Snowsight Git\n(Native)")]
    D -->|CI/CD| J[("Use CI/CD with Git\n(GitHub Actions/GitLab CI)")]
    D -->|Scripting| K[("Use Snowflake CLI + Git\n(Automation)")]
    D -->|Third-Party Tools| L[("Use GitHub/GitLab/Bitbucket\n(Collaboration)")]

    E --> M{File Size?}
    M -->|< 5GB| N[("Direct Upload/Download")]
    M -->|> 5GB| O[("Presigned URLs")]

    F --> P{File Count?}
    P -->|< 100/day| Q[("PUT/GET Commands")]
    P -->|> 100/day| R[("Storage API or SDKs")]

    G --> S{Cloud Provider?}
    S -->|AWS| T[("AWS S3 SDK")]
    S -->|Azure| U[("Azure Blob SDK")]
    S -->|GCP| V[("GCS SDK")]

    I --> W{Team Size?}
    W -->|1-5| X[("Snowsight Git")]
    W -->|5-50| Y[("Snowsight Git + CI/CD")]
    W -->|50+| Z[("Snowsight Git + CI/CD + IaC")]

    J --> AA{Tool?}
    AA -->|GitHub| AB[("GitHub Actions")]
    AA -->|GitLab| AC[("GitLab CI")]
    AA -->|Azure DevOps| AD[("Azure DevOps")]
    AA -->|Jenkins| AE[("Jenkins")]

    K --> AF{Environment?}
    AF -->|Local| AG[("Snowflake CLI")]
    AF -->|CI/CD| AH[("Snowflake CLI + GitHub Actions")]

    L --> AI{Provider?}
    AI -->|GitHub| AJ[("GitHub")]
    AI -->|GitLab| AK[("GitLab")]
    AI -->|Bitbucket| AL[("Bitbucket")]

    %% --- Annotations ---
    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59,60,61,62,63,64,65,66,67,68,69,70,71,72,73,74,75,76,77,78,79,80,81,82,83,84,85,86,87,88,89,90,91,92,93,94,95,96,97,98,99,100 stroke:#333,stroke-width:2px;
    classDef default fill:#f9f9f9,stroke:#333;
    classDef data fill:#4285f4,stroke:#1976d2;
    classDef code fill:#f4511e,stroke:#d84315;
    classDef storage fill:#009688,stroke:#00796b;
    classDef git fill:#9c27b0,stroke:#7b1fa2;
    classDef size fill:#ff9800,stroke:#f57c00;
    classDef count fill:#00bcd4,stroke:#0097a7;
    classDef cloud fill:#e91e63,stroke:#c2185b;
    classDef team fill:#3f51b5,stroke:#303f9f;
    classDef tool fill:#673ab7,stroke:#5e35b1;
    classDef env fill:#ff5722,stroke:#e64a19;
    classDef provider fill:#795548,stroke:#5d4037;
    class C,E,F,G,H data;
    class D,I,J,K,L code;
    class M,N,O,P,Q,R storage;
    class S,T,U,V cloud;
    class W,X,Y,Z team;
    class AA,AB,AC,AD,AE tool;
    class AF,AG,AH env;
    class AI,AJ,AK,AL provider;
```

## **6. Key Engineering Principles**

### **A. Core Principles for Storage API Integrations**

1. **Programmatic First**:
   - Use **Storage API** for **automated workflows** where SQL commands (PUT/GET) are insufficient.
   - Reserve **PUT/GET** for **ad-hoc operations** or **SQL-based workflows**.

2. **Cloud-Native Access**:
   - For **high-volume operations**, use **cloud storage SDKs** (AWS S3, Azure Blob, GCS) for **resumable uploads** and **advanced features**.
   - Use **Snowflake Storage API** for **unified access** across cloud providers.

3. **Security by Default**:
   - Always use **TLS 1.2+** for all Storage API requests.
   - Use **JWT tokens** or **OAuth 2.0** for authentication (avoid hardcoded credentials).
   - Apply **RBAC** and **network policies** to restrict access.

4. **Error Handling and Retries**:
   - Implement **retry logic** with **exponential backoff** for transient errors.
   - Use **presigned URLs** for large file transfers to avoid timeouts.
   - Log **all operations** for debugging and auditing.

5. **Performance Optimization**:
   - Use **parallel uploads/downloads** for large datasets.
   - Enable **compression** to reduce data transfer size.
   - **Batch operations** to minimize API calls.

6. **Cost Awareness**:
   - Monitor **Storage API usage** (requests, data transfer).
   - Use **cloud storage lifecycle policies** to reduce costs (e.g., move old files to cold storage).
   - **Cache frequently accessed files** to reduce cloud storage requests.

7. **Idempotency**:
   - Design **idempotent operations** to handle retries safely.
   - Use **unique identifiers** for files to avoid duplicates.

### **B. Core Principles for Git Integrations**

1. **Version Everything**:
   - Store **all Snowflake SQL code** (worksheets, stored procedures, functions, views) in Git.
   - Use **branches** for feature development and **tags** for releases.

2. **Automate Testing and Deployment**:
   - Implement **CI/CD pipelines** to **lint, test, and deploy** Snowflake code.
   - Use **pre-commit hooks** for local validation (e.g., SQL linting).
   - **Test in staging** before deploying to production.

3. **Collaborate Effectively**:
   - Use **pull requests/merge requests** for code reviews.
   - Enforce **approvals** for production changes.
   - Document **changes** in commit messages and pull request descriptions.

4. **Infrastructure as Code (IaC)**:
   - Manage **Snowflake objects** (databases, schemas, tables, stages) as **Terraform/Pulumi code**.
   - Store **IaC templates** in Git and **version control** them.
   - Use **CI/CD pipelines** to apply IaC changes.

5. **Security and Compliance**:
   - Use **OAuth 2.0** or **Personal Access Tokens (PAT)** for Git authentication (avoid passwords).
   - Apply **least-privilege access** to Git repositories.
   - **Audit all changes** (who made changes, when, and why).

6. **Separation of Concerns**:
   - **Code**: Store in Git (worksheets, stored procedures, functions, views).
   - **Data**: Store in Snowflake tables (not in Git).
   - **Infrastructure**: Store as IaC (Terraform/Pulumi) in Git.

7. **Disaster Recovery**:
   - **Backup Git repositories** regularly.
   - Use **Git tags** to mark stable releases.
   - **Document recovery procedures** for Snowflake objects.

### **C. Combined Principles for Storage API + Git**

1. **End-to-End Automation**:
   - Use **Git** for **code versioning** and **CI/CD** for **testing/deployment**.
   - Use **Storage API** for **programmatic file access** in data pipelines.
   - Combine both for **fully automated workflows**.

2. **Immutable Artifacts**:
   - Store **SQL scripts** in Git as **immutable artifacts**.
   - Store **data files** in cloud storage (S3/Azure Blob/GCS) as **immutable objects**.
   - Use **versioning** (Git tags, cloud storage versioning) to track changes.

3. **Reproducible Workflows**:
   - **Version control** all components (code, data, infrastructure).
   - Use **deterministic scripts** (same input → same output).
   - **Document dependencies** (e.g., Snowflake version, cloud provider SDKs).

4. **Observability**:
   - **Log all operations** (Storage API calls, Git commits, CI/CD runs).
   - **Monitor performance** (latency, throughput, errors).
   - **Alert on failures** (e.g., failed Storage API calls, CI/CD pipeline errors).

5. **Scalability**:
   - Use **parallel processing** for high-volume Storage API operations.
   - Use **CI/CD pipelines** to scale testing and deployment.
   - **Partition data** in cloud storage for efficient access.

## **7. Production Checklist**

### **A. Storage API Integrations**

#### **General**
- [ ] **Assess Requirements**: Determine if **Storage API**, **PUT/GET**, or **cloud storage SDKs** are the best fit.
- [ ] **Review Limitations**: Understand **file size limits**, **rate limits**, and **unsupported features**.
- [ ] **Plan Authentication**: Choose **JWT**, **OAuth 2.0**, or **cloud credentials** for authentication.
- [ ] **Design Error Handling**: Implement **retry logic** with **exponential backoff** for transient errors.
- [ ] **Monitor Usage**: Set up **logging** and **monitoring** for Storage API operations.

#### **Snowflake Storage API (Preview)**
- [ ] **Request Access**: Contact Snowflake to **enable Storage API** for your account.
- [ ] **Set Up Authentication**:
  - Generate **JWT tokens** for service accounts.
  - Or, configure **OAuth 2.0** for user authentication.
- [ ] **Test Connectivity**:
  - Verify **API endpoints** are accessible.
  - Test **list, upload, download, delete** operations.
- [ ] **Implement Retries**:
  - Use **exponential backoff** for transient errors (429, 503).
  - Set **max retries** (e.g., 3) and **timeout** (e.g., 30 seconds).
- [ ] **Monitor Performance**:
  - Track **latency**, **throughput**, and **error rates**.
  - Set up **alerts** for failed operations.
- [ ] **Secure Access**:
  - Use **network policies** to restrict API access.
  - Use **PrivateLink/PSC** for private connectivity.

#### **PUT/GET Commands**
- [ ] **Set Up Stages**:
  - Create **internal stages** for temporary files.
  - Create **external stages** for cloud storage (S3, Azure Blob, GCS).
- [ ] **Configure Credentials**:
  - Use **IAM roles** (AWS), **managed identity** (Azure), or **service accounts** (GCP) for external stages.
  - Avoid **hardcoded credentials** (use environment variables or secret managers).
- [ ] **Optimize Performance**:
  - Use **`AUTO_COMPRESS = TRUE`** for compression.
  - Use **`PARALLEL = 10`** for parallel uploads/downloads.
  - Use **`OVERWRITE = TRUE`** to replace existing files.
- [ ] **Monitor Usage**:
  - Track **PUT/GET operations** in `ACCOUNT_USAGE.QUERY_HISTORY`.
  - Set up **alerts** for failed operations.
- [ ] **Secure Access**:
  - Use **RBAC** to restrict stage access.
  - Use **network policies** to restrict IP access.

#### **Cloud Storage SDKs**
- [ ] **Choose SDK**:
  - **AWS S3 SDK** for S3.
  - **Azure Blob SDK** for Azure Blob Storage.
  - **GCS SDK** for Google Cloud Storage.
- [ ] **Configure Credentials**:
  - Use **IAM roles** (AWS), **managed identity** (Azure), or **service accounts** (GCP).
  - Store credentials **securely** (e.g., AWS Secrets Manager, Azure Key Vault, GCP Secret Manager).
- [ ] **Implement Resumable Uploads**:
  - Use **multipart uploads** (AWS S3) for files >5GB.
  - Use **block blob uploads** (Azure Blob) for resumable uploads.
  - Use **resumable uploads** (GCS) for large files.
- [ ] **Optimize Performance**:
  - Use **parallel uploads/downloads** for large datasets.
  - Enable **compression** (e.g., Gzip, Snappy).
  - Use **batch operations** to minimize API calls.
- [ ] **Monitor Usage**:
  - Track **SDK operations** (uploads, downloads, errors).
  - Set up **alerts** for failed operations.
- [ ] **Secure Access**:
  - Use **TLS 1.2+** for all SDK requests.
  - Use **private connectivity** (PrivateLink, Private Service Connect) for sensitive data.

#### **External Stage API**
- [ ] **Automate Stage Management**:
  - Use **SQL scripts** or **REST API** to create/manage stages.
  - Store **stage definitions** in Git for version control.
- [ ] **Dynamic Stage Creation**:
  - Create **stages on-the-fly** for temporary workflows.
  - Use **environment variables** for stage configurations.
- [ ] **Monitor Stage Usage**:
  - Track **stage operations** (file uploads/downloads).
  - Set up **alerts** for failed operations.
- [ ] **Secure Stage Access**:
  - Use **RBAC** to restrict stage access.
  - Use **network policies** to restrict IP access.

### **B. Git Integrations**

#### **General**
- [ ] **Assess Requirements**: Determine if **Snowsight Git**, **CI/CD**, or **Snowflake CLI + Git** is the best fit.
- [ ] **Review Limitations**: Understand **supported object types**, **Git provider dependencies**, and **conflict resolution**.
- [ ] **Plan Authentication**: Choose **OAuth 2.0**, **PAT**, or **SSH keys** for Git authentication.
- [ ] **Design Workflow**: Define **branching strategy** (e.g., Git Flow, trunk-based development).
- [ ] **Document Processes**: Document **code review**, **testing**, and **deployment** processes.

#### **Snowsight Git**
- [ ] **Set Up Git Provider**:
  - Create **GitHub/GitLab/Bitbucket repositories**.
  - Configure **OAuth apps** or **PATs** for authentication.
- [ ] **Connect Snowsight**:
  - Connect **Snowsight to Git** for each user.
  - Test **push/pull** operations.
- [ ] **Define Branching Strategy**:
  - Use **`main`** for production code.
  - Use **feature branches** (e.g., `feature/*`) for development.
  - Use **release branches** (e.g., `release/*`) for staging.
- [ ] **Enforce Code Reviews**:
  - Require **pull requests/merge requests** for `main` branch.
  - Require **approvals** from at least one team member.
- [ ] **Monitor Usage**:
  - Track **Git operations** (pushes, pulls, conflicts).
  - Set up **alerts** for failed operations.

#### **CI/CD with Git**
- [ ] **Choose CI/CD Tool**:
  - **GitHub Actions** for GitHub repositories.
  - **GitLab CI** for GitLab repositories.
  - **Azure DevOps** for Azure repositories.
  - **Jenkins** for self-hosted CI/CD.
- [ ] **Set Up CI/CD Pipeline**:
  - Configure **triggers** (push, pull request, schedule).
  - Define **jobs** (linting, testing, deployment).
  - Store **secrets** securely (e.g., Snowflake credentials).
- [ ] **Implement Testing**:
  - **Lint SQL** (SQLFluff, Snowflake Linter).
  - **Validate SQL** (syntax, schema, permissions).
  - **Run Unit Tests** (pytest, custom scripts).
  - **Run Integration Tests** (end-to-end workflows).
- [ ] **Implement Deployment**:
  - **Deploy to Dev/Test**: On push to feature branches.
  - **Deploy to Staging**: On pull request merge to `main`.
  - **Deploy to Production**: On push to `main` (with approvals).
- [ ] **Monitor CI/CD**:
  - Track **pipeline runs** (success/failure rates).
  - Set up **alerts** for failed pipelines.
  - **Log all deployments** for audit trails.

#### **Snowflake CLI + Git**
- [ ] **Install Snowflake CLI**:
  - Install **Snowflake CLI** (`snowflake` or `snowsql`).
  - Configure **connection profiles** for different environments.
- [ ] **Script Workflows**:
  - Write **shell scripts** or **Python scripts** for Snowflake operations.
  - Store scripts in **Git repositories**.
- [ ] **Automate with Git Hooks**:
  - Use **pre-commit hooks** for SQL linting.
  - Use **post-commit hooks** for automated deployments.
- [ ] **Integrate with CI/CD**:
  - Run **Snowflake CLI commands** in CI/CD pipelines.
  - Use **environment variables** for credentials.
- [ ] **Monitor Scripts**:
  - Track **script executions** (success/failure rates).
  - Set up **alerts** for failed scripts.

#### **Third-Party Git Tools**
- [ ] **Set Up Git Provider**:
  - **GitHub**: Create repositories, configure OAuth apps.
  - **GitLab**: Create repositories, configure CI/CD.
  - **Bitbucket**: Create repositories, configure pipelines.
- [ ] **Integrate with Snowsight**:
  - Connect **Snowsight to Git** for version control.
  - Test **push/pull** operations.
- [ ] **Set Up CI/CD**:
  - Configure **pipelines** for testing and deployment.
  - Store **secrets** securely.
- [ ] **Enforce Policies**:
  - Require **code reviews** for `main` branch.
  - Require **approvals** for production deployments.
- [ ] **Monitor Usage**:
  - Track **Git operations** (pushes, pulls, merges).
  - Set up **alerts** for failed operations.

## **8. Production-Ready Snippets**

### **A. Storage API Snippets**

#### **1. Python: Full Storage API Workflow**
```python
import requests
import jwt
import time
import os
from cryptography.hazmat.primitives import serialization

# Configuration
ACCOUNT = os.getenv('SNOWFLAKE_ACCOUNT')
USER = os.getenv('SNOWFLAKE_USER')
PRIVATE_KEY_PATH = os.getenv('SNOWFLAKE_PRIVATE_KEY_PATH')
STAGE_NAME = 'MY_STAGE'
FILE_PATH = 'data.csv'
LOCAL_FILE_PATH = '/local/path/data.csv'

# Load private key
with open(PRIVATE_KEY_PATH, 'rb') as key:
    private_key = serialization.load_pem_private_key(
        key.read(),
        password=None
    )

# Generate JWT token
def generate_jwt_token():
    payload = {
        'iss': f'{USER}@{ACCOUNT}',
        'sub': f'{USER}@{ACCOUNT}',
        'iat': int(time.time()),
        'exp': int(time.time()) + 3600,
        'scope': 'session:role:MY_ROLE'
    }
    token = jwt.encode(payload, private_key, algorithm='RS256')
    return token

# Get session token
def get_session_token():
    jwt_token = generate_jwt_token()
    url = f'https://{ACCOUNT}.snowflakecomputing.com/api/v2/authentication/request'
    headers = {
        'X-Snowflake-Authorization-Token-Type': 'KEYPAIR_JWT',
        'Content-Type': 'application/json'
    }
    data = {
        'token': jwt_token,
        'warehouse': 'MY_WH',
        'database': 'MY_DB',
        'schema': 'MY_SCHEMA',
        'role': 'MY_ROLE'
    }
    response = requests.post(url, headers=headers, json=data)
    response.raise_for_status()
    return response.json()['data']['sessionToken']

# List stages
def list_stages(session_token):
    url = f'https://{ACCOUNT}.snowflakecomputing.com/api/v2/storage/stages'
    headers = {
        'Authorization': f'Bearer {session_token}',
        'Content-Type': 'application/json'
    }
    params = {'type': 'INTERNAL'}
    response = requests.get(url, headers=headers, params=params)
    response.raise_for_status()
    return response.json()['data']

# List files in stage
def list_files(session_token, stage_name, prefix=''):
    url = f'https://{ACCOUNT}.snowflakecomputing.com/api/v2/storage/stages/{stage_name}/files'
    headers = {
        'Authorization': f'Bearer {session_token}',
        'Content-Type': 'application/json'
    }
    params = {'prefix': prefix, 'limit': 1000}
    response = requests.get(url, headers=headers, params=params)
    response.raise_for_status()
    return response.json()['data']

# Upload file
def upload_file(session_token, stage_name, file_path, file_content):
    url = f'https://{ACCOUNT}.snowflakecomputing.com/api/v2/storage/stages/{stage_name}/files/{file_path}'
    headers = {
        'Authorization': f'Bearer {session_token}',
        'Content-Type': 'application/octet-stream'
    }
    params = {'overwrite': 'true'}
    response = requests.put(url, headers=headers, params=params, data=file_content)
    response.raise_for_status()
    return response.json()['data']

# Download file
def download_file(session_token, stage_name, file_path):
    url = f'https://{ACCOUNT}.snowflakecomputing.com/api/v2/storage/stages/{stage_name}/files/{file_path}'
    headers = {
        'Authorization': f'Bearer {session_token}'
    }
    response = requests.get(url, headers=headers)
    response.raise_for_status()
    return response.content

# Generate presigned URL for download
def generate_presigned_url(session_token, stage_name, file_path, expiry=3600):
    url = f'https://{ACCOUNT}.snowflakecomputing.com/api/v2/storage/stages/{stage_name}/files/{file_path}/download'
    headers = {
        'Authorization': f'Bearer {session_token}',
        'Content-Type': 'application/json'
    }
    params = {'expiry': expiry}
    response = requests.get(url, headers=headers, params=params)
    response.raise_for_status()
    return response.json()['data']['presignedUrl']

# Main workflow
def main():
    session_token = get_session_token()

    # List stages
    stages = list_stages(session_token)
    print("Stages:", stages)

    # List files in stage
    files = list_files(session_token, STAGE_NAME)
    print("Files in stage:", files)

    # Upload file
    with open(LOCAL_FILE_PATH, 'rb') as f:
        file_content = f.read()
    upload_result = upload_file(session_token, STAGE_NAME, FILE_PATH, file_content)
    print("Upload result:", upload_result)

    # Download file
    downloaded_content = download_file(session_token, STAGE_NAME, FILE_PATH)
    with open(f'/local/path/{FILE_PATH}_downloaded', 'wb') as f:
        f.write(downloaded_content)
    print("File downloaded")

    # Generate presigned URL
    presigned_url = generate_presigned_url(session_token, STAGE_NAME, FILE_PATH)
    print("Presigned URL:", presigned_url)

if __name__ == '__main__':
    main()
```

#### **2. Bash: PUT/GET Workflow with Error Handling**
```bash
#!/bin/bash
# snowflake_put_get.sh - PUT/GET workflow with error handling

# Configuration
CONNECTION="my_conn"
STAGE="MY_STAGE"
LOCAL_FILE="/local/path/data.csv"
REMOTE_FILE="data.csv"
LOG_FILE="/var/log/snowflake_put_get.log"

# Function to log messages
log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a $LOG_FILE
}

# Function to handle errors
handle_error() {
    log "ERROR: $1"
    exit 1
}

# Upload file
upload_file() {
    log "Uploading $LOCAL_FILE to @$STAGE/$REMOTE_FILE..."
    if ! snowflake stage upload @$STAGE $LOCAL_FILE --connection $CONNECTION; then
        handle_error "Failed to upload $LOCAL_FILE"
    fi
    log "Successfully uploaded $LOCAL_FILE"
}

# Download file
download_file() {
    log "Downloading @$STAGE/$REMOTE_FILE to $LOCAL_FILE..."
    if ! snowflake stage download @$STAGE/$REMOTE_FILE $LOCAL_FILE --connection $CONNECTION; then
        handle_error "Failed to download $REMOTE_FILE"
    fi
    log "Successfully downloaded $REMOTE_FILE"
}

# List files
list_files() {
    log "Listing files in @$STAGE..."
    if ! snowflake stage list @$STAGE --connection $CONNECTION; then
        handle_error "Failed to list files in @$STAGE"
    fi
}

# Main workflow
main() {
    # Upload file
    upload_file

    # List files
    list_files

    # Download file
    download_file
}

# Run main workflow
main
```

#### **3. Python: AWS S3 SDK with Resumable Uploads**
```python
import boto3
import os
from botocore.exceptions import ClientError

# Configuration
BUCKET_NAME = 'my-bucket'
STAGE_PATH = 'snowflake-stage/'
FILE_PATH = '/local/path/large_file.csv'  # >5GB
S3_KEY = f'{STAGE_PATH}large_file.csv'
CHUNK_SIZE = 8 * 1024 * 1024  # 8MB chunks

# Initialize S3 client
s3 = boto3.client(
    's3',
    aws_access_key_id=os.getenv('AWS_ACCESS_KEY_ID'),
    aws_secret_access_key=os.getenv('AWS_SECRET_ACCESS_KEY'),
    region_name='us-east-1'
)

def multipart_upload(file_path, bucket_name, s3_key):
    """
    Upload a large file to S3 using multipart upload.
    """
    try:
        # Initiate multipart upload
        mpu = s3.create_multipart_upload(
            Bucket=bucket_name,
            Key=s3_key,
            ServerSideEncryption='aws:kms',
            SSEKMSKeyId=os.getenv('AWS_KMS_KEY_ID')
        )
        mpu_id = mpu['UploadId']
        log(f"Initiated multipart upload for {s3_key} with ID {mpu_id}")

        # Upload parts
        part_tags = []
        with open(file_path, 'rb') as f:
            part_number = 1
            while True:
                data = f.read(CHUNK_SIZE)
                if not data:
                    break
                part = s3.upload_part(
                    Bucket=bucket_name,
                    Key=s3_key,
                    PartNumber=part_number,
                    UploadId=mpu_id,
                    Body=data
                )
                part_tags.append({'PartNumber': part_number, 'ETag': part['ETag']})
                log(f"Uploaded part {part_number} for {s3_key}")
                part_number += 1

        # Complete multipart upload
        s3.complete_multipart_upload(
            Bucket=bucket_name,
            Key=s3_key,
            UploadId=mpu_id,
            MultipartUpload={'Parts': part_tags}
        )
        log(f"Completed multipart upload for {s3_key}")
        return True
    except ClientError as e:
        log(f"Error uploading {s3_key}: {e}")
        # Abort multipart upload on error
        s3.abort_multipart_upload(
            Bucket=bucket_name,
            Key=s3_key,
            UploadId=mpu_id
        )
        return False

def download_file(bucket_name, s3_key, file_path):
    """
    Download a file from S3.
    """
    try:
        s3.download_file(
            bucket_name,
            s3_key,
            file_path
        )
        log(f"Downloaded {s3_key} to {file_path}")
        return True
    except ClientError as e:
        log(f"Error downloading {s3_key}: {e}")
        return False

def list_files(bucket_name, prefix=''):
    """
    List files in S3 bucket with prefix.
    """
    try:
        paginator = s3.get_paginator('list_objects_v2')
        for page in paginator.paginate(Bucket=bucket_name, Prefix=prefix):
            for obj in page.get('Contents', []):
                log(f"File: {obj['Key']}, Size: {obj['Size']} bytes")
        return True
    except ClientError as e:
        log(f"Error listing files in {bucket_name}: {e}")
        return False

def log(message):
    """
    Log a message.
    """
    print(message)

# Main workflow
if __name__ == '__main__':
    # Upload file
    if not multipart_upload(FILE_PATH, BUCKET_NAME, S3_KEY):
        exit(1)

    # List files
    if not list_files(BUCKET_NAME, STAGE_PATH):
        exit(1)

    # Download file
    if not download_file(BUCKET_NAME, S3_KEY, '/local/path/large_file_downloaded.csv'):
        exit(1)
```

#### **4. Python: External Stage API Workflow**
```python
import snowflake.connector
import os

# Configuration
CONN_PARAMS = {
    'user': os.getenv('SNOWFLAKE_USER'),
    'password': os.getenv('SNOWFLAKE_PASSWORD'),
    'account': os.getenv('SNOWFLAKE_ACCOUNT'),
    'warehouse': os.getenv('SNOWFLAKE_WAREHOUSE'),
    'database': os.getenv('SNOWFLAKE_DATABASE'),
    'schema': os.getenv('SNOWFLAKE_SCHEMA'),
    'role': os.getenv('SNOWFLAKE_ROLE')
}

STAGE_NAME = 'MY_S3_STAGE'
S3_BUCKET = 'my-bucket'
S3_PATH = 'snowflake-data/'
AWS_KEY_ID = os.getenv('AWS_KEY_ID')
AWS_SECRET_KEY = os.getenv('AWS_SECRET_KEY')

# Connect to Snowflake
def get_connection():
    return snowflake.connector.connect(**CONN_PARAMS)

# Create external stage
def create_external_stage(conn, stage_name, s3_bucket, s3_path, aws_key_id, aws_secret_key):
    cursor = conn.cursor()
    try:
        cursor.execute(f"""
            CREATE STAGE IF NOT EXISTS {stage_name}
            URL = 's3://{s3_bucket}/{s3_path}'
            CREDENTIALS = (AWS_KEY_ID = '{aws_key_id}' AWS_SECRET_KEY = '{aws_secret_key}')
            FILE_FORMAT = (TYPE = 'PARQUET')
        """)
        log(f"Created external stage {stage_name}")
        return True
    except Exception as e:
        log(f"Error creating stage {stage_name}: {e}")
        return False

# List stages
def list_stages(conn):
    cursor = conn.cursor()
    try:
        cursor.execute("SHOW STAGES")
        stages = cursor.fetchall()
        for stage in stages:
            log(f"Stage: {stage[0]}, Type: {stage[1]}, URL: {stage[2]}")
        return True
    except Exception as e:
        log(f"Error listing stages: {e}")
        return False

# Describe stage
def describe_stage(conn, stage_name):
    cursor = conn.cursor()
    try:
        cursor.execute(f"DESCRIBE STAGE {stage_name}")
        stage_details = cursor.fetchall()
        for detail in stage_details:
            log(f"{detail[0]}: {detail[1]}")
        return True
    except Exception as e:
        log(f"Error describing stage {stage_name}: {e}")
        return False

# List files in stage
def list_files_in_stage(conn, stage_name):
    cursor = conn.cursor()
    try:
        cursor.execute(f"LIST @{stage_name}")
        files = cursor.fetchall()
        for file in files:
            log(f"File: {file[0]}, Size: {file[1]}, Last Modified: {file[2]}")
        return True
    except Exception as e:
        log(f"Error listing files in stage {stage_name}: {e}")
        return False

# Alter stage
def alter_stage(conn, stage_name, new_s3_path):
    cursor = conn.cursor()
    try:
        cursor.execute(f"""
            ALTER STAGE {stage_name}
            SET URL = 's3://{S3_BUCKET}/{new_s3_path}'
        """)
        log(f"Altered stage {stage_name} to new path {new_s3_path}")
        return True
    except Exception as e:
        log(f"Error altering stage {stage_name}: {e}")
        return False

# Drop stage
def drop_stage(conn, stage_name):
    cursor = conn.cursor()
    try:
        cursor.execute(f"DROP STAGE IF EXISTS {stage_name}")
        log(f"Dropped stage {stage_name}")
        return True
    except Exception as e:
        log(f"Error dropping stage {stage_name}: {e}")
        return False

def log(message):
    print(message)

# Main workflow
def main():
    conn = None
    try:
        conn = get_connection()

        # Create external stage
        if not create_external_stage(conn, STAGE_NAME, S3_BUCKET, S3_PATH, AWS_KEY_ID, AWS_SECRET_KEY):
            exit(1)

        # List stages
        if not list_stages(conn):
            exit(1)

        # Describe stage
        if not describe_stage(conn, STAGE_NAME):
            exit(1)

        # List files in stage
        if not list_files_in_stage(conn, STAGE_NAME):
            exit(1)

        # Alter stage
        if not alter_stage(conn, STAGE_NAME, 'new-snowflake-data/'):
            exit(1)

        # Drop stage
        if not drop_stage(conn, STAGE_NAME):
            exit(1)

    finally:
        if conn:
            conn.close()

if __name__ == '__main__':
    main()
```

### **B. Git Integrations Snippets**

#### **1. Snowsight Git Workflow**
```markdown
# Snowsight Git Workflow

## 1. Set Up Git Repository
1. Create a new repository in GitHub/GitLab/Bitbucket:
   ```bash
   gh repo create my-snowflake-project --public --clone
   cd my-snowflake-project
   git init
   echo "# Snowflake Project" > README.md
   git add README.md
   git commit -m "Initial commit"
   git push -u origin main
   ```

2. Configure Git provider authentication:
   - **GitHub**: Create a Personal Access Token (PAT) with `repo` permissions.
   - **GitLab**: Create a PAT with `api` and `read_repository`/`write_repository` permissions.
   - **Bitbucket**: Create an App Password with `read`/`write` permissions.

## 2. Connect Snowsight to Git
1. Open Snowsight and navigate to **Worksheets**.
2. Click the **Git icon** in the top-right corner.
3. Select **Connect to Git**.
4. Choose your Git provider (GitHub, GitLab, Bitbucket).
5. Authenticate with OAuth or PAT.
6. Enter the repository URL (e.g., `https://github.com/myorg/my-snowflake-project`).
7. Select the branch (e.g., `main`).
8. Click **Connect**.

## 3. Work with Git in Snowsight
### Push Changes
1. Make changes to a worksheet, stored procedure, function, or view.
2. Click the **Git icon** in the top-right corner.
3. Enter a **commit message** (e.g., "Add user events query").
4. Click **Push** to save changes to Git.

### Pull Changes
1. Click the **Git icon** in the top-right corner.
2. Click **Pull** to load the latest changes from Git.
3. Resolve any **merge conflicts** if they exist.

### Switch Branches
1. Click the **Git icon** in the top-right corner.
2. Select **Switch Branch**.
3. Choose an existing branch or create a new one.

## 4. Collaborate with Team
1. Team members **clone the repository** and connect Snowsight.
2. Make changes and **push to feature branches** (e.g., `feature/add-user-events`).
3. Submit **pull requests/merge requests** for code reviews.
4. **Approve and merge** changes into `main` after review.

## 5. Best Practices
- Use **feature branches** for development.
- **Rebase or merge** feature branches into `main`.
- **Squash commits** to keep history clean.
- **Tag releases** (e.g., `v1.0.0`) for stable versions.
- **Document changes** in commit messages and pull request descriptions.
```

#### **2. GitHub Actions: Snowflake SQL Linting + Testing + Deployment**
```yaml
# .github/workflows/snowflake-ci-cd.yml
name: Snowflake CI/CD

on:
  push:
    branches: [ main ]
    paths:
      - '**.sql'
      - '.github/workflows/snowflake-ci-cd.yml'
  pull_request:
    branches: [ main ]
    paths:
      - '**.sql'

env:
  SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
  SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
  SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}
  SNOWFLAKE_WAREHOUSE: ${{ secrets.SNOWFLAKE_WAREHOUSE }}
  SNOWFLAKE_DATABASE: ${{ secrets.SNOWFLAKE_DATABASE }}
  SNOWFLAKE_SCHEMA: ${{ secrets.SNOWFLAKE_SCHEMA }}
  SNOWFLAKE_ROLE: ${{ secrets.SNOWFLAKE_ROLE }}

jobs:
  lint:
    name: Lint SQL
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install sqlfluff

      - name: Lint SQL files
        run: |
          sqlfluff lint --dialect snowflake --rules L001,L003,L004,L009,L010,L014,L016,L021,L024,L025,L026,L027,L028,L029,L030,L031,L032,L033,L034,L035,L036,L037,L038,L039,L040,L042,L043,L044 . || exit 1

  test:
    name: Test SQL
    needs: lint
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install snowflake-connector-python pytest

      - name: Run tests
        run: |
          pytest tests/ --tb=short

  deploy-dev:
    name: Deploy to Dev
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install snowflake-connector-python

      - name: Deploy to Dev
        run: |
          python deploy.py --environment dev

  deploy-prod:
    name: Deploy to Prod
    needs: deploy-dev
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install snowflake-connector-python

      - name: Deploy to Prod
        run: |
          python deploy.py --environment prod
```

```python
# deploy.py
import snowflake.connector
import os
import glob
import argparse

# Parse command-line arguments
parser = argparse.ArgumentParser(description='Deploy Snowflake objects')
parser.add_argument('--environment', required=True, choices=['dev', 'prod'], help='Environment (dev/prod)')
args = parser.parse_args()

# Configuration
ENV_CONFIG = {
    'dev': {
        'warehouse': os.getenv('SNOWFLAKE_DEV_WAREHOUSE', os.getenv('SNOWFLAKE_WAREHOUSE')),
        'database': os.getenv('SNOWFLAKE_DEV_DATABASE', os.getenv('SNOWFLAKE_DATABASE')),
        'schema': os.getenv('SNOWFLAKE_DEV_SCHEMA', os.getenv('SNOWFLAKE_SCHEMA')),
        'role': os.getenv('SNOWFLAKE_DEV_ROLE', os.getenv('SNOWFLAKE_ROLE'))
    },
    'prod': {
        'warehouse': os.getenv('SNOWFLAKE_PROD_WAREHOUSE'),
        'database': os.getenv('SNOWFLAKE_PROD_DATABASE'),
        'schema': os.getenv('SNOWFLAKE_PROD_SCHEMA'),
        'role': os.getenv('SNOWFLAKE_PROD_ROLE')
    }
}

# Connect to Snowflake
def get_connection(env):
    return snowflake.connector.connect(
        user=os.getenv('SNOWFLAKE_USER'),
        password=os.getenv('SNOWFLAKE_PASSWORD'),
        account=os.getenv('SNOWFLAKE_ACCOUNT'),
        warehouse=ENV_CONFIG[env]['warehouse'],
        database=ENV_CONFIG[env]['database'],
        schema=ENV_CONFIG[env]['schema'],
        role=ENV_CONFIG[env]['role']
    )

# Deploy stored procedures
def deploy_stored_procedures(conn):
    cursor = conn.cursor()
    for sp_file in glob.glob('stored_procedures/*.sql'):
        with open(sp_file, 'r') as f:
            sp_sql = f.read()
        sp_name = os.path.basename(sp_file).replace('.sql', '')
        try:
            cursor.execute(f"CREATE OR REPLACE PROCEDURE {sp_name}() AS $$ {sp_sql} $$")
            print(f"Deployed stored procedure: {sp_name}")
        except Exception as e:
            print(f"Error deploying {sp_name}: {e}")
            raise

# Deploy functions
def deploy_functions(conn):
    cursor = conn.cursor()
    for func_file in glob.glob('functions/*.sql'):
        with open(func_file, 'r') as f:
            func_sql = f.read()
        func_name = os.path.basename(func_file).replace('.sql', '')
        try:
            cursor.execute(f"CREATE OR REPLACE FUNCTION {func_name}() AS $$ {func_sql} $$")
            print(f"Deployed function: {func_name}")
        except Exception as e:
            print(f"Error deploying {func_name}: {e}")
            raise

# Deploy views
def deploy_views(conn):
    cursor = conn.cursor()
    for view_file in glob.glob('views/*.sql'):
        with open(view_file, 'r') as f:
            view_sql = f.read()
        view_name = os.path.basename(view_file).replace('.sql', '')
        try:
            cursor.execute(f"CREATE OR REPLACE VIEW {view_name} AS {view_sql}")
            print(f"Deployed view: {view_name}")
        except Exception as e:
            print(f"Error deploying {view_name}: {e}")
            raise

# Main workflow
def main():
    env = args.environment
    conn = None
    try:
        conn = get_connection(env)
        print(f"Deploying to {env} environment...")

        # Deploy stored procedures
        deploy_stored_procedures(conn)

        # Deploy functions
        deploy_functions(conn)

        # Deploy views
        deploy_views(conn)

        print(f"Successfully deployed to {env} environment")
    finally:
        if conn:
            conn.close()

if __name__ == '__main__':
    main()
```

#### **3. GitLab CI: Snowflake Testing + Deployment**
```yaml
# .gitlab-ci.yml
stages:
  - lint
  - test
  - deploy-dev
  - deploy-prod

variables:
  SNOWFLAKE_ACCOUNT: $SNOWFLAKE_ACCOUNT
  SNOWFLAKE_USER: $SNOWFLAKE_USER
  SNOWFLAKE_PASSWORD: $SNOWFLAKE_PASSWORD
  SNOWFLAKE_DEV_WAREHOUSE: $SNOWFLAKE_DEV_WAREHOUSE
  SNOWFLAKE_DEV_DATABASE: $SNOWFLAKE_DEV_DATABASE
  SNOWFLAKE_DEV_SCHEMA: $SNOWFLAKE_DEV_SCHEMA
  SNOWFLAKE_DEV_ROLE: $SNOWFLAKE_DEV_ROLE
  SNOWFLAKE_PROD_WAREHOUSE: $SNOWFLAKE_PROD_WAREHOUSE
  SNOWFLAKE_PROD_DATABASE: $SNOWFLAKE_PROD_DATABASE
  SNOWFLAKE_PROD_SCHEMA: $SNOWFLAKE_PROD_SCHEMA
  SNOWFLAKE_PROD_ROLE: $SNOWFLAKE_PROD_ROLE

lint:
  stage: lint
  image: python:3.10
  before_script:
    - pip install sqlfluff
  script:
    - sqlfluff lint --dialect snowflake --rules L001,L003,L004,L009,L010,L014,L016,L021,L024,L025,L026,L027,L028,L029,L030,L031,L032,L033,L034,L035,L036,L037,L038,L039,L040,L042,L043,L044 .
  only:
    - merge_requests
    - main

test:
  stage: test
  image: python:3.10
  before_script:
    - pip install snowflake-connector-python pytest
  script:
    - pytest tests/ --tb=short
  only:
    - merge_requests
    - main

deploy-dev:
  stage: deploy-dev
  image: python:3.10
  before_script:
    - pip install snowflake-connector-python
  script:
    - python deploy.py --environment dev
  only:
    - main

deploy-prod:
  stage: deploy-prod
  image: python:3.10
  before_script:
    - pip install snowflake-connector-python
  script:
    - python deploy.py --environment prod
  when: manual
  only:
    - main
```

#### **4. Snowflake CLI + Git: Automated Data Loading**
```bash
#!/bin/bash
# snowflake_data_loading.sh - Automated data loading with Snowflake CLI + Git

# Configuration
CONNECTION="my_conn"
STAGE="MY_STAGE"
LOCAL_DATA_DIR="/local/path/data"
GIT_REPO_DIR="/local/path/my-snowflake-repo"
LOG_FILE="/var/log/snowflake_data_loading.log"

# Function to log messages
log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a $LOG_FILE
}

# Function to handle errors
handle_error() {
    log "ERROR: $1"
    git -C $GIT_REPO_DIR checkout main
    exit 1
}

# Function to load data
load_data() {
    local file=$1
    local file_name=$(basename "$file")

    log "Loading $file_name to @$STAGE..."
    if ! snowflake stage upload @$STAGE "$file" --connection $CONNECTION; then
        handle_error "Failed to upload $file_name"
    fi

    log "Copying $file_name to MY_TABLE..."
    if ! snowflake query "COPY INTO MY_TABLE FROM @$STAGE/$file_name FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1)" --connection $CONNECTION; then
        handle_error "Failed to copy $file_name to MY_TABLE"
    fi

    log "Successfully loaded $file_name"
}

# Function to commit changes to Git
commit_changes() {
    local message=$1
    git -C $GIT_REPO_DIR add .
    git -C $GIT_REPO_DIR commit -m "$message"
    git -C $GIT_REPO_DIR push
    log "Committed changes to Git: $message"
}

# Main workflow
main() {
    # Checkout feature branch
    git -C $GIT_REPO_DIR checkout -b "feature/data-loading-$(date +%Y%m%d%H%M%S)" || handle_error "Failed to checkout feature branch"

    # Load all CSV files in directory
    for file in $LOCAL_DATA_DIR/*.csv; do
        load_data "$file"
    done

    # Commit changes to Git
    commit_changes "Automated data loading: $(date '+%Y-%m-%d %H:%M:%S')"

    # Create pull request
    git -C $GIT_REPO_DIR push --set-upstream origin $(git -C $GIT_REPO_DIR branch --show-current)
    gh pr create --repo myorg/my-snowflake-repo --title "Automated Data Loading: $(date '+%Y-%m-%d')" --body "Automated data loading from $LOCAL_DATA_DIR" || log "Failed to create pull request (GH CLI not installed)"
}

# Run main workflow
main
```

#### **5. Terraform + GitHub Actions: Infrastructure as Code**
```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  push:
    branches: [ main ]
    paths:
      - 'terraform/**'
  pull_request:
    branches: [ main ]
    paths:
      - 'terraform/**'

env:
  TF_VAR_snowflake_account: ${{ secrets.SNOWFLAKE_ACCOUNT }}
  TF_VAR_snowflake_user: ${{ secrets.SNOWFLAKE_USER }}
  TF_VAR_snowflake_password: ${{ secrets.SNOWFLAKE_PASSWORD }}
  TF_VAR_aws_access_key: ${{ secrets.AWS_ACCESS_KEY_ID }}
  TF_VAR_aws_secret_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

jobs:
  terraform:
    name: Terraform
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: terraform

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.3.0

      - name: Terraform Init
        id: init
        run: terraform init

      - name: Terraform Format
        id: fmt
        run: terraform fmt -check

      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color
        continue-on-error: true

      - name: Terraform Apply (Main Branch)
        if: github.ref == 'refs/heads/main' && steps.plan.outcome == 'success'
        run: terraform apply -auto-approve

      - name: Terraform Plan (Pull Request)
        if: github.ref != 'refs/heads/main'
        run: terraform plan -no-color
```

```hcl
# terraform/main.tf
terraform {
  required_providers {
    snowflake = {
      source  = "Snowflake-Labs/snowflake"
      version = "~> 0.68"
    }
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "snowflake" {
  account  = var.snowflake_account
  user     = var.snowflake_user
  password = var.snowflake_password
}

provider "aws" {
  region     = "us-east-1"
  access_key = var.aws_access_key
  secret_key = var.aws_secret_key
}

# Create a database
resource "snowflake_database" "my_db" {
  name = "MY_DB"
}

# Create a schema
resource "snowflake_schema" "my_schema" {
  name     = "MY_SCHEMA"
  database = snowflake_database.my_db.name
}

# Create a warehouse
resource "snowflake_warehouse" "my_wh" {
  name           = "MY_WH"
  warehouse_size = "XSMALL"
  auto_suspend   = 60
}

# Create a role
resource "snowflake_role" "my_role" {
  name = "MY_ROLE"
}

# Grant privileges to role
resource "snowflake_role_grants" "my_role_grants" {
  role_name = snowflake_role.my_role.name
  privileges = [
    "USAGE ON WAREHOUSE ${snowflake_warehouse.my_wh.name}",
    "USAGE ON DATABASE ${snowflake_database.my_db.name}",
    "USAGE ON SCHEMA ${snowflake_schema.my_schema.name}",
    "CREATE TABLE ON SCHEMA ${snowflake_schema.my_schema.name}",
    "CREATE STAGE ON SCHEMA ${snowflake_schema.my_schema.name}"
  ]
}

# Create a user
resource "snowflake_user" "my_user" {
  name         = "MY_USER"
  display_name = "My User"
  password     = var.snowflake_password
  default_warehouse = snowflake_warehouse.my_wh.name
  default_namespace = "${snowflake_database.my_db.name}.${snowflake_schema.my_schema.name}"
  default_role = snowflake_role.my_role.name
}

# Grant role to user
resource "snowflake_role_membership" "my_user_role" {
  role_name = snowflake_role.my_role.name
  user_name = snowflake_user.my_user.name
}

# Create an external stage
resource "snowflake_stage" "my_stage" {
  name     = "MY_STAGE"
  database = snowflake_database.my_db.name
  schema   = snowflake_schema.my_schema.name
  url      = "s3://my-bucket/path/"
  credentials = {
    aws_key_id     = var.aws_access_key
    aws_secret_key = var.aws_secret_key
  }
  file_format = {
    type = "PARQUET"
  }
}

# Create a table
resource "snowflake_table" "my_table" {
  name     = "MY_TABLE"
  database = snowflake_database.my_db.name
  schema   = snowflake_schema.my_schema.name
  column {
    name = "ID"
    type = "INTEGER"
  }
  column {
    name = "NAME"
    type = "STRING"
  }
  column {
    name = "VALUE"
    type = "FLOAT"
  }
}

# Outputs
output "snowflake_account" {
  value = var.snowflake_account
}

output "database_name" {
  value = snowflake_database.my_db.name
}

output "schema_name" {
  value = snowflake_schema.my_schema.name
}

output "warehouse_name" {
  value = snowflake_warehouse.my_wh.name
}

output "stage_name" {
  value = snowflake_stage.my_stage.name
}

output "table_name" {
  value = snowflake_table.my_table.name
}
```

#### **6. Pre-Commit Hook for SQL Linting**
```bash
#!/bin/sh
# .git/hooks/pre-commit - Pre-commit hook for SQL linting

# Install SQLFluff if not present
if ! command -v sqlfluff &> /dev/null; then
    echo "Installing SQLFluff..."
    pip install sqlfluff --user
    export PATH=$PATH:$HOME/.local/bin
fi

# Lint all staged SQL files
STAGED_FILES=$(git diff --cached --name-only | grep '\.sql$')
if [ -z "$STAGED_FILES" ]; then
    exit 0
fi

echo "Linting staged SQL files..."
sqlfluff lint --dialect snowflake $STAGED_FILES

# Exit with error if linting fails
if [ $? -ne 0 ]; then
    echo ""
    echo "SQL linting failed. Commit aborted."
    echo "Fix the issues and try committing again."
    exit 1
fi

exit 0
```

## **9. Final Notes**

### **For Further Reading**
- [Snowflake Storage API Documentation (Preview)](https://docs.snowflake.com/en/developer-guide/storage-api)
- [Snowflake PUT/GET Commands](https://docs.snowflake.com/en/user-guide/data-load-local-file-system)
- [AWS S3 SDK for Python (Boto3)](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html)
- [Azure Blob Storage SDK for Python](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-python)
- [Google Cloud Storage SDK for Python](https://cloud.google.com/storage/docs/reference/libraries#client-libraries-install-python)
- [Snowsight Git Integration](https://docs.snowflake.com/en/user-guide/snowsight-git)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
- [Terraform Snowflake Provider](https://registry.terraform.io/providers/Snowflake-Labs/snowflake/latest/docs)
- [SQLFluff: The SQL Linter](https://www.sqlfluff.com/)

### **Open Questions for Your Environment**
1. **Storage API**:
   - Do you need **programmatic access** to Snowflake stages?
   - What **cloud storage providers** are you using (AWS, Azure, GCP)?
   - What **file sizes** and **volumes** do you expect to handle?
   - Do you need **resumable uploads** or **multi-part uploads**?

2. **PUT/GET Commands**:
   - Do you need **SQL-based file transfers** for ad-hoc operations?
   - What **compression** and **parallelism** settings are optimal for your workloads?
   - Do you need **automation** for PUT/GET operations?

3. **Cloud Storage SDKs**:
   - Do you need **advanced cloud storage features** (e.g., S3 Batch, Blob Lease)?
   - Do you need **resumable uploads** for large files?
   - What **cloud provider SDK** are you using (Boto3, Azure Blob, GCS)?

4. **Snowsight Git**:
   - Do you need **version control** for Snowflake SQL objects?
   - What **Git provider** are you using (GitHub, GitLab, Bitbucket)?
   - Do you need **code reviews** and **collaboration** features?

5. **CI/CD with Git**:
   - Do you need **automated testing** for Snowflake SQL code?
   - Do you need **automated deployment** to Snowflake?
   - What **CI/CD tool** are you using (GitHub Actions, GitLab CI, Azure DevOps, Jenkins)?

6. **Snowflake CLI + Git**:
   - Do you need **scripted workflows** for Snowflake operations?
   - Do you need **local development** with Snowflake CLI?
   - Do you need **Git hooks** for pre-commit validation?

7. **Security**:
   - What **authentication methods** do you use (JWT, OAuth, PAT, SSH)?
   - Do you need **private connectivity** (PrivateLink, Private Service Connect)?
   - Do you need **encryption** (CMK, TLS)?

8. **Performance**:
   - What **latency** and **throughput** requirements do you have?
   - Do you need **parallel uploads/downloads**?
   - Do you need **compression** for file transfers?

9. **Compliance**:
   - Do you have **compliance requirements** (HIPAA, GDPR, PCI DSS)?
   - Do you need **audit trails** for Storage API and Git operations?
   - Do you need **data retention policies**?

10. **Team Collaboration**:
    - How many **team members** need access to Storage API and Git?
    - Do you need **code reviews** and **approvals** for changes?
    - Do you need **documentation** for Storage API and Git workflows?
