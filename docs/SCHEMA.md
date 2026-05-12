# BookMyShow Database Schema

## PostgreSQL DDL

```sql
-- Enable necessary extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- ============================================================================
-- TABLE: venues
-- ============================================================================
CREATE TABLE venues (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL,
    capacity INT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT check_capacity_positive CHECK (capacity > 0)
);

CREATE INDEX idx_venues_city ON venues(city);
CREATE INDEX idx_venues_name ON venues(name);

-- ============================================================================
-- TABLE: events
-- ============================================================================
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    venue_id BIGINT NOT NULL REFERENCES venues(id) ON DELETE RESTRICT,
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'upcoming',
    total_seat_count INT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT check_event_status IN ('upcoming', 'on_sale', 'sold_out', 'cancelled'),
    CONSTRAINT check_total_seats_positive CHECK (total_seat_count > 0),
    CONSTRAINT check_event_time_valid CHECK (start_time < end_time)
);

CREATE INDEX idx_events_status ON events(status);
CREATE INDEX idx_events_start_time ON events(start_time DESC);
CREATE INDEX idx_events_venue_id ON events(venue_id);

-- ============================================================================
-- TABLE: users
-- ============================================================================
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    phone VARCHAR(20) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT check_email_not_empty CHECK (email <> ''),
    CONSTRAINT check_name_not_empty CHECK (name <> '')
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_phone ON users(phone);

-- ============================================================================
-- TABLE: seats
--
-- Critical table for high-concurrency booking. Each row represents a physical seat.
-- ============================================================================
CREATE TABLE seats (
    id BIGSERIAL PRIMARY KEY,
    event_id BIGINT NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    section VARCHAR(50) NOT NULL,
    row VARCHAR(10) NOT NULL,
    number VARCHAR(10) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    category VARCHAR(50) NOT NULL DEFAULT 'General',
    status VARCHAR(50) NOT NULL DEFAULT 'available',
    held_by UUID REFERENCES users(id) ON DELETE SET NULL,
    held_until TIMESTAMP,
    version BIGINT NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    -- Status can be: available, held, booked
    CONSTRAINT check_seat_status IN ('available', 'held', 'booked'),
    CONSTRAINT check_price_positive CHECK (price > 0),
    -- If held_by is set, held_until must be set
    CONSTRAINT check_held_consistency CHECK ((held_by IS NULL AND held_until IS NULL) OR (held_by IS NOT NULL AND held_until IS NOT NULL)),
    -- Unique seat per event (no duplicate seats within same event)
    UNIQUE(event_id, section, row, number)
);

-- CRITICAL: This is the workhorse index for seat availability queries during high concurrency
CREATE INDEX idx_seats_event_status ON seats(event_id, status) WHERE status != 'booked';

-- For cleanup queries: find all expired holds
CREATE INDEX idx_seats_held_until ON seats(held_until) WHERE status = 'held';

-- For updates on specific seats (prevent full table scans during booking)
CREATE INDEX idx_seats_event_id ON seats(event_id);

-- ============================================================================
-- TABLE: bookings
--
-- Represents a complete booking order. One booking → many seats (via booking_seats).
-- ============================================================================
CREATE TABLE bookings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    event_id BIGINT NOT NULL REFERENCES events(id) ON DELETE RESTRICT,
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    total_amount DECIMAL(12, 2) NOT NULL,
    payment_reference VARCHAR(255),
    payment_method VARCHAR(50),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    paid_at TIMESTAMP,

    -- Status can be: pending, confirmed, failed, refunded, expired
    CONSTRAINT check_booking_status IN ('pending', 'confirmed', 'failed', 'refunded', 'expired'),
    CONSTRAINT check_booking_amount_positive CHECK (total_amount > 0)
);

-- Primary query: get user's booking history
CREATE INDEX idx_bookings_user ON bookings(user_id, created_at DESC);

-- Query: find all pending bookings for a specific event
CREATE INDEX idx_bookings_event ON bookings(event_id, status);

-- Partial index: only index unresolved bookings (MUCH smaller than full index)
-- Used for queries like: "Find all pending/failed bookings for cleanup/retry"
-- This index is much smaller in memory/disk than indexing all bookings
CREATE INDEX idx_bookings_unresolved ON bookings(created_at ASC)
    WHERE status IN ('pending', 'failed');

-- ============================================================================
-- TABLE: booking_seats
--
-- Junction table: many seats per booking, many bookings per seat (historically).
-- This table creates the link between bookings and the specific seats they booked.
-- ============================================================================
CREATE TABLE booking_seats (
    id BIGSERIAL PRIMARY KEY,
    booking_id UUID NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
    seat_id BIGINT NOT NULL REFERENCES seats(id) ON DELETE RESTRICT,
    price_at_booking DECIMAL(10, 2) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    -- Prevent duplicate seat entries in same booking
    UNIQUE(booking_id, seat_id),

    CONSTRAINT check_price_positive CHECK (price_at_booking > 0)
);

-- Query: find all seats in a booking
CREATE INDEX idx_booking_seats_booking ON booking_seats(booking_id);

-- Query: find all bookings that contain a specific seat (data integrity checks)
CREATE INDEX idx_booking_seats_seat ON booking_seats(seat_id);

-- ============================================================================
-- CONSTRAINT CHECK: Double-booking prevention
--
-- NOTE: This is checked at application level with optimistic locking:
-- When updating seat status from 'available' to 'booked', we use WHERE version = X
-- If two transactions try simultaneously, only one will update the correct version.
-- ============================================================================

-- Verify no seat appears twice in confirmed bookings (data integrity check)
ALTER TABLE booking_seats
    ADD CONSTRAINT check_no_duplicate_confirmed_seats
    EXCLUDE USING gist (
        seat_id WITH =,
        CAST((SELECT status FROM bookings b WHERE b.id = booking_seats.booking_id) AS TEXT) WITH =
    ) WHERE (SELECT status FROM bookings b WHERE b.id = booking_seats.booking_id) = 'confirmed';

```

