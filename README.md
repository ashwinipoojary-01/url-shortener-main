# URL Shortener with Analytics.

A production-style URL shortener built to demonstrate system design fundamentals:
custom base62 encoding, cache-aside Redis caching, an async analytics ingestion
pipeline, geo/device tracking, and Redis-backed rate limiting.

## Stack.

- **API**: Node.js + Express
- **Frontend**: React (Vite) — `client/`
- **Source of truth**: MongoDB (Mongoose) — `Url` collection (normal) + `ClickEvent`
  collection (time-series)
- **Cache / queue broker / rate-limit store**: Redis (ioredis + BullMQ)
- **Async workers**: BullMQ worker process, deployed separately from the API

## Quick start.

Everything at once (Mongo + Redis + API + worker + React client):

```bash
cp .env.example .env
cp client/.env.example client/.env
docker compose up --build
# client:  http://localhost:5173
# API:     http://localhost:3000
```

Or run pieces individually (needs Mongo + Redis running locally first):

```bash
npm install
npm start            # API server        -> :3000
npm run worker       # analytics worker  (separate process)

cd client
npm install
npm run dev           # React client      -> :5173
```

### Frontend

A single-page app (`client/`) covering the whole flow: paste a URL, get a
short one, browse your links, expand any link to see click analytics
(clicks-by-day bar chart, device donut chart, top countries) — built with
plain React + Recharts, no extra state library needed at this scope.

Since the backend has no real auth system (see "What's intentionally out of
scope" below), the client generates a random id on first visit, persists it
in `localStorage`, and sends it as `X-API-Key` on every request — enough to
scope "my links" to a browser without building a login flow.

## API

| Method | Path                        | Description                          |
|--------|-----------------------------|---------------------------------------|
| POST   | `/api/shorten`               | Create a short URL                    |
| GET    | `/:shortCode`                 | Redirect + fire-and-forget analytics  |
| DELETE | `/api/urls/:shortCode`        | Deactivate a short URL                |
| GET    | `/api/analytics/:shortCode`   | Aggregated click analytics            |
| GET    | `/health`                     | Liveness check                        |

```bash
curl -X POST http://localhost:3000/api/shorten \
  -H "Content-Type: application/json" \
  -d '{"longUrl": "https://example.com/some/very/long/path"}'
# -> {"shortCode":"1CgEiz","shortUrl":"http://localhost:3000/1CgEiz",...}

curl -L http://localhost:3000/1CgEiz   # redirects, and enqueues a click event

curl http://localhost:3000/api/analytics/1CgEiz
```

## Design decisions & tradeoffs (interview talking points)

### 1. Short code generation — counter vs. random

Two strategies, both implemented (`SHORT_CODE_STRATEGY` env var):

- **`counter`** (default): a Redis `INCR` produces a globally-unique,
  monotonically increasing integer, which is base62-encoded. Zero collision
  probability by construction — no retry loop. To avoid exposing an obviously
  sequential, enumerable ID (`1, 2, 3...` -> `b, c, d...`), the counter is
  permuted with a reversible linear map before encoding. At extreme write
  volume, a single Redis key becomes a hot key; the mitigation documented in
  `base62.js`/`urlService.js` is sharded counters or pre-allocated ID blocks
  per app instance.
- **`random`**: generate a random 7-char base62 string directly (62^7 ≈ 3.5
  trillion combinations), check uniqueness in Mongo, retry on the rare
  collision. Harder to enumerate/scrape than sequential IDs, at the cost of a
  uniqueness check + retry loop on write.

### 2. Caching strategy — cache-aside with write-through on create

Reads go cache-first, falling back to Mongo on miss and repopulating the
cache (classic cache-aside / lazy loading). Writes populate the cache
immediately (write-through) because a just-created link is disproportionately
likely to be clicked right after creation (share-and-click pattern) — waiting
for the first read to populate the cache would guarantee a miss on the very
first, most time-sensitive click.

TTL (default 1hr) trades staleness against hit ratio. Deactivation explicitly
invalidates the cache key rather than relying on TTL expiry, since that's a
deliberate consistency-sensitive action.

### 3. Async analytics ingestion

The redirect handler is the hottest path in the system — it should do the
minimum possible work before returning a 302. Click analytics (geo lookup, UA
parsing, DB write) is enqueued to BullMQ (Redis-backed) and processed by an
independently-scalable worker process, decoupling redirect latency from
analytics processing entirely. Job durability comes from Redis persistence +
BullMQ retry/backoff; a dead Redis (not just a dead worker) is the one
scenario that loses in-flight events — an acceptable tradeoff for
best-effort analytics vs. the operational cost of a stronger broker like
Kafka.

### 4. Click event storage — time-series collection

`ClickEvent` uses a MongoDB time-series collection (bucketed by `shortCode`,
timed on `clickedAt`). This workload is high-write, append-only, and queried
by time range per link — exactly the profile time-series collections are
built for (better compression, faster range scans than a normal collection
at this write volume). The `Url.clickCount` field is a denormalized fast
counter, incremented by the same worker, so "total clicks" reads don't
require a full aggregation.

### 5. Rate limiting

Redis-backed (`rate-limiter-flexible`) rather than in-memory, because an
in-memory limiter's state is per-process — once there's more than one app
instance behind a load balancer, each instance enforces the limit
independently and the effective limit becomes `N × instance_count`. Two
separate limiter buckets: a generous one for redirects (protects against
scraping without punishing legitimately popular links) and a stricter one
for URL creation (protects the more expensive write path from spam).

### 6. Horizontal scaling

- **API tier**: fully stateless — any instance can serve any request, since
  all shared state (cache, rate-limit counters, ID counter, queue) lives in
  Redis/Mongo, not in process memory. Scale by adding instances behind a
  load balancer (`docker compose up --scale api=3`).
- **Worker tier**: scales independently of the API tier based on analytics
  ingestion volume, not request volume.
- **MongoDB**: read replicas for analytics query load; the `Url` collection
  is small and hot enough to mostly live in Redis cache regardless. At very
  large scale, shard `Url` by `shortCode` hash and `ClickEvent` by
  `shortCode` (already the time-series `metaField`).
- **Redis**: the ID counter is the one piece of genuinely centralized
  state. Redis Cluster with hash-tagged sharded counters, or a move to
  pre-allocated ID blocks per instance, removes that single point of
  contention if it becomes a bottleneck.

## What's intentionally out of scope

- Auth/API keys are stubbed via an `X-API-Key` header, not a real user
  system — out of scope for the system-design exercise this project targets.
- IP addresses are stored raw in `ClickEvent` for simplicity; production
  would truncate or hash them for privacy compliance.
- No CDN/edge-redirect layer (e.g. resolving hot short codes at a CDN edge
  worker) — a natural next step once redirect volume justifies it.
