# QA Notes

Use this file during the live panel session to capture each question, a brief answer, and whether the answer felt complete.

## Session Notes

- Date:
- Presenter:
- Topic:

## Questions Asked

| # | Question | Brief Answer | Complete or Gap? | Follow-up |
|---|----------|--------------|------------------|-----------|
| 1 | What happens if Redis crashes mid-lock? | Redis is only a lock/optimization layer. If it crashes before the lock is stored, the request fails safely. If it crashes after lock acquisition, the DB check and seat versioning still prevent double-booking. Lock TTL and DB as source of truth keep the system safe. | Complete | Mention fallback if Redis is unavailable for a while. |
| 2 | At what concurrent users does the DB become the bottleneck? | With `SELECT ... FOR UPDATE`, the modeled ceiling is about 2,840 RPS before the 500-connection pool exhausts. That is far below the 25K RPS target, which is why the design avoids holding DB connections during waits. | Complete | Quote the 0.176X connection formula if needed. |
| 3 | What stops one user from holding 200 seats? | The API should enforce a per-booking seat cap and reserve seats with Redis `SETNX` plus a TTL. The DB only finalizes after the lock and version check, so a single user cannot bypass the limit without the app layer allowing it. | Partial | Call out whether the current UI/API already has a hard seat cap. |
| 4 | Your auto-scaling spikes your bill to $3,200 this month - what's your plan? | First, identify the driver: API RPS, Redis ops, DB writes, or worker retries. Then cap spend with scale policies, queue depth alarms, and reserved capacity for the steady baseline. If the spike is caused by abuse, apply ALB rate limits and CloudFront caching more aggressively. | Partial | Need a concrete monthly guardrail and alert threshold. |
| 5 | Why not PostgreSQL row locking instead of Redis? | Row locking is correct but it blocks and ties up connections during payment latency. At peak, that hits the pool limit long before 25K RPS. Redis `SETNX` moves contention out of the database hot path while DB remains the final source of truth. | Complete | Be ready to explain the fallback if Redis is degraded. |

## Quick Prompts

- Restate the question briefly.
- Acknowledge the scenario is real.
- Walk through the design response and name the component involved.
- State the current limit.
- Say what would change if the scenario happens.
