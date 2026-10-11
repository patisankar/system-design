# Reliable Webhook Delivery — Mock Interview Rehearsal

## Problem

Design a reliable webhook platform that sends payment events such as `authorized`, `settled`, `declined`, and `refunded` to merchant endpoints at approximately 10,000 events per second.

## Requirements and guarantees

- Durable delivery: do not permanently lose an accepted payment event.
- At-least-once delivery with idempotent processing.
- Ordering for events belonging to the same transaction or subscription.
- Eventual consistency; the initial attempt should be fast, while retries may continue longer.
- Exponential backoff with jitter for retryable failures.
- Merchant-visible delivery status and correlation ID; detailed internal logs stay private.
- Dead-letter handling, alerting, and authorized replay after the retry window is exhausted.

## High-level design

```text
Payment service
  -> payment DB + transactional outbox
  -> outbox relay
  -> Kafka, partitioned by ordering key
  -> webhook delivery workers
  -> merchant endpoint

Retry scheduler -> delivery workers
Failed terminal deliveries -> DLQ
Delivery metadata -> status API / internal operations
```

## Interview questions and confident answers

### 1. What delivery guarantee would you provide?

“I would provide at-least-once delivery with a stable event ID. Network failures make true end-to-end exactly-once delivery impractical, so both our consumers and the merchant must process the event idempotently.”

### 2. How would you preserve ordering without creating hot partitions?

“I would first clarify the scope of ordering. Assuming lifecycle events must be ordered per transaction, I would partition by `merchant_id + transaction_id`. That keeps `authorized`, `settled`, and `refunded` ordered for one transaction while allowing unrelated transactions to run in parallel. Merchant-wide ordering would serialize a high-volume merchant, so I would challenge that requirement.”

Do not use `merchant_id + date`: it can break ordering across day boundaries and still create a daily hotspot for a large merchant.

### 3. What happens when the merchant returns HTTP 503?

“Since 503 is retryable, I record the failed attempt and schedule exponential backoff with jitter—for example, around 1, 2, and 4 minutes. The worker does not sleep while waiting. Every retry uses the same event ID. A 2xx response marks the delivery successful; exceeding the retry window moves it to the DLQ, marks it failed, and raises an internal alert for investigation or replay.”

### 4. What if the merchant processed the request but the 200 response was lost?

“The outcome is unknown, so I retry with the same event ID. The merchant checks that ID in its idempotency store. If it already processed the event, it skips the business operation and returns success. This is duplicate delivery with effectively-once business processing.”

Important precision: an idempotency key does not prevent duplicate delivery; it prevents duplicate business effects.

### 5. What if the payment is committed but the process crashes before publishing to Kafka?

“I use the transactional outbox pattern. In the same database transaction, I update the payment and insert an outbox row. Either both commit or neither commits. A relay publishes unpublished rows to Kafka and marks them published only after Kafka acknowledges them. If Kafka is unavailable, the durable outbox row remains available for retry.”

### 6. What if the relay publishes to Kafka but crashes before marking the outbox row published?

“The relay may publish the same event again after restarting. I retain the stable event ID and make consumers idempotent with an inbox or deduplication store. The consumer may receive the event twice but applies its business effect only once. I prefer a safely ignored duplicate over a permanently lost payment event.”

### 7. How do you prevent one failed event from blocking unrelated events?

“Later events for the same ordering key wait so their order remains intact, but unrelated transaction keys continue concurrently. Failed attempts move into a delayed retry mechanism instead of holding a delivery worker or blocking the entire Kafka partition.”

### 8. How do you prevent a retry storm after a large merchant recovers?

“I combine exponential backoff with jitter, per-merchant rate limits, bounded concurrency, and a controlled ramp-up. A circuit breaker reduces calls while the endpoint is unhealthy. When it recovers, queued retries are released gradually rather than all at once. New traffic and retry traffic can also have separate capacity budgets so the backlog does not starve current events.”

### 9. What merchant information would you expose?

“I would expose a high-level delivery state, attempt count, timestamps, correlation ID, and a safe summarized error, plus authorized replay where appropriate. Detailed infrastructure logs and sensitive internal errors remain internal.”

## Thirty-second design summary

“I would commit each payment change and webhook event atomically using a transactional outbox. An outbox relay publishes events to Kafka using a stable event ID. Kafka is partitioned by the required ordering key, such as merchant plus transaction, which preserves transaction lifecycle order while enabling parallelism. Delivery workers use at-least-once delivery, idempotency, exponential backoff with jitter, per-merchant rate limits, and circuit breakers. Successful 2xx responses are marked delivered; exhausted retries go to a DLQ with alerting and controlled replay. Merchants receive a safe status view while detailed diagnostics remain internal.”

## Confidence framework

Use this sentence structure:

> “Because **failure or constraint**, I will **design decision**. If **success condition**, I will **success action**. Otherwise, I will **failure action**. The trade-off is **explicit trade-off**.”

Before answering, pause for three seconds and identify:

1. The guarantee or invariant.
2. The failure being handled.
3. The decision and its trade-off.

Avoid vague phrases such as “some kind of dedup.” Say “consumer-side deduplication using a stable event ID and an inbox table.”

## Next rehearsal

Answer each question aloud without reading the sample answer. Keep each response to 20–30 seconds, then finish with the thirty-second design summary.


