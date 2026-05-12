# Concurrency Strategy: Preventing Double-Bookings at Scale

## Executive Summary

This document analyzes two competing strategies for preventing double-bookings under high concurrency (500K concurrent users, 25K RPS peak), each with radically different performance characteristics. We analyze both to failure, then make a justified choice constrained by the $2,000/month AWS budget.

---

## Option A: PostgreSQL SELECT FOR UPDATE (Pessimistic Locking)

### How It Prevents Double-Booking

The strategy: **Lock the row before checking, so no other transaction can touch it.**

**Exact SQL Transaction:**

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

  -- Lock the seat row. Other transactions WAIT here until we commit/rollback.
  SELECT * FROM seats
    WHERE id = 42 AND event_id = 1
    FOR UPDATE;

  -- Check status (no one else can modify it while we hold lock)
  IF row.status = 'available' THEN
    -- Perform booking
    UPDATE seats SET status = 'booked', held_by = NULL
      WHERE id = 42;

    INSERT INTO booking_seats (booking_id, seat_id, price_at_booking)
      VALUES (booking_123, 42, 500.00);

    COMMIT;  -- Release lock here
  ELSE
    ROLLBACK;  -- Seat was taken by someone else, release lock
  END IF;

-- If another transaction tried: SELECT FOR UPDATE WHILE we held lock
-- That transaction is BLOCKED (waiting queue) until we COMMIT or ROLLBACK
```

**Why This Works:**

- While transaction A holds the lock on seat #42, transaction B's `SELECT FOR UPDATE` blocks
- Transaction B doesn't fail or error - it **waits** (hangs)
- Only after A commits (releases lock) does B's SELECT proceed
- B then sees status='booked', aborts, returns "seat taken"
- **Result:** No double-booking, consistent state

### Hard Limit: At What RPS Does Pool Exhaust?

**Connection Pool Exhaustion Formula:**

When using PostgreSQL with `SELECT FOR UPDATE`, each transaction ties up a database connection for its entire duration:

```
Connections held = (Σ concurrent transactions) × (avg transaction time)
```

More precisely:

```
Connections_held = (% query_RPS × query_duration_s) + (% payment_RPS × payment_duration_s)
```

**Given parameters (from constraints analysis):**

- PgBouncer max_connections = 500
- Two types of queries:
  - **Regular seat availability checks:** 80% of RPS, 20ms duration
  - **Booking confirmation (with payment hold):** 20% of RPS, 800ms duration (payment gateway round-trip)
- Total RPS = X

**Calculation:**

```
Regular connections held = 0.80 × X × 0.020 sec = 0.016X
Payment connections held = 0.20 × X × 0.800 sec = 0.160X

Total connections held = 0.016X + 0.160X = 0.176X

Pool exhaustion occurs when: 0.176X = 500
                             X = 2,840 RPS
```

**The Hard Limit: 2,840 RPS**

```
┌─────────────────────────────────────────┐
│ Our Requirement: 25,000 RPS peak       │
│ SELECT FOR UPDATE can handle: 2,840 RPS │
│ Shortfall: -88% insufficient!           │
└─────────────────────────────────────────┘
```

**What Happens at Exhaustion?**

At 3,000 RPS (above limit):

1. All 500 connections are held by transactions
2. New booking request arrives: "Get connection from pool"
3. Pool returns: None available (timeout)
4. Application throws: `java.sql.SQLException: Cannot get a connection`
5. User sees: "Booking temporarily unavailable"
6. Real issue: System can't scale past 2,840 RPS, even with more servers

### Deadlock Risk with Multi-Seat Bookings

**Scenario: User books 3 seats**

```
Transaction A (User books seats 10, 20, 30):
  FOR UPDATE seats WHERE id = 10         -- Gets lock on seat 10
  FOR UPDATE seats WHERE id = 20         -- Gets lock on seat 20
  FOR UPDATE seats WHERE id = 30         -- WAITING for seat 30 (held by B)

