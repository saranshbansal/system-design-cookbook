# API Design — extended interview checklist & principles ✅

### When to choose REST vs GraphQL vs gRPC
- **REST**: Default for public/external APIs; simple, cache-friendly, and ubiquitous. Sketch resource URLs and standard methods quickly in interviews.
- **GraphQL**: Choose when many client shapes need flexible queries and you can tolerate backend complexity (query planning, N+1 issues). Mention batching/dataloader patterns to avoid N+1 problems.
- **gRPC**: Use for internal, low-latency service-to-service calls where binary RPCs and strong typing (protobuf) matter. Note limited browser support.

### Design primitives & best practices
- **Idempotency**: Ensure non-idempotent endpoints (POST) accept an idempotency key to prevent double-processing on retries (Idempotency-Key header).
- **Pagination & cursors**: Prefer cursor-based for feeds; offset is OK for simple pagination and small datasets. Return next/prev cursors and total only when needed.
- **Versioning strategies**: Use header or URL versioning (e.g., `Accept: application/vnd.example.v2+json` or `/v2/resource`) and have a deprecation plan (timelines, compatibility notes).
- **Caching**: Use HTTP caching headers (Cache-Control, ETag) and CDNs for public resources. For private APIs, consider response caching with cache keys based on auth scope.
- **Rate limiting & throttling**: Use token-bucket/leaky-bucket algorithms; scope limits by API key, user, IP, or tenant. Provide `Retry-After` and clear error codes when throttled.
- **Backoff & retries**: Clients should use exponential backoff with jitter. Make retries safe with idempotency keys or design endpoints to be idempotent.
- **Security & auth**: OAuth 2.0 / OpenID Connect for user auth, JWT for stateless sessions, mTLS for private services, and API keys for server-to-server. Always validate input and avoid sensitive data in URLs.
- **Error handling**: Use proper status codes (4xx for client, 5xx for server), include structured error payloads with an error code, message, and actionable fields.
- **Documentation & contract-first**: Prefer OpenAPI (Swagger) and contract-testing (Pact/consumer-driven tests) for backward compatibility and client expectations.

### Operational concerns
- **API Gateway**: Centralize auth, rate limiting, logging, and routing at the gateway. Use service mesh for internal mTLS and observability rather than public-facing routing.
- **Monitoring & SLOs**: Track request latency P95/P99, error rates, throttling events, and capacity metrics. Alert on saturation and elevated 5xx rates.
- **Testing**: Unit + integration tests, contract tests (consumer-driven), and end-to-end scenarios. Load test critical endpoints to understand bottlenecks.

---

## Quick endpoint sketch — what to draw in an interview (example: booking service)
- GET /events/{id} → get event details
- POST /events/{id}/bookings  (Idempotency-Key: <uuid>) → create booking
- GET /users/{id}/bookings?cursor=... → user bookings (cursor-based pagination)
- DELETE /bookings/{id} → cancel booking (idempotent behavior)

Include error codes and a note about optimistic locking or version checks for concurrency-sensitive operations.

---

## Senior Q&A — interview-style (principled, short answers) 🎯

1. **Q: How do you evolve a public API without breaking clients?**
   A: Use additive changes (new fields, new endpoints), support multiple versions concurrently with a deprecation policy, provide feature flags, and use contract tests to detect breaking changes early.

2. **Q: How do you design for eventual consistency when operations are asynchronous?**
   A: Surface eventual consistency to clients (202 Accepted + location for status), use webhooks or long-polling for completion notifications, and provide read-after-write consistency via read-your-writes patterns on request or a short-lived cache.

3. **Q: Public API vs internal API — what differences matter most?**
   A: Public APIs require strong backwards-compatibility guarantees, richer documentation, stability SLAs, and stricter security and rate limiting. Internal APIs can evolve faster, favor productivity (fewer backward-compat constraints) and may use RPC/gRPC.

4. **Q: How to choose rate limits and enforce fair usage across tenants?**
   A: Start with reasonable defaults per user/API key, monitor usage patterns, apply tiered limits by plan/tenant, provide burst allowances, and fail-fast with clear `429` responses and `Retry-After` headers. Consider global vs per-tenant throttling depending on fairness requirements.

5. **Q: A client complains of intermittent 500s after a new deploy — how do you investigate?**
   A: Check release changelog and feature flags, examine logs/traces for correlation with the deploy time, look at error budgets/SLOs, roll back or disable risky changes if correlated, and add golden metrics and more granular tracing.

6. **Q: How do you guarantee idempotency for complex, multi-step operations?**
   A: Use idempotency keys with deduplication storage tied to the key and operation, persist operation states (pending/complete/failed), design compensating actions for partial failures, and make retries safe by reconciling state server-side.

7. **Q: When is GraphQL a bad idea?**
   A: When backend complexity, N+1 risks, and authorization-per-field costs outweigh client flexibility—especially for simple CRUD APIs or when caching and CDN benefits of REST are important.

8. **Q: API versioning — path vs header vs content negotiation — which do you prefer?**
   A: Prefer header or content negotiation for minor/semantic versioning and path versioning for major, clearly breaking versions. Important: document the deprecation policy and keep client migration simple.

9. **Q: How do you design an idempotent webhook consumer?**
   A: Use idempotency tokens provided by the sender, keep deduplication stores with TTL, acknowledge delivery quickly, and design processing to be re-entrant with idempotent side-effects.

---

### TL;DR 💡
In interviews: sketch 4–5 endpoints, state your choices (REST/GraphQL/gRPC), call out pagination/auth/versioning/caching, and walk through trade-offs. For senior roles, emphasize operational practices: backward compatibility, idempotency, monitoring, and clear deprecation/versioning policies.
