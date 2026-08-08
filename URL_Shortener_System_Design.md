# System Design: URL Shortener (e.g. TinyURL)
> Source: *System Design Interview* — Chapter 8

---

## Step 1 — Clarify Scope

### Clarification Q&A (simulate in interview)

| Question | Answer |
|---|---|
| Example of how it works? | `https://www.systeminterview.com/q=chatsystem&c=loggedin&v=v3&l=long` → `https://tinyurl.com/y7keocwj` |
| Traffic volume? | 100 million URLs/day |
| Length of short URL? | As short as possible |
| Allowed characters? | `[0-9, a-z, A-Z]` (62 characters) |
| Can URLs be deleted/updated? | No — assume immutable |

### Core Use Cases
1. **URL shortening** — given a long URL → return a short URL
2. **URL redirecting** — given a short URL → redirect to the original long URL
3. **High availability, scalability, fault tolerance**

---

## Step 2 — Back-of-the-Envelope Estimation

- **Write**: 100 million URLs/day → 100M / 24 / 3600 ≈ **1,160 writes/sec**
- **Read**: 10:1 read-to-write ratio → **11,600 reads/sec**
- **Storage**: 10 years × 365 billion records × 100 bytes avg URL length = **365 TB**

---

## Step 3 — High-Level Design

### API Endpoints (REST)

**URL Shortening:**
```
POST /api/v1/data/shorten
  Request body: { longUrl: longURLString }
  Response: shortURL
```

**URL Redirecting:**
```
GET /api/v1/{shortUrl}
  Response: longURL (HTTP redirect)
```

---

### URL Redirecting: 301 vs 302

| | 301 (Permanent) | 302 (Temporary) |
|---|---|---|
| Browser caches? | Yes — subsequent requests go **directly to long URL** (bypasses tinyurl server) | No — all requests go **through tinyurl server** |
| Server load | Lower (cached after first hit) | Higher (every request hits tinyurl) |
| Analytics tracking | ❌ Hard (requests bypass server) | ✅ Easy (every click tracked server-side) |
| **When to use** | Reduce server load | Track click analytics |

**Redirect flow (Figure 8-2):**
```
Client → GET short URL → TinyURL Server
TinyURL Server → 301/302 (Location: long URL)
Client → GET long URL → Origin Server
```

---

### URL Shortening: Hash Function

Short URL format: `www.tinyurl.com/{hashValue}`

**Requirements for hash function:**
- Each `longURL` maps to exactly one `hashValue`
- Each `hashValue` maps back to the `longURL`

---

## Step 4 — Deep Dive

### Data Model

Store `<shortURL, longURL>` in a relational database:

```
┌────────────────────────┐
│        url Table        │
├────────────────────────┤
│ PK  id (auto increment) │
│     shortURL            │
│     longURL             │
└────────────────────────┘
```

---

### Hash Value Length

Characters: `[0-9, a-z, A-Z]` = 10 + 26 + 26 = **62 possible characters**

| n (length) | Max URLs supported |
|---|---|
| 1 | 62 |
| 2 | 3,844 |
| 3 | 238,328 |
| 4 | 14,776,336 |
| 5 | 916,132,832 |
| 6 | 56,800,235,584 |
| **7** | **3,521,614,606,208 (~3.5 trillion)** ✅ |

**n = 7** is sufficient — 3.5 trillion >> 365 billion needed.

---

### Hash Approach 1: Hash + Collision Resolution

Use well-known hash functions (CRC32, MD5, SHA-1):

| Hash Function | Hash Value (Hex) |
|---|---|
| CRC32 | `5cb54054` |
| MD5 | `5a62509a84df9ee03fe1230b9df8b84e` |
| SHA-1 | `0eeae7916c06853901d9ccbefbfcaf4de57ed85b` |

Problem: Even CRC32 (shortest) is 8 chars — too long. **Take first 7 characters.**

**Collision handling flow (Figure 8-5):**
```
longURL → hash function → shortURL
         ↓ hash collision?
         yes → append predefined string → rehash
         no  → save to DB
```

Drawback: DB query on every request to check collision → **use Bloom filter** to speed this up.

---

### Hash Approach 2: Base 62 Conversion ✅ (Preferred)

Convert a unique numeric ID (auto-increment primary key) to a base-62 string.

**Example:** Convert `11157₁₀` to base 62:
```
11157 ÷ 62 = 179, remainder 59  → "X" (59 → 'X' since 0-9=0-9, 10-35=a-z, 36-61=A-Z)
  179 ÷ 62 =   2, remainder 55  → "T"
    2 ÷ 62 =   0, remainder  2  → "2"

Result: "2TX" → short URL: https://tinyurl.com/2TX
```