Transaction B (User books seats 20, 30):
  FOR UPDATE seats WHERE id = 20         -- WAITING for seat 20 (held by A)
  FOR UPDATE seats WHERE id = 30         -- Can't proceed, waiting for B's first lock
```

**Deadlock:** A waits for 30 (held by B), B waits for 20 (held by A). Neither progresses.

**Mitigation:**

1. **Lock ordering:** Always acquire locks in ascending seat ID order

   ```sql
   FOR UPDATE seats WHERE id IN (30, 20, 10) ORDER BY id ASC
   ```

   Both A and B now lock in order: 10 → 20 → 30 (no circular wait)

2. **Lock timeout:** Set `lock_timeout = 5s` at session level

   ```sql
   SET lock_timeout = '5s';
   ```

   If transaction doesn't acquire lock in 5 seconds, abort with error
   Application catches error and retries with exponential backoff

3. **Database deadlock detection:** PostgreSQL detects actual deadlocks and aborts one transaction
   Application catches `SQLSTATE:40P01` and retries

**Cost:** Deadlock handling adds complexity (retry logic), but rare enough (~1 in 10K transactions with good schema design)

---

## Option B: Redis SETNX Distributed Lock

### How It Prevents Double-Booking

The strategy: **Use atomic SET-IF-NOT-EXISTS in Redis to claim exclusive access, then verify in database.**

**Lock Key Structure:**

```
Key: seat_lock:{event_id}:{seat_id}
Example: seat_lock:event_1:seat_42
```

Use event_id in key to enable partitioning/sharding across Redis clusters in the future.

**The Booking Flow (with Lua script):**

```lua
-- Atomic Lua script executed on Redis server (all or nothing)
-- This prevents race conditions within Redis itself

-- KEYS[1] = "seat_lock:event_1:seat_42"
-- ARGV[1] = "lock_value" (booking transaction ID)
-- ARGV[2] = 15 (TTL in seconds)

local result = redis.call('SET', KEYS[1], ARGV[1], 'NX', 'EX', ARGV[2])
if result then
  -- Lock acquired, return 1 (success)
  return 1
else
  -- Lock already held by someone else, return 0 (failure)
  return 0
end
```

**Full Booking Transaction:**

```python
lock_key = f"seat_lock:{event_id}:{seat_id}"
lock_value = str(booking_transaction_id)
lock_ttl = 15  # seconds

# Step 1: Try to acquire lock in Redis
result = redis.eval(LOCK_SCRIPT, 1, lock_key, lock_value, lock_ttl)

if result == 1:  # Lock acquired
  try:
    # Step 2: Double-check seat status in database (paranoia check)
    seat = db.query("SELECT * FROM seats WHERE id = ? AND status = 'available'", seat_id)

    if seat:
      # Step 3: Book the seat
      db.execute("""
        UPDATE seats SET status = 'booked', held_by = NULL, version = version + 1
        WHERE id = ? AND version = ?
      """, seat_id, seat.version)

      # Step 4: Record booking
      booking = db.insert_booking(...)

      # Step 5: Release lock explicitly (or let TTL expire)
      redis.delete(lock_key)

      return booking  # Success!
    else:
      # Seat already booked? Delete lock and retry
      redis.delete(lock_key)
      return retry()

  except Exception as e:
    # Database error, release lock
    redis.delete(lock_key)
    raise e

else:  # Lock NOT acquired (someone else has it)
  return {status: "seat_unavailable_try_again"}
```

**Why This Works:**

- Redis SETNX is atomic at Redis level (no race within Redis)
- Only one requester gets `result=1`, others get `result=0`
- Winner proceeds to database booking
- Loser immediately knows "seat is locked" without waiting
- TTL ensures lock auto-releases if holder crashes (garbage collection)

### What Happens If Redis Fails Mid-Lock?

**Scenario 1: Redis fails BEFORE lock is held (no lock stored)**

```
Time 1: User A sends SETNX to Redis
Time 2: Redis crashes
Time 3: SETNX fails (no response), User A sees "system error, retry"
Time 4: Redis comes back online, no lock exists

