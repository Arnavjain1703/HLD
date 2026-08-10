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

### Stage 1 — Directory Namespace (Figure 15-3)

```mermaid
graph TD
    drive[/drive]
    u1[/user1]
    u2[/user2]
    u3[/user3]
    recipes[/recipes]
    chicken[chicken_soup.txt]
    football[football.mov]
    sports[sports.txt]
    pic[best_pic_ever.png]

    drive --> u1
    drive --> u2
    drive --> u3
    u1 --> recipes
    recipes --> chicken
    u2 --> football
    u2 --> sports
    u3 --> pic
```

### Stage 2 — Storage Full, Shard by user_id (Figures 15-4, 15-5)

```mermaid
graph TD
    disk["/drive — 10 MB free of 1 TB ⚠️ FULL"]
    hash{"user_id % 4"}
    s1[Server 1]
    s2[Server 2]
    s3[Server 3]
    s4[Server 4]

    disk -->|"solution: shard"| hash
    hash -->|"0"| s1
    hash -->|"1"| s2
    hash -->|"2"| s3
    hash -->|"3"| s4
```

### Stage 3 — S3 Replication (Figure 15-6)

```mermaid
graph LR
    subgraph same["Same-Region Replication (Region A)"]
        b1[Bucket]
        r1[Replica]
        r2[Replica]
        r3[Replica]
        b1 -->|replication| r1
        b1 -->|replication| r2
        b1 -->|replication| r3
    end

    subgraph cross["Cross-Region Replication"]
        subgraph ra["Region A"]
            ba[Bucket]
        end
        subgraph rb["Region B"]
            bb[Bucket]
            rb1[Replica]
            rb2[Replica]
        end
        ba -->|replication| bb
        bb -->|replication| rb1
        bb -->|replication| rb2
    end
```

### Stage 4 — Decoupled Architecture (Figure 15-7)

```mermaid
graph TD
    user["👤 User\n(Browser / Mobile)"]
    lb[Load Balancer]
    api[API Servers]
    db[(Metadata DB)]
    fs[(File Storage\nAmazon S3)]

    user --> lb
    lb --> api
    api --> db
    api --> fs
```

---

## 5. Detailed Architecture — Full High-Level Design (Figure 15-10)

```mermaid
graph TD
    user["👤 User\n(Browser / Mobile)"]
    lb[Load Balancer]
    api[API Servers]
    bs[Block Servers]
    cs[(Cloud Storage\nS3)]
    cold[(Cold Storage\nS3 Glacier)]
    mdb[(Metadata DB)]
    mc[(Metadata Cache\nRedis)]
    ns[Notification Service]
    obq[Offline Backup Queue]

    user -->|HTTPS| lb
    lb --> api
    api -->|upload blocks| bs
    api -->|read/write metadata| mdb
    api -->|read cache| mc
    api -->|long polling| ns
    bs -->|store blocks| cs
    cs -->|cold tier| cold
    ns -->|push changes| obq
    ns -->|notify clients| user
```

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

### Block Server Upload Pipeline — New File (Figure 15-11)

```mermaid
graph LR
    file[📄 File]
    subgraph bs[Block Servers]
        split1[Split → Block 1]
        split2[Split → Block 2]
        splitn[Split → Block N]
        comp1[Compress]
        comp2[Compress]
        compn[Compress]
        enc1[Encrypt 🔒]
        enc2[Encrypt 🔒]
        encn[Encrypt 🔒]
    end
    cs[(Cloud Storage)]

    file --> split1 --> comp1 --> enc1 --> cs
    file --> split2 --> comp2 --> enc2 --> cs
    file --> splitn --> compn --> encn --> cs
```

Block size limit: **4 MB** (Dropbox reference)

### Delta Sync — Only Modified Blocks Transferred (Figure 15-12)

```mermaid
graph LR
    subgraph bsrv[Block Servers]
        b1[Block 1]
        b2[Block 2\n✏️ modified]
        b3[Block 3]
        b4[Block 4]
        b5[Block 5\n✏️ modified]
        b6[Block 6]
        b7[Block 7]
        b8[Block 8]
        b9[Block 9]
        b10[Block 10]
    end
    cs[(Cloud Storage)]

    b2 -->|"changed only"| cs
    b5 -->|"changed only"| cs
```

### Compression Algorithms
- Text files → **gzip / bzip2**
- Images and videos → format-specific algorithms

---

## 7. Metadata Database Schema (Figure 15-13)

> **Why Relational DB (not NoSQL)?**
> Strong consistency (ACID) is required — the same file must look identical to all clients at all times. NoSQL defaults to eventual consistency. Relational DB gives ACID natively without application-layer workarounds.

