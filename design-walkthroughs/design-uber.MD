## Uber — System Design Walkthrough

### Goal
Design a large-scale ride-hailing service (like Uber) that supports matching riders with drivers, real-time tracking, fare calculation, and high availability.

### High-level Requirements
- Functional
	- Rider can request a ride (pickup, destination).
	- Driver can accept/decline requests and update availability.
	- Find nearby drivers and match them to rider requests quickly.
	- Real-time location updates for driver and rider tracking.
	- Fare calculation, trip lifecycle (requested → accepted → en-route → completed → paid).
	- Basic surge pricing.
	- User and driver profiles, trip history, ratings.
- Non-functional
	- Low latency for matching (< 1s for initial shortlist).
	- High availability and fault tolerance.
	- Scalable to millions of users and drivers across regions.
	- Secure and consistent handling of payments and sensitive data.

### High-level Components
- Client apps (iOS/Android) for riders and drivers.
- API Gateway / Mobile backend.
- Matching Service (dispatch).
- Location Service (ingests location streams).
- Geospatial Index (to query nearby drivers).
- Trip Service (orchestrates state transitions).
- Pricing Service (fares, surge).
- Notifications/Push Service.
- Payments Service.
- Data stores: user DB, driver DB, trip DB, geospatial store, metrics and logs.
- Real-time stream platform (e.g., Kafka) for events and location updates.
- Monitoring & alerting.

### Core Flows

1. Driver heartbeats & location updates
	 - Driver app sends location updates periodically (e.g., every 2–5s) to a Location Ingest service.
	 - Location Ingest publishes to a streaming platform and updates the Geospatial Index (Redis / Geo-based DB).

2. Rider requests a ride
	 - Rider sends request with pickup and destination.
	 - Trip Service validates and persists the request, calls the Matching Service.

3. Matching
	 - Matching Service queries Geospatial Index for nearby available drivers within radius R and returns a shortlist (k drivers).
	 - Could use various strategies: nearest-first, batch request, multi-criteria (ratings, acceptance rate, ETA, dynamic pricing).
	 - Dispatch either:
		 - Sends requests to drivers sequentially (offers to driver 1 → wait T → driver 2), or
		 - Sends concurrent offers and chooses the first acceptor.
	 - Once a driver accepts, Trip Service updates state and notifies client apps.

4. Trip lifecycle
	 - Driver picks up rider, trip state transitions; Location Service streams updates for ETA calculation and UI.
	 - On completion, Trip Service calculates fare via Pricing Service and triggers Payments.

### Data Modeling (simplified)
- User: user_id, name, phone, payment_methods, rating, etc.
- Driver: driver_id, name, vehicle_info, status (available/busy), rating, current_location_id, last_active
- Trip: trip_id, rider_id, driver_id, pickup_location, dropoff_location, start_time, end_time, status, fare, route
- Location events: driver_id, timestamp, lat, lon
Indexes:
- Geospatial index on driver locations
- Secondary indexes for driver availability and region

### Matching Strategies & Algorithms
- Simple radius nearest-neighbor: Use geospatial index (Redis GEO or specialized quad-tree).
- Batched/Multicast offers: choose k nearest and include predicted acceptance probability.
- Skill-based routing: VIP riders get priority; drivers filtered by vehicle type.
- Multi-criteria cost function: combine ETA, acceptance rate, driver rating.

### Storage Choices
- User/Driver/Trip metadata: relational DB or document store with strong consistency for writes (Postgres or MySQL with shards).
- Geospatial store & hot location data: in-memory store like Redis (GEO commands) or a dedicated geospatial DB (GeoMesa, Elasticsearch).
- Event stream: Kafka for durability and decoupling (location updates, trip events).
- Long-term analytics: data lake (S3) with Parquet for batch processing.

### Scaling & Sharding
- Partition by geographic region (city, area) to reduce search space and locality of data.
- Use service instances per region for matching and location ingest.
- Shard user/trip data by user_id or region to parallelize writes.
- Autoscale matching service horizontally; maintain leader-election or consistent hashing for routing.

### Consistency & Concurrency
- Use optimistic locking or state-machine transitions in Trip Service to avoid double-booking.
- Centralize critical state transitions in Trip Service to act as source-of-truth.
- Use idempotent APIs and event-sourcing to replay events during failure recovery.

### Real-time Considerations
- Compress or throttle location updates from client to balance accuracy vs. cost (adaptive sampling).
- Edge aggregation: do basic filtering or aggregation in gateways before publishing to core systems.
- Use WebSockets or mobile push notifications for low-latency updates to clients.

### Fault Tolerance & Recovery
- Use retries with exponential backoff for transient failures.
- Design matching to be tolerant to missing drivers (fallback to expanded radius).
- Persistent event logs (Kafka) allow replay for state reconstruction.
- Circuit-breakers around external services (payments, notification).

### Security and Privacy
- Encrypt sensitive data at rest and in transit.
- Token-based auth for mobile clients (OAuth2/JWT) with refresh tokens.
- Rate-limit and detect anomalous behavior (fraud detection).
- Minimize PII in logs; redact or hash identifiers where possible.

### Monitoring & Metrics
- Track: request rates, match latency, acceptance rates, trip success/failure, driver availability, ETA accuracy.
- SLOs: 99.9% availability for matching API, median match latency < 1s in cities.
- Alerts on error-rate spikes, lagging Kafka consumers, low driver supply.

### Cost & Trade-offs
- In-memory geospatial store is faster but more expensive; persistent geospatial DB cheaper but higher latency.
- Precomputing match candidates reduces latency but might use stale data.
- Strong consistency increases complexity and reduces throughput; eventual consistency is acceptable for metrics/analytics but not for trip assignment.

### Additional Features (optional)
- Pooling/shared rides: needs route optimization and multi-stop matching.
- Estimated Time of Arrival (ETA) improvements with ML.
- Driver incentives & dynamic pricing models.
- Real-time surge visualization dashboards.

### Interview Tips
- Clarify requirements and constraints (scale, target markets).
- Start with a simple core design (rider requests → nearest driver match), then iterate with scaling and failure scenarios.
- Be explicit about data flows, bottlenecks, and where to add caches or queues.
- Discuss trade-offs and pick measurable SLOs.
