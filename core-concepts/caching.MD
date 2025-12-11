# Caching

Caching comes up in almost every system design interview, usually when you identify that your database is getting hammered with reads. The idea is simple. Store frequently accessed data in fast memory (like Redis) so you can skip the database entirely for most reads.

The performance difference is massive. A cache hit on Redis takes around 1ms compared to 20-50ms for a typical database query. When you're serving millions of requests, that 20-50x speedup matters. You also reduce load on your database, letting it handle more write traffic and avoiding the need to scale it prematurely.

## Caching Strategies

- **Cache-aside (Lazy loading):** Application checks cache first; on miss, it reads from the database, writes into the cache, and returns the value. Simple and widely used.
- **Read-through:** Cache layer is responsible for fetching from the backing store on misses (often implemented inside a cache client or middleware). The application only interacts with the cache.
- **Write-through:** Writes go to the cache, which synchronously writes to the backing store. Ensures cache and store are always consistent (higher write latency).
- **Write-behind (Write-back):** Writes update the cache and are batched/asynchronously persisted to the backing store. Improves write throughput at the cost of possible data loss on cache failure.
- **Refresh-ahead (Proactive refresh):** Cache proactively refreshes keys before expiration to avoid cold misses and stampedes.

Choose the strategy based on read/write ratio, acceptable staleness, and operational complexity.

## Cache Eviction Strategies

When cache size is limited, eviction decides what to drop. Common policies:

- **LRU (Least Recently Used):** Evict the least recently accessed item. Good default for many workloads.
- **LFU (Least Frequently Used):** Evict items accessed least often. Better when hot items are repeatedly accessed over time.
- **FIFO (First In First Out):** Evict oldest entries. Simple but often suboptimal.
- **TTL / Time-based expiry:** Items expire after a fixed time-to-live. Works well for time-sensitive data.
- **Random eviction:** Choose keys to evict at random. Simple and can avoid pathological behavior in some workloads.
- **Size-based / cost-aware:** Evict items based on memory footprint or cost of recomputation.

Many cache systems combine policies (e.g., LRU + TTL). Tune TTLs and eviction policy based on hit-rate goals and memory constraints.

## Distributed Caching

For large-scale services you need caches that span multiple machines. Key concepts:

- **Sharding / Partitioning:** Split cache keyspace across nodes so each node stores a subset. Reduces per-node memory needs and increases aggregate capacity. Common approaches: client-side hashing, consistent hashing, or dedicated proxy/cluster managers.
- **Replication:** Keep copies of cache entries on multiple nodes for availability and faster regional reads. Replication increases memory cost and complicates consistency.
- **Consistency models:** Most distributed caches are eventually consistent. Decide whether your system requires strong consistency (expensive) or can accept stale reads.
- **Cache coherence & invalidation:** Invalidate or update remote caches after writes using pub/sub, invalidation messages, or write-through paths. Be mindful of message loss and ordering.

Popular systems and patterns:

- **Redis Cluster:** Shards data across nodes; supports replication and failover. Good for rich data structures and operations.
- **Memcached:** Simple, in-memory key-value store; often used with client-side sharding.
- **Client-side caching:** Keep a small LRU cache in the application process for ultra-low latency (beware memory bloat and stale data).

Design considerations:

- Plan for node failures: use replication and fast re-sharding or rebalancing.
- Use consistent hashing to minimize key reallocation when nodes join/leave.
- Beware hotspot keys: some keys are much hotter and can overwhelm a single shard. Techniques: replication for hot keys, request routing, or splitting logical keys.

## Cache Stampede (Thundering Herd) and Mitigation

When many clients simultaneously miss and recompute the same key, the backing store can be overwhelmed. Mitigation techniques:

- **Mutex / Locking:** Use a lock per key (e.g., Redis SETNX) so only one process recomputes while others wait or return stale data.
- **Request coalescing:** Let the first requester recompute and others wait for the result (short waits / subscribers).
- **Stale-while-revalidate:** Serve stale data while asynchronously refreshing the cache.
- **Probabilistic early expiration:** Start refreshing keys slightly before TTL expiration based on access frequency.
- **Backoff + circuit breaker:** If backing store is failing, return degraded responses and avoid retry storms.