```mermaid
erDiagram
    USER {
        bigint user_id PK
        varchar user_name
        timestamp created_at
    }
    DEVICE {
        uuid device_id PK
        bigint user_id FK
        timestamp last_logged_in
    }
    WORKSPACE {
        bigint id PK
        bigint owner_id FK
        boolean is_shared
        timestamp created_at
    }
    FILE {
        bigint id PK
        varchar file_name
        varchar relative_path
        boolean is_directory
        bigint latest_version
        bigint checksum
        bigint workspace_id FK
        timestamp created_at
        timestamp last_modified
    }
    FILE_VERSION {
        bigint id PK
        bigint file_id FK
        uuid device_id FK
        bigint version_number
        timestamp last_modified
    }
    BLOCK {
        bigint block_id PK
        bigint file_version_id FK
        int block_order
    }

    USER ||--o{ DEVICE : "has"
    USER ||--o{ WORKSPACE : "owns"
    WORKSPACE ||--o{ FILE : "contains"
    FILE ||--o{ FILE_VERSION : "has revisions"
    FILE_VERSION ||--o{ BLOCK : "consists of"
```

### Table Purpose Reference

| Table | Purpose |
|---|---|
| **user** | Basic user info: username, email, profile photo |
| **device** | Device info; push_id for mobile notifications. One user → multiple devices |
| **workspace** | Root directory; can be shared |
| **file** | Latest file state |
| **file_version** | Immutable revision history; rows never updated after insert |
| **block** | Reconstruct any version by fetching all blocks for a `file_version_id` sorted by `block_order` |

---

## 8. Upload Flow (Figure 15-14)

Two requests sent **in parallel** from Client 1: add metadata + upload content.

```mermaid
sequenceDiagram
    participant C1 as Client 1
    participant BS as Block Servers
    participant CS as Cloud Storage
    participant API as API Servers
    participant MDB as Metadata DB
    participant NS as Notification Service
    participant C2 as Client 2

    par Add file metadata
        C1->>API: 1. add file metadata
        API->>MDB: 2. store metadata, status=PENDING
        API->>NS: 3. notify: file being added
        NS-->>C2: 4. notify changes
    and Upload file content
        C1->>BS: 2.1 upload file
        BS->>CS: 2.2 upload blocks (chunked, compressed, encrypted)
        CS-->>API: 2.3 upload complete callback
        API->>MDB: 2.4 update status=UPLOADED
        API->>NS: 2.5 notify: file uploaded
        NS-->>C2: 2.6 notify changes
    end
```

---

## 9. Download Flow (Figure 15-15)

```mermaid
sequenceDiagram
    participant C2 as Client 2
    participant BS as Block Servers
    participant CS as Cloud Storage
    participant API as API Servers
    participant MDB as Metadata DB
    participant NS as Notification Service

    NS-->>C2: 1. notify changes (file changed elsewhere)
    C2->>API: 2. get changes (fetch metadata)
    API->>MDB: 3. get changes
    MDB-->>API: 4. return changes
    API-->>C2: 5. metadata returned
    C2->>BS: 6. download blocks
    BS->>CS: 7. download blocks from cloud
    CS-->>BS: 8. blocks returned
    BS-->>C2: 9. blocks returned
    Note over C2: reconstruct file from blocks
```

**How does a client know a file changed?**
- **Online** → notification service informs the client; client pulls latest changes
- **Offline** → changes cached in offline backup queue; client pulls when back online

---

## 10. Notification Service

### Long Polling vs WebSocket

| Option | Best For | Google Drive? |
|---|---|---|
| **Long polling** | Infrequent, unidirectional server→client | ✅ Chosen |
| **WebSocket** | Real-time bi-directional (chat, gaming) | ❌ Overkill |

**Why long polling?**
- File change notifications are infrequent — no continuous data burst
- The flow is unidirectional (server tells client about changes)
- WebSocket is purpose-built for chat-style apps, not file sync
- Dropbox uses the same approach

```mermaid
sequenceDiagram
    participant C as Client
    participant NS as Notification Service
    participant API as API Servers

    C->>NS: open long poll connection
    Note over NS: waiting for changes...
    NS-->>C: file changed! (close connection)
    C->>API: pull latest metadata
    API-->>C: metadata returned
    C->>NS: open new long poll connection
```

---

## 11. Save Storage Space

```mermaid
graph TD
    problem["Storage fills up\n(500 PB at scale)"]
    d1["1. De-duplicate blocks\n(same hash = same block)"]
    d2["2. Cap version history\n(drop oldest when limit hit)"]
    d3["3. Cold storage tiering\n(inactive files → S3 Glacier\n~10× cheaper)"]

    problem --> d1
    problem --> d2
    problem --> d3
```

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
| **Notification service** | Server fails | 1M+ long poll connections drop; clients gradually reconnect to different server |
| **Offline backup queue** | Queue fails | Queues are replicated; consumers re-subscribe |

