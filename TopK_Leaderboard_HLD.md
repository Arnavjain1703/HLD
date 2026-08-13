# Top K Leaderboard System — High Level Design

---

## Full Architecture Diagram

```mermaid
graph TB
    Client["🎮 Client\n(User Device)"]
    LB["Load Balancer\n& API Gateway"]
    LWS["Leaderboard\nWrite Service\n(Validation + Buffer)"]
    Kafka["Apache Kafka\n(Event Bus / Shock Absorber)"]
    Workers["Score Process\nWorkers"]
    Redis["Redis Cluster\n(Sharded Z-Sets)\n🔥 Hot Store"]
    SQL["SQL Database\n❄️ Cold Store / Source of Truth"]
    LRS["Leaderboard\nRead Service\n(Scatter-Gather)"]
    RAC["Read-Aside Cache\n(Short TTL)"]
    AggSvc["Aggregation Service\n(Cron Job)"]
    IdempotencyStore["Redis\n(Idempotency Keys)"]

    Client -->|"POST /v1/scores\n(+ idempotency key)"| LB
    Client -->|"GET /v1/leaderboards/:id"| LB
    LB --> LWS
    LB --> LRS

    LWS -->|"Check duplicate key"| IdempotencyStore
    LWS -->|"Flush aggregated score\nafter 5s or buffer full"| Kafka

    Kafka -->|"Partitioned by hash(userID)"| Workers
    Workers -->|"ZINCRBY / ZADD\nO(log N)"| Redis
    Workers -->|"Append score history"| SQL

    LRS -->|"Cache hit?"| RAC
    RAC -->|"Cache miss —\nScatter-Gather"| Redis
    Redis -->|"Top-K per shard"| LRS
    LRS -->|"Merge + sort in memory\nStore result"| RAC
    LRS -->|"Return top-K JSON"| Client

    AggSvc -->|"ZUNIONSTORE\ndaily ZSets → rolling week/month"| Redis

    SQL -.->|"Rebuild on disaster recovery"| Redis

    style Redis fill:#dc2626,color:#fff
    style Kafka fill:#f59e0b,color:#000
    style RAC fill:#16a34a,color:#fff
    style SQL fill:#2563eb,color:#fff
    style AggSvc fill:#7c3aed,color:#fff
```

---

## 1. Requirements

### Functional Requirements

| # | Requirement | Description |
|---|-------------|-------------|
| 1 | **Update Score** | Users can submit score updates after a game/level |
| 2 | **Get Top K** | View top 10 or top 100 globally ranked users |
| 3 | **Get User Rank** | Any user can see their own rank + neighbors, even if outside top K |

### Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| **Scalability** | 10M DAU, 50K writes/sec peak, 500K reads/sec |
| **Write Visibility** | Score visible in leaderboard within **1–2 seconds** |
| **Read Latency** | Top-K fetch **< 20ms P99** |
| **Availability** | **99.9%** uptime on read path (stale reads acceptable) |
| **Storage** | ~1–2 GB in Redis (100 bytes/user × 10M users) |

---

## 2. Data Model

```mermaid
erDiagram
    USERS {
        string user_id PK
        string username
        string email
        timestamp created_at
    }
    SCORES {
        string score_id PK
        string user_id FK
        int    delta
        int    total_score
        timestamp updated_at
        string idempotency_key
    }
    USERS ||--o{ SCORES : "has"
```

> **Redis** is used as the live ranking index — not source of truth.  
> **SQL** is the durable source of truth and disaster-recovery store.

---

## 3. API Design (REST)

### Endpoints

```
POST   /v1/scores
GET    /v1/leaderboards/:leaderboardId?k=100
GET    /v1/leaderboards/:leaderboardId/users/:userId?range=5
```

### Request / Response Flows

```mermaid
sequenceDiagram
    participant C as Client
    participant GW as API Gateway
    participant LWS as Write Service
    participant LRS as Read Service

    Note over C,LRS: Score Update Flow
    C->>GW: POST /v1/scores {userId, score, idempotencyKey}
    GW->>LWS: forward + auth
    LWS->>LWS: Validate score (anti-cheat)
    LWS->>LWS: Buffer increment in memory
    LWS-->>C: 202 Accepted

    Note over C,LRS: Read Leaderboard Flow
    C->>GW: GET /v1/leaderboards/global?k=10
    GW->>LRS: forward
    LRS->>LRS: Check read-aside cache
    LRS-->>C: 200 [{rank, userId, score}, ...]
```

