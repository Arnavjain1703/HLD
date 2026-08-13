# ⚽ Soccer Ticket Booking Platform — High Level Design

> **System Design Interview Reference**
> Microservices · PostgreSQL · Redis · Elasticsearch · Stripe · SSE · Virtual Waiting Queue

---

## Table of Contents

1. [Requirements](#1-requirements)
2. [Core Entities](#2-core-entities)
3. [API Design](#3-api-design)
4. [High Level Architecture](#4-high-level-architecture)
5. [Two-Phase Booking Flow](#5-two-phase-booking-flow)
6. [Deep Dives](#6-deep-dives)
7. [Design Summary](#7-design-summary)

---

## 1. Requirements

### Functional Requirements

| # | Requirement |
|---|---|
| FR-1 | Users can **search** for matches by team, competition, city, or date |
| FR-2 | Users can **view a match** — teams, venue, competition, interactive seat map |
| FR-3 | Users can **reserve a seat** — exclusive 10-minute hold |
| FR-4 | Users can **confirm purchase** and receive a digital e-ticket |

> **Out of scope:** Admin event creation · team/player management · resale marketplace · multi-seat selection

---

### Non-Functional Requirements

| Quality | Requirement | Why it matters |
|---|---|---|
| 🔒 **Strong Consistency** | No double booking. One seat → one user. | ACID writes on ticket status; Redis NX atomic lock |
| ⚡ **High Availability** | Search/browse highly available; eventual consistency OK (seconds) | Booking failures must not affect browsing |
| 🌊 **Surge Scalability** | El Clásico / World Cup Finals → 10–100M concurrent users | Virtual queue gates traffic before it reaches the backend |
| 🔍 **Low Latency Search** | <200ms P99 for search queries | Full-text + geospatial via Elasticsearch; CDN caches top queries |
| 📖 **Read >> Write** | Read:Write ≈ 100:1 | Cache static event data aggressively; tickets are the only dynamic entity |

---

## 2. Core Entities

### PostgreSQL Schema

```
┌─────────────────────────┐     ┌─────────────────────────┐
│         Match           │     │         Venue           │
├─────────────────────────┤     ├─────────────────────────┤
│ PK  id: UUID            │     │ PK  id: UUID            │
│ FK  homeTeamId: UUID    │     │     name: VARCHAR       │
│ FK  awayTeamId: UUID    │     │     city, country       │
│ FK  venueId: UUID  ─────┼────▶│     latitude: DECIMAL   │
│     kickoffTime: TSTZ   │     │     longitude: DECIMAL  │
│     competition: VARCHAR│     │     capacity: INTEGER   │
│     status: ENUM        │     │     seatMapUrl: TEXT    │
│     name, description   │     └─────────────────────────┘
└────────────┬────────────┘
             │ 1:N
             ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│         Ticket          │     │         Team            │
├─────────────────────────┤     ├─────────────────────────┤
│ PK  id: UUID            │     │ PK  id: UUID            │
│ FK  matchId: UUID       │     │     name: VARCHAR(100)  │
│     section: VARCHAR    │     │     logoUrl: TEXT       │
│     row: VARCHAR        │     │     country: VARCHAR    │
│     seatNumber: INTEGER │     │     leagueId: VARCHAR   │
│     price: DECIMAL(10,2)│     └─────────────────────────┘
│     status: ENUM        │
│       available|booked  │     ┌─────────────────────────┐
│ FK? bookedByUserId:UUID │     │        Booking          │
└─────────────────────────┘     ├─────────────────────────┤
                                │ PK  id: UUID            │
┌─────────────────────────┐     │ FK  ticketId: UUID UNQ  │
│          User           │     │ FK  userId: UUID        │
├─────────────────────────┤     │     stripeIntentId: UNQ │
│ PK  id: UUID            │     │     status: ENUM        │
│     email: UNIQUE       │     │       pending|confirmed │
│     name: VARCHAR       │     │       |failed           │
│     passwordHash: TEXT  │     │     createdAt: TSTZ     │
│     stripeCustomerId    │     └─────────────────────────┘
│     createdAt: TSTZ     │
└─────────────────────────┘
```

### Redis Data Structures

```
# ── Seat Lock (distributed, atomic, auto-expiring) ──────────────
SET  ticket:{ticketId}:lock  "1"  EX 600  NX
# OK  → reserved successfully
# nil → already locked → return 409 Conflict

DEL  ticket:{ticketId}:lock        # on confirm success
# Timeout: key auto-evicts at 600s → seat returns to pool

# ── Virtual Waiting Queue (surge control) ────────────────────────
ZADD   waitingQueue:{matchId}  {unix_timestamp}  {userId}
ZRANK  waitingQueue:{matchId}  {userId}          # queue position
ZPOPMIN waitingQueue:{matchId}  500              # admit next 500

# ── Match Cache (immutable, no TTL) ─────────────────────────────
SET  match:{matchId}  {json}
```

---

## 3. API Design

| Method | Endpoint | Description | Request | Response |
|---|---|---|---|---|
| `GET` | `/matches/{matchId}` | View match + seat map | Path: `matchId` | `{ match, venue, homeTeam, awayTeam, tickets[] }` |
| `GET` | `/search/matches` | Search (Elasticsearch) | `?q=Barcelona&location=Madrid&date=2026-09-01&competition=UCL` | `[{ id, name, kickoffTime, venue, teams, minPrice }]` |
| `POST` | `/bookings/reserve` | Phase 1 — lock seat 10 min | Body: `{ ticketId }` · Header: `Authorization: Bearer <JWT>` | `200: { reservationId, expiresAt }` · `409: already reserved` |
| `POST` | `/bookings/confirm` | Phase 2 — complete purchase | Body: `{ ticketId, stripePaymentIntentId }` · Header: `Authorization: Bearer <JWT>` | `200: { bookingId, eTicketUrl }` · `402: payment failed` |
| `POST` | `/bookings/stripe-callback` | Stripe webhook (internal) | `Stripe-Signature` + event payload | `200 ACK` → updates Postgres + sends e-ticket |
| `GET` | `/matches/{matchId}/seat-updates` | SSE real-time seat stream | `Accept: text/event-stream` | Streaming: `data: { ticketId, status }` on each change |

> ⚠️ User identity is **never** passed in the request body.
> It flows exclusively through `Authorization: Bearer <JWT>` to prevent spoofing.

---

## 4. High Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                  CLIENT                                      │
│                    Browser / Mobile App — React SPA                          │
│               SSE connection · Seat map · 10-min countdown                   │
└─────────────────┬───────────────────────────────────────┬───────────────────┘
                  │ HTTPS / SSE                           │ static assets
                  ▼                                       ▼
┌─────────────────────────────────┐         ┌────────────────────────────┐
│           API Gateway           │         │            CDN             │
│  JWT Auth · Rate Limit · Route  │         │   CloudFront / Fastly      │
│       Load Balancing            │         │   Static + Search cache    │
│                                 │         │        (30s TTL)           │
└────┬──────────────┬─────────────┘         └────────────────────────────┘
     │              │              │
     ▼              ▼              ▼
┌──────────┐  ┌──────────┐  ┌──────────────┐
│  Match   │  │  Search  │  │   Booking    │
│ Service  │  │ Service  │  │   Service    │
│          │  │          │  │              │
│ View     │  │ Full-text│  │ Reserve      │
│ match    │  │ + Geo    │  │ Confirm      │
│ Seat map │  │ search   │  │ Stripe hook  │
└────┬─────┘  └────┬─────┘  └──────┬───────┘
     │              │               │
     │ reads        │ queries       │ writes
     ▼              ▼               ▼
┌──────────┐  ┌──────────┐  ┌──────────────┐
│Postgres  │  │Elastic-  │  │   Redis      │
│          │  │search    │  │              │
│Primary   │  │          │  │ Seat lock    │
│data store│  │Inverted  │  │ (NX TTL 10m) │
│ACID      │  │index     │  │              │
│          │  │Geo index │  │ Virtual      │
│+ Read    │  │          │  │ queue        │
│Replicas  │  │AWS Open- │  │ (Sorted Set) │
│          │  │Search    │  │              │
└────┬─────┘  └──────────┘  │ Match cache  │
     │                       │ Pub/sub      │
     │ CDC stream             └──────────────┘
     └──────────────────────▶ Elasticsearch
                               (sync)

┌──────────────────────┐     ┌──────────────────────────┐
│       Stripe         │     │   Notification Service   │
│                      │     │                          │
│ PaymentIntents       │     │ Email / SMS              │
│ Idempotency keys     │     │ e-Ticket delivery        │
│ Webhook callback     │     │ On booking confirmed     │
└──────────────────────┘     └──────────────────────────┘

┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐
  Virtual Waiting Queue  (admin-gated per match)
│ Redis Sorted Set · Position via SSE           │
  El Clásico · World Cup Finals · UCL Final
└ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘
```

---

## 5. Two-Phase Booking Flow

```
CLIENT          BOOKING SVC       REDIS LOCK        POSTGRES / STRIPE
  │                  │                 │                    │
  │  [1] Clicks seat │                 │                    │
  │──────────────────▶                 │                    │
  │                  │                 │                    │
  │  [2] POST /bookings/reserve        │                    │
  │──────────────────▶                 │                    │
  │                  │  [3] SET NX     │                    │
  │                  │  EX 600         │                    │
  │                  │────────────────▶│                    │
  │                  │                 │                    │
  │                  │◀──── OK ────────│                    │
  │◀── 200 { reservationId, expiresAt }│                    │
  │                  │                 │                    │
  │  [4] 10-min countdown starts       │                    │
  │      (payment form shown)          │                    │
  │                  │                 │                    │
  │  [5] POST /bookings/confirm        │                    │
  │──────────────────▶                 │                    │
  │                  │  [6] POST payment intent             │
  │                  │────────────────────────────────────▶ │
  │                  │                 │   Stripe processes │
  │                  │                 │   (async)          │
  │                  │  [7] Webhook: payment succeeded      │
  │                  │◀──────────────────────────────────── │
  │                  │                 │                    │
  │                  │  [7a] UPDATE tickets SET status='booked'
  │                  │────────────────────────────────────▶ │
  │                  │  [7b] DEL lock  │                    │
  │                  │────────────────▶│                    │
  │                  │                 │                    │
  │◀── 200 { bookingId, eTicketUrl }   │                    │
  │                  │                 │                    │

  ─ ─ ─ ─ TIMEOUT PATH ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
  If user closes browser / timer expires (600s):
    Redis TTL evicts key automatically
    → Seat returns to available pool
    → No DB write, no cron job needed
  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
```

---

## 6. Deep Dives

### 6.1 Low Latency Search — Elasticsearch

**Problem:** SQL `LIKE '%term%'` requires a full table scan — O(n), unusable at scale.

**Solution:** Elasticsearch with inverted index (text) + geospatial index (location).

```json
// Elasticsearch query (simplified)
{
  "query": {
    "bool": {
      "must": [{
        "multi_match": {
          "query": "Barcelona",
          "fields": ["name", "homeTeam", "awayTeam", "competition"]
        }
      }],
      "filter": [
        {
          "geo_distance": {
            "distance": "50km",
            "venue.location": { "lat": 40.4, "lon": -3.7 }
          }
        },
        {
          "range": { "kickoffTime": { "gte": "now" } }
        }
      ]
    }
  }
}
```

| Layer | Mechanism | Benefit |
|---|---|---|
| **Inverted index** | Tokenise name, team, competition → term → [matchId…] | O(1) lookup instead of O(n) scan |
| **Geo index** | `geo_point` field on venue lat/lng + geo-hash | Fast radius queries ("matches near Madrid") |
| **Node query cache** | AWS OpenSearch top-10K queries cached per shard | Zero extra infra; config flag |
| **CDN edge cache** | `GET /search?q=...` cached at edge for 30s TTL | Eliminates origin load during sale spikes |
| **CDC sync** | Change Data Capture stream Postgres → ES | Matches/venues rarely change; no queue needed |

---

### 6.2 Real-Time Seat Map — Server-Sent Events

**Problem:** Seat map loads once, immediately goes stale. Users click already-reserved seats → confusing errors.

**Solution:** SSE persistent connection from client to Match Service.

```js
// Client — open SSE on page load
const es = new EventSource(`/matches/${matchId}/seat-updates`);

es.onmessage = (e) => {
  const { ticketId, status } = JSON.parse(e.data);
  updateSeatMap(ticketId, status);  // grey out seat instantly
};

// Server — on reserve / confirm
// Booking Service publishes to Redis pub/sub:
//   PUBLISH seat-updates:{matchId}
//           '{"ticketId":"abc","status":"reserved"}'
//
// Match Service subscribes and forwards
// to all SSE clients viewing that match
```

**SSE vs WebSocket:** Unidirectional (server → client) is sufficient. SSE is simpler, native browser support, auto-reconnects, works over standard HTTP/2.

**Fallback:** Poll every 5s for sessions under 2 minutes — simpler, acceptable lag.

---

### 6.3 Surge Handling — Virtual Waiting Queue

**Problem:** 10–100M users simultaneously when El Clásico tickets go live. Backend collapses. Seat map goes black immediately.

**Solution:** Virtual Waiting Queue gated per match in Redis.

```
10M Users
    │
    ▼
┌─────────────────────────────────┐
│   Virtual Waiting Queue         │
│   Redis Sorted Set              │
│   score = arrival timestamp     │
│   ZADD / ZRANK / ZPOPMIN        │
└────────────────┬────────────────┘
                 │ drain N users when
                 │ N reservations complete
                 ▼
         ┌───────────────┐
         │ Booking Svc   │  ← controlled, steady load
         │ (normal flow) │
         └───────────────┘
```

```bash
# User arrives → join queue
ZADD waitingQueue:{matchId}  {unix_timestamp}  {userId}

# Push position to user via SSE:
# "You are #4,821 in the queue (~12 min)"
ZRANK waitingQueue:{matchId}  {userId}

# When 500 reservations complete → admit next 500
ZPOPMIN waitingQueue:{matchId}  500

# Issue each user a short-lived admission token (JWT)
# Push token via SSE → client navigates to event page
# Booking Service validates token before allowing /reserve
```

| Design choice | Reason |
|---|---|
| Admin-gated per match | Regular matches bypass queue; no overhead |
| Score = arrival timestamp | Natural FIFO; `ZRANK` position in O(log n) |
| Event-driven admission | After N confirmations complete → admit N more; steady flow |
| SSE for position updates | No polling; same connection already open for seat map |
| Optional: randomise score | Fairness — prevents first-come-network-wins across regions |

---

### 6.4 Preventing Double Booking — Strong Consistency

**Three layers of protection:**

```
Layer 1 — Redis SET NX (atomic lock)
─────────────────────────────────────────────────────
SET ticket:{ticketId}:lock "1" EX 600 NX
  OK  → you got the lock, proceed
  nil → already locked → return 409 Conflict immediately

NX flag makes this atomic — no race condition possible.

Layer 2 — PostgreSQL row-level lock on confirm
─────────────────────────────────────────────────────
UPDATE tickets
SET    status = 'booked', bookedByUserId = $1
WHERE  id = $2
AND    status = 'available'   ← guard clause

If Redis fails and two requests race:
  → Only one DB write wins (0 rows updated = error for loser)
  → No data corruption

Layer 3 — Stripe idempotency
─────────────────────────────────────────────────────
stripePaymentIntentId has UNIQUE constraint in bookings table.
Duplicate webhook retries → no-op.
```

**Redis failure scenario:** Lock lost → two users may reach confirm. Postgres ACID catches the race. One succeeds, one gets an error. Bad UX for a brief window; **zero data corruption**. Acceptable trade-off (agreed with product).

---

### 6.5 Read Scalability — 100:1 Read/Write

```js
// Cache-aside for match details (near-immutable data)
async function getMatch(id) {
  const hit = await redis.get(`match:${id}`);
  if (hit) return JSON.parse(hit);  // O(1) Redis hit

  const row = await db.query(`
    SELECT m.*, v.*, ht.name AS homeTeam, at.name AS awayTeam
    FROM   matches m
    JOIN   venues v  ON v.id  = m.venueId
    JOIN   teams  ht ON ht.id = m.homeTeamId
    JOIN   teams  at ON at.id = m.awayTeamId
    WHERE  m.id = $1
  `, [id]);

  // No TTL — invalidate only on admin update
  await redis.set(`match:${id}`, JSON.stringify(row));
  return row;
}

// Tickets are NOT cached — always read from Postgres
// + cross-reference Redis lock to determine availability
```

| Strategy | What it solves |
|---|---|
| Redis cache (match/venue/team) | Near-immutable data; eliminates repeat DB joins |
| Tickets never cached | Dynamic entity; stale ticket data = bad UX |
| Postgres read replicas | All SELECT → replica; writes → primary; linear read scaling |
| Shard on `venueId` | High correlation between user location and match; fewer shards hit per query |
| CDN | Team logos, stadium images, seat-map SVGs served at edge; zero origin load |
| ES sharding on `venueId` | Geo-filtered searches resolve to a single shard in most cases |

---

## 7. Design Summary

### Functional Requirements Checklist

| # | Requirement | How it's satisfied |
|---|---|---|
| ✅ FR-1 | Search matches | Elasticsearch full-text + geospatial index + CDN edge cache |
| ✅ FR-2 | View match + seat map | Match Service → Postgres join + Redis lock cross-ref |
| ✅ FR-3 | Reserve a seat | `Redis SET NX EX 600` — atomic, distributed, auto-expiring |
| ✅ FR-4 | Confirm purchase + e-ticket | Stripe webhook → Postgres write → Notification Service |

### Non-Functional Requirements Checklist

| # | Requirement | How it's satisfied |
|---|---|---|
| ✅ NFR-1 | No double booking | Redis NX atomic lock + Postgres ACID row-level guard |
| ✅ NFR-2 | High availability (search/browse) | Elasticsearch cluster + CDN + Postgres read replicas |
| ✅ NFR-3 | Low latency search (<200ms P99) | ES inverted index + node query cache + CDN edge |
| ✅ NFR-4 | Surge scalability (El Clásico) | Virtual queue — Redis Sorted Set + SSE position feed |
| ✅ NFR-5 | Real-time seat map | SSE persistent connection + Redis pub/sub fan-out |

### Technology Decisions

| Technology | Role | Why chosen |
|---|---|---|
| **PostgreSQL** | Primary data store | ACID transactions, row-level locking, read replicas, sharding on venueId |
| **Redis** | Seat lock + cache + virtual queue + pub/sub | Atomic NX, TTL auto-expiry, O(log n) sorted set, in-memory speed |
| **Elasticsearch** | Full-text + geospatial search | Inverted index + geo_point, AWS OpenSearch node query cache |
| **Stripe** | Payment processing | PaymentIntents, async webhook, idempotency keys, PCI compliance |
| **SSE** | Real-time seat updates | Unidirectional push, native browser support, auto-reconnect, HTTP/2 |
| **CDN** | Static assets + search cache | Edge delivery, 30s TTL on search API responses, zero origin load |

---

*Soccer Ticket Booking Platform · High Level Design · System Design Interview Reference*