## Cache Population & Warming

- **Lazy population:** Populate on demand (cache-aside). Simple but can cause cold-start latency.
- **Eager population / warming:** Preload cache with known hot keys at startup or during deployment.
- **Batch precomputation:** Periodically compute and refresh costly aggregates in the cache.

Warming helps avoid cold-start latency for critical paths but increases system complexity during deploys.

## Observability & Metrics

Track and alert on cache health and behavior:

- **Hit rate / Miss rate:** Primary indicators of effectiveness.
- **Eviction rate:** High evictions suggest insufficient capacity or poor key choice.
- **Latency (p95/p99):** Cache should reduce end-to-end latency; monitor tail latencies.
- **Load on origin DB:** Ensure database queries drop when cache is used.
- **Error rates and instrumentation of invalidation messages.**

Expose metrics from your cache layer (client and server) and dashboard them alongside DB and app metrics.

## Consistency & Invalidation Patterns

Invalidation is the hardest part of caching. Common patterns:

- **Synchronous invalidation after write:** Immediately delete or update cache entries as part of the write flow. Ensures freshness but adds latency to writes.
- **Asynchronous invalidation (event-driven):** Emit events (e.g., via a message queue) to invalidate caches. Faster writes but eventual consistency.
- **Short TTLs:** Accept small staleness windows and rely on TTL to naturally expire data.
- **Versioned keys / namespacing:** Include a version token in keys (e.g., `user:123:v42`) and bump version on major updates to quickly invalidate related sets.
- **Write-through / write-behind:** Use caching layer to keep store and cache in sync automatically.

Choose a pattern aligned with your consistency needs. For strongly consistent workflows (e.g., financial transfers), rely less on caching or use synchronous patterns.

## Security, Privacy, and Cost Considerations

- **Sensitive data:** Avoid caching highly sensitive data in shared caches unless encrypted and access-controlled.
- **Cache poisoning:** Validate inputs used for cache keys and guard against untrusted clients writing arbitrary cache entries.
- **Cost:** Memory costs can be significant. Balance hit-rate gains vs. added infrastructure cost. Use cost-effective tiers and eviction policies.

## CDN vs Application Cache

- **CDN:** Optimized for static assets and edge caching (images, JS, videos). Works at the HTTP layer and reduces latency by geo-proximity.
- **App cache (Redis/Memcached):** For dynamic application data and computation results. Provides programmatic access patterns and richer eviction/TTL control.

Use both where appropriate: CDNs for static delivery, application caches for dynamic data and computed results.

## Operational Best Practices

- Start by profiling: measure whether reads are the real bottleneck before adding complexity.
- Cache only data that is read frequently and expensive to compute or fetch.
- Use feature flags to enable/disable caching paths for testing.
- Automate cache rebalancing and node replacement.
- Add graceful degradation: design systems to tolerate cache failures (circuit breakers, reduced functionality).

## Common Interview Questions (Staff-Level)

These are examples you may be asked; for each, be ready to discuss trade-offs, alternatives, and operational details.

- **Design a caching layer for a user profile service with 100M users and 1M QPS reads.**
	- Talk about sharding (consistent hashing), replication for availability, hot key handling, TTL strategy, invalidation on writes, cache warming, and metrics to monitor.

- **How would you prevent a cache stampede when a popular key expires?**
	- Discuss mutex/lock per key, stale-while-revalidate, request coalescing, and probabilistic early refresh.

- **Compare write-through, write-behind, and cache-aside. When would you choose each?**
	- Explain latency vs durability trade-offs, operational complexity, and failure modes.

- **How do you handle cache invalidation for data that is updated by multiple services?**
	- Discuss centralized event streams for invalidation, versioned keys, idempotent invalidation messages, and eventual consistency implications.

- **How would you scale Redis for cross-region low-latency reads?**
	- Talk about read replicas in regions, geo-replication, TTLs tuned per region, and fallbacks to origin when replication lag is high.