---

## 4. Redis Sorted Sets (Z-Sets) — Deep Dive

Redis Z-Sets use **two internal structures** operating in tandem:

```mermaid
graph LR
    subgraph "Z-Set Internal Structure"
        HM["HashMap\nmember → score\nO(1) point lookup"]
        SL["Skip List\nscore-ordered\nO(log N) range queries"]
    end
    HM -->|"ZSCORE leaderboard Bob"| Out1["Returns: 150\n(no list traversal)"]
    SL -->|"ZRANGEBYSCORE leaderboard 100 +inf"| Out2["Returns: Bob(150), Carol(300)\n(skip list traversal)"]
```

### Skip List Traversal — `ZRANGEBYSCORE > 100`

```mermaid
flowchart TD
    L3["Level 3: HEAD → Carol(300)"]
    L2["Level 2: HEAD → Bob(150)"]
    L1["Level 1: HEAD → Alice(100) → Bob(150) → Carol(300)"]

    L3 -->|"300 > 100 → drop down"| L2
    L2 -->|"150 > 100 → drop down"| L1
    L1 -->|"Alice(100) = baseline\nReturn next pointer → Bob(150)"| Result["✅ Result: Bob(150)"]
```

### Operation Complexity

| Operation | Complexity | Redis Command |
|-----------|-----------|---------------|
| Insert / Update score | O(log N) | `ZINCRBY` |
| Get user score | O(1) | `ZSCORE` |
| Get top K | O(log N + K) | `ZREVRANGE` |
| Get user rank | O(log N) | `ZREVRANK` |
| Get neighbors (range) | O(log N + M) | `ZREVRANGEBYSCORE` |

---

## 5. Write Path — Real-Time Counting at Scale

```mermaid
flowchart LR
    Client["Client\nscore events"]
    Buffer["Write Service\nIn-Memory Buffer\n5s window"]
    Kafka["Kafka Topic\nPartitioned by hash(userID)"]
    Workers["Score Process\nWorkers"]
    Redis["Redis Z-Set"]
    SQL["SQL DB"]

    Client -->|"kill+10, kill+10, kill+10\nflush as +30"| Buffer
    Buffer -->|"Aggregated delta\n50K → ~200K writes/sec"| Kafka
    Kafka -->|"Ordered per user"| Workers
    Workers --> Redis
    Workers --> SQL
```

### Idempotency — Handling Server Crashes

```mermaid
sequenceDiagram
    participant C as Client
    participant LWS as Write Service
    participant IR as Redis (Idempotency Store)

    C->>LWS: POST /score {delta:10, key:"uuid-123"}
    LWS->>IR: Has "uuid-123" been processed?
    IR-->>LWS: No
    LWS->>LWS: Process score
    LWS->>IR: Store "uuid-123"
    LWS-->>C: 200 OK

    Note over C,LWS: Server crashes — client retries
    C->>LWS: POST /score {delta:10, key:"uuid-123"}
    LWS->>IR: Has "uuid-123" been processed?
    IR-->>LWS: YES → ignore (deduplicate)
    LWS-->>C: 200 OK (no double count)
```

---

## 6. Scaling the Leaderboard — Sharding Strategies

### Strategy A: Fixed Range Partitioning

```mermaid
graph LR
    S1["Shard 1\nScores 0–1,000\n⚠️ Hotspot risk"]
    S2["Shard 2\nScores 1,001–5,000"]
    S3["Shard 3\nScores 5,001+"]
    Q["Top-K Query"] --> S3
```

- ✅ Easy top-K: just query the highest shard  
- ❌ Hotspot: beginners flood Shard 1

### Strategy B: Hash Partitioning — Scatter-Gather ✅ Recommended