**Comparison:**

| Property | Hash + Collision Resolution | Base 62 Conversion |
|---|---|---|
| Short URL length | Fixed (7 chars) | Variable (grows with ID) |
| Unique ID generator needed? | No | **Yes** |
| Collision possible? | Yes — must resolve | **No** (ID is unique) |
| Predictable next URL? | No | Yes — security concern |

---

### URL Shortening Flow (Base 62) — Figure 8-7

```
1. Input: longURL
2. Is longURL already in DB?
   → YES: return existing shortURL
   → NO:
     3. Generate new unique ID (primary key)
     4. Convert ID → shortURL via base 62
     5. Save {ID, shortURL, longURL} to DB
     6. Return shortURL
```

**Concrete example:**
- Input: `https://en.wikipedia.org/wiki/Systems_design`
- Unique ID generated: `2009215674938`
- Base 62 conversion: `2009215674938` → `"zn9edcu"`
- Short URL: `https://tinyurl.com/zn9edcu`

> **Note:** The **distributed unique ID generator** is a critical component. It must generate globally unique IDs in a distributed environment. Refer to Chapter 7 (Design a Unique ID Generator in Distributed Systems).

---

### URL Redirecting Flow — Figure 8-8

```
User → GET https://tinyurl.com/zn9edcu
     → Load Balancer
     → Web Servers
          ↓ shortURL in cache?
          YES → return longURL directly (fast)
          NO  → fetch longURL from database
                return longURL to user
```

**Steps:**
1. User clicks short URL
2. Load balancer routes to web servers
3. If shortURL is in **cache** → return longURL immediately
4. If not in cache → fetch from **database** → return longURL
5. The longURL is returned to the user

Cache is critical here — **reads >> writes** (10:1), so cache hit rate will be high.

---

## Step 5 — Wrap Up (Additional Talking Points)

| Topic | Key Points |
|---|---|
| **Rate Limiting** | Filter malicious users sending huge volumes of shortening requests. Filter by IP or other rules. See: "Design a Rate Limiter" |
| **Web Server Scaling** | Web tier is **stateless** → easy horizontal scaling (add/remove servers) |
| **Database Scaling** | Use **replication** (read replicas for 10:1 read load) and **sharding** |
| **Analytics** | Track clicks, timing, geo. Use 302 redirect for accurate click tracking |
| **Availability & Consistency** | Core system design fundamentals — CAP theorem, replication lag |

---

## Quick Cheat Sheet

```
Scale: 100M writes/day → ~1,160 writes/sec, ~11,600 reads/sec
Storage: 365 TB over 10 years
Short URL length: 7 chars (62^7 = 3.5 trillion)
Hash approach: Base 62 (preferred) or Hash+Collision
Redirect type: 302 (analytics) vs 301 (performance)
Cache: shortURL → longURL (high read:write ratio makes this effective)
DB schema: id (PK, auto-increment) | shortURL | longURL
```

---

## Potential Interview Questions & Model Answers

### Clarification / Requirement Questions (Interviewer → You)

| Question | Your Answer |
|---|---|
| What is the expected scale? | 100 million URLs shortened per day |
| How long should the short URL be? | As short as possible — 7 characters is optimal |
| Can users customize their short URL? | Clarify — if yes, store custom alias; check uniqueness before saving |
| Should URLs expire? | Clarify — default no TTL; if yes, add `expiry_timestamp` column |
| Should we support analytics (click tracking)? | Clarify — affects 301 vs 302 redirect choice |
| Should the system be highly available or strongly consistent? | Availability > consistency — it's OK to serve a slightly stale cached URL |

---

### Deep Dive Questions (Interviewer probing your design)

#### On Hash Function

**Q: Why not just use MD5 / SHA-1 directly?**
> Their output is 128-bit / 160-bit hex — way longer than 7 chars. Even truncating to 7 chars causes collisions. You'd need to recursively append a predefined string and re-hash until no collision — expensive DB check on every write.

**Q: How does Base 62 avoid collisions?**
> Because the input is a globally unique auto-incremented integer ID. Two different IDs always produce two different base-62 strings. No DB collision check needed.

**Q: What if two users shorten the same long URL? Do they get the same short URL?**
> Depends on design choice:
> - Check if `longURL` already exists in DB before inserting → return existing `shortURL` (dedup, saves space)
> - Or always generate a new ID → two different short URLs for the same long URL (simpler, wastes space)
> Base 62 flow in the book does dedup (step 2 checks if longURL is in DB).

**Q: How do you generate unique IDs in a distributed environment?**
> Options: (1) Twitter Snowflake — 64-bit ID with timestamp + datacenter + machine + sequence; (2) DB auto-increment with multiple masters risks collision → use ticket server; (3) UUID — no coordination needed but 128-bit is too long for direct use.