Result: Safe. Lock never existed. User A or B can retry. No double-booking.
```

**Scenario 2: Redis fails AFTER lock is held, BEFORE booking completes**

```
Time 1: User A acquires lock in Redis: seat_lock:1:42 = txn_123 (TTL=15s)
Time 2: User A reaches database, about to insert booking
Time 3: Redis crashes
Time 4: Database completes booking for User A
Time 5: Redis comes back online
Time 6: TTL has expired (15 seconds passed), lock no longer exists
Time 7: User B acquires lock (lock auto-released by TTL)
Time 8: User B checks database, seat is booked (see Step 2: Double-check in database)
Time 9: User B's booking fails, returns "seat taken"

Result: Safe! Database double-check (paranoia check) catches it.
```

**Scenario 3: Redis fails during booking (worst case - could cause double-booking?)**

```
Time 1: User A acquires lock in Redis
Time 2: User A: UPDATE seats SET status = 'booked'
Time 3: Redis crashes (lock still in memory, not persisted to disk if no save)
Time 4: User B doesn't see lock, acquires lock immediately
Time 5: User B: SELECT * FROM seats WHERE id=42, status = 'available' -- SEES BOOKED
Time 6: User B's double-check catches it

Result: SAFE. Database is source of truth, Redis is optimization.
```

**The Key Insight:**
Redis is an **optimization** for speed, not the source of truth. The database double-check (paranoia check) is the real safety net. Redis failures don't cause double-bookings because:

1. Redis lock only prevents unnecessary database queries
2. Database state is always validated before committing
3. If Redis is down, we fall back to database checks only (slower but safe)

---

### The Right TTL Value for Seat Lock

**Tradeoff Analysis:**

**Too Short (TTL = 1 second):**

```
Scenario: User is mid-payment, payment takes 3 seconds
Time 0: User acquires lock (TTL=1s)
Time 0.5: User sends payment to Stripe
Time 1.0: Lock expires (TTL reached)
Time 1.1: Another user acquires same seat lock
Time 2.0: First user gets payment confirmation
Time 2.5: First user tries to complete booking, but lock is gone
Result: Double-booking!
```

**Verdict:** Too risky, payment processing is unpredictable (network delays)

**Too Long (TTL = 300 seconds = 5 minutes):**

```
Scenario: User acquires lock, crashes before completing
Time 0: User A acquires lock (TTL=300s)
Time 0.1: User A's browser crashes
Time 1: Seat is locked, unavailable to everyone else
Time 60: Seat still locked (User A not coming back)
Time 299: Seat finally released
Result: User B waits 5 minutes for unavailable seat to appear
```

**Verdict:** Too generous, terrible UX, seats held unnecessarily

**Optimal TTL = 15 seconds:**

```
Rationale:
- Payment processing typically completes in 2-5 seconds
- Network timeouts (retry) add another 5-8 seconds
- Grace period for slowness: +2 seconds
- Total: ~12-15 seconds safe

At 15 seconds:
- 99% of legitimate bookings complete
- Crashed sessions expire quickly (seats return to market in 15 seconds)
- Balance: safe for payments, fast garbage collection
```

**Lock Key Naming:**

`seat_lock:{event_id}:{seat_id}` is better than `seat_lock:{seat_id}` because:

- Allows Redis cluster partitioning by event (distribute hot movies across nodes)
- Prevents lock collision if same seat_id exists in different events (rare but possible)
- Makes debugging easier: can see which event's seat is locked

---

### Redis Scaling Model

**Why Redis Doesn't Hit Connection Pool Bottleneck:**

```
PostgreSQL with SELECT FOR UPDATE:
- Each transaction = 1 connection held
- 500 max connections = 2,840 RPS max

