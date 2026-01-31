# Twitter Feed (Home Timeline) — Design Walkthrough 🔧✨

## Overview

**Goal:** Design a scalable, low-latency home timeline (feed) service that delivers a personalized, ordered stream of tweets to users in near real-time.

**Scope:** posting tweets, follow/unfollow, fetch home timeline, pagination, and basic ranking. Out of scope: full search, ads, or external integrations.

---

## Assumptions & Scale

- Typical target: 200M active users.
- Example scale (adjustable): 500k writes/sec peak, 2M reads/sec peak.
- Fanout distribution: most users have modest follower counts; a small fraction (celebrities) have millions.

> Ask product: confirm expected active users, reads/writes per second, and personalization depth.

---

## Requirements

### Functional
- Post a tweet and make it visible to followers.
- Fetch `GET /feed?user_id&limit&cursor` with sub-second latency.
- Support follow/unfollow affecting subsequent feeds.
- Pagination and streaming/real-time updates (optional push notifications).

### Non-functional
- Highly available (99.9%+), low read latency (p95/p99 targets), durable writes.
- Scalable for large fanouts and spikes.
- Freshness: new tweets visible in seconds.

---

## API (examples)

- POST /tweet
  - Request: `{ "user_id": "u123", "tweet_id": "t123", "text": "hello" }`
  - Response: `201` with created tweet metadata

- POST /follow
  - Request: `{ "follower_id":"u1", "followee_id":"u2" }` → `200`

- GET /feed?user_id={id}&limit={n}&cursor={c}
  - Response: `{ "tweets": [{tweet}], "next_cursor": "..." }`

---

## Data Model (conceptual)

- Tweet: `{ tweet_id, user_id, text, created_at, metadata }`
- Follow: `{ follower_id, followee_id, created_at }`
- Timeline entry: `{ user_id, tweet_id, author_id, created_at, score }`

Storage choices: Cassandra for timeline storage (append-friendly, tunable consistency), Redis for hot caches, S3 for long-term cold storage (media & old tweets), Kafka for durable event ingestion.

---

## High-level strategies (tradeoffs)

### Fan-out-on-write (push)
- On tweet: write timeline entries into each follower's timeline partition.
- Pros: very fast reads.
- Cons: expensive writes (storage duplication), problematic for very large follower counts.

### Fan-out-on-read (pull)
- Store tweets per author; on read, fetch recent tweets from the followees and merge/sort.
- Pros: cheap writes, no duplication.
- Cons: expensive and higher-latency reads; merging complexity.

### Hybrid (recommended)
- Push for users with small follower counts. For high-fanout authors use pull or partial push (sampled or cached recent tweets).
- Dynamic thresholding and background fanout workers.

---

## Read path (detailed)
1. Client → API Gateway → Timeline Service.
2. Timeline Service tries Redis hot cache (`feed:<user_id>`).
3. If cache miss → read precomputed slice from Cassandra partition for `user_id`.
4. Hybrid: merge in recent celebrity tweets (pull) and call ranking service to reorder top-K.
5. Return paginated results; optionally push via WebSocket/Push for real-time subscribers.

---

## Write path (detailed)
1. User posts tweet → Auth + Write Service validates & persists tweet to durable store and publishes to Kafka (`tweets` topic).
2. Fanout workers consume `tweets`:
   - For small-fanout authors: write entries into follower timeline partitions in Cassandra and update Redis caches.
   - For large-fanout authors: write to author timeline and optionally push to a sampled subset or to a high-speed cache (avoid mass writes).
3. Notify push/real-time service for subscribed clients; update monitoring metrics.

---

## Ranking & Personalization
- Lightweight ranking: recency + author weight + engagement boosts.
- Advanced ranking: feature gen + ML ranking service that scores candidate tweets (offline features + online features).
- Keep ranking compute minimal on read: fetch candidates and call Ranker for top-K reordering.

---

## Hot users & scaling techniques
- Partition follower lists; parallelize fanout across workers.
- Threshold-based strategy: above threshold use on-read merge or selective push.
- Salt partition keys for very hot timeline partitions.
- Use back-pressure and degrade gracefully (serve stale cache or partial feed when under overload).

---

## Consistency, Idempotency & Fault Tolerance
- Use `tweet_id` for idempotent writes; dedupe in ingestion.
- Kafka as durable stream (replayable); fanout workers are idempotent.
- Dead-letter queue + reconciliation job to re-fanout on error.
- Periodic batch reconciliation for correctness after code changes.

---

## Caching & TTLs
- Redis: per-user `feed:<user_id>` list (most recent N tweets), TTL few minutes.
- Cache invalidation on new tweets or follow/unfollow events.
- Pre-aggregate multi-granularity slices for heavy queries.

---

## Monitoring & SLOs
- Metrics: read latency p50/p95/p99, fanout worker lag, Kafka consumer lag, cache hit ratio.
- Alerts: queue depth, p99 latency spikes, fanout failures.
- Example SLOs: 99% reads < 200ms; fanout worker lag < 10s.

---

## Anti-abuse & Rate Limits
- Rate-limit posting, follows, and feed reads per IP & account.
- Spam detection and throttling for suspicious accounts.
- Throttle fanout for accounts flagged as abusive.

---

## Example schemas

Cassandra timeline (conceptual):

```
CREATE TABLE user_timeline (
  user_id text,
  created_at timestamp,
  tweet_id uuid,
  author_id text,
  score float,
  PRIMARY KEY (user_id, created_at)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

Redis keys:

- `feed:<user_id>` -> list of tweet_ids (most recent N)
- `tweet:<tweet_id>` -> tweet payload (cache)

---

## Diagrams (placeholders)
- Component diagram: `../resources/img/design-twitter-feed-component.svg` (add when ready)
- Sequence diagram (posting → fanout → read): `../resources/img/design-twitter-feed-sequence.svg`

---

## Tradeoffs summary
- Push (fanout-write): fast reads, expensive writes → good for most users.
- Pull (fanout-read): cheap writes, heavier reads → required for celebrities.
- Hybrid balances latency and cost, adds operational complexity.

---

## Next steps
1. Add component & sequence diagrams to `resources/img/`.
2. Add example Ranker API + sample payloads.
3. Add a small reconciliation script and verification queries for offline jobs.

---

*Drafted by GitHub Copilot (Raptor mini (Preview)).*