---

#### On Redirecting

**Q: When would you choose 301 over 302?**
> 301 = permanent redirect — browser caches it, all subsequent requests skip your server entirely. Good for reducing server load. Bad for analytics since clicks after the first aren't tracked.
> 302 = temporary redirect — every click goes through your server. Good for click tracking, A/B testing, and analytics. Higher server load but manageable with caching.

**Q: How does caching help URL redirecting?**
> Redirecting is read-heavy (10:1 ratio). Cache stores `shortURL → longURL` mappings. On a cache hit you skip the DB entirely. Cache eviction policy: LRU (Least Recently Used) works well — popular URLs stay warm.

---

#### On Scalability

**Q: The system gets 10x traffic. What breaks first and how do you fix it?**
> 1. **Web servers** — stateless, so just add more behind the load balancer.
> 2. **Cache** — scale out with a distributed cache (Redis cluster). Shard by shortURL hash.
> 3. **Database reads** — add read replicas. Route all GET requests to replicas.
> 4. **Database writes** — if write volume is high, shard the DB (e.g. by ID range or consistent hashing).

**Q: How would you shard the database?**
> Option 1: **Range-based sharding** on ID — simple but hotspots if recent IDs are all on one shard.
> Option 2: **Hash-based sharding** on shortURL or ID — distributes load evenly; use consistent hashing to minimise resharding cost when adding nodes.

**Q: What happens if the unique ID generator goes down?**
> This is a single point of failure. Fix: run multiple ID generator instances. Use Snowflake-style IDs where each machine has a unique `machine_id` baked in — no coordination needed between generators.

---

#### On Data Model

**Q: How would you support URL expiry (TTL)?**
> Add an `expiry_timestamp` column to the `url` table. On redirect, check if `NOW() > expiry_timestamp` → return 404. Run a background cleanup job to delete expired rows and free up short URLs.

**Q: How would you support custom aliases (e.g. tinyurl.com/my-brand)?**
> Add a `custom_alias` column. Before inserting, check uniqueness. If taken, return an error. Store it like any other shortURL in the same table — only generation differs (user-provided vs system-generated).

**Q: What if someone tries to shorten a malicious URL?**
> Integrate with a URL safety API (e.g. Google Safe Browsing). Check the longURL before shortening. If flagged, reject the request. Also implement rate limiting per IP to prevent abuse.

---

#### On System Design Breadth

**Q: How would you add analytics — e.g. how many times a URL was clicked?**
> - **Synchronous (simple):** increment a click counter in DB on every redirect. Problem: DB write on every read — high contention.
> - **Asynchronous (scalable):** on redirect, publish a click event to a message queue (Kafka/SQS). A consumer service aggregates and writes to an analytics store (ClickHouse, Redshift). No impact on redirect latency.

**Q: How would you handle a "thundering herd" — a short URL goes viral and gets millions of hits at once?**
> - Cache warm-up: pre-populate cache for known popular URLs.
> - CDN: serve the redirect at the edge — no request hits your origin servers at all.
> - Cache stampede protection: use a mutex/lock so only one request populates the cache on a miss; others wait rather than all hitting the DB simultaneously.

**Q: What if someone keeps generating short URLs to exhaust the ID space?**
> - Rate limiting: cap requests per API key / IP per minute.
> - Require authentication for shortening (logged-in users only).
> - Recycle expired short URLs back into the available pool.

---

### "What would you do differently?" Questions

**Q: Would you use a SQL or NoSQL database?**
> NoSQL (e.g. DynamoDB, Cassandra) fits well here:
> - Data is simple key-value: `shortURL → longURL`
> - No complex joins needed
> - Easy horizontal scaling
> - High read throughput with low latency
> 
> SQL works too if you need ACID guarantees or want to support complex queries (analytics). For pure shortening + redirecting, NoSQL is a better fit at scale.

**Q: Where would you put a CDN in this architecture?**
> In front of the web servers for redirect traffic. The CDN caches `GET /{shortUrl}` responses (the 301/302 + Location header). For 301 redirects especially — the CDN can serve the redirect entirely from the edge with no origin hit after the first request.

**Q: How would you make this multi-region?**
> - Deploy web servers and cache in each region (US, EU, APAC).
> - Use GeoDNS to route users to the nearest region.
> - Replicate the DB across regions (active-active or active-passive).
> - Writes can go to the nearest region; use async replication to sync globally.
> - Caveat: cross-region replication lag means a URL shortened in US may not be visible in EU for a few hundred ms — acceptable for this use case.

