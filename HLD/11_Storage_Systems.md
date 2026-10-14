# 💾 Storage Systems: Object, Block & File Storage

> *"Data is the new oil. Storage is the refinery."*

---

## 📋 Table of Contents
1. [Storage Types Overview](#storage-types)
2. [Block Storage](#block-storage)
3. [File Storage](#file-storage)
4. [Object Storage Deep Dive](#object-storage)
5. [Amazon S3 Internals](#s3-internals)
6. [Distributed File Systems: HDFS](#hdfs)
7. [Database Storage Engines: LSM vs B-Tree](#lsm-vs-btree)
8. [Write-Ahead Log (WAL)](#wal)
9. [Erasure Coding vs Replication](#erasure-coding)
10. [Real-Life Scenario](#real-life-scenario)
11. [🔥 Big Tech Story: Facebook Haystack](#big-tech-story)
12. [Trade-offs & Best Practices](#trade-offs)
13. [Key Takeaways](#key-takeaways)

---

## 🗂️ Storage Types Overview

| Feature | Block Storage | File Storage | Object Storage |
|---|---|---|---|
| Access method | Raw blocks (LUN) | File path (NFS/SMB) | HTTP API (GET/PUT) |
| Structure | No structure (raw) | Hierarchical (directories) | Flat namespace (key-value) |
| Filesystem | You add your own | Built in | None |
| Scalability | Limited to one server | Limited | Virtually unlimited |
| Metadata | Minimal | Filesystem metadata | Rich custom metadata |
| Performance | Very high IOPS | Good | Moderate (latency for small files) |
| Use case | Databases, VMs | Shared file access, home dirs | Images, videos, backups, logs |
| Examples | AWS EBS, Azure Disk | AWS EFS, Azure Files, NFS | AWS S3, GCS, Azure Blob |

---

## 🟦 Block Storage

Block storage provides raw storage volumes at the block level (512-byte or 4KB blocks). No filesystem built in — you attach the block device and format it with a filesystem.

**How it works:**
```
Application → Filesystem (ext4, NTFS, XFS)
                    ↓
            Block Device Driver
                    ↓
            Block Storage Volume (EBS, SAN)
                    ↓
            Physical Disks (SSD, HDD)
```

**Use cases:**
- **Databases:** PostgreSQL, MySQL, MongoDB need raw block devices for maximum I/O control
- **Virtual Machines:** Each VM's disk is a block volume
- **Boot volumes:** EC2 root volumes are EBS

**AWS EBS (Elastic Block Store):**
```
Volume Types:
  gp3 (General Purpose SSD): 3,000-16,000 IOPS, 125-1,000 MB/s, default choice
  io2 Block Express: 256,000 IOPS, 4,000 MB/s, for high-performance databases
  st1 (Throughput Optimized HDD): Sequential reads, 500 MB/s, for big data, logs
  sc1 (Cold HDD): 250 MB/s, cheapest, archive data
  
Multi-Attach: Attach one EBS volume to multiple EC2 instances (io1/io2 only)
Snapshots: Point-in-time backup to S3 (incremental, only changed blocks)
```

**Block storage characteristics:**
- Attached to ONE server at a time (exclusive access, except Multi-Attach)
- Low latency (NVMe EBS: < 1ms)
- Highest IOPS of any storage type
- Not accessible via HTTP — must be mounted as a disk

---

## 📁 File Storage

File storage provides a shared filesystem accessible by multiple machines simultaneously.

**NFS (Network File System):**
```
Server A ──┐
Server B ──┤── NFS Mount → /mnt/shared ── NFS Server → Storage
Server C ──┘
```

All servers see the same files as if they were local.

**AWS EFS (Elastic File System):**
- Fully managed NFS
- Scales automatically (no provisioning)
- Multi-AZ available
- Performance: Elastic mode scales to 3+ GB/s
- Expensive compared to S3 (~10x)

**Use cases:**
- Shared code/config across multiple app servers
- CMS media files that multiple web servers need to read
- ML training datasets shared across GPU instances
- Legacy apps that use filesystem APIs

**Limitations:**
- Throughput limited vs object storage at scale
- File locking coordination can be complex
- POSIX filesystem operations (rename, chmod) add complexity

---

## 📦 Object Storage Deep Dive

Object storage is the dominant storage paradigm for modern cloud applications.

**Core concepts:**
- **Flat namespace:** No directories — just buckets and keys (though keys can contain `/` to simulate directories)
- **Immutable objects:** Write once, read many. You can replace an object but not modify it in-place.
- **Rich metadata:** Each object can have arbitrary key-value metadata
- **HTTP API:** Access via standard GET/PUT/DELETE HTTP calls

```python
# Accessing S3 via Python boto3
import boto3

s3 = boto3.client('s3')

# Upload
s3.upload_file('local_photo.jpg', 'my-bucket', 'photos/2024/alice_birthday.jpg')

# Download  
s3.download_file('my-bucket', 'photos/2024/alice_birthday.jpg', 'local_copy.jpg')

# Generate pre-signed URL (temporary access)
url = s3.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'my-bucket', 'Key': 'photos/private/photo.jpg'},
    ExpiresIn=3600  # 1 hour
)
```

**Key features:**
- Virtually unlimited storage
- 11 nines durability (99.999999999%) via replication/erasure coding
- Built-in versioning, lifecycle policies, access control
- Event notifications (S3 → Lambda on upload)
- Multipart upload for large files (up to 5TB per object)

---

## 🔬 Amazon S3 Internals

S3 is one of the most remarkable engineering achievements in cloud computing. Storing trillions of objects with 11 nines durability is non-trivial.

### Architecture

S3 is built on Amazon's internal storage system. Each object is:
1. Split into chunks
2. Erasure coded (not just replicated)
3. Stored across multiple AZs (Availability Zones) within a region
4. Indexed in a distributed metadata store

### S3 Storage Classes

| Storage Class | Retrieval | Min Duration | Use Case | Cost |
|---|---|---|---|---|
| S3 Standard | Instant | None | Frequently accessed data | $$$ |
| S3 Intelligent-Tiering | Instant | None | Unknown access patterns | $$$ + monitoring fee |
| S3 Standard-IA | Instant | 30 days | Infrequently accessed | $$ |
| S3 One Zone-IA | Instant | 30 days | Reproducible infrequent | $ |
| S3 Glacier Instant | Instant | 90 days | Archive, instant retrieval | $ |
| S3 Glacier Flexible | 1-12 hours | 90 days | Archive | $¢ |
| S3 Glacier Deep Archive | 12-48 hours | 180 days | Long-term archive | ¢ |

**Intelligent-Tiering:** S3 monitors access patterns. Objects not accessed for 30 days automatically moved to IA (lower cost). If accessed, moved back to Standard. Zero retrieval cost.

### S3 Consistency (Since December 2020)

S3 now provides **strong read-after-write consistency** for all operations:
- PUT followed by GET → always returns the new version
- DELETE followed by LIST → deleted object not listed

Previously S3 was eventually consistent for overwrites and deletes, causing subtle bugs.

### S3 Performance Optimization

**Request rate:** S3 automatically scales to thousands of requests/second per prefix.

**For very high throughput:** Use multiple prefixes (avoid too many objects under a single prefix).

```
# Low throughput (single prefix bottleneck):
photos/user_123/photo_1.jpg
photos/user_123/photo_2.jpg
...

# High throughput (hash prefix distribution):
a3/photos/user_123/photo_1.jpg  (prefix = hash[:2])
f7/photos/user_123/photo_2.jpg
```

**Multipart Upload:** For objects > 100MB, use multipart upload:
1. Initiate multipart upload → get upload ID
2. Upload parts in parallel (each ≥ 5MB, up to 10,000 parts)
3. Complete multipart upload → S3 assembles parts

```python
# Multipart upload
mpu = s3.create_multipart_upload(Bucket='my-bucket', Key='large-file.mp4')
upload_id = mpu['UploadId']

parts = []
for i, chunk in enumerate(read_in_chunks('large-file.mp4', size=50*1024*1024)):
    part = s3.upload_part(
        Bucket='my-bucket', Key='large-file.mp4',
        UploadId=upload_id, PartNumber=i+1, Body=chunk
    )
    parts.append({'PartNumber': i+1, 'ETag': part['ETag']})

s3.complete_multipart_upload(
    Bucket='my-bucket', Key='large-file.mp4',
    UploadId=upload_id,
    MultipartUpload={'Parts': parts}
)
```

### S3 Lifecycle Policies

Automatically transition objects between storage classes or delete them.

```json
{
  "Rules": [{
    "ID": "ArchiveOldPhotos",
    "Filter": { "Prefix": "photos/" },
    "Status": "Enabled",
    "Transitions": [
      { "Days": 90,  "StorageClass": "STANDARD_IA" },
      { "Days": 365, "StorageClass": "GLACIER" }
    ],
    "Expiration": { "Days": 3650 }
  }]
}
```

After 90 days → moved to Standard-IA (lower cost)
After 1 year → moved to Glacier (archive)
After 10 years → deleted

---

## 🐘 Distributed File Systems: HDFS

HDFS (Hadoop Distributed File System) was designed for storing and processing very large datasets (petabytes) across commodity hardware.

### Architecture

```
               ┌──────────────┐
               │   NameNode   │ ← Metadata only (file → block mappings)
               │  (master)    │   Stored in memory for fast access
               └──────┬───────┘
                      │ (block locations)
     ┌────────────────┼───────────────────┐
     ▼                ▼                   ▼
┌─────────┐     ┌─────────┐         ┌─────────┐
│DataNode1│     │DataNode2│         │DataNode3│
│ Block 1 │     │ Block 2 │         │ Block 3 │
│ Block 4 │     │ Block 1 │         │ Block 2 │
│ Block 7 │     │ Block 5 │         │ Block 4 │
└─────────┘     └─────────┘         └─────────┘
(actual file data, replicated 3x)
```

**NameNode:** Master. Stores the entire filesystem namespace in memory:
- File to block mapping: `file.txt → [Block1, Block2, Block3]`
- Block to DataNode mapping: `Block1 → [DataNode1, DataNode2, DataNode5]`

**DataNode:** Workers. Actually store the data blocks. Report block list to NameNode periodically.

**Block size:** 128MB (vs filesystem's 4KB). Large blocks reduce NameNode memory usage and optimize for sequential reads (MapReduce workloads).

**Replication factor:** Default 3. Each block is stored on 3 DataNodes (across different racks for rack failure resilience).

### Write Path

```
Client → NameNode: "Create file.txt"
NameNode: "Write block 1 to DataNode1, DataNode3, DataNode5"
Client → DataNode1: Write block data
DataNode1 → DataNode3: Replicate (pipeline)
DataNode3 → DataNode5: Replicate (pipeline)
DataNode5 → DataNode3: ACK
DataNode3 → DataNode1: ACK
DataNode1 → Client: ACK
Client → NameNode: "Block 1 written"
```

### Write-Once-Read-Many

HDFS does NOT support random writes. Files are written once and then read (sequentially). This is by design — optimized for batch analytics (MapReduce, Spark) that read entire files sequentially.

**NameNode is a SPOF:** The original HDFS had a single NameNode. HDFS HA was added later (active/standby NameNode pair with ZooKeeper for failover).

---

## 🌲 Database Storage Engines: LSM Tree vs B-Tree

How databases physically store data on disk is one of the most critical performance factors.

### B-Tree (Used by: PostgreSQL, MySQL InnoDB, SQLite)

B+ Trees (see document 04) store data sorted on disk. Updates modify existing pages in place.

```
Write: Find page on disk → read into memory → modify → write back
Read:  Find page via tree traversal → read into memory → return
```

**Write amplification:** One logical write (e.g., UPDATE a row) may cause many disk writes:
1. Write the modified data page
2. Write the WAL entry
3. Write the B-tree index pages (multiple if multiple indexes)
4. Write the free space map if needed

B-Tree is **write-amplified** but **read-optimized**: data is at a predictable location; reads are fast.

**Best for:** Read-heavy workloads, random access patterns, OLTP (many small reads/writes)

### LSM Tree (Log-Structured Merge Tree)

Used by: Apache Cassandra, RocksDB, LevelDB, TiKV, Kafka (its log), InfluxDB, ScyllaDB

**Key insight:** Sequential writes are 10-100x faster than random writes (especially on HDDs; also significant on SSDs). LSM trees turn all writes into sequential writes.

```
Write Path:
  1. Write to WAL (append-only, sequential) ← crash safety
  2. Write to Memtable (in-memory sorted structure)
  3. Return to client (write is done!)

Background:
  When memtable is full → flush to SSTable (Sorted String Table) on disk
  SSTables are immutable sorted files

Read Path:
  1. Check Memtable (newest, in memory)
  2. Check recent SSTables (newer first)
  3. Check older SSTables...
  (Bloom filters used to quickly skip SSTables that don't contain the key)
```

**Compaction:** Background process that merges SSTables, removes deleted keys (tombstones), deduplicates updates.

```
Before compaction:
  SSTable1: key=A:1, key=B:old, key=C:3
  SSTable2: key=B:new, key=D:5
  SSTable3: key=A:2, key=E:7

After compaction (merge into one sorted SSTable):
  Merged:   key=A:2(latest), key=B:new(latest), key=C:3, key=D:5, key=E:7
```

### Comparison Table

| Feature | B-Tree | LSM Tree |
|---|---|---|
| Write performance | Moderate (random writes) | Very high (sequential writes) |
| Read performance | Very high (direct access) | Moderate (may check multiple SSTables) |
| Write amplification | Low | High (compaction rewrites data) |
| Read amplification | Low | Medium (Bloom filters mitigate) |
| Space amplification | Low | High (duplicates until compaction) |
| Disk I/O pattern | Random | Sequential |
| Best for | Read-heavy, OLTP | Write-heavy, time-series, analytics |
| Delete performance | Good | Slow (tombstones, must wait for compaction) |
| Examples | PostgreSQL, MySQL | Cassandra, RocksDB, LevelDB |

**Rule of thumb:** If writes are your bottleneck → LSM tree. If reads are your bottleneck → B-Tree.

---

## 📝 Write-Ahead Log (WAL)

The WAL (also called journal or redo log) is the foundation of database durability.

**The core idea:** Before modifying any data page, write the change to an append-only log. If the database crashes, replay the log on recovery.

```
Transaction:
  1. Write WAL entry: "UPDATE accounts SET balance=balance-100 WHERE id=1"
  2. fsync() WAL to disk (guaranteed to persist)
  3. Apply to in-memory data page
  4. Send COMMIT to client

On crash recovery:
  Read WAL entries since last checkpoint
  Replay committed transactions
  Roll back uncommitted transactions
```

**Why WAL is efficient:**
- WAL writes are **sequential** (append-only) → fast
- Data page writes can be **batched** (multiple transactions → one page write)
- Actual data pages can be written **lazily** (during checkpoints)
- The WAL is the bottleneck for write performance in most databases

**PostgreSQL WAL:**
- Called WAL (Write-Ahead Log) or XLOG
- Used for: crash recovery, streaming replication, point-in-time recovery
- `wal_level = logical` enables logical replication (row-level change capture)

**MySQL Redo Log (InnoDB):**
- Similar WAL concept
- Fixed-size circular log files (ib_logfile0, ib_logfile1)
- Flushed on transaction commit (`innodb_flush_log_at_trx_commit = 1` for full durability)

---

## 🧩 Erasure Coding vs Replication

When storing data across multiple nodes, how do you protect against disk/node failures?

### Replication

Store N identical copies of data on N different nodes.

```
Replication factor 3:
  Data: [Block A]
  → Stored on: Node1 (Block A), Node2 (Block A), Node3 (Block A)

Can tolerate: 2 node failures
Storage overhead: 3x (200% overhead)
Read: Read from any of the 3 nodes (read amplification = 1)
```

**Used by:** HDFS (default RF=3), Cassandra, MongoDB, most databases

**Overhead:** 3x replication = 200% overhead

### Erasure Coding

Data is encoded into N+K chunks using mathematical transforms. Any K chunks can reconstruct the original.

```
RS (Reed-Solomon) 6+3 erasure code:
  Data: [Block A] (6 data chunks)
  + 3 parity chunks (computed from data)
  = 9 total chunks stored across 9 nodes

Can tolerate: 3 node failures (any 3 of 9)
Storage overhead: 9/6 = 1.5x (50% overhead)
Read: Read 6 of 9 chunks (read amplification = 6 instead of 1)
```

**The math:** Reed-Solomon codes (basis of QR codes, CDs, DVDs, RAID) can reconstruct original data from any 6 of 9 chunks.

**Advantages of erasure coding:**
- Much lower storage overhead (50% vs 200% for 3-node tolerance)
- Same durability guarantee

**Disadvantages:**
- Higher read amplification (must read from multiple nodes)
- CPU overhead for encoding/decoding
- Write latency is higher

**Used by:** Amazon S3 (within each AZ, then replicated across AZs), Facebook Warm Storage, Google Colossus

**S3's durability math:** S3's 11 nines durability comes from:
1. Erasure coding within an AZ (tolerates disk failures)
2. Replication across 3 AZs (tolerates AZ failures)
3. With EC: can survive multiple simultaneous disk failures in one AZ

---

## 📸 Real-Life Scenario: Photo-Sharing App Storing Billions of Photos

**The challenge:** You're building a photo-sharing app like Instagram. Users upload 10 million photos/day, each averaging 3MB. After 5 years, that's:

```
10M photos/day × 5 years × 365 days × 3MB = 54.75 PB
```

Plus thumbnails (3 sizes), so ~5x = 274 PB of storage needed.

**Naive approach: Store photos on application servers**
```
10M photos/day = 30TB/day
Server with 10TB disk → replace every 8 hours!
Not scalable. No redundancy. No CDN.
```

**Production approach: Object Storage + CDN**

```
Upload Flow:
  1. User uploads photo to app server (POST /upload)
  2. App server validates photo
  3. App server uploads to S3:
     s3://my-bucket/photos/original/{user_id}/{photo_id}.jpg
  4. Publish "photo_uploaded" event to Kafka
  5. Return photo URL to user

Async Processing:
  Thumbnail Service reads Kafka event:
    → Generate thumbnail 150x150px → upload to S3 /thumbs/150/{photo_id}.jpg
    → Generate thumbnail 640x640px → upload to S3 /thumbs/640/{photo_id}.jpg
    → Update DB: photo.thumbnail_url = "https://cdn.example.com/thumbs/..."

CDN:
  CloudFront distribution in front of S3
  Photos: Cache-Control: public, max-age=86400
  Thumbnails: Cache-Control: public, max-age=604800
  
Storage Lifecycle:
  Standard: First 90 days (hot data, recently uploaded)
  Standard-IA: 90-365 days (cooler data)
  Glacier: > 1 year (rarely accessed old photos)
```

**Cost estimation:**
```
Data storage: 274 PB × $0.023/GB = ~$6M/month (S3 Standard)
With tiering (50% in IA, 30% in Glacier): ~$2.5M/month

CDN transfers: 1 billion views/day × 200KB avg thumbnail = 200TB/day
CloudFront: 200TB × $0.008/GB = $1,600/day = $48K/month

Database (photo metadata): PostgreSQL with photo_id, user_id, S3_key, timestamps
With sharding by user_id: manageable
```

**The key architecture decisions:**
1. App servers are stateless (no local file storage)
2. S3 is the source of truth for all media
3. CDN reduces S3 GET requests by 95%+
4. Async thumbnail generation decouples upload latency from processing
5. Lifecycle policies automatically tier cold data to cheaper storage

---

## 🔥 Big Tech Story: Facebook Haystack — Solving the Small File Problem

### The Problem (2009)

Facebook had a photo problem. Not a storage problem per se — they were using NFS (Network File System) — but a performance problem caused by how NFS handles millions of tiny files.

By 2009, Facebook was storing 65 billion photos. Each photo had 4 versions (thumbnail, small, medium, large) = 260 billion files in NFS.

**NFS's fundamental problem with many small files:**

To serve a photo from NFS:
1. Translate filename to inode via filesystem namespace lookup
2. Read inode from disk (disk seek #1)
3. Read directory block (disk seek #2)
4. Read data from inode's block map (disk seek #3)
5. Finally read the actual photo data (disk seek #4)

For each photo: **3-4 disk seeks** before any actual data is read.

NFS metadata operations (lookups, permission checks) went to a CDN. But cache had limited space. For "long-tail" photos (old, rarely viewed), the CDN had no cache hit. Every request was 3-4 disk seeks on NFS.

**The result:** Photo serving was a major bottleneck. NFS metadata operations (not data!) were the bottleneck.

### The Haystack Solution

Facebook engineers Beaver, Kumar, Li, et al. designed Haystack with one core insight:

**Eliminate all disk seeks except the one to read actual photo data.**

**Haystack architecture:**

```
Haystack Store:
  One large file (haystack) per machine containing billions of photos
  
  File structure:
  [Header][Photo1][Photo2][Photo3]...[PhotoN]
  
  Index file (in memory!):
  photo_id → {offset_in_file, size, flags}
  
Serving a photo:
  1. Look up photo_id in in-memory index → get (offset, size)  [0 disk seeks!]
  2. Read (offset, size) from the large haystack file [1 disk seek]
  Done! Only 1 disk seek total.

vs NFS: 3-4 disk seeks
```

**The needle and haystack metaphor:** Each photo is a "needle" in the "haystack" (the large file).

**The index in memory:** The index is small (8-byte key, 4-byte offset, 4-byte size = 16 bytes per photo). For 1 billion photos: 16 GB of RAM for the index. Feasible with modern servers.

**Deletion:** Photos are marked deleted in the index (soft delete). Compaction periodically rewrites the haystack file without deleted needles.

**Replication:** Each haystack is replicated 3x across servers and data centers.

**The CDN in front:** Haystack served the "long tail" — photos that missed CDN cache. With Haystack, each such request required only 1 disk seek.

### The Results

- Photo serving latency dropped by **50%**
- NFS servers replaced by far fewer Haystack machines
- Storage efficiency improved (no filesystem overhead per file)

### Published Paper

Facebook published "Finding a needle in Haystack: Facebook's photo storage" at OSDI 2010. It became one of the most cited papers in the storage systems field.

### What Came After

- Facebook later built **f4** — a lower-cost "warm storage" system using erasure coding for older photos (traded more CPU for 33% less storage cost)
- Facebook's **Tectys** project further evolved their storage infrastructure
- The Haystack principles influenced how cloud object stores handle small objects (batching, indexing)

### The Lesson

The most important insight from Haystack: **understand your access patterns and optimize for them.**

NFS was designed for general-purpose shared file access. Facebook's workload was: write once, read many, delete rarely, millions of tiny files.

NFS's general-purpose design (flexible metadata, permission hierarchy, directory navigation) was a mismatch. Haystack stripped away all the general-purpose flexibility and optimized for the specific workload — resulting in a 4x improvement in disk seeks per operation.

**System design insight:** The best storage system for your use case is almost always a specialized one. Don't use a general-purpose solution when a domain-specific one exists. And if it doesn't exist, build it.

---

## ⚖️ Trade-offs & Best Practices

### Storage Type Trade-offs

| Type | Performance | Cost | Scalability | Flexibility |
|---|---|---|---|---|
| Block | Highest | Medium | Limited (per-server) | Low (single attach) |
| File | Good | High | Medium | Medium |
| Object | Medium (latency) | Low | Unlimited | High (rich metadata) |

### Best Practices

1. **Use object storage (S3/GCS) as the default for media and large files.** It's cheaper than block/file storage, infinitely scalable, and has built-in redundancy.

2. **Never store user files on application server disks.** Application servers should be stateless. Files must be in object storage.

3. **Use presigned URLs for secure media access.** Generate temporary signed URLs (15 minutes) instead of making buckets public. Prevents unauthorized access.

4. **Enable versioning on critical S3 buckets.** Versioning allows recovery from accidental deletes or overwrites. Essential for compliance-sensitive data.

5. **Implement lifecycle policies.** Move data through storage classes automatically. Objects not accessed for 30 days should move to IA; rarely accessed to Glacier.

6. **For databases, choose B-tree storage if reads dominate; LSM if writes dominate.**

7. **Use multipart upload for objects > 100MB.** Enables parallel uploads, retrying failed parts, and better progress tracking.

8. **Understand your metadata access pattern.** Facebook built Haystack because NFS metadata was the bottleneck. Profile your actual access patterns before assuming the standard solution is the right one.

9. **For high write throughput databases, tune WAL settings.** PostgreSQL's `wal_compression=on`, `synchronous_commit=on`, `commit_delay` can significantly impact write throughput.

10. **Use erasure coding for cold storage.** For data that is rarely read but must be durable (archive, backup), erasure coding offers the same durability as 3x replication at half the cost.

---

## 🎯 Key Takeaways

| Concept | Core Insight |
|---|---|
| **Block storage** | Raw, high-performance, one server at a time; use for databases and VMs |
| **File storage** | Shared filesystem access; use for multi-server shared files (NFS/EFS) |
| **Object storage** | Flat, HTTP, unlimited scale; use for everything else (media, backups, logs) |
| **S3 durability** | 11 nines via erasure coding + multi-AZ replication |
| **Lifecycle policies** | Automate cost reduction: Standard → IA → Glacier over time |
| **HDFS** | Distributed file system for big data; large blocks, write-once-read-many |
| **B-Tree vs LSM** | B-Tree = read-optimized; LSM = write-optimized; match to your workload |
| **WAL** | Sequential log enables fast writes and crash recovery; foundation of all ACID databases |
| **Erasure coding** | Same durability as 3x replication at 1.5x overhead; ideal for cold storage |
| **Facebook's lesson** | Understand your access patterns deeply; specialized solutions beat general-purpose |

---

*Next: [12_Security.md](12_Security.md) — OAuth, JWT, mTLS, Zero Trust architecture, and the OWASP Top 10.*