---

## 13. Design Tradeoffs

### Alternative: Upload Directly to Cloud (Bypass Block Servers)

```mermaid
graph LR
    subgraph current["Current Design"]
        c1[Client] -->|upload| cbs[Block Servers] -->|blocks| ccs[Cloud Storage]
    end
    subgraph alt["Alternative"]
        a1[Client] -->|upload direct| acs[Cloud Storage]
    end
```

| Aspect | Block Servers (Current) | Direct to Cloud |
|---|---|---|
| Upload speed | Slightly slower (extra hop) | Faster — file transferred once |
| Logic location | Centralized in block servers | Must re-implement on iOS, Android, Web |
| Security | Server-side encryption (safer) | Client-side encryption (risky) |
| Maintenance | Single codebase | Three platform implementations |

**Verdict**: Block servers win — centralized security and maintainability.

---

## 14. Interview Q&A — DB Choice & Why

---

### Q1: Why relational DB for metadata? Why not NoSQL?

**Core issue: consistency.**

The system needs **strong consistency** — the same file must look identical to all clients simultaneously.

- **Relational DB (MySQL/PostgreSQL)**: ACID natively supported — consistency guaranteed
- **NoSQL (DynamoDB, Cassandra, MongoDB)**: defaults to eventual consistency — replicas can diverge momentarily
- To use NoSQL, you would implement consistency in application-layer sync logic — complex and error-prone

**Scaling the relational DB:**
- **Read replicas** for horizontal read scaling (1:1 read:write ratio)
- **Sharding** on `user_id` or `workspace_id` for write scaling
- **Metadata cache** (Redis) for hot data — always invalidate cache on write

---

### Q2: What is the metadata cache and why not just always query the DB?

Metadata (file paths, version numbers, checksums) is read far more often than written. The cache (Redis/Memcached) reduces latency for hot data.

**Consistency rule:**
- On every DB write → invalidate or update the corresponding cache entry
- Cache and DB master must always hold the same value

---

### Q3: Why S3 (object storage) for file blocks instead of a database or filesystem?

- S3 is designed for binary blobs of arbitrary size at petabyte scale
- Built-in **same-region and cross-region replication** — files survive regional outages
- Far cheaper per GB than block storage or relational DB at 500 PB scale
- Access pattern is simple: `PUT` / `GET` by key — no need for querying or indexing
- A relational DB with binary blob columns would be massively inefficient at this scale

---

### Q4: Why split files into blocks instead of storing whole files?

- **Delta sync** — only modified blocks transferred on update → bandwidth savings
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
1. First version processed → saved as canonical version
2. Later conflicting version → flagged as a conflict copy
3. User presented with both copies → option to merge or override
4. Example: `SystemDesignInterview_user2_conflicted_copy_2019-05-01`

---

### Q7: How do you guarantee no data loss?

- **Block replication within a region** (S3 same-region replication)
- **Cross-region replication** — regional failure cannot cause data loss
- **Metadata DB replication** — master + slaves; promote slave on master failure
- **Offline backup queue** — change intent persisted; synced when client reconnects

---

### Q8: How do you reduce storage costs at 500 PB scale?

1. **Block de-duplication** — identical hash = shared block storage
2. **Versioning caps** — limit stored revisions; oldest dropped at cap
3. **Cold storage tiering** — inactive files → S3 Glacier (~10× cheaper than S3 Standard)

---

### Q9: What are the most interesting DB schema decisions?

- **`file_version` rows are immutable** — never updated after insert; preserves revision history integrity
- **`block` has `block_order`** — reconstruct any file version by fetching blocks for a `file_version_id` sorted by `block_order`
- **`checksum` on `file`** — enables de-duplication and integrity verification
- **Separate `device` table** — one user can have multiple devices; push notifications are device-scoped
- **`file` vs `file_version`** — `file` = current state; `file_version` = immutable history

---

### Q10: Why are two requests sent in parallel during upload?

1. **Metadata flow**: Client → API servers → Metadata DB (store entry, status = "pending")
2. **Content flow**: Client → Block servers → Cloud storage

They are independent — metadata storage does not depend on content upload completing. Parallelism minimizes total upload latency. Final status set to "uploaded" only after cloud storage confirms completion.

---

### Q11: How would the design change if files could be larger than 10 GB?

- Resumable uploads become even more critical
- S3 multipart upload natively handles files up to 5 TB
- Block size tuned upward to reduce block count per file
- Block servers need higher parallelism in the pipeline
- `block` table may need partitioning by `file_version_id` at extreme scale

---

*Source: System Design Interview — An Insider's Guide, Vol. 1 (Alex Xu)*
