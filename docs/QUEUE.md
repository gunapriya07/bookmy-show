# Async Order Flow: Payment Processing Queue Design

## Executive Summary

This document defines how payment processing moves off the critical API path using AWS SQS. Payments are sent to a queue and processed asynchronously, keeping the API responsive during peak load while maintaining transactional safety.

**Core Principle:** Decouple payment processing from booking confirmation. Booking is immediately confirmed (seats locked), payment is processed in background queue. If payment fails, we have time to contact user before ticket delivery.

---

## Section 1: Why Async? The Math

### Synchronous Payment Processing (Why It Fails)

If we made payment calls **synchronous** (API waits for payment response before returning):

**Current Pool Formula (from CONCURRENCY.md):**

```
Connections_held = (% query_RPS × query_duration_s) + (% payment_RPS × payment_duration_s)
```

At current 80% regular queries + 20% payment queries:

```
Connections_held = 0.80 × X × 0.020 + 0.20 × X × 0.800
                = 0.016X + 0.160X
                = 0.176X
```

Pool exhaustion at 500 connections:

```
0.176X = 500
X = 2,840 RPS
```

**Now what if payments were SYNCHRONOUS?** (API thread blocks entire duration)

At 25K RPS peak with 20% payment requests:

```
Payment requests/sec = 0.20 × 25,000 = 5,000 requests/second
Each held for 800ms = 5,000 × 0.8 = 4,000 connections

Total needed:
  Regular queries: 0.80 × 25,000 × 0.020 = 400 connections
  Payment (sync):  0.20 × 25,000 × 0.800 = 4,000 connections
  Total:                                    4,400 connections

We have: 500 connections
Shortfall: -3,900 connections (-88% insufficient)
```

**The Disaster:**

```
At 25K RPS, payments are the MULTIPLIER that breaks the system:
- 400 connections for queries: Fine (out of 500)
- 4,000 connections for payments: CATASTROPHIC (8x over limit)
→ Every payment request fails with "No connections available"
→ All users experience: "Payment processing unavailable" at peak
```

### Asynchronous Payment Processing (The Solution)

With async queue-based processing:

```
API path: POST /api/bookings
  1. Acquire Redis lock ✓ (milliseconds)
  2. Verify seats available ✓ (database query, ~20ms)
  3. Book seats ✓ (database update, ~10ms)
  4. Publish to SQS ✓ (network call, ~50ms)
  5. Return "Booking confirmed, payment pending" to user ✓ (total ~80ms)
  → API connection freed immediately

Separate worker pool (not API servers):
  Background: Process payments from queue (no impact on API connections)
```

**New connection model:**

```
API connections: ~100-200 (for booking flow, no payment waits)
Worker connections: ~10-20 (can retry indefinitely without impacting users)
Total: ~130 connections (vs 4,400 with sync)
```

**The Math:**

