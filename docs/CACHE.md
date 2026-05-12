# Cache Layer Design

## Executive Summary

The cache layer is critical for reducing database load at 25K RPS peak. We cannot query the database for every seat availability check—it would exhaust the connection pool. This document specifies exactly what to cache, for how long, and what invalidates each cache entry.

**Core Principle:** Cache-Aside Pattern (Delete on invalidation, Next read repopulates)

---

## Cache Entry 1: Event Details

### Redis Key Format

```
event:{event_id}
```

Example:

```
event:123
```

### Data Structure (JSON in Redis string)

```json
{
  "id": 123,
  "name": "Avengers: Endgame",
  "description": "Epic superhero film",
  "venue_id": 5,
  "venue_name": "Inox Mumbai",
  "start_time": "2026-05-20T12:00:00Z",
  "end_time": "2026-05-20T15:30:00Z",
  "status": "on_sale",
  "total_seat_count": 500,
  "created_at": "2026-05-01T10:00:00Z"
}
```

### TTL: 1 Hour (3,600 seconds)

**Justification:**

- Event details change infrequently (status, name updates are rare)
- Hard to predict when event details change (not tied to seat bookings)
- 1 hour is long enough to avoid repeated database queries
- Short enough to catch status changes (on_sale → sold_out) within reasonable time
- 500K users all requesting same event details = massive read:write ratio favors TTL-only

**Trade-off Analysis:**

- Too short (5 min): Every 5 minutes, 100K+ users fetch from DB (wasted queries)
- Too long (24 hr): Sold-out event still shows "on_sale" for hours (poor UX)
- **1 hour: Sweet spot** between DB load and UX freshness

### Invalidation Trigger

**Event-driven invalidation occurs when:**

1. Event status changes: `upcoming` → `on_sale` → `sold_out` → `cancelled`
2. Event details updated (price adjustment, venue change - rare)

**Code Logic:**

```python
def update_event_status(event_id, new_status):
    # Update database (source of truth)
    db.execute("""
        UPDATE events SET status = ? WHERE id = ?
    """, new_status, event_id)

    # Invalidate cache
    redis.delete(f"event:{event_id}")

    # Next read will fetch from DB and repopulate cache with 1-hour TTL
```

### Invalidation Strategy: TTL + Event-Driven

- **Primary:** Event-driven (delete when status changes)
- **Backup:** 1-hour TTL (if we miss the event-driven trigger, stale data expires)
- **Why both:** TTL alone misses status transitions; Event-driven alone risks stale cache if application crashes before invalidation

**Decision:** Use event-driven deletion + TTL as safety net

---

## Cache Entry 2: Seat Availability Count per Event per Category

### Redis Key Format

```
availability:{event_id}:{category}
```

Examples:

```
availability:123:VIP        → 45 (45 VIP seats available)
availability:123:General    → 128
availability:123:Premium    → 67
```

### Data Structure (Redis integer counter)

```
INCRBY availability:123:VIP -1    (Decrement when seat booked)
INCRBY availability:123:VIP 1     (Increment when hold expires - rare)
GET availability:123:VIP          (Read availability)
```

### TTL: 30 Seconds

**Justification:**