```mermaid
flowchart TB
    User["User Update\nhash(userID) mod N"]
    S1["Redis Shard 1\nUsers: A, D, G..."]
    S2["Redis Shard 2\nUsers: B, E, H..."]
    S3["Redis Shard 3\nUsers: C, F, I..."]

    User --> S1
    User --> S2
    User --> S3

    subgraph "Read — Top-K Scatter-Gather"
        LRS["Leaderboard Read Service"]
        S1 -->|"Top-K from shard"| LRS
        S2 -->|"Top-K from shard"| LRS
        S3 -->|"Top-K from shard"| LRS
        LRS -->|"Merge N×K, sort in memory\nReturn top K"| RAC["Read-Aside Cache\nTTL ~1s"]
    end
```

- ✅ Perfectly balanced write load  
- ⚠️ Read is more expensive (N×K merge), acceptable since K ≤ 100

---

## 7. Historical Data & Time Windows

```mermaid
graph TD
    subgraph "Static Fixed Window — e.g. October Leaderboard"
        FW["leaderboard:october-2026\nZSET key"]
        FW -->|"Period ends"| RO["Read-Only Key\nO(1) access forever"]
    end

    subgraph "Rolling Window — e.g. Last 7 Days"
        D1["leaderboard:day:2026-10-01"]
        D2["leaderboard:day:2026-10-02"]
        D3["leaderboard:day:2026-10-03"]
        D4["... daily ZSets ..."]
        AggSvc["Aggregation Service\nCron every 5 min"]
        D1 --> AggSvc
        D2 --> AggSvc
        D3 --> AggSvc
        D4 --> AggSvc
        AggSvc -->|"ZUNIONSTORE"| RW["leaderboard:rolling-week\nCache Key"]
        RW -->|"Users query"| Result["Instant O(1) read"]
    end
```

| Window Type | Mechanism | Read Complexity |
|-------------|-----------|----------------|
| Fixed (monthly) | Dedicated Z-Set key per period | O(1) after period ends |
| Rolling (7 days) | Daily Z-Sets + `ZUNIONSTORE` via cron | O(1) from merged cache key |

---

## 8. Complete Data Flow

```mermaid
sequenceDiagram
    participant U as User
    participant LWS as Write Service
    participant K as Kafka
    participant W as Score Workers
    participant R as Redis Sharded
    participant DB as SQL DB
    participant LRS as Read Service
    participant C as Read Cache

    U->>LWS: Score 50 points
    LWS->>LWS: Buffer + validate (anti-cheat)
    LWS->>K: Flush aggregated delta
    K->>W: Consume batch (ordered by userID)
    W->>R: ZINCRBY — O(log N), re-sorted instantly
    W->>DB: Append to score history log

    Note over U,C: Another user queries top 10
    U->>LRS: GET /leaderboards/global?k=10
    LRS->>C: Cache hit?
    alt Cache Miss
        LRS->>R: Scatter — top 10 from all shards
        R-->>LRS: 10 results × N shards = N×10 entries
        LRS->>LRS: Merge + sort → final top 10
        LRS->>C: Cache result (TTL ~1s)
    end
    C-->>U: 200 JSON [{rank, userId, score}]
```

---

## 9. Component Summary

| Component | Technology | Role |
|-----------|-----------|------|
| Load Balancer / API GW | AWS ALB / Kong | Traffic distribution, auth, rate limiting |
| Write Service | Stateless backend | Validation, anti-cheat, in-memory aggregation buffer |
| Event Bus | Apache Kafka | Shock absorber, decouples ingestion from processing |
| Score Workers | Consumer group | Batch-consume Kafka → update Redis + SQL |
| Hot Store | Redis Z-Sets (sharded) | Real-time sorted rankings, O(log N) ops |
| Cold Store | PostgreSQL / MySQL | Source of truth, score history, disaster recovery |
| Read Service | Stateless backend | Scatter-gather across Redis shards, merge results |
| Read-Aside Cache | Redis / Memcached | Short-TTL cache of computed top-K results |
| Aggregation Service | Cron job | Builds rolling-window Z-Sets via ZUNIONSTORE |
| Idempotency Store | Redis | Deduplicates retried score submissions |

---

*Generated from system design walkthrough — Top K Leaderboard*