- User: Gets response in 80ms (seats locked, booking confirmed)
- Payment: Processed asynchronously, user notified of status later
- Pool exhaustion: Eliminated (payments don't hold API connections)

**Result:** ✓ System scales to 25K RPS, ✓ Payments process reliably (no timeouts), ✓ API stays responsive

---

## Section 2: The Queue Message Format

### SQS Message Body (Complete JSON)

```json
{
  "messageVersion": "1.0",
  "bookingId": "550e8400-e29b-41d4-a716-446655440000",
  "userId": "650e8400-e29b-41d4-a716-446655440001",
  "eventId": 123,
  "totalAmount": 2500.0,
  "currency": "INR",
  "paymentToken": "pm_1A1A1A1A1A1A1A1A1A1A1A1A",
  "paymentMethod": "stripe",
  "seatIds": [42, 43, 44],
  "seatDetails": [
    { "seat_id": 42, "row": "A", "number": "1", "price": 800.0 },
    { "seat_id": 43, "row": "A", "number": "2", "price": 800.0 },
    { "seat_id": 44, "row": "A", "number": "3", "price": 900.0 }
  ],
  "userEmail": "user@example.com",
  "userPhone": "+919876543210",
  "idempotencyKey": "booking_550e8400_1715514000",
  "createdAt": "2026-05-12T12:00:00Z",
  "expiresAt": "2026-05-12T12:15:00Z"
}
```

### Field Explanations

| Field            | Type      | Purpose                                | Why Included                                                                       |
| ---------------- | --------- | -------------------------------------- | ---------------------------------------------------------------------------------- |
| `messageVersion` | string    | API versioning for worker              | If message format changes, worker can handle multiple versions                     |
| `bookingId`      | UUID      | Unique booking identifier              | Primary key to track this order end-to-end                                         |
| `userId`         | UUID      | User who booked                        | Required to mark booking confirmed/failed in DB                                    |
| `eventId`        | int       | Event being booked                     | Required for audit trail, seat release logic                                       |
| `totalAmount`    | decimal   | Amount to charge                       | Payment gateway needs exact amount                                                 |
| `currency`       | string    | Currency code (INR, USD)               | Payment gateway needs this (prevents charge in wrong currency)                     |
| `paymentToken`   | string    | Stripe/payment token from frontend     | Actual token to charge; already validated by frontend                              |
| `paymentMethod`  | string    | Which gateway (stripe, razorpay, etc.) | Worker routes to correct payment processor                                         |
| `seatIds`        | int[]     | Seat IDs booked                        | Used to mark seats as "booked" after payment success                               |
| `seatDetails`    | object[]  | Full seat data (row, number, price)    | For invoice/confirmation email; avoids DB query                                    |
| `userEmail`      | string    | User's email                           | Send confirmation email without DB query                                           |
| `userPhone`      | string    | User's phone                           | Send SMS notification without DB query                                             |
| `idempotencyKey` | string    | Unique key for deduplication           | **CRITICAL:** Payment gateway must support this; prevents double-charging on retry |
| `createdAt`      | timestamp | When booking was created               | Audit trail, timing analytics                                                      |
| `expiresAt`      | timestamp | When to stop retrying                  | Don't retry charges after 15 minutes (seats held too long)                         |

### Design Principle: Self-Contained Message

**The message must contain everything the worker needs without a database query:**

Why? Because:

1. **Reduced latency:** Worker doesn't wait for DB lookup, calls payment API immediately
2. **Resilience:** If DB is slow/down, payment processing continues
3. **Audit trail:** Message itself is a complete record of what was attempted
4. **Idempotency:** Easy to detect duplicate messages (idempotencyKey is in message)

---

## Section 3: The Worker Logic - Success and Failure Paths

### Worker Pseudocode (Complete Flow)

```python
class PaymentWorker:
    """
    SQS worker that processes payment queue messages.
    Handles success, failure, retries, and dead letter queue routing.
    """

    def process_message(self, message):
        """Main worker entry point."""

        # Step 1: Parse and validate message
        try:
            payload = json.loads(message.body)
            booking_id = payload['bookingId']
            idempotency_key = payload['idempotencyKey']
        except json.JSONDecodeError:
            # Message format invalid, send to DLQ immediately
            self.send_to_dlq(message, "Invalid message format")
            return

        # Step 2: Idempotency check (prevent duplicate charges)
        if self.idempotency_cache.get(idempotency_key):
            # This message was already processed successfully
            log.info(f"Duplicate payment request, skipping: {idempotency_key}")
            message.delete()  # Remove from queue
            return

        # Step 3: Call payment gateway with timeout
        try:
            payment_result = self.charge_payment(
                idempotency_key=payload['idempotencyKey'],
                amount=payload['totalAmount'],
                currency=payload['currency'],
                token=payload['paymentToken'],
                payment_method=payload['paymentMethod'],
                timeout=10  # seconds
            )
        except PaymentTimeoutError:
            # Payment gateway didn't respond (could be success or failure)
            # Don't retry immediately - check status instead
            return self.handle_payment_timeout(message, payload)
        except PaymentNetworkError as e:
            # Transient network error (connection refused, DNS timeout)
            # Retry this message
            return self.handle_transient_error(message, payload, e)
        except Exception as e:
            # Unexpected error
            log.error(f"Unexpected error processing payment: {e}")
            return self.handle_unexpected_error(message, payload, e)

        # Step 4a: PAYMENT SUCCESS
        if payment_result.status == 'succeeded':
            try:
                # Mark booking as confirmed in database
                self.db.execute("""
                    UPDATE bookings
                    SET status = 'confirmed',
                        payment_reference = ?,
                        paid_at = NOW()
                    WHERE id = ? AND status = 'pending'
                """, payment_result.transaction_id, booking_id)

                # Update seat status to 'booked' (seats already in 'held' or 'booked')
                self.db.execute("""
                    UPDATE seats
                    SET status = 'booked', held_by = NULL, held_until = NULL
                    WHERE id IN (SELECT seat_id FROM booking_seats WHERE booking_id = ?)
                """, booking_id)

                # Cache idempotency key (prevent reprocessing for 24 hours)
                self.idempotency_cache.set(idempotency_key, True, ttl=86400)

                # Send confirmation email/SMS
                self.send_confirmation(payload)

                # Delete message from queue (processing complete)
                message.delete()

                log.info(f"Payment confirmed for booking {booking_id}")

            except Exception as e:
                # Database error after payment succeeded
                # This is serious: payment was charged but we can't confirm it
                log.error(f"Database error after successful payment: {e}")
                self.send_dlq(message, f"DB error after payment: {e}")

        # Step 4b: PAYMENT FAILED (User doesn't have funds, invalid card, etc)
        elif payment_result.status == 'failed':
            try:
                # Mark booking as failed
                self.db.execute("""
                    UPDATE bookings
                    SET status = 'failed',
                        payment_reference = ?
                    WHERE id = ? AND status = 'pending'
                """, payment_result.transaction_id, booking_id)

                # Release the seats back to available
                self.db.execute("""
                    UPDATE seats
                    SET status = 'available', held_by = NULL, held_until = NULL
                    WHERE id IN (SELECT seat_id FROM booking_seats WHERE booking_id = ?)
                """, booking_id)

                # Send failure notification to user
                self.send_failure_notification(payload, payment_result.reason)

                # Delete message from queue (no point retrying - payment genuinely failed)
                message.delete()

                log.info(f"Payment failed for booking {booking_id}: {payment_result.reason}")

            except Exception as e:
                log.error(f"Database error marking payment as failed: {e}")
                # Send to DLQ if we can't mark it failed in DB
                self.send_to_dlq(message, f"DB error marking failure: {e}")

    # ========== ERROR HANDLING PATHS ==========

    def handle_payment_timeout(self, message, payload):
        """
        Payment gateway returned no response (timeout).
        Can't tell if payment succeeded or failed.
        """
        booking_id = payload['bookingId']
        idempotency_key = payload['idempotencyKey']

        # Try to query payment gateway for transaction status
        try:
            status = self.payment_gateway.query_transaction_status(idempotency_key)

            if status == 'confirmed':
                # Payment actually succeeded (network was slow, but it went through)
                # Treat as success path
                return self.handle_payment_success(message, payload,
                    transaction_id=idempotency_key)

            elif status == 'not_found':
                # Payment never reached gateway (didn't go through)
                # Will retry on next message receive (check visibility timeout)
                log.warning(f"Payment timeout, status unknown, will retry: {booking_id}")
                # Don't delete message, let it be retried (go back in queue)
                # SQS will re-deliver after visibility timeout expires
                return

            elif status == 'failed':
                # Payment gateway rejected it
                return self.handle_payment_failed(message, payload,
                    reason="Payment rejected by gateway")

        except Exception as e:
            # Can't even query status, treat as transient error
            log.error(f"Error querying payment status: {e}")
            return self.handle_transient_error(message, payload, e)

    def handle_transient_error(self, message, payload, error):
        """
        Transient error (network timeout, connection reset, etc).
        Message will be retried by SQS up to max_receive_count times.
        """
        log.warning(f"Transient error processing payment: {error}")
        # Don't delete message from queue
        # SQS will redelivery message after visibility timeout
        # Message.receive_count will increment
        # After max retries, SQS automatically sends to DLQ
        return

    def handle_unexpected_error(self, message, payload, error):
        """
        Unexpected error that we don't know how to handle.
        Send to DLQ for manual inspection.
        """
        log.error(f"Unexpected error: {error}")
        self.send_to_dlq(message, f"Unexpected error: {error}")

    def send_to_dlq(self, message, reason):
        """
        Send message to Dead Letter Queue for manual review.
        """
        dlq_message = {
            "original_message": json.loads(message.body),
            "failure_reason": reason,
            "failed_at": datetime.now().isoformat(),
            "receive_count": message.attributes.get('ApproximateReceiveCount', 0)
        }

        self.dlq.send_message(
            MessageBody=json.dumps(dlq_message),
            MessageAttributes={
                'FailureReason': {'StringValue': reason, 'DataType': 'String'}
            }
        )

        # Delete original message from main queue
        message.delete()

    def send_confirmation(self, payload):
        """Send payment confirmation email and SMS to user."""
        email_body = f"""
        Your booking is confirmed!
        Booking ID: {payload['bookingId']}
        Event: [event name from eventId]
        Seats: {', '.join(map(str, payload['seatIds']))}
        Amount: {payload['totalAmount']} {payload['currency']}
        """
        send_email(payload['userEmail'], "Booking Confirmed", email_body)
        send_sms(payload['userPhone'], "Booking confirmed! Check your email.")

    def send_failure_notification(self, payload, reason):
        """Notify user that payment failed and seats are available again."""
        email_body = f"""
        Payment could not be processed.
        Booking ID: {payload['bookingId']}
        Reason: {reason}

        Your seats have been released and are available for rebooking.
        Please try again with a different payment method.
        """
        send_email(payload['userEmail'], "Payment Failed", email_body)
```

### Worker State Machine

```
Message received from SQS
    ↓
Parse message
    ├─ [INVALID FORMAT] → DLQ
    └─ [VALID]
        ↓
Check idempotency cache
    ├─ [ALREADY PROCESSED] → Delete message
    └─ [NEW REQUEST]
        ↓
Call payment gateway
    ├─ [TIMEOUT]
    │   ├─ Query status
    │   ├─ [CONFIRMED] → Success path
    │   ├─ [FAILED] → Failure path
    │   └─ [NOT_FOUND] → Wait for retry (visibility timeout)
    │
    ├─ [NETWORK ERROR] → Wait for retry (message back in queue)
    │
    └─ [RESPONSE RECEIVED]
        ├─ [SUCCESS (200 OK)]
        │   ├─ Update booking status = 'confirmed'
        │   ├─ Mark seats as 'booked'
        │   ├─ Cache idempotency key
        │   ├─ Send confirmation email
        │   └─ Delete message
        │
        └─ [FAILURE (402, 422, etc)]
            ├─ Update booking status = 'failed'
            ├─ Release seats to 'available'
            ├─ Send failure notification email
            └─ Delete message
```

---

## Section 4: Edge Cases

### Edge Case 1: Server Crashes After SQS Publish But Before API Response

**Scenario:**

```
Time 1: API server publishes message to SQS (successful)
Time 2: API server crashes before returning response to user
Time 3: User sees timeout error (no response from server)
```

**What Happens:**

1. **SQS message is safe** (already persisted in SQS queue)
2. **Booking is not confirmed yet** (waiting for worker to process payment)
3. **Seats are held** (held_until timestamp prevents rebooking for 15 minutes)
4. **Payment worker processes message** (after server comes back up)
5. **Booking eventually confirms** (user can refresh page or check email)

**User Experience:**

```
Attempt 1: "Booking failed (timeout)" → User sees error
Attempt 2: User refreshes page after 1-2 minutes
           → Booking shows as "confirmed"
           → Email arrives: "Your booking is confirmed!"

OR

User waits for support to contact them
→ Support checks database: "Booking is confirmed"
→ Issue resolved (payment went through, seats locked)
```

**Design Principle:** Better to lose the API response than lose the payment. User can always check booking status later.

### Edge Case 2: Payment Gateway Returns Timeout (Neither Success Nor Failure)

**Scenario:**

```
Time 1: Worker calls Stripe payment API
Time 2: Request sent successfully
Time 3: Stripe charges the card
Time 4: Stripe tries to send response back
Time 5: Network connection drops (timeout)
Time 6: Worker receives: "Connection timeout after 10 seconds"
```

**Problem:** Did the charge actually go through?

- Worker doesn't know
- Can't just retry (might double-charge)
- Can't just fail (charge might be pending)

**Solution: Query Payment Gateway Status**

```python
# Worker code (from pseudocode above)
def handle_payment_timeout(self, message, payload):
    idempotency_key = payload['idempotencyKey']

    # Query Stripe: Did you charge this idempotency key?
    status = self.payment_gateway.query_transaction_status(idempotency_key)

    if status == 'confirmed':
        # Stripe: "Yes, we charged and confirmed it"
        return self.handle_payment_success(...)

    elif status == 'not_found':
        # Stripe: "No record of this charge"
        # Request timeout = request never reached Stripe
        # Don't retry yet, let SQS redelivery handle it (visibility timeout)
        return

    elif status == 'failed':
        # Stripe: "We tried to charge but it failed"
        return self.handle_payment_failure(...)
```

**Why Idempotency Key?**

The `idempotencyKey` is the magic that makes this safe:

```
Attempt 1: Worker sends payment with idempotencyKey="booking_550e8400_1715514000"
           → Stripe charges card, returns "confirmed"
           → Network timeout (worker doesn't get response)

Attempt 2: SQS redelivers message
           → Worker sends payment with same idempotencyKey
           → Stripe sees: "Oh, we already processed this key"
           → Stripe returns: "Already confirmed" (doesn't charge again)
           → Worker treats as success

Result: User charged once, not twice ✓
```

**Why Not Just Retry Blindly?**

```
WITHOUT idempotency key:

Attempt 1: Charge $2500 → Success → Timeout (response lost)
Attempt 2: Retry with same request → Charge $2500 again
           → User charged $5000 instead of $2500 ✗ DISASTER
```

---

## Section 5: SQS Configuration

### Visibility Timeout: 30 Seconds

**Definition:** Time that SQS hides a message from other workers after first delivery.

**Calculation:**

Worst-case payment processing time:

```
Query payment API:           5 seconds (normal payment processing)
Payment API timeout buffer:  5 seconds (network slowness)
Query payment status:        2 seconds (if timeout, checking status)
Database update:             3 seconds (if DB is slow)
Send confirmation email:     5 seconds (email service call)
Safety margin:              10 seconds (unexpected delays)

Total: ~30 seconds
```

**Configuration:**

```
VisibilityTimeout = 30 seconds
```

**What Happens:**

```
Worker A receives message at T=0
  → Processing starts
  → Worker has 30 seconds to finish

If message not deleted by T=30:
  → SQS assumes Worker A crashed
  → Message becomes visible again at T=30
  → Worker B picks up same message (retry)
  → Process repeats until:
     a) Message successfully processes (deleted)
     b) Retry count exceeds max (sent to DLQ)
```

**Why 30 seconds?**

- **Too short (5 sec):** Payment processing takes 5 sec min, so message gets redelivered before first worker finishes (duplicate processing, chaos)
- **Too long (300 sec):** If worker crashes, user waits 5 minutes for retry (slow)
- **30 seconds:** Covers worst case, retries within reasonable time

### Max Receive Count: 3

**Definition:** Number of times SQS will redeliver a message before sending to DLQ.

**SQS Retry Behavior:**

```
Attempt 1 (T=0):        Worker receives, processes, doesn't delete
                        → Error (transient)

Visibility timeout expires (T=30)

Attempt 2 (T=30):       Message redelivered to another worker
                        → Same error (transient)

Visibility timeout expires (T=60)

Attempt 3 (T=60):       Message redelivered again
                        → Same error persists

Visibility timeout expires (T=90)

After attempt 3:        Message has been received 3 times
                        → Max receive count exceeded
                        → SQS sends to DLQ

Attempt 4+ on DLQ:      DLQ has separate configuration
                        → Alarm fires for manual review
```

**Configuration:**

```
MaxReceiveCount = 3
```

**Why 3?**

```
Transient error examples (recover on retry):
- Payment gateway temporarily slow (usually recovers within 1-2 retries)
- Database connection pool temporarily exhausted
- Network packet loss (TCP retransmits, typically succeeds 2nd or 3rd time)

After 3 retries = 90 seconds elapsed:
- If problem persists, it's not transient (systemic issue)
- Send to DLQ for investigation
- Alert ops team
- Don't retry forever (wastes resources)
```

**Trade-offs:**

- **Too low (1):** A single transient glitch sends to DLQ (noisy alerts)
- **Too high (10):** Retries for 5+ minutes, expensive
- **3 retries:** ~90 seconds total, good balance

### Full SQS Configuration Summary

```yaml
Queue: PaymentProcessing
  VisibilityTimeout: 30 seconds
  MessageRetentionPeriod: 1209600 seconds (14 days)
  RedrivePolicy:
    DeadLetterTargetArn: arn:aws:sqs:region:account:PaymentProcessing-DLQ
    MaxReceiveCount: 3

DLQ: PaymentProcessing-DLQ
  VisibilityTimeout: 300 seconds (5 minutes)
  MessageRetentionPeriod: 1209600 seconds (14 days)

Monitoring:
  Alert if PaymentProcessing-DLQ has messages
  Alert if PaymentProcessing ApproximateAgeOfOldestMessage > 60 seconds
```

---

## Complete Worker Deployment Architecture

### Worker Pool Configuration

```
EC2 instance for payment workers:
  - Type: t3.large (1 CPU, 8GB RAM is overkill, but simple)
  - Count: 2-3 instances (redundancy, can handle 25K RPS peak)
  - Each handles: ~10K-15K messages per second (simple queue polling)

Auto-scaling:
  - Scale up if ApproximateNumberOfMessagesVisible > 1000
  - Scale down if < 100
  - Max 5 instances (cost cap)

Cost at peak:
  - 2 instances × $0.10/hour = $50/month ✓ (fits in budget)
```

### Worker Code Skeleton (Python with boto3)

```python
import boto3
import json
import time
from datetime import datetime

class PaymentWorker:
    def __init__(self):
        self.sqs = boto3.resource('sqs')
        self.payment_queue = self.sqs.Queue('<queue-url>')
        self.dlq = self.sqs.Queue('<dlq-url>')
        self.db = DatabaseConnection()
        self.payment_api = PaymentGatewayClient()

    def run(self):
        """Main worker loop - process messages indefinitely."""
        while True:
            messages = self.payment_queue.receive_messages(
                MaxNumberOfMessages=10,  # Batch 10 at a time
                WaitTimeSeconds=20  # Long polling (reduce CPU)
            )

            for message in messages:
                try:
                    self.process_message(message)
                except Exception as e:
                    log.error(f"Unexpected error: {e}")
                    # Message stays in queue, will retry

            time.sleep(1)  # Small delay between batches

# Run worker
if __name__ == '__main__':
    worker = PaymentWorker()
    worker.run()
```

---

## Handling Special Cases

### What If SQS Is Down?

```
SQS failure mode:
  - API publishes message to SQS → Fails (SQS unreachable)
  - API catches exception
  - API returns to user: "Booking created, payment pending (will process later)"
  - Booking sits in 'pending' status

Recovery:
  - SQS comes back online
  - User retries booking payment from UI ("Complete Payment" button)
  - New message published to SQS
  - Payment processes

Mitigation:
  - SQS is AWS-managed (99.99% uptime SLA)
  - Unlikely to happen, but API gracefully handles it
```

### What If All Payment Workers Are Crashed?

```
Problem:
  - Messages queue up in SQS
  - ApproximateNumberOfMessagesVisible grows to 10,000+
  - Users see: "Payment pending" status

Alarm:
  - Alert fires when ApproximateAgeOfOldestMessage > 60 seconds
  - On-call engineer: "Workers down, restart them"
  - Workers come back
  - Backlog of messages processed (all confirmations sent)

SLA:
  - Worst case: 1 hour before messages expire from queue (14-day retention)
  - Typical case: 5 minutes downtime triggers alert, workers restarted
```

---

## Monitoring and Alerts

Set up CloudWatch monitoring:

```
Metrics to track:
  - ApproximateNumberOfMessagesVisible (should be < 100)
  - ApproximateAgeOfOldestMessage (should be < 30 seconds)
  - NumberOfMessagesSent (messages published)
  - NumberOfMessagesReceived (messages processed)
  - NumberOfMessagesDeleted (successful completions)
  - DLQ size (should be ~0 during normal operation)

Alarms:
  - ApproximateNumberOfMessagesVisible > 1000 (workers overloaded)
  - ApproximateAgeOfOldestMessage > 60 seconds (processing lag)
  - DLQ has messages (failure rate > 0)
  - Worker pool CPU > 80% (scale up)

Logs:
  - Each message processing logs entry/exit
  - Each payment call logs request/response
  - Failures logged to CloudWatch
```

---

## Summary: Why This Design Works

| Aspect                      | Why It Works                                           |
| --------------------------- | ------------------------------------------------------ |
| **Async processing**        | Keeps API responsive (80ms vs 800ms per booking)       |
| **SQS queue**               | Decouples payment from booking (payment can take time) |
| **Self-contained messages** | Worker doesn't depend on DB, faster processing         |
| **Idempotency keys**        | Prevents double-charging on retries                    |
| **Visibility timeout**      | Redelivers messages if worker crashes                  |
| **Max receive count**       | Sends persistent failures to DLQ (not forever)         |
| **Separate worker pool**    | Doesn't impact API performance                         |
| **Timeout handling**        | Queries payment status instead of retrying blindly     |
| **Email confirmations**     | Users know status even if they close browser           |

---

## Implementation Roadmap

**Phase 1 (Day 1):** SQS queue setup + basic worker
**Phase 2 (Day 2):** Error handling + DLQ monitoring  
**Phase 3 (Day 3):** Idempotency cache + status query logic
**Phase 4 (Day 4):** Load testing + autoscaling setup
**Phase 5 (Day 5+):** Production monitoring + oncall runbook