- **What metrics would you include in a runbook for cache incidents?**
	- Hit/miss rate, eviction rate, cache latency, backend DB QPS/latency, error rates on invalidation messages, memory pressure, replication lag.

For staff-level interviews, go beyond definitions—sketch an architecture, show trade-offs, and describe operational runbooks and failure scenarios.

### Sample Answers / Talking Points

- **Design a caching layer for a user profile service with 100M users and 1M QPS reads.**
	- Start with read-heavy optimization: use Redis cluster with client-side sharding (consistent hashing) and regional read-replicas. Keep cold data in the DB, hot data in cache. Use TTLs for non-critical fields and synchronous invalidation for critical fields (e.g., privacy settings). Add a small in-process LRU for sub-ms reads. Plan for hot keys using replication or dedicated caches.

- **How would you prevent a cache stampede when a popular key expires?**
	- Use a distributed lock (Redis SETNX) for the first recompute, serve stale data with `stale-while-revalidate` for others, and implement probabilistic early refresh based on key access frequency.

- **Compare write-through, write-behind, and cache-aside.**
	- Write-through: strong consistency between cache and store, higher write latency.
	- Write-behind: high write throughput, risk of data loss on failure; requires careful batching and durability guarantees.
	- Cache-aside: simple, commonly used, places invalidation responsibility on application.

## Diagram: Common Flows

Cache-aside (lazy load):

Client -> App -> Check Cache
									|-- hit --> Return
									|-- miss --> Read DB -> Populate Cache -> Return

Write-through:

Client -> App -> Write Cache -> Cache writes synchronously to DB -> Return

Stale-While-Revalidate (serve stale while refreshing):

Client -> App -> Cache
						 |-- fresh --> Return
						 |-- stale --> Return stale; async refresh in background

## Pseudo-code Examples

Cache-aside (simple):

```pseudo
function get(key):
	value = cache.get(key)
	if value != null:
		return value
	value = db.read(key)
	cache.set(key, value, ttl)
	return value
```

Stale-While-Revalidate (with background refresh):

```pseudo
function get(key):
	entry = cache.getEntry(key) // {value, expiresAt}
	if entry == null:
		value = db.read(key)
		cache.set(key, {value, expiresAt=now+ttl})
		return value

	if entry.expiresAt > now:
		return entry.value

	// expired but still present: return stale and refresh
	spawn_background_task(refreshKey, key)
	return entry.value

function refreshKey(key):
	lock = redis.setnx("lock:"+key, 1, lockTTL)
	if not lock:
		return
	value = db.read(key)
	cache.set(key, {value, expiresAt=now+ttl})
	redis.del("lock:"+key)
```

Distributed lock (Redis SETNX pattern):

```pseudo
function getWithLock(key):
	value = cache.get(key)
	if value != null: return value

	if redis.setnx("lock:"+key, clientId, lockTTL):
		// we own the lock, recompute
		value = db.read(key)
		cache.set(key, value, ttl)
		redis.del("lock:"+key)
		return value
	else:
		// wait or return stale
		sleep(short)
		return cache.get(key) // maybe null
```

## Rollout Checklist for Production Cache

- **Measure first:** Confirm read hotspot via metrics and tracing.
- **Prototype / Local testing:** Validate cache client behavior and eviction policy locally.
- **Capacity plan:** Estimate working set size, pick instance sizes and cluster topology.
- **Authentication & network:** Ensure TLS and VPC/network rules for caches.
- **Monitoring:** Add hit/miss, eviction, latency, memory pressure, replication lag metrics.
- **Instrumentation:** Add tracing for cache-miss paths to find origin load.
- **Failover plan:** Define behavior if cache is unavailable (circuit breaker, degraded mode).
- **Backfill/warm:** Preload critical keys before switching traffic.
- **Gradual rollout:** Feature-flag cache usage and ramp traffic slowly.
- **Runbook:** Document steps to rollback, re-warm cache, and handle hotspots.
