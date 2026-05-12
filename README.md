# BookMyShow - High-Scale Booking System

## Step 2: Constraints Analysis

### Understanding the Three Hard Constraints

This section documents the analysis of three critical constraints that will drive all architectural decisions. This is the foundation for the entire system design.

---

## Constraint 1: 5 Lakh Concurrent Users at 12:00:00 Noon

### The Challenge

5 lakh = **500,000 concurrent users** arriving simultaneously when tickets go live.

### Peak Request Rate (RPS) Calculation

**Assumption: Average API requests per user during peak**

- Conservative estimate: 1-2 requests/user/60 seconds (users searching, browsing, checking availability)
- Peak spike: Some users making rapid requests (click to check, refresh, place booking)
- Spike scenario: At the exact moment tickets go live (12:00:00), many users generate write requests

**RPS Calculation:**

```
Baseline RPS = 500,000 concurrent users × (1-2 requests / 60 seconds)
            = 500,000 × 0.017 to 0.033
            = 8,500 to 16,500 RPS (sustained read-heavy browsing)

Peak RPS (ticket sale moment) = 500,000 × (spike: assume 30% of users make immediate requests)
                               = 150,000 simultaneous requests
                               = 2,500 RPS (if spike spreads over 60 seconds)
                               = 25,000 RPS (if concentrated in 6 seconds)

Worst case (all users click "buy now" in same second):
                               = 500,000 RPS (not sustainable, must be handled via queueing)
```

### Which Component Hits Its Limit First?

**At 25,000 RPS write requests for seat bookings:**

1. **Database Layer** ← BOTTLENECK #1
   - Each booking requires: SELECT with row-lock + UPDATE + INSERT (3 queries minimum)
   - Even with optimized indexes, a single RDS instance struggles
   - Single t3.xlarge RDS can handle ~5,000-7,000 OLTP transactions/sec max
   - **Verdict:** Database locks up first under write contention

2. **Application Layer** (API servers)
   - 25,000 RPS across web servers is manageable (scaling to 10-15 t3.xlarge instances)
   - Can handle at ~2,500 RPS per instance
   - **Verdict:** Can scale horizontally, not the bottleneck

3. **Network/Load Balancer**
   - ALB can handle 400K new connections/sec
   - **Verdict:** Not the bottleneck

### Design Implication

**The database lock contention during write-heavy operations is the critical bottleneck. Solutions:**

- Implement optimistic locking or row-level pessimistic locking efficiently
- Use database read replicas for searching/availability checks (write separation)
- Cache ticket availability aggressively to reduce database queries
- Queue booking requests to serialize writes and prevent lock storms

---

## Constraint 2: Zero Acceptable Double-Bookings

### Technical Definition

A **double-booking** occurs when two rows exist in the `booking_seats` table with:

- Same `seat_id`
- Same `showing_id` (same movie, same theater, same time)
- Both marked as confirmed/paid

This would mean overselling a seat - a critical data integrity violation.

### Root Cause of Double-Bookings

```
Race Condition Scenario:
Time T1: User A queries - Seat #42 is available (not locked)
Time T2: User B queries - Seat #42 is available (not locked)
Time T3: User A books - INSERT booking_seats (seat #42)
Time T4: User B books - INSERT booking_seats (seat #42)  ← Double-booking!
```

### Prevention Mechanisms

**Option 1: Pessimistic Locking (SQL Row Locks)**

```sql
BEGIN TRANSACTION;
  SELECT * FROM seats
    WHERE seat_id = 42 AND showing_id = 123
    FOR UPDATE;  -- ← Locks the row

  -- Check if already booked
  IF available THEN
    INSERT INTO booking_seats (...);
  END IF;
COMMIT;
```

- User A: Gets lock, books successfully
- User B: Waits for lock, then sees seat is booked, transaction aborts
- **Pros:** Simple, guaranteed consistency
- **Cons:** Lock contention = slow at 500K concurrent users, causes timeout storms

**Option 2: Optimistic Locking (Version Numbers)**

```sql
UPDATE booking_seats
  SET status = 'booked', booking_id = X, version = version + 1
  WHERE seat_id = 42
    AND showing_id = 123
    AND status = 'available'
    AND version = 5;  -- Version check

-- If affected rows = 0, seat was already booked or version changed
```

- Multiple transactions can attempt simultaneously
- Only one succeeds (atomic UPDATE condition fails for others)
- **Pros:** No locks, high concurrency
- **Cons:** Higher failure rate, requires retry logic

**Option 3: Hybrid Approach (Recommended for this scale)**

- Use **SELECT ... FOR UPDATE SKIP LOCKED** in PostgreSQL or MySQL 8.0+
- Skip locked rows and allow users to book available seats from remaining pool
- Reduces lock wait time from "infinite" to near-zero
- Only locks seats the user is actually trying to book

### Design Implication

**Pessimistic locking alone won't scale to 500K concurrent users.** Must implement:

- Optimistic locking with version numbers as primary mechanism
- Row-level pessimistic locking only for specific high-contention seats (surge pricing, limited inventory)
- Intelligent retry logic with exponential backoff for failed bookings
- Transaction isolation level: READ COMMITTED or SERIALIZABLE (database-specific)