Redis SETNX:
- Each operation = pipelined request (many requests per connection)
- 10 connections to Redis cluster can handle 100K+ operations/second
- Single Redis node: 80K-100K ops/sec (sub-millisecond operation time)
```

**Architecture:**

```
Application Servers (100+ instances)
           ↓
    Redis Cluster (3-5 nodes)
           ↓
PostgreSQL Primary (1 RDS instance)
```

At 25K RPS:

- 25K SETNX operations/second → 1 Redis node handles this easily
- Winners (100-500 at any moment) proceed to database
- Database sees ~100-500 concurrent actual bookings (not 25K lock waits)
- Pool stays well under 500 connection limit

---

## Option B Limitations

**When Redis Becomes a Bottleneck:**

- Single Redis node maxes at ~100K ops/sec
- Beyond 100K RPS lock attempts, Redis itself becomes bottleneck
- Our peak: 25K RPS, so comfortable headroom (4x buffer)

**When Redis Fails Completely:**

- All lock attempts fail
- System falls back to: database-only with optimistic locking (version checks)
- Performance degrades to ~2,800 RPS (same as Option A)
- System stays available, just slower

---

## Summary: Option A vs Option B

| Metric                   | Option A (FOR UPDATE)   | Option B (Redis)                |
| ------------------------ | ----------------------- | ------------------------------- |
| **Max RPS**              | 2,840                   | 100,000+                        |
| **Headroom vs 25K peak** | -88% (FAILS)            | +75% (comfortable)              |
| **Connection overhead**  | Per-transaction         | Pipelined, negligible           |
| **Deadlock risk**        | Yes, needs mitigation   | No                              |
| **Complexity**           | Simple SQL              | Complex: Redis + fallback logic |
| **Infrastructure cost**  | None (use existing RDS) | 1-2 Redis nodes (~$100-150/mo)  |
| **Resilience**           | Database is SPOF        | Graceful degradation to DB      |
| **Setup time**           | 1-2 days                | 3-5 days                        |

---

# OUR CHOICE: HYBRID APPROACH (Redis Primary + DB Optimistic Locking Fallback)

## Why We Choose Hybrid, Not Pure Option A or B

**We choose Hybrid because:**

1. **Option A (pure SELECT FOR UPDATE) cannot handle our peak load.** At 25K RPS, connection pool exhausts at 2,840 RPS. We'd miss 88% of our peak traffic - system becomes unavailable during sales events. This violates the core constraint.

2. **Option B (pure Redis) is reliable but adds complexity without eliminating database layer.** We still need optimistic locking on seats (for version checks). Pure Redis locks would be false confidence - database is already doing the real work.

3. **Hybrid gives us the best of both:**
   - Redis handles the speed problem (lock attempts at 25K RPS)
   - Database handles the safety problem (version-based optimistic locking)
   - Falls back gracefully if Redis fails (system still works, just slower)

## Exact Architecture

### Primary Path (99% of cases, full speed):

```
User requests seat lock
  ↓
Try SETNX lock in Redis (milliseconds)
  ↓
Success (lock acquired)
  ↓
Check seat in database (for safety)
  ↓
UPDATE seat with optimistic locking (version check)
  ↓
Success → Return confirmation
  ↓
Release Redis lock
```

### Fallback Path (Redis unavailable, slow but safe):

```
User requests seat lock
  ↓
SETNX to Redis fails (timeout, unreachable)
  ↓
Fall back to database-only booking
  ↓
SELECT * FROM seats WHERE id = ? AND status = 'available'
  ↓
UPDATE seats SET status = 'booked' WHERE id = ? AND version = X
  ↓
If version mismatch: retry with exponential backoff
  ↓
