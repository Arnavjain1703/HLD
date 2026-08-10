# Chapter 15: Design Google Drive — System Design Interview Notes

> Source: *System Design Interview* (Alex Xu), Chapter 15, pp. 250–271

---

## Table of Contents
1. [Requirements & Scope](#1-requirements--scope)
2. [Back of the Envelope Estimation](#2-back-of-the-envelope-estimation)
3. [APIs](#3-apis)
4. [High-Level Design Evolution](#4-high-level-design-evolution)
5. [Detailed Architecture](#5-detailed-architecture)
6. [Deep Dive: Block Servers](#6-deep-dive-block-servers)
7. [Metadata Database Schema](#7-metadata-database-schema)
8. [Upload Flow](#8-upload-flow)
9. [Download Flow](#9-download-flow)
10. [Notification Service](#10-notification-service)
11. [Save Storage Space](#11-save-storage-space)
12. [Failure Handling](#12-failure-handling)
13. [Design Tradeoffs](#13-design-tradeoffs)
14. [Interview Q&A — DB Choice & Why](#14-interview-qa--db-choice--why)

---

## 1. Requirements & Scope

### Functional Requirements
- **Add files** — drag and drop into Google Drive
- **Download files**
- **Sync files across devices** — file added on one device auto-syncs to others
- **See file revisions**
- **Share files** with friends, family, coworkers
- **Send notifications** when a file is edited, deleted, or shared

### Out of Scope
- Google Docs real-time collaborative editing

### Clarifying Questions (Candidate ↔ Interviewer)
| Candidate | Interviewer |
|---|---|
| Most important features? | Upload/download, file sync, notifications |
| Mobile app, web app, or both? | Both |
| Supported file formats? | Any file type |
| Files need to be encrypted? | Yes |
| File size limit? | 10 GB or smaller |
| How many users? | 10 million DAU |

### Non-Functional Requirements
- **Reliability** — data loss is unacceptable
- **Fast sync speed** — slow sync causes abandonment
- **Low bandwidth usage** — critical for mobile users
- **Scalability** — handle high traffic volumes
- **High availability** — works even when servers are offline or have network errors

---

## 2. Back of the Envelope Estimation

| Metric | Value |
|---|---|
| Total signed-up users | 50 million |
| DAU | 10 million |
| Free space per user | 10 GB |
| Files uploaded per user per day | 2 |
| Average file size | 500 KB |
| Read : Write ratio | 1 : 1 |
| Total storage | 50M × 10 GB = **500 Petabytes** |
| Upload QPS | 10M × 2 / 24h / 3600s ≈ **240** |
| Peak QPS | 240 × 2 = **480** |

---

## 3. APIs

All APIs require user authentication over HTTPS.

### 3.1 Upload a File

Two upload modes:
- **Simple upload** — small files
- **Resumable upload** — large files or unreliable networks

```
POST https://api.example.com/files/upload?uploadType=resumable
Params:
  - uploadType=resumable
  - data: local file to upload
```

Resumable upload — 3 steps:
1. Send initial request → retrieve resumable URL
2. Upload data and monitor upload state
3. If interrupted → resume from last checkpoint

### 3.2 Download a File

```
GET https://api.example.com/files/download
Params:
  - path: "/recipes/soup/best_soup.txt"
```

### 3.3 Get File Revisions

```
GET https://api.example.com/files/list_revisions
Params:
  - path: "/recipes/soup/best_soup.txt"
  - limit: 20
```

---

## 4. High-Level Design Evolution

### Stage 1 — Single Server

Start simple: Apache web server + MySQL + `/drive` directory.

```
/drive
  /user1
    /recipes
      chicken_soup.txt
  /user2
    football.mov
    sports.txt
  /user3
    best_pic_ever.png
```

> Figure 15-3: Drive directory namespace

### Stage 2 — Storage Full → Shard the DB

```
+--------------------------------------+
|  /drive       [10 MB free of 1 TB]   |  <- disk full!
+--------------------------------------+
```

> Figure 15-4: Single server storage full

Shard by `user_id`:

```
          user_id % 4
         /    |    |    \
     Server1 Server2 Server3 Server4
```

> Figure 15-5: DB sharding by user_id

### Stage 3 — Move Files to S3

```
Same-region replication:           Cross-region replication:

  Bucket                            Region A (Bucket)
    +-- replica                           |
    +-- replica            --------->  Region B (Bucket)
    +-- replica                           +-- replica
  (all in Region A)                       +-- replica
```

> Figure 15-6: S3 same-region vs cross-region replication

### Stage 4 — Decouple Everything

```
       User (Browser / Mobile)
                |
          Load balancer
                |
           API servers
          /             \
   Metadata DB        File storage (S3)
```

> Figure 15-7: Decoupled architecture

---

## 5. Detailed Architecture

```
                   User
             (Browser / Mobile)
                    |
              Load balancer
                    |
              API servers <--------- long polling
             /      |      \               |
     Block        Metadata    Notification
     servers        DB         Service -----> Offline Backup Queue
        |           |
   Cloud          Metadata
   storage          Cache
        |
   Cold storage
```

> Figure 15-10: Full high-level design

### Component Summary

| Component | Role |
|---|---|
| **Load balancer** | Distributes traffic evenly; redistributes when a server goes down |
| **API servers** | User auth, upload flow management, file metadata updates |
| **Metadata database** | Stores metadata for users, files, blocks, versions — NOT actual file content |
| **Metadata cache** | Caches hot metadata for fast reads |
| **Block servers** | Split, compress, encrypt files; handle delta sync on updates |
| **Cloud storage** | Object storage (S3); stores actual encrypted file blocks |
| **Cold storage** | S3 Glacier for inactive files — much cheaper than S3 standard |
| **Notification service** | Pub/sub — notifies clients when files are added/edited/deleted |
| **Offline backup queue** | Persists pending changes for offline clients to sync later |

---

## 6. Deep Dive: Block Servers

For large frequently updated files, uploading the whole file every time wastes bandwidth.

### Delta Sync — Only Modified Blocks Transferred

```
Block servers (10 blocks):          Cloud storage:

+----------+----------+
| Block 1  | Block 2  |<-- changed        Block 2
|          |(modified)| ------------>
+----------+----------+                   Block 5
| Block 3  | Block 4  |
+----------+----------+
| Block 5  | Block 6  |<-- changed
|(modified)|          |
+----------+----------+
| Block 7  | Block 8  |
+----------+----------+
| Block 9  | Block 10 |
+----------+----------+
```

> Figure 15-12: Delta sync — only Block 2 and Block 5 transferred

### Compression
- Text files → **gzip / bzip2**
- Images and videos → format-specific algorithms

### Block Server Upload Pipeline (new file)

```
         +--split--> Block 1 --compress--> [  ] --encrypt--> [lock] --+
File ----|--split--> Block 2 --compress--> [  ] --encrypt--> [lock] --|--> Cloud storage
         |   ...                                                       |
         +--split--> Block N --compress--> [  ] --encrypt--> [lock] --+
```

> Figure 15-11: Block server pipeline

Block size limit: **4 MB** (Dropbox reference)

---

## 7. Metadata Database Schema

> **Why Relational DB (not NoSQL)?**
> Strong consistency (ACID) is required — the same file must look identical to all clients at all times. NoSQL defaults to eventual consistency, which would require complex application-level sync logic to work around. Relational DB gives ACID natively.

```
+-------------------+       +---------------------+
|       user        |       |        file         |
|-------------------|       |---------------------|
| user_id    bigint |--+    | id           bigint |
| user_name  varchar|  |    | file_name    varchar |
| created_at        |  |    | relative_path varchar|
+-------------------+  |    | is_directory boolean |
                       |    | latest_version bigint|
+-------------------+  |    | checksum     bigint  |
|      device       |  |    | workspace_id bigint  |
|-------------------|  |    | created_at           |
| device_id  uuid   |  |    | last_modified        |
| user_id    bigint |--+    +---------------------+
| last_logged_in    |
+-------------------+       +---------------------+
                            |    file_version      |
+-------------------+       |---------------------|
|     workspace     |       | id            bigint|
|-------------------|       | file_id       bigint|
| id         bigint |       | device_id     uuid  |
| owner_id   bigint |       | version_number bigint|
| is_shared  boolean|       | last_modified        |
| created_at        |       +---------------------+
+-------------------+
                            +---------------------+
                            |       block         |
                            |---------------------|
                            | block_id      bigint|
                            | file_version_id     |
                            | block_order   int   |
                            +---------------------+
```

> Figure 15-13: Metadata DB schema

### Table Purpose Reference

| Table | Purpose |
|---|---|
| **user** | Basic user info: username, email, profile photo |
| **device** | Device info; push_id for mobile notifications. One user can have multiple devices |
| **workspace** | Root directory; can be shared |
| **file** | Latest file state |
| **file_version** | Immutable revision history; rows never updated after insert |
| **block** | One row per block; reconstruct any version by fetching all blocks for a file_version_id ordered by block_order |

---

## 8. Upload Flow

> Figure 15-14: Upload sequence diagram

Two requests sent **in parallel** from Client 1:

```
Client 1     Block servers   Cloud storage   API servers   Metadata DB   Notification svc
   |                                              |
   |---(1) add file metadata-------------------->|
   |                                              |---(2) store metadata, status=PENDING--->|
   |                                              |---(3) notify changes------------------------------------>|
   |                                              |    (Client 2 notified: file being added)
   |---(2.1) upload file--->|
   |                        |---(2.2) upload blocks--->|
   |                                              |<---(2.3) upload complete callback-----------|
   |                                              |---(2.4) update metadata: status=UPLOADED--->|
   |                                              |---(2.5) notify changes------------------------------------>|
```

### Steps
1. Client 1 adds file metadata → API servers → Metadata DB sets status = **"pending"**
2. Notification service informed → notifies Client 2 a file is being added
3. Client 1 uploads content to block servers
4. Block servers chunk → compress → encrypt → upload blocks to cloud storage
5. Cloud storage triggers upload completion callback to API servers
6. File status updated to **"uploaded"** in Metadata DB
7. Notification service notifies all relevant clients the file is fully uploaded

---

## 9. Download Flow

> Figure 15-15: Download sequence diagram

Triggered when a file is added or edited by another client.

**How does a client know a file changed?**
- **Online** → notification service informs the client; client pulls latest changes
- **Offline** → changes cached in offline backup queue; client pulls when back online

```
Client 2     Block servers   Cloud storage   API servers   Metadata DB   Notification svc
   |                                                                          |
   |<--(1) notify changes-----------------------------------------------------|
   |---(2) get changes------------------------------------------------>|
   |                                                                   |---(3) get changes--->|
   |                                                                   |<--(4) return changes--|
   |<--(5) metadata returned-------------------------------------------|
   |---(6) download blocks--->|
   |                          |---(7) download blocks--->|
   |                          |<--(8) blocks returned----|
   |<--(9) blocks returned----|
   (reconstruct file from blocks)
```

---

## 10. Notification Service

Purpose: notify clients of file changes made by other clients to minimize sync conflicts.

### Long Polling vs WebSocket

| Option | Best For | Google Drive? |
|---|---|---|
| **Long polling** | Infrequent, unidirectional server→client | Chosen |
| **WebSocket** | Real-time bi-directional (chat, gaming) | Overkill |

**Why long polling?**
- File change notifications are infrequent — no continuous data burst
- The flow is unidirectional (server tells client about changes; client does not push back)
- WebSocket is purpose-built for chat-style apps, not file sync
- Dropbox uses the same approach

### How Long Polling Works
1. Each client opens a persistent long poll connection to notification service
2. On file change detection, connection is closed; client connects to API servers to download latest changes
3. After response received or timeout, client immediately sends a new long poll request

---

## 11. Save Storage Space

Multiple versions of the same file stored across multiple data centers fills storage fast.

### Strategy 1 — De-duplicate Data Blocks
- Two blocks are identical if they have the **same hash value**
- Identical blocks across users and versions → stored once

### Strategy 2 — Intelligent Backup Limits
- **Version cap** — limit stored revisions; oldest replaced when cap is reached
- **Weight recent versions** — a heavily edited doc might be saved 1000+ times in a day; recent versions matter more

### Strategy 3 — Cold Storage Tiering
- Files not accessed for months/years → **Amazon S3 Glacier**
- S3 Glacier is ~10× cheaper than S3 Standard

---

## 12. Failure Handling

| Component | Failure | Recovery |
|---|---|---|
| **Load balancer** | Primary fails | Secondary active via heartbeat monitoring |
| **Block server** | Server fails | Other block servers pick up pending jobs |
| **Cloud storage** | Regional outage | Files in multiple regions; fetch from healthy region |
| **API server** | Server fails | Stateless — load balancer redirects to other servers |
| **Metadata cache** | Node goes down | Other replicas serve; new node spun up |
| **Metadata DB master** | Goes down | Promote slave to master; bring up new slave |
| **Metadata DB slave** | Goes down | Use other slaves for reads; replace failed slave |
| **Notification service** | Server fails | 1M+ long poll connections drop; clients gradually reconnect to a different server |
| **Offline backup queue** | Queue fails | Queues are replicated; consumers re-subscribe |

---

## 13. Design Tradeoffs

### Alternative: Upload Directly to Cloud (Bypass Block Servers)

| Aspect | Block Servers (Current Design) | Direct to Cloud |
|---|---|---|
| Upload speed | Slightly slower (extra hop) | Faster — file transferred once |
| Logic location | Centralized in block servers | Must re-implement on iOS, Android, Web |
| Security | Server-side encryption (safer) | Client-side encryption (risky) |
| Maintenance | Single codebase | Three platform implementations |

**Verdict**: Block servers are better — centralized security and maintainability win.

### Alternative: Separate Presence Service
- Extract online/offline detection from notification servers into a dedicated **Presence Service**
- Benefit: reusable by other services (chat, analytics, etc.)

---

## 14. Interview Q&A — DB Choice & Why

---

### Q1: Why relational DB for metadata? Why not NoSQL?

**Core issue: consistency.**

The system needs **strong consistency** — the same file must look identical to all clients simultaneously. If User A uploads a file and User B sees a stale version on a different device, that is a critical correctness bug.

- **Relational DB (MySQL/PostgreSQL)**: ACID is natively supported — atomicity, consistency, isolation, durability are guaranteed
- **NoSQL (DynamoDB, Cassandra, MongoDB)**: defaults to eventual consistency — different replicas can hold different values at the same moment in time
- To use NoSQL here, you would need to implement consistency guarantees in application-layer synchronization logic — complex, error-prone, and re-inventing what SQL gives for free

**Scaling the relational DB:**
- **Read replicas** — horizontal read scaling (1:1 read:write from estimation)
- **Sharding** on `user_id` or `workspace_id` for write scaling
- **Metadata cache** (Redis) for hot data — always invalidate on write

---

### Q2: What is the metadata cache and why not just always query the DB?

Metadata (file paths, version numbers, checksums) is read far more often than written. The cache (Redis/Memcached) reduces latency for hot data.

**Consistency rule:**
- On every DB write → invalidate or update the corresponding cache entry
- Cache and DB master must always hold the same value
- This ensures strong consistency even with a caching layer

---

### Q3: Why S3 (object storage) for file blocks instead of a database or filesystem?

- S3 is designed for binary blobs of arbitrary size at petabyte scale
- Built-in **same-region and cross-region replication** — files survive regional outages
- Far cheaper per GB than block storage or relational DB at 500 PB scale
- Access pattern is simple: `PUT` / `GET` by key — no need for querying or indexing
- A relational DB with binary blob columns would be massively inefficient at this scale

---

### Q4: Why split files into blocks instead of storing whole files?

- **Delta sync** — only modified blocks transferred on update → saves bandwidth
- **Parallel upload** — blocks uploaded concurrently → faster
- **Resumability** — interrupted upload resumes from last successful block, not from scratch
- **De-duplication** — identical blocks (same hash) stored once, shared across users and versions

---

### Q5: Why long polling for notifications instead of WebSocket?

- Google Drive notifications are **infrequent and unidirectional** (server notifies client of a change)
- WebSocket provides **bi-directional real-time streams** — right for chat, wrong for file sync
- Long polling is simpler, sufficient, and less resource-intensive
- Dropbox uses the same approach

---

### Q6: How are sync conflicts handled?

**First write wins:**
1. First version processed by the system → saved as canonical version
2. Later conflicting version → flagged as a conflict copy
3. User presented with both copies → option to merge or override
4. Example conflict copy name: `SystemDesignInterview_user2_conflicted_copy_2019-05-01`

---

### Q7: How do you guarantee no data loss?

- **Block replication within a region** (S3 same-region replication)
- **Cross-region replication** — regional failure cannot cause data loss
- **Metadata DB replication** — master + slaves; promote slave on master failure
- **Offline backup queue** — change intent persisted; synced when client reconnects

---

### Q8: How do you reduce storage costs at 500 PB scale?

1. **Block de-duplication** — identical hash = shared block storage across users
2. **Versioning caps** — limit stored revisions; oldest dropped at cap
3. **Cold storage tiering** — inactive files → S3 Glacier (~10× cheaper than S3 Standard)

---

### Q9: What are the most interesting DB schema decisions?

- **`file_version` rows are immutable** — never updated after insert; preserves revision history integrity
- **`block` has `block_order`** — reconstruct any file version by fetching all blocks for a `file_version_id` sorted by `block_order`
- **`checksum` on `file`** — enables de-duplication and integrity verification
- **Separate `device` table** — one user can have multiple devices; push notifications are device-scoped, not user-scoped
- **`file` vs `file_version`** — `file` = current state; `file_version` = immutable history; clean separation of concerns

---

### Q10: Why are two requests sent in parallel during upload?

When Client 1 uploads:
1. **Metadata flow**: Client → API servers → Metadata DB (store entry, status = "pending")
2. **Content flow**: Client → Block servers → Cloud storage

They are independent — metadata storage does not depend on content upload completing. Parallelism minimizes total upload latency. Final status is set to "uploaded" only after cloud storage confirms completion.

---

### Q11: How would the design change if files could be larger than 10 GB?

- Resumable uploads become even more critical
- S3 multipart upload natively handles files up to 5 TB
- Block size might be tuned upward to reduce block count per file
- Block servers need higher parallelism in the pipeline
- `block` table may need partitioning by `file_version_id` at extreme scale

---

*Source: System Design Interview — An Insider's Guide, Vol. 1 (Alex Xu)*