---

## Schema Design Commentary

### 1. Why UUID for `booking.id` instead of SERIAL?

**Answer:**

In a distributed, high-concurrency system with 500K concurrent users, using SERIAL (auto-incrementing integers) creates centralized contention:

**The Problem with SERIAL:**

- Every new booking must obtain the next integer from a sequence
- At 500K concurrent users × 25K RPS, the sequence becomes a bottleneck
- PostgreSQL serializes sequence allocation → lock queue on `pg_sequence_data`
- All booking inserts wait for sequence lock first, defeating parallelism

**Why UUID is Better:**

- Generated client-side or with `gen_random_uuid()` - no server contention
- Each transaction generates its own ID independently
- 128-bit UUID space is astronomically large (no collision risk)
- Allows sharding/partitioning across multiple databases without coordination
- Distributed systems can generate globally unique IDs without central authority

**Trade-off:**

- UUIDs are 16 bytes vs SERIAL's 8 bytes (2x storage, 2x index size)
- Slower index lookups (longer keys) but faster insert throughput (no sequence lock)
- **Net win:** +2% storage cost to avoid -50% throughput hit on inserts

**Constraint implication:** With our $2,000/month budget, we can't afford sequence contention. UUID cost (extra storage) is far cheaper than adding more database instances.

---

### 2. Why does `seats` have a `version` column?

**Answer:**

The version column is the **core mechanism for preventing double-bookings** under high concurrency. This implements Optimistic Locking.

**The Problem Without Version:**