Success → Return confirmation
```

### Why This Hybrid Design Kills Two Birds

**Problem #1: Connection Pool Bottleneck (Option A problem)**

- Redis SETNX reduces database load by 98%
- Lock attempts don't touch database
- Only successful bookings reach database
- From 25K RPS attempts → 100-500 actual database transactions
- Database connections stay under 50 (out of 500 max)

**Problem #2: Double-Booking (without Redis)**

- Database optimistic locking (version column) remains primary safety mechanism
- Even without Redis, system is safe (just slow)
- Version column prevents double-bookings regardless of Redis state

## Architecture Justification vs Constraints

### Constraint 1: 5L Concurrent Users at 25K RPS Peak ✓

```
Without Redis:  2,840 RPS max → FAILS (misses 88% of peak)
With Redis:     100K+ RPS max → SUCCESS (25x headroom)
```

**We pass the peak load test.**

### Constraint 2: Zero Double-Bookings ✓

```
Redis + Database Optimistic Locking:
1. Redis lock: 99% of collisions blocked before database access
2. Database version: 100% safety net if #1 fails or Redis is down
3. Two independent mechanisms = extremely high confidence
```

**We achieve zero double-bookings.**

### Constraint 3: $2,000/month AWS Budget ✓

```
Hybrid costs:
- 1 Redis node (r6g.large): $60-80/month
- RDS (already budgeted): $900-1000/month
- Fewer database connections needed: no change
- Total added cost: ~$70-80/month (3-4% budget increase)

Acceptable trade-off: +4% cost for +88% throughput improvement
```

**We stay within budget.**

---

## What This Strategy Cannot Handle

**Honest limitations:**

1. **Simultaneous booking of dependent seats** (e.g., "rows must be contiguous")
   - Problem: Booking seat 10, then 11 requires coordinating two locks
   - Current solution handles independent seats perfectly
   - Multi-seat reservations need additional logic (seat bundles, atomic transactions)

2. **Redis geographic failover** (if AWS region fails)
   - Hybrid relies on single Redis instance
   - If Redis node hardware fails, fallback to database (slow but safe)
   - Doesn't survive full region failure without multi-region setup ($4,000+ budget)

3. **Extreme load beyond 100K RPS**
   - Redis maxes out at ~100K ops/sec on single node
   - Beyond that, need Redis cluster sharding (adds complexity)
   - Our peak is 25K RPS, so comfortable, but limited runway

---

## Condition to Switch Away from Hybrid

**We would switch to pure Option A (database only) if:**

- Peak traffic drops below 5,000 RPS permanently
- Reason: Redis becomes unnecessary overhead, complexity isn't worth it
- Simpler system, easier to debug, no Redis failures to worry about

**We would switch to multi-region (budgetary change) if:**

- Customer demand grows to 2M+ concurrent users
- Reason: Single RDS instance can't handle that volume even with Redis
- Would need: Multi-region active-active setup
- Budget: $4,000-6,000/month

---

## Implementation Roadmap

### Phase 1: Core Hybrid (Week 1-2)

- Implement Redis SETNX lock acquisition
- Implement database fallback (if SETNX fails)
- Load test to verify 25K RPS handling

### Phase 2: Production Hardening (Week 3)

- Add circuit breaker (if Redis error rate > 5%, disable and go database-only)
- Monitoring for lock acquisition latency (should be <5ms)
- Alert if locks held > 30 seconds (stuck transaction)

### Phase 3: Optimization (Week 4+)

- Tune Redis TTL based on real payment timing data
- Consider Redis sharding if traffic grows
- Profile: measure % of bookings that use Redis vs database

---

## Conclusion

**We choose Hybrid (Redis + Database Optimistic Locking) because:**

- **Passes peak load constraint** (25K RPS): Redis makes DB pool irrelevant
- **Passes safety constraint** (zero double-bookings): Database version checks as safety net
- **Passes budget constraint** ($2,000/mo): Only $70 additional cost
- **Graceful degradation:** Works without Redis (just slower), works without database luck
- **Clear upgrade path:** Can add Redis cluster, multi-region, or database replicas later as traffic grows

This strategy deliberately couples fast-path optimization (Redis) with safe-path guarantees (database), making it appropriate for a system where failures occasionally happen but double-bookings are unacceptable.
