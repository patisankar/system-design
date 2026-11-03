# Financial System Design Interview: Deep Study Plan

Prepare from production case studies and practice defending the mechanisms behind them. Company descriptions below come from the linked sources; questions and exercises are proposed interview preparation.

## 1. Payment correctness: idempotency and uncertain outcomes

### Read

- [Stripe: Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency)
- [PayPal: Idempotency](https://developer.paypal.com/api/rest/reference/idempotency/) — API documentation that adds a concrete contract to the blog.

Stripe discusses failures before, during, and after request execution, including successful operations whose responses are lost. PayPal documents that retries can return the latest request status, retention depends on the API, and simultaneous requests with the same key can cause the second request to fail.

### Depth questions

- What identifies the business operation versus an individual network attempt?
- How do two concurrent requests atomically claim the same key?
- What happens when the same key arrives with a different amount?
- Where do you persist the operation, processor reference, and result?
- The processor charges successfully, then your process crashes before saving the result. What recovers it?
- What happens after the provider’s idempotency retention expires?
- Can you switch processors while the first processor’s outcome remains unknown?

### Exercise

Design the API, tables, unique constraints, and recovery flow. Walk through a crash at every boundary between database writes and the processor call.

## 2. Ledger correctness: more than debits equal credits

### Read

- [Uber: Zero-Sum by Design—10 Years of Uber’s Payments Platform](https://www.uber.com/mu/en/blog/ubers-payments-platform/)
- [Stripe: Ledger—Tracking and validating money movement](https://stripe.dev/blog/ledger-stripe-system-for-tracking-and-validating-money-movement)

Uber describes immutable money orders, zero-sum validation, strongly consistent balances, and entity changelogs. Stripe describes modeling producer workflows as movements between accounts, with data completeness checks and a data quality platform. These provide two perspectives: executing money movement and proving that financial activity is represented correctly.

### Depth questions

- Can a balanced ledger still contain an incorrect or duplicated payment?
- Which invariants prevent duplicate postings and unauthorized overdrafts?
- How do pending, available, and settled balances differ?
- What is atomic within one database transaction? What spans services?
- How do you represent partial capture, partial refund, fees, and reversals?
- How do you reconstruct yesterday’s balance after a late correction arrives?
- How do you distinguish a missing event from a legitimately delayed event?

### Exercise

Model a $100 payment, a $3 fee, and a later $20 partial refund. Show journal entries, balance effects, identifiers, and correction history. State your accounting assumptions explicitly.

## 3. Consistency: follow the read path

### Read

- [Uber: How LedgerStore Supports Trillions of Indexes](https://www.uber.com/se/en/blog/how-ledgerstore-supports-trillions-of-indexes/)

Uber gives a concrete financial consequence of stale indexes: a credit-card hold may be invisible to a subsequent lookup, potentially causing another charge while the hold remains. The post explains an index-intent protocol using two-phase commit to maintain consistency between records and indexes.

### Depth questions

- Is a strongly consistent primary record sufficient if the application discovers it through a stale index?
- What happens when an index intent succeeds but the record write fails?
- What does a reader do when it encounters an unresolved intent?
- Which reads require strong consistency: authorization lookup, available balance, payment history, analytics?
- How does stronger consistency affect latency and availability?

### Exercise

Draw the write and read protocols. Identify the durable state after each possible crash and explain how a reader resolves it.

## 4. Scaling: contention on one account

### Read

- The **Hot Entity Problem** section in [Uber’s payments platform article](https://www.uber.com/mu/en/blog/ubers-payments-platform/).

Uber describes serialized batch writes to address frequent mutations against the same ledger entity while preserving strong consistency. This is a useful case where adding partitions alone does not address contention on one logical account.

### Depth questions

- What limits throughput when many requests update the same balance?
- Would optimistic locking cause excessive retries?
- Can batching preserve the required operation ordering?
- How does the batch owner recover after crashing?
- How do you apply backpressure without starving quieter accounts?
- Can you divide the account safely, or would that weaken the balance invariant?

### Exercise

Compare row locking, optimistic concurrency, per-account serialization, and batching. Specify the workload under which you would choose each.

## 5. Event infrastructure: separate delivery from business correctness

### Read

- [PayPal: Scaling Kafka to Support PayPal’s Data Growth](https://medium.com/paypal-tech/scaling-kafka-to-support-paypals-data-growth-a0b4da420fab)

The accessible introduction describes Kafka’s use across tracking, database synchronization, risk, analytics, and compliance. The remainder is member-only: only the introduction was verified, not its detailed implementation.

The following are design exercises, rather than answers attributed to PayPal.

### Depth questions

- What partition key preserves the ordering your workflow needs?
- The consumer commits a database update and crashes before acknowledging the message. What happens next?
- How do you atomically record a financial update and an event to publish?
- How do you replay events without issuing another external payment?
- How do you distinguish event deduplication from business-operation deduplication?

### Exercise

Trace one payment through database commit, event publication, consumption, and webhook delivery. Mark every duplicate and loss window.

## 6. Migration: preserve correctness while changing the system

### Read

- [Stripe: Online migrations at scale](https://stripe.com/blog/online-migrations)

Stripe describes a subscription migration using staged dual writes, backfills, read comparisons with Ruby’s Scientist library, and incremental changes to write paths. This is particularly relevant to Ruby and subscription work.

### Depth questions

- What if one side of a dual write fails?
- Can a backfill overwrite a newer live update?
- How do you prove completeness and semantic equivalence?
- Which comparisons are safe to shadow, and which operations have financial side effects?
- What conditions permit cutover?
- After cutover, what makes rollback safe or unsafe?

### Exercise

Prepare a migration decision with rollout stages, correctness checks, stop conditions, and rollback boundaries.

## Eight-week study schedule

Keep 30-minute weekday sessions.

| Period | Focus | Required evidence |
| --- | --- | --- |
| Weeks 1–2 | Idempotency and payment recovery | API/schema design, concurrency walkthrough, crash-recovery matrix |
| Weeks 3–4 | Ledger and reconciliation | Worked money flows, explicit invariants, discrepancy investigation |
| Week 5 | Consistency and hot accounts | Read/write protocol and contention trade-off analysis |
| Week 6 | Events and replay | Publication/consumption failure matrix |
| Week 7 | Safe migration | Cutover and rollback decision |
| Week 8 | Integration | Two 45-minute mocks; revise weak answers |

## SMART goal

> Over eight weeks, I will complete six source-backed financial system design case studies and two timed mock interviews. For each case, I will explain the business invariant, data model, transaction boundary, failure recovery, rejected alternative, and operational verification. In the final mock, I will independently resolve at least four of five injected failure scenarios without violating the stated money invariants.

## How to study each article

Keep three distinct notes:

1. What the company explicitly says.
2. What you infer.
3. What you would choose for your interview requirements.

This prevents memorizing an architecture without understanding when it applies.

## First interview scenario

> The payment succeeds at the processor, the response is lost, and the customer retries. How does your design prevent a second charge and establish the final payment status?

Spend several sessions on this scenario: API contract, concurrency, persistence, recovery, and reconciliation.