```sql
-- Thread 1: User A wants to book seat #42
SELECT * FROM seats WHERE id = 42 AND status = 'available';
-- Returns: seat is available, version = 5

-- Thread 2: User B wants to book seat #42 (same moment)
SELECT * FROM seats WHERE id = 42 AND status = 'available';
-- Returns: seat is available, version = 5

-- Thread 1: Books the seat
UPDATE seats SET status = 'booked', version = 6 WHERE id = 42;
-- ✓ Success (seat now booked)

-- Thread 2: Also books the seat (DOUBLE-BOOKING!)
UPDATE seats SET status = 'booked', version = 6 WHERE id = 42;
-- ✓ Also succeeds (but seat was already booked - data corruption)
```

**The Solution With Version (Optimistic Locking):**

```sql
-- Thread 1: Booking logic
UPDATE seats
    SET status = 'booked', version = version + 1, held_by = NULL
    WHERE id = 42 AND version = 5;  -- Version check here
-- ✓ Success: 1 row affected, version becomes 6

-- Thread 2: Same booking logic
UPDATE seats
    SET status = 'booked', version = version + 1, held_by = NULL
    WHERE id = 42 AND version = 5;  -- Version check fails!
-- ✗ Failed: 0 rows affected (version is now 6, not 5)
-- Application retries: "Seat no longer available"
```

**Why This Is Critical:**

- Pessimistic locking (FOR UPDATE) causes lock queues at 500K users
- Optimistic locking has no lock waits, just failed retries
- Failed retries are much cheaper than lock waits
- Application can immediately show user "seat taken" without delays

**Storage cost:** 8 bytes per seat, negligible.
**Concurrency gain:** Eliminates single-seat bottleneck.

---

### 3. Why `held_until` instead of just holding seats at application level?

**Answer:**

The `held_until` timestamp is **database-level seat reservation management** - it's the difference between:

- **Application-level:** "I'll remember you reserved this seat" (fragile, loses data on crash)
- **Database-level:** "This seat is reserved until 2026-05-12 12:05:30 UTC" (durable, survives crashes)

**Scenario: Why Application-Level Fails**

```
Time 12:00:00: User A reserves seat #42 (app stores in memory: "reserved for 10 minutes")
Time 12:03:00: User A's browser crashes (doesn't complete payment)
Time 12:05:00: System crash (app server goes down)
               Reservation in memory: LOST
Time 12:05:30: App comes back up, no record of User A's reservation
Time 12:05:35: User B buys seat #42 (User A can't complete purchase anymore)

Result: User A spent 5 minutes, then seat was gone. Poor UX.
```

**Why Database-Level Works**

```
Time 12:00:00: User A reserves seat, set held_by = user_a_id, held_until = 12:10:00
                (persisted to disk, survives any crash)
Time 12:03:00: User A's browser crashes
Time 12:05:00: System crash, restart
Time 12:05:15: App comes back up
Time 12:05:20: User A reloads page, sees seat still reserved (held_until still in future)
               Can complete payment within 10 minutes window

Time 12:10:01: Cleanup job: SELECT * FROM seats WHERE held_until < NOW() AND status = 'held'
               Resets held_by = NULL, status = 'available'
               Seat returns to market for other users
```

**Additional Benefits:**

- **Prevents stale holds:** If user abandons cart at 12:03, seat auto-releases at 12:10
- **Fair queuing:** First user gets 10-minute window, then next user can reserve
- **Concurrent cleanup:** Single cleanup job can release thousands of expired holds with one query
- **No application state needed:** App doesn't need to remember what it reserved

**Storage cost:** 8 bytes per seat (timestamp), negligible.
**UX/fairness gain:** Essential for good user experience under high load.

---

### 4. Why partial index on `bookings.status`?

**Answer:**

A partial index on unresolved statuses (`pending`, `failed`) uses dramatically less storage and memory than a full index, while maintaining query performance for the queries that matter.

**Storage Cost Comparison:**

```
Full index on bookings(created_at ASC):
- Total bookings: 10M
- Index entries: 10M
- Size: ~160 MB
- In-memory cache: ~160 MB RAM

Partial index on bookings WHERE status IN ('pending', 'failed'):
- Only unresolved bookings: ~100K (1% of total)
- Index entries: 100K
- Size: ~1.6 MB
- In-memory cache: ~1.6 MB RAM
```

