## Decision: Concurrency strategy

**Context:** We need to prevent double-booking under 500K concurrent users and 25K RPS peak without exhausting the database pool or blowing the $2,000/month AWS budget.

**Options considered:**
1. PostgreSQL `SELECT ... FOR UPDATE` - considered because it gives strict row-level correctness, but it blocks transactions and ties up a DB connection for the full booking/payment window. From [docs/CONCURRENCY.md](docs/CONCURRENCY.md), the modeled ceiling is about 2,840 RPS before the 500-connection pool is exhausted, which is far below peak.
2. Redis `SETNX` distributed lock - chosen because it makes the hot path non-blocking, keeps most contention out of PostgreSQL, and allows the app to fail fast when a seat is already locked.

**Why chosen:** Redis `SETNX` keeps the booking path within Redis-scale contention instead of PostgreSQL-scale contention. The concurrency analysis shows that lock attempts can stay in the tens of thousands of ops/sec on a small Redis cluster while the database only sees the much smaller set of winning bookings, keeping us inside the 500-connection limit and within budget.

**Tradeoffs accepted:** Redis becomes an availability dependency, lock management is more complex, and the database must still verify state before committing because Redis is an optimization rather than the source of truth.

**Revision trigger:** Revisit if peak lock traffic approaches Redis cluster capacity, if Redis reliability becomes unacceptable, or if the booking flow no longer needs to absorb high contention.

## Decision: Cache invalidation approach

**Context:** We need fast reads for event pages and availability while keeping stale data bounded when seat status or event state changes.

**Options considered:**
1. TTL-only invalidation - considered because it is simpler and requires less application plumbing, but it allows stale event or availability data to persist until expiry.
2. Event-driven invalidation - chosen because booking and status changes are explicit domain events, so the cache can be deleted immediately and repopulated on the next read.

**Why chosen:** Event-driven invalidation is the safer fit for seat inventory and booking history because correctness matters more than letting a stale cached value live out its full TTL. The cache doc shows that hot entries such as seat availability are only valid for short windows anyway, so deleting on change avoids showing seats as available after they were already booked.

**Tradeoffs accepted:** We must maintain invalidation code paths for booking, cancellation, and status updates, and a missed invalidation can still create temporary staleness until TTL expiry.

**Revision trigger:** Revisit if invalidation fan-out becomes too expensive, or if we introduce cache entries that are truly immutable and better served by TTL-only caching.

## Decision: Booking ID type

**Context:** Booking creation happens in a distributed system under high concurrency, and the identifier must stay unique without adding a centralized allocation bottleneck.

**Options considered:**
1. SERIAL / BIGSERIAL - considered because it is compact and familiar, but every insert must serialize sequence allocation, which creates contention under heavy concurrent writes.
2. UUID - chosen because each server can generate an ID independently, which avoids central sequence contention and works cleanly with future sharding or cross-service references.

**Why chosen:** UUID removes the sequence lock from the booking hot path. Given the scale target and the database budget, avoiding a centralized ID allocator is cheaper than scaling around it, even though UUID indexes are larger than integer keys.

**Tradeoffs accepted:** UUIDs take more storage and slightly slower index lookups than SERIAL, and they are less human-friendly when read aloud or debugged manually.

**Revision trigger:** Revisit if booking volume drops enough that sequence contention is no longer relevant, or if index/storage cost becomes materially higher than the contention cost.

## Decision: SQS visibility timeout value

**Context:** Payment processing is asynchronous, but a worker crash or transient gateway failure must not cause duplicate processing or premature redelivery.

**Options considered:**
1. Short timeout like 30s-60s - considered because it reduces the time a stuck message remains hidden, but it risks redelivery while a slow payment attempt is still in flight.
2. 300s visibility timeout - chosen because it gives the worker enough time to complete a payment attempt, DB updates, and notification publishing without premature retry, while still bounding retry delay.

**Why chosen:** 300 seconds is long relative to the typical payment round-trip in [docs/QUEUE.md](docs/QUEUE.md), which is about 800ms for the blocking gateway call in the rejected synchronous model, but still short enough that failed messages reappear quickly and can move to the DLQ after repeated attempts. This keeps reliability high without leaving failed work invisible for too long.

**Tradeoffs accepted:** Failed messages may stay hidden for several minutes before retry, and a worker that exceeds the timeout must be idempotent to avoid duplicate side effects if the message is retried.

**Revision trigger:** Revisit if payment latency distribution changes materially, if workers routinely need more than 5 minutes, or if DLQ volume shows the timeout is too aggressive or too lenient.
