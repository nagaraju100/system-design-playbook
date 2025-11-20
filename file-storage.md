# File Storage System Design

Design a distributed file storage system like Google Drive, Dropbox, or AWS S3 that can store and serve billions of files.

## Requirements

### Functional Requirements
- Upload files of any size
- Download files
- Delete files
- List files in directory
- Support file versioning
- Support file sharing
- Support file synchronization

### Non-Functional Requirements
- High availability (99.99% uptime)
- Durability (99.999999999% - 11 9's)
- Scalability (exabytes of storage)
- Low latency for file access
- Handle millions of files per second

## Capacity Estimation

### Scale Estimates
- 500 million users
- Average 100 files per user
- Total: 50 billion files
- Average file size: 100KB
- Total storage: 5PB
- Peak upload: 10,000 files/second
- Peak download: 100,000 files/second

## System APIs

```
# Upload file
POST /api/v1/files
Headers: Content-Type: multipart/form-data
Body: file data

Response: {
  "file_id": "file123",
  "url": "https://storage.com/files/file123"
}

# Download file
GET /api/v1/files/{file_id}

# Delete file
DELETE /api/v1/files/{file_id}

# List files
GET /api/v1/files?directory=/path&limit=100

# Share file
POST /api/v1/files/{file_id}/share
Body: {
  "permissions": "read" | "write",
  "expires_at": "2024-12-31"
}
```

## High-Level Design

```
┌─────────┐     ┌──────────────┐     ┌─────────────┐
│ Client  │────▶│ Load Balancer│────▶│ API Servers │
└─────────┘     └──────────────┘     └──────┬──────┘
                                             │
                    ┌────────────────────────┼────────────────────────┐
                    ▼                        ▼                        ▼
            ┌──────────────┐        ┌──────────────┐        ┌──────────────┐
            │  Metadata    │        │  Storage     │        │  CDN         │
            │  Service     │        │  Service     │        │  (Cache)     │
            └──────────────┘        └──────────────┘        └──────────────┘
                    │                        │
                    ▼                        ▼
            ┌──────────────┐        ┌──────────────┐
            │  Database    │        │  Object      │
            │  (File Info) │        │  Storage     │
            │              │        │  (S3, HDFS)  │
            └──────────────┘        └──────────────┘
```

## Detailed Design

### File Storage Architecture

**Metadata vs. Data Separation:**
- **Metadata Service**: Stores file information (name, size, path, permissions)
- **Storage Service**: Stores actual file data (binary content)
- Separation enables independent scaling

### Metadata Database

**File Metadata Table:**
```
- file_id: VARCHAR(64) (primary key)
- user_id: BIGINT
- file_name: VARCHAR(255)
- file_path: VARCHAR(1024)
- file_size: BIGINT
- content_type: VARCHAR(100)
- storage_path: VARCHAR(512)  # Path in object storage
- created_at: TIMESTAMP
- updated_at: TIMESTAMP
- version: INT
- INDEX (user_id, file_path)
```

**Directory Structure:**
- Store as tree structure in database
- Use materialized path or nested set model
- Enable fast directory listing

### Object Storage

**Storage Servers:**
- Store files as objects (blobs)
- Use distributed file system (HDFS) or object storage (S3)
- Replicate files across multiple servers
- Handle large files efficiently

**File Chunking:**
- Split large files into chunks (e.g., 64MB)
- Store chunks separately
- Enables parallel upload/download
- Better fault tolerance

### File Upload Flow

1. Client initiates upload
2. API server authenticates request
3. For large files:
   - Split into chunks
   - Upload chunks in parallel
   - Store chunk metadata
4. Store file metadata in database
5. Store file data in object storage
6. Replicate to multiple storage servers
7. Return file_id to client

### File Download Flow

1. Client requests file download
2. API server validates permissions
3. Lookup file metadata from database
4. Get file location from metadata
5. If file in CDN cache:
   - Serve from CDN
6. If not in cache:
   - Fetch from object storage
   - Serve to client
   - Optionally cache in CDN

### File Replication

**Replication Strategy:**
- Replicate each file to N servers (typically 3)
- Use quorum for reads/writes
- Automatic failover on server failure
- Geographic replication for disaster recovery

**Replication Levels:**
- **Same Data Center**: Low latency replication
- **Different Data Centers**: Disaster recovery
- **Different Regions**: Global availability

### Deduplication

**Content-Based Deduplication:**
- Hash file content (SHA-256)
- Store hash in metadata
- If hash exists, reuse existing file
- Save storage space

**Implementation:**
- Calculate hash during upload
- Check if hash exists in database
- If exists, create new metadata entry pointing to existing file
- If not, store new file

### File Versioning

**Version Storage:**
- Store each version as separate file
- Link versions in metadata
- Keep last N versions (configurable)
- Archive old versions to cheaper storage

**Version Metadata:**
```
- file_id: VARCHAR(64)
- version: INT
- storage_path: VARCHAR(512)
- created_at: TIMESTAMP
- created_by: BIGINT
```

### File Sharing

**Sharing Model:**
- Generate shareable link with token
- Store sharing permissions in database
- Support read-only and read-write access
- Optional expiration date

**Sharing Table:**
```
- share_id: VARCHAR(64)
- file_id: VARCHAR(64)
- token: VARCHAR(64) (unique)
- permissions: ENUM('read', 'write')
- created_by: BIGINT
- expires_at: TIMESTAMP
- access_count: INT
```

### File Synchronization

**Sync Strategy:**
- Client maintains local file cache
- Periodically sync with server
- Detect changes using file hashes
- Upload/download only changed files

**Change Detection:**
- Compare local file hash with server hash
- Upload if local hash differs
- Download if server hash differs
- Handle conflicts (last-write-wins or manual resolution)

## Storage Optimization

### Compression
- Compress files before storage
- Reduce storage and bandwidth
- Trade-off: CPU usage
- Good for text files, less effective for already compressed files

### Tiered Storage
- **Hot Storage**: Frequently accessed files (SSD)
- **Warm Storage**: Occasionally accessed files (HDD)
- **Cold Storage**: Rarely accessed files (tape, glacier)
- Move files between tiers based on access patterns

### CDN Integration
- Cache popular files in CDN
- Serve from edge locations
- Reduce origin server load
- Lower latency for global users

## Database Sharding

### Sharding Strategy

**Shard by User ID:**
- Distribute users across shards
- All user's files in same shard
- Simple but may cause hotspots

**Shard by File ID:**
- Hash file_id to determine shard
- Even distribution
- Need to query multiple shards for user's files

**Hybrid Approach:**
- Shard metadata by user_id
- Shard storage by file_id
- Balance between query efficiency and distribution

## Performance Optimizations

### Parallel Upload/Download
- Split large files into chunks
- Upload/download chunks in parallel
- Faster transfer for large files
- Better fault tolerance

### Caching
- Cache file metadata in Redis
- Cache frequently accessed files
- Reduce database and storage load

### Batch Operations
- Batch metadata updates
- Batch file operations
- Reduce overhead

## Security Considerations

### Encryption
- **Encryption at Rest**: Encrypt files in storage
- **Encryption in Transit**: Use HTTPS/TLS
- **Client-Side Encryption**: Encrypt before upload

### Access Control
- User authentication
- File-level permissions
- Role-based access control (RBAC)
- Audit logging

### Data Integrity
- Verify file integrity using checksums
- Detect corruption
- Automatic repair using replicas

## Monitoring and Metrics

### Key Metrics
- Upload/download latency
- Storage utilization
- Replication lag
- Cache hit ratio
- Error rates
- Throughput (files per second)

### Alerts
- High storage utilization
- Replication failures
- High error rates
- Slow upload/download
- Storage server failures

## Use Cases

1. **Cloud Storage**: Personal file storage (Google Drive, Dropbox)
2. **Object Storage**: Application data storage (AWS S3)
3. **Backup Storage**: System backups
4. **Content Delivery**: Media file storage and delivery
5. **Archive Storage**: Long-term data archival

## Popular File Storage Systems

- **AWS S3**: Object storage service
- **Google Cloud Storage**: Object storage
- **Azure Blob Storage**: Object storage
- **HDFS**: Distributed file system
- **Ceph**: Distributed storage system

