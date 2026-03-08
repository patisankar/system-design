webhooks being dropped due to the legacy rate limit
======

Webhooks were being dropped under congestion due to legacy rate limiting and retry behavior.

Two core issues:

1. **Internal contention (lock/token acquisition failures)** was treated the same as **merchant delivery failures**.
2. A single `failure_count` controlled retries for both, meaning:
   - After ~23 hours / 15 attempts, we “gave up”.
   - Lock contention could cause permanent webhook loss.
   - Retry logic was duplicated and difficult to reason about.

The system lacked a clear separation between:
- **Admission control** (internal capacity constraints)
- **Delivery failure** (external merchant/system issues)


Advise:
======
Workers must only perform useful delivery work; delay logic must not consume execution slots.

Merchant concurrency caps remain in place to prevent downstream overload.

DLQ applies only to delivery failures, never to internal lock contention.

Retry semantics must distinguish between admission failures and delivery failures to preserve correctness.

Approach
======

## Immediate Fix (Current PR)

**Goal:** Stabilize behavior without introducing risky schema changes.

### Changes

1. **Keep the existing job schema**
   - No new required fields (rollback-safe).
   - No new job type introduced.

2. **Maintain per-merchant concurrency limits**
   - Token-based semaphore around delivery.
   - Prevents overwhelming merchants.
   - No worker sleeping; retries re-enqueue with delay.

3. **Retain current `failure_count` cutoff (~23h / 15 attempts)**
   - Applies to both failure types for now.
   - On exhaustion:
     - Non-paging Sentry alert
     - DogStats metric emitted
     - Datadog monitor (to be terraformed) to page Webhooks/IPS

4. **Refactor retry logic**
   - Simplify structure.
   - Reduce duplicated retry blocks.
   - Clarify failure handling paths.

### Result

- Reduced webhook drops under congestion.
- Controlled merchant concurrency.
- Improved observability.
- Safe rollback (no schema incompatibility).

This addresses the immediate instability while minimizing deployment risk.

---

## Follow-Up Plan (Correct Long-Term Semantics)

To fully align behavior with intended failure semantics:

1. **Separate failure tracking**
   - Introduce `rate_limiting_retry_count` (or similar).
   - Keep `delivery_retry_count` separate.

2. **Different retry policies**
   - **Lock failures (internal contention):**
     - Short jittered delay.
     - No DLQ.
   - **Delivery failures (merchant-facing):**
     - Capped exponential backoff.
     - DLQ after max attempts or age threshold.

3. **Unified metrics with tags**
   - Use `failure_type=lock|delivery`.
   - Separate monitors if needed (same metric family).

4. **Schema cleanup**
   - Remove deprecated fields once migration is complete.

---

## Summary

We stabilize webhook delivery by enforcing controlled concurrency and structured retries now (without risky schema changes), and follow up by cleanly separating internal contention from true delivery failures so only genuine merchant errors can exhaust retries.