- This is the **hottest** query during peak: "How many VIP seats left?"
- Availability changes frequently (every booking changes count)
- 30 seconds balances:
  - **Too short (5 sec):** At 25K RPS with 80% being availability checks = 20K reads/sec × 5 sec = refreshes every 100ms (overkill, doesn't scale with TTL)
  - **Too long (5 min):** User books VIP, but counts still show seats available for 300 seconds (terrible UX, false inventory)
  - **30 seconds:** Acceptable staleness (seat might be booked but still showing available for max 30 sec), balances load

**Read Pattern:**

```
User clicks "Filter by VIP seats"
  → Check Redis: availability:123:VIP (typically hits)
  → Returns "45 seats available" in <5ms

User doesn't see "Currently 100 VIP seats" then later "50 seats"
  → Instead sees availability window update every ~30 seconds max
  → Acceptable staleness for browsing experience
```

### Invalidation Trigger

**Occurs on:**

1. Seat status changes: `available` → `booked` or `available` → `held`
2. Hold expires: `held` → `available` (seat released back to inventory)

**Code Logic:**

```python
def book_seat(user_id, seat_id, event_id):
    # Get seat details
    seat = db.query("SELECT * FROM seats WHERE id = ?", seat_id)

    # Update DB (source of truth)
    db.execute("""
        UPDATE seats
        SET status = 'booked', version = version + 1, held_by = NULL
        WHERE id = ? AND version = ?
    """, seat_id, seat.version)

    # Invalidate cache: availability count changed
    redis.delete(f"availability:{event_id}:{seat.category}")

    # Next read will COUNT from DB and repopulate cache with 30-sec TTL
    # Query: SELECT COUNT(*) FROM seats WHERE event_id = ? AND category = ? AND status = 'available'
```

**When hold expires (background job):**

```python
def release_expired_holds():
    # Database: Mark expired holds as available
    db.execute("""
        UPDATE seats
        SET status = 'available', held_by = NULL, held_until = NULL
        WHERE held_until < NOW() AND status = 'held'
    """)

    # For each affected event+category, invalidate cache
    affected_events_categories = db.query("""
        SELECT DISTINCT event_id, category FROM seats
        WHERE held_until < NOW() AND status = 'available'
    """)

    for event_id, category in affected_events_categories:
        redis.delete(f"availability:{event_id}:{category}")
        # Next read will recount and repopulate
```

### Invalidation Strategy: Event-Driven + TTL (Hybrid)

**Primary:** Event-driven deletion (when seat status changes)
**Backup:** 30-second TTL (expiration-based refresh)

**Why:**

- Event-driven: Catches invalidation immediately on booking
- TTL: Handles case where invalidation logic has a bug or crashes
- 30 seconds worst-case staleness is acceptable for availability

---

## Cache Entry 3: Static Seat Map Layout

### Redis Key Format

```
seatmap:{event_id}
```

Example:

```
seatmap:123
```

### Data Structure (JSON in Redis string)

```json
{
  "event_id": 123,
  "sections": [
    {
      "section": "A",
      "rows": [
        {
          "row": "1",
          "seats": [
            { "number": "1", "category": "General", "price": 500 },
            { "number": "2", "category": "General", "price": 500 },
            { "number": "3", "category": "Premium", "price": 800 }
          ]
        }
      ]
    }
  ]
}
```

### TTL: 24 Hours (86,400 seconds)

**Justification:**

- Seat layout is **completely static** once event is created
- Seat positions don't change mid-event
- Theater layout defined at event creation, never modified during sales
- 24-hour TTL is just paranoia for "what if someone manually updates seats" (almost never happens)
- Main benefit: Browser-side caching (CDN, client-side cache use this)

**Scale Benefit:**

- Each user downloads seatmap once (large JSON ~50-200KB)
- Then clicking on seats is client-side rendering (no server hits)
- Reduces read load on API servers

### Invalidation Trigger

**Never changes during normal operation**

- Only invalidates if: Event is cancelled, Seats are manually adjusted (administrative action)

**Code Logic:**

```python
def update_event_seats(event_id, new_seat_config):
    # Rarely used: Only for admin changes
    db.execute("UPDATE seats SET ... WHERE event_id = ?", event_id)

    # Invalidate cache
    redis.delete(f"seatmap:{event_id}")

    # Next user will fetch fresh seatmap from DB
```

### Invalidation Strategy: TTL-Only (with Manual Event-Driven Optional)

**Primary:** 24-hour TTL (expiration-based)
**Optional:** Manual invalidation for admin changes

**Why TTL-Only is Sufficient:**

- Seat layout literally never changes during normal operations
- Even if you miss an invalidation event, 24-hour staleness is not a big deal (seats have fixed positions)
- Simplifies code: no invalidation logic needed

---

## Cache Entry 4: User Booking History (Personal)

### Redis Key Format

```
user_bookings:{user_id}
```

Example:

```
user_bookings:uuid_12345
```

### Data Structure (JSON in Redis string)

```json
{
  "user_id": "uuid_12345",
  "bookings": [
    {
      "booking_id": "booking_uuid_789",
      "event_id": 123,
      "event_name": "Avengers: Endgame",
      "status": "confirmed",
      "total_amount": 2500,
      "created_at": "2026-05-10T14:30:00Z"
    }
  ],
  "cached_at": "2026-05-12T12:00:00Z"
}
```

### TTL: 5 Minutes (300 seconds)

**Justification:**

- Personal booking history changes infrequently (user books ~1-2 times per month)
- But when user views "My Bookings", they expect recent accuracy
- 5 minutes: User refreshes page, sees most recent bookings
- Balances: Personalization (everyone has own copy) vs staleness

### Invalidation Trigger

**Occurs when:**

1. User completes a booking: Insert new booking
2. User cancels/refunds: Update booking status

**Code Logic:**

```python
def complete_booking(user_id, booking_id):
    # Update DB
    db.execute("INSERT INTO bookings ... VALUES (...)")

    # Invalidate user's booking history cache
    redis.delete(f"user_bookings:{user_id}")

    # Next read will fetch from DB and repopulate
```

### Invalidation Strategy: Event-Driven + TTL

**Primary:** Event-driven (delete when booking changes)
**Backup:** 5-minute TTL (safety net)

---

## Cache Entry 5: Event Search Results (Filtered List)

### Redis Key Format

```
events_search:{city}:{genre}:{date}
```

Examples:

```
events_search:Mumbai:Action:2026-05-20
events_search:Delhi:Comedy:2026-05-21
```

### Data Structure (JSON array in Redis string)

```json
{
  "query": "Mumbai + Action + 2026-05-20",
  "results": [
    { "id": 123, "name": "Avengers", "venue": "Inox", "start_time": "12:00" },
    {
      "id": 124,
      "name": "Fast & Furious",
      "venue": "PVR",
      "start_time": "15:30"
    }
  ],
  "cached_at": "2026-05-12T12:00:00Z"
}
```

### TTL: 10 Minutes (600 seconds)

**Justification:**

- Search results are moderately expensive (3-way join in DB)
- But highly cacheable (results same for all users with same filters)
- 10 minutes: Between availability (30s, changes often) and event-details (1h, static)

### Invalidation Trigger

**Occurs when:**

- New event added to city/genre (rare, admin action)
- Event status changes (on_sale → sold_out)
- Old events expire (automatic by TTL)

---

## WHAT WE EXPLICITLY DO NOT CACHE

### Individual Seat Status (Per-Seat: available/held/booked)

**Redis Key That Would Look Like:**

```
seat:123:42           → "available"  (we DON'T do this)
seat:123:43           → "booked"
seat:123:44           → "held"
```

### Why NOT to Cache Individual Seat Status

**Reason 1: Double-Booking Risk**

```
User A checks cache: seat:123:42 → "available"
User A acquires lock in Redis, posts booking request

User B checks cache: seat:123:42 → "available"  (stale from cache)
User B also acquires lock, posts booking request

Both A and B think they have the right to book seat 42
Result: Double-booking or one of them fails inconsistently
```

Without ultra-fast cache invalidation (<1ms), there's a race window.

**Reason 2: Cache Invalidation Overhead**

- At 25K RPS, every booking changes 1-3 seat statuses
- That's 25K-75K cache invalidations per second
- Redis can handle ~100K ops/sec, but we'd use 75% of capacity just invalidating
- Doesn't scale

**Reason 3: Version Number Handles It Better**

- Database `seats.version` column prevents double-bookings atomically
- Version check is in the database UPDATE WHERE clause (guaranteed atomic)
- No cache can be more reliable than this

**Therefore:** Individual seat status lives in database only. Queries for "is this seat available?" hit database, but are cached as counts (availability:{event_id}:{category}) not per-seat.

---

## Overall Invalidation Strategy

### Pattern: Cache-Aside (Delete on Change)

```
When data changes in database:
  1. Update database (source of truth)
  2. Delete Redis key (explicit invalidation)
  3. Don't update cache (leave it empty)
  4. On next read: Query database, repopulate cache with TTL
```

### Why Cache-Aside Instead of Write-Through?

**Write-Through (update cache during write):**

```
User books seat
  → Update database
  → Update cache immediately
  → Send response to user
```

Problems:

- Double the work per transaction
- If database update succeeds but cache update fails: stale data that outlives TTL
- Harder to debug

**Cache-Aside (delete cache on write):**

```
User books seat
  → Update database (source of truth)
  → Delete Redis key
  → Send response to user
  → Next reader queries database, repopulates cache
```

Benefits:

- Simpler: Database is always source of truth
- Failures: If Redis delete fails, next TTL expiration fixes it
- Safe: No data in cache that isn't in database

**Our Choice:** Cache-Aside because database is the source of truth and cache is purely an optimization.

---

## Code-Level Invalidation Example

### Scenario: User Books a Seat

```python
def book_seat(user_id, seat_id, event_id):
    """Complete booking and handle cache invalidation."""

    # Step 1: Get seat details (for category info)
    seat = db.query("""
        SELECT id, event_id, category, status, version
        FROM seats WHERE id = ?
    """, seat_id)

    if not seat:
        return {"error": "Seat not found"}

    # Step 2: Acquire Redis lock (from CONCURRENCY.md)
    lock_key = f"seat_lock:{event_id}:{seat_id}"
    lock_acquired = redis.eval(LOCK_SCRIPT, 1, lock_key, user_id, 15)

    if not lock_acquired:
        return {"error": "Seat temporarily unavailable, try again"}

    try:
        # Step 3: Paranoia check - confirm seat still available
        seat_check = db.query("""
            SELECT * FROM seats WHERE id = ? AND status = 'available'
        """, seat_id)

        if not seat_check:
            return {"error": "Seat already booked"}

        # Step 4: Book the seat with optimistic locking
        update_result = db.execute("""
            UPDATE seats
            SET status = 'booked', version = version + 1, held_by = NULL
            WHERE id = ? AND version = ?
        """, seat_id, seat.version)

        if update_result.rows_affected == 0:
            # Version mismatch = seat was booked by someone else
            return {"error": "Seat taken, try another"}

        # Step 5: Create booking record
        booking = db.insert_booking(user_id, event_id, seat_id)

        # Step 6: CRITICAL - Invalidate cache
        # Delete availability count (will be recounted on next read)
        redis.delete(f"availability:{event_id}:{seat.category}")

        # Also delete event details (might affect sold-out status)
        redis.delete(f"event:{event_id}")

        # Step 7: Release Redis lock
        redis.delete(lock_key)

        return {
            "status": "success",
            "booking_id": booking.id,
            "confirmation": f"Booked seat {seat.number}"
        }

    except Exception as e:
        # Error during booking, release lock and propagate error
        redis.delete(lock_key)
        raise e
```

### Scenario: Hold Expires (Background Job)

```python
def release_expired_holds():
    """Run every 30 seconds: Release holds that have expired."""

    # Step 1: Find all expired holds
    expired_seats = db.query("""
        SELECT DISTINCT event_id, category FROM seats
        WHERE status = 'held' AND held_until < NOW()
    """)

    # Step 2: Update them in database
    db.execute("""
        UPDATE seats
        SET status = 'available', held_by = NULL, held_until = NULL
        WHERE status = 'held' AND held_until < NOW()
    """)

    # Step 3: Invalidate cache for each affected event+category
    invalidation_count = 0
    for event_id, category in expired_seats:
        redis.delete(f"availability:{event_id}:{category}")
        invalidation_count += 1

    log.info(f"Released {len(expired_seats)} seat holds, invalidated {invalidation_count} cache keys")
```

---

## Cache Sizing Estimate

**Memory Usage at Peak (500K concurrent users):**

| Cache Entry                   | Key Count                        | Avg Size  | Total       |
| ----------------------------- | -------------------------------- | --------- | ----------- |
| `event:*`                     | 1,000 events                     | 500 bytes | 500 KB      |
| `availability:*:*`            | 5,000 (1K events × 5 categories) | 20 bytes  | 100 KB      |
| `seatmap:*`                   | 1,000 events                     | 100 KB    | 100 MB      |
| `user_bookings:*`             | 100K active users                | 2 KB      | 200 MB      |
| `events_search:*`             | 50K search queries               | 5 KB      | 250 MB      |
| Seat locks (from CONCURRENCY) | 50K active seats                 | 100 bytes | 5 MB        |
| **Total**                     | -                                | -         | **~555 MB** |

**AWS r6g.large ElastiCache:**

- Memory: 16 GB
- Cost: ~$100/month
- Utilization: 555 MB / 16 GB = 3.5% (comfortable headroom)

**Fits easily in budget.**

---

## Monitoring Alerts

Set up monitoring to catch cache issues:

```
Alert if:
  - Cache hit rate < 70% (cache strategy not working)
  - Redis memory usage > 50% (running out of cache space)
  - Availability cache stale > 60 sec (invalidation failing)
  - Lock acquisition latency > 10ms (Redis overloaded)
  - DB query latency after cache miss > 100ms (N+1 queries problem)
```

---

## Summary Table

| Cache Entry       | TTL      | Size  | Invalidation       | Pattern | Why                                                         |
| ----------------- | -------- | ----- | ------------------ | ------- | ----------------------------------------------------------- |
| `event:*`         | 1h       | 500B  | Event-driven + TTL | Both    | Status changes important, but hard to predict               |
| `availability:*`  | 30s      | 20B   | Event-driven + TTL | Both    | HOTTEST query, changes frequently, 30s staleness acceptable |
| `seatmap:*`       | 24h      | 100KB | TTL-only           | TTL     | Static data, never changes in normal ops                    |
| `user_bookings:*` | 5m       | 2KB   | Event-driven + TTL | Both    | Personal data, changes infrequently                         |
| `events_search:*` | 10m      | 5KB   | TTL-only           | TTL     | Search results relatively stable                            |
| `seat:{seat_id}`  | ❌ NEVER | N/A   | N/A                | N/A     | Too risky (double-booking), version column handles it       |

---

## Implementation Roadmap

**Phase 1 (Week 1):** Implement availability count cache

- Most important for reducing DB load
- Biggest performance gain with smallest complexity

**Phase 2 (Week 2):** Implement event details cache

- Adds user-facing improvements (faster event loads)
- Simple invalidation logic

**Phase 3 (Week 3):** Implement seatmap cache

- Browser-side optimization (reduce API traffic)
- Add Cache-Control headers for CDN distribution

**Phase 4 (Week 4):** Implement personal booking history cache

- Nice-to-have optimization
- Requires user authentication context
