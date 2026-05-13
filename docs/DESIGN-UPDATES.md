## Post-Roast Design Updates

## Update 1: Per-user hold cap

**Triggered by:** Panel Question 3 - What stops one user from holding 200 seats?

**What changed:**
Added a Redis counter key, `holds:{userId}:count`, that increments when a seat hold is created and decrements when the hold expires or converts into a booking. The booking API now rejects new holds once a user reaches 8 concurrent active holds.

**Why this is necessary:**
The original design prevented double-booking, but it did not prevent a single user or scripted client from consuming an excessive share of inventory. The counter adds a hard application-level limit so a user cannot monopolize seats even if they can still obtain individual `SETNX` locks.

**What it costs:**
One extra Redis read/write pair on hold creation and release, plus a small amount of logic in the booking API and cleanup job. It also introduces the need to keep the counter in sync with expiry and rollback paths.

**What it still doesn't solve:**
It does not stop a botnet of many accounts from distributing holds across users, and it still depends on the hold-release path staying correct under retries and worker failures.

## Update 2: Queue depth and spend guardrails

**Triggered by:** Panel Question 4 - Your auto-scaling spikes your bill to $3,200 this month - what's your plan?

**What changed:**
Added CloudWatch alarms on SQS queue depth, worker retry rate, and monthly spend forecast. When queue depth crosses a defined threshold, the ECS worker service scales out first; if spend forecast exceeds the monthly cap, the API enters a protected mode that keeps booking flows alive but disables non-essential cache warmups and lowers the CloudFront TTL for lower-priority pages.

**Why this is necessary:**
The original design could scale technically, but it did not include a cost-control path if a traffic spike, retry storm, or abusive pattern pushed the bill above budget. These guardrails give us a way to preserve the core booking path while cutting non-essential load before overspend becomes a surprise.

**What it costs:**
Extra operational overhead from alarm tuning, dashboarding, and a small amount of policy logic in the API and deployment pipeline. The protected mode can also reduce cache efficiency and increase origin traffic during a budget event.

**What it still doesn't solve:**
It does not eliminate the underlying traffic spike, and if the spike is caused by legitimate demand we may still need to accept degraded non-critical behavior until the month resets or capacity is increased.