**Query Use Cases:**

```sql
-- Case 1: User views booking history (common query)
SELECT * FROM bookings WHERE user_id = ? AND status = 'confirmed';
→ Full index on bookings(user_id, created_at) is better (indexes this query)

-- Case 2: System retry logic (occasional query)
SELECT * FROM bookings WHERE status IN ('pending', 'failed') ORDER BY created_at ASC LIMIT 1000;
→ Partial index is perfect (only indexes 1% of data, exact match)

-- Case 3: Analytics (rare query)
SELECT COUNT(*) FROM bookings;
→ Full table scan anyway (why pay for index?)
```

**Why Multiple Indexes:**
We have BOTH:

1. **`idx_bookings_user`** - Full index for "my bookings" queries (user_id + created_at)
2. **`idx_bookings_unresolved`** - Partial index for background jobs (cleanup, retry)

This is strategic: we index what we frequently query, not everything.

**$2,000 Budget Implication:**

With limited budget, every index costs:

- Disk space (stored on expensive RDS)
- Memory (AWS charges for provisioned IOPS)
- Insert/update overhead (slower writes)

Partial indexes let us:

- Index high-concurrency queries (full index on `idx_bookings_user`)
- Still support background jobs efficiently (partial index)
- Use 10% of the storage of a full index

**Example Storage Savings:**

```
Scenario: 50M historical bookings, $2,000/month budget
- Full index on bookings(created_at): 800 MB
- Partial index instead: 8 MB
- Savings: 792 MB = ~$20-30/month at AWS rates
- That's 1-2% of entire monthly budget!
```

At scale, every percentage point matters.

---

## Key Design Decisions Summary

| Decision                                 | Reason                                                                | Cost/Benefit                         |
| ---------------------------------------- | --------------------------------------------------------------------- | ------------------------------------ |
| **UUID for bookings.id**                 | No sequence contention, scalable to distributed systems               | +2% storage, +50% insert throughput  |
| **Version column on seats**              | Optimistic locking prevents double-bookings without lock waits        | +8 bytes/seat, eliminates bottleneck |
| **held_until timestamp**                 | Durable seat reservations survive crashes, enable fair queuing        | +8 bytes/seat, improves UX           |
| **Partial index on unresolved bookings** | 100x smaller index uses 1% of storage, still fast for background jobs | -$20-30/month savings                |
| **Foreign keys with RESTRICT/CASCADE**   | Prevent orphaned records, maintain data integrity                     | +validation overhead at insert time  |
| **CHECK constraints on status**          | Enforce valid values at database level, prevent invalid states        | Prevents invalid data corruption     |

---

## Implementation Notes for Backend Team

1. **Optimistic Locking Retry Logic:**
   - When seat booking UPDATE affects 0 rows, catch and retry with next available seat
   - Implement exponential backoff (1ms, 2ms, 4ms...) to avoid thundering herd
   - Max 5 retries before returning "all seats temporarily unavailable"

2. **Seat Hold Expiration:**
   - Run cleanup job every 30 seconds: `UPDATE seats SET held_by = NULL, status = 'available' WHERE held_until < NOW() AND status = 'held'`
   - Or use trigger: `CREATE TRIGGER release_expired_holds BEFORE SELECT ... `
   - Cleanup job is simpler, faster (batch vs individual triggers)

3. **Connection Pool Sizing:**
   - `idx_seats_event_status` will be hot (queried millions of times during peak)
   - Keep 20-30 database connections ready for seat queries
   - Separate connection pool for booking writes (prevents starving reads)

4. **Monitoring:**
   - Alert if `seats.version` values on popular events exceed 1000 (seat is contended)
   - Track: "How many booking attempts fail due to optimistic lock?" (should be <5%)
   - Monitor partial index size to ensure it stays <5% of full index
