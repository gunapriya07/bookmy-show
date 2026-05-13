# System Architecture — Full Diagram (ASCII)

Note: every component and connection is labelled with the data that flows across it.

Client
|
| (HTTPS requests; browser fetches)
v
CloudFront CDN - Serves: static assets (JS/CSS/images), cached event pages - Passes through: API calls to ALB when cache-miss - Cache hit path: serve edge-cached page -> Client - Cache miss path: forward request -> ALB (label: cache-miss)
<- CDN caching strategy (see [docs/CACHE.md](docs/CACHE.md))
|
v
Application Load Balancer (ALB) - SSL termination - Health check interval: 30s - Rate limit rule: 100 rps per IP (edge rate-limit) - Routes: /api/\* -> Node.js API AutoScale Group
<- ALB behavior and security (see [docs/DESIGN-DECISIONS.md](docs/DESIGN-DECISIONS.md))
|
v
+---------------------------+
| Node.js API servers |
| (Auto-Scaling Group) |
| - Handles user API, seat |
| selection, order create |
+---------------------------+
/ | \
 / | \
 / | \
 v v v
Redis Cluster PostgreSQL Primary SQS Payment Queue
(clustered) (writes only) (payment FIFO/standard)
/ \ ^ ^ ^ |
/ \ | | | | message format:
v v | | | | {user_id,order_id,amount,seat_ids,payment_method}
(a) Cache keys (b) Seat | | | | visibility-timeout: 300s
with TTL (TTL) locks | | | | DLQ path: PaymentDLQ ← after maxReceiveCount - read for page SETNX | | | <- Async queue design (see [docs/QUEUE.md](docs/QUEUE.md))
rendering & lookups lock TTL | | |  
 <- Cache design (see [docs/CACHE.md](docs/CACHE.md))
| | |
| | +--> PostgreSQL Read Replica #1 (route SELECT/reads)
| | <- Read replica (see [docs/SCHEMA.md](docs/SCHEMA.md): reads separated from writes)
| +----> PostgreSQL Read Replica #2 (route SELECT/reads)
| <- Read replica (see [docs/SCHEMA.md](docs/SCHEMA.md))
|
+------ writes (INSERT/UPDATE/DELETE) --> PostgreSQL Primary
<- Writes routed to primary (see [docs/SCHEMA.md](docs/SCHEMA.md))

Node.js API servers (detailed labels): - Reads from Redis: cached page fragments, session tokens, seat availability cache - Uses Redis SETNX for seat locks (lock TTL): to reserve seats during checkout - Writes to DB: create orders, payment status updates (writes -> Primary) - Publishes to SQS Payment Queue: on order created -> Payment processing async
<- Node scaling and concurrency (see [docs/CONCURRENCY.md](docs/CONCURRENCY.md))

SQS Payment Queue (detailed): - Message format (summary): {user_id, order_id, amount_cents, seat_ids[], payment_method, created_at} - Visibility timeout: 300s (5 minutes) - DLQ: PaymentDLQ after maxReceiveCount - Consumers: Payment Worker (ECS)
<- Async queue justification (see [docs/QUEUE.md](docs/QUEUE.md))

Payment Worker (ECS)
Steps (labelled): 1) Read SQS message (receiveMessage) 2) Call payment gateway (external HTTP/S) — await confirmation/decline 3) Update DB: write payment result -> PostgreSQL Primary 4) Publish SNS topic: PaymentConfirmation (payload: {order_id,status,timestamp}) 5) Delete SQS message (on success) - Visibility timeout and retry semantics ensure at-least-once processing
<- Worker flow and retry strategy (see [docs/QUEUE.md](docs/QUEUE.md))

SNS Topic: PaymentConfirmation - Subscribers: SES Email service (transactional emails), SMS gateway - Trigger: Payment Worker publishes on confirmed payment
<- Notification design (see [docs/DESIGN-UPDATES.md](docs/DESIGN-UPDATES.md))

SES Email + SMS Gateway - Sends: payment confirmations, receipts, alerts - Triggered by: SNS -> subscription filter on PaymentConfirmation
<- Email/SMS rationale (see [docs/QUEUE.md](docs/QUEUE.md) and [docs/DESIGN-DECISIONS.md](docs/DESIGN-DECISIONS.md))

PostgreSQL Primary - Accepts: all writes (orders, payments, seat reservation finalization) - Replication: async -> Read Replicas x2
<- Write routing and schema decisions (see [docs/SCHEMA.md](docs/SCHEMA.md))

PostgreSQL Read Replicas (x2) - Accepts: read-only queries (seat availability checks, reporting, list events) - Route: Node.js API servers send SELECT queries here where possible
<- Read replica use (see [docs/SCHEMA.md](docs/SCHEMA.md))

Dead-letter and error paths - SQS Payment Queue -> PaymentDLQ after maxReceiveCount - Failed payment processing alerts -> SNS/ops topic for manual investigation
<- DLQ strategy (see [docs/QUEUE.md](docs/QUEUE.md))

Cache hit vs cache miss summary - CloudFront cache HIT: CloudFront -> Client (serves cached event page or static asset) - CloudFront cache MISS: CloudFront -> ALB -> Node.js -> (may read Redis/DB) -> response -> CloudFront caches
<- Edge caching choices (see [docs/CACHE.md](docs/CACHE.md))

Concurrency / seat-lock summary - Seat reservation uses Redis SETNX lock with TTL to avoid double-booking - If lock acquired: proceed to create order & publish to SQS - If lock not acquired: return conflict to client
<- Seat-lock rationale (see [docs/CONCURRENCY.md](docs/CONCURRENCY.md))

Notes / Callouts - All network arrows represent JSON over HTTPS unless otherwise noted. - Monitoring & health checks: ALB health checks -> Node.js /health endpoint (interval 30s) - Autoscaling: Node.js instances scale by CPU and request latency thresholds (see [docs/CONCURRENCY.md](docs/CONCURRENCY.md)).
