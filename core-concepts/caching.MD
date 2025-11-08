# Caching

Caching comes up in almost every system design interview, usually when you identify that your database is getting hammered with reads. The idea is simple. Store frequently accessed data in fast memory (like Redis) so you can skip the database entirely for most reads.
The performance difference is massive. A cache hit on Redis takes around 1ms compared to 20-50ms for a typical database query. When you're serving millions of requests, that 20-50x speedup matters. You also reduce load on your database, letting it handle more write traffic and avoiding the need to scale it prematurely.

External Caching
The pattern you'll use 90% of the time is cache-aside with Redis. On a read, check the cache first. If the data is there, return it. If not, query the database, store the result in the cache with a TTL, and return it. This is straightforward to implement and works for most read-heavy systems.

But caching introduces real complexity. The hardest part is invalidation. When a user updates their profile in the database, you need to delete or update the cached copy. Otherwise the next read returns stale data. There are a few strategies here. You can invalidate the cache entry immediately after writes, use short TTLs and accept some staleness, or combine both. The right choice depends on how fresh your data needs to be.

You also need to think about cache failures. If Redis goes down, every request suddenly hits your database. Can it handle that traffic spike? This is called a cache stampede and it can take down your whole system. Some approaches include keeping a small in-process cache as a fallback, using circuit breakers to prevent overwhelming the database, or accepting degraded performance until Redis comes back up.

> A common mistake is caching everything. Cache only data that's read frequently and doesn't change often. If you're caching data that changes on every request, you're just adding latency and complexity for no benefit. Profile your system first, then cache the hot paths.

CDN caching is different. It's for static assets like images, videos, and JavaScript files served from edge locations close to users. In-process caching works for small values that change rarely, like feature flags or config data. But for your core application data, external caching with Redis is the default.