---

## Constraint 3: $2,000/Month AWS Budget

### AWS Bill Breakdown (typical month)

Assuming this budget allocation:

| Component                 | Estimated Cost | Qty/Specs                       |
| ------------------------- | -------------- | ------------------------------- |
| EC2 Compute (API servers) | $600-800       | 4-6 t3.xlarge instances         |
| RDS Database (Primary)    | $800-1000      | 1 r6g.xlarge instance + storage |
| ElastiCache (Redis)       | $300-400       | 1-2 r6g.large clusters          |
| SQS (Queue)               | $50-100        | Booking queue, retry queue      |
| ALB (Load Balancer)       | $30-50         | 1 Application Load Balancer     |
| Data Transfer             | $100-200       | Inter-region, internet egress   |
| RDS Backups & Storage     | $50-100        | Automated backups, PIOPS        |
| **Total**                 | **~$2,000**    |                                 |

### What This Budget Actually Buys

**Adequate for:**

- 500K concurrent users with proper architectural choices
- Read replicas (if smaller instances than primary)
- Basic disaster recovery (automated snapshots)
- ~3TB database storage
- ~10-15 GB cache

**NOT included:**

- Second AZ active-active setup (would need $4,000+)
- Very large analytics warehouse
- Premium support
- Extensive monitoring/APM tools (DataDog, New Relic)

### Critical Budget Constraints

1. **Single RDS instance** (can't afford read replicas at this budget)
   - All writes go to primary
   - Read replicas NOT affordable, so cache everything instead

2. **Limited compute scaling**
   - Can't spin up unlimited EC2 instances
   - Must rely on efficient code + aggressive caching
   - If traffic exceeds 15K RPS, system becomes unstable

3. **No redundancy across regions**
   - Single region deployment only
   - No failover to another region
   - High availability via multiple AZs (within same region), but not across regions

### Impact Analysis: What if Budget Were $500/Month?

At $500/month, we'd need **fundamentally different architecture:**

| Current ($2,000)         | New ($500/month)                           |
| ------------------------ | ------------------------------------------ |
| EC2 t3.xlarge × 5        | Lambda functions only (pay-per-invocation) |
| RDS r6g.xlarge           | RDS t4g.small or Aurora Serverless         |
| ElastiCache cluster      | DynamoDB DAX or no cache                   |
| Single region, 3 AZs     | Single AZ only                             |
| Native PostgreSQL        | Serverless SQL (Aurora Serverless)         |
| SQS for queuing          | DynamoDB Streams or SNS                    |
| Sustained for 500K users | Sustained for ~50K concurrent users max    |

**Key changes:**

- Shift from servers to functions (Lambda) = lower baseline cost
- Use managed databases over provisioned instances
- Accept higher latency in exchange for lower cost
- Implement tier-based quality of service (premium users get faster booking)
- May need to implement booking lottery instead of first-come-first-served

### Design Implication

**The $2,000 budget forces:**

- Every architectural choice must maximize resource utilization
- Aggressive caching (Redis) to reduce database load
- Efficient indexing and query optimization
- Batch operations where possible (reduce round-trips)
- No room for experimental features or redundancy
- Must be 99.5% efficient, not 90% efficient

---

## How the Three Constraints Interact

### Constraint 1 + Constraint 2 → The Core Tension

- **500K concurrent users** generate massive write traffic (25K+ RPS peak)
- **Zero double-bookings** requires atomic seat reservations (database serialization)
- **Traditional locking breaks under this load** (lock wait queues, timeouts)
- **Solution:** Optimistic locking + intelligent queueing

### Constraint 2 + Constraint 3 → Resource Efficiency

- **Zero double-bookings** requires transaction safety (costs CPU, memory, disk I/O)
- **$2,000 budget** means we can't throw more hardware at the problem
- **Must optimize at code level:** efficient queries, connection pooling, caching strategy
- **Can't afford traditional HA with read replicas** → must use cache-aside pattern

### Constraint 1 + Constraint 3 → The Scalability vs. Cost Tradeoff

- **500K concurrent users** would normally cost $5,000-10,000/month for redundancy
- **$2,000 budget** requires accepting single-point failures in exchange for cost
- **Solution:** Stateless application layer + managed databases
- **Risk:** If primary RDS fails, entire system goes down (no read replica to failover to)

---

## Summary

| Constraint               | Hard Limit                   | Most Constrained Component  |
| ------------------------ | ---------------------------- | --------------------------- |
| **5L concurrent @ noon** | ~25K booking RPS peak        | Database (lock contention)  |
| **Zero double-bookings** | Atomic transactions required | Row-level locking mechanism |
| **$2,000 budget**        | Single RDS instance only     | Infrastructure provisioning |

**Next Steps:** These constraints will determine:

1. Database schema design (optimistic locking fields)
2. Caching strategy (cache-aside for availability)
3. API rate limiting (prevent overwhelming database)
4. Queue depth limits (manage spike smoothing)
5. Connection pooling (maximize DB throughput)
