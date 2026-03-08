Webhooks being dropped due to the legacy rate limit
==

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

The core architectural issue
We modeled:

error → increment failure_count → backoff → give up after N

Instead of:

```
if internal contention:
    delay and retry (does not count toward delivery exhaustion)

if merchant delivery failure:
    increment delivery retry count
    apply bounded retry policy
    eventually DLQ
```

Advise:
======
Workers must only perform useful delivery work; delay logic must not consume execution slots.

Merchant concurrency caps remain in place to prevent downstream overload.

DLQ applies only to delivery failures, never to internal lock contention.

Retry semantics must distinguish between admission failures and delivery failures to preserve correctness.

Approach
======

## Immediate Fix 

## Before

When a webhook job executed:

1. Attempt delivery.
2. If lock/token unavailable → retry logic.
3. If merchant HTTP failure → retry logic.
4. Both failure types incremented `failure_count`.
5. After ~15 attempts (~23 hours), job stopped retrying.

### Problems

- Workers could sleep or loop during lock contention.
- Internal lock failures counted toward retry exhaustion.
- Retry logic was duplicated and hard to reason about.
- System recovery under load was inefficient.


## After 

### 1. Enforced Concurrency Cap (Token-Based Semaphore)

Before delivering:

```pseudo
if token_acquired:
    deliver_webhook()
else:
    reenqueue_with_delay()
    return
```

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
