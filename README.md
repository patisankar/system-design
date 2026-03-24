# Minimum Skills
https://www.allthingsdistributed.com/2016/03/10-lessons-from-10-years-of-aws.html

## 1. Core Distributed Systems (Non-negotiable)
- Consistency models
  - Strong vs eventual consistency
  - Read-your-writes, monotonic reads
- Idempotency (critical for payments, retries, webhooks)
- Concurrency control
  - Optimistic vs pessimistic locking
- Failure handling
  - Retries, exponential backoff, circuit breakers
- Data partitioning
  - Sharding (user-based, account-based)

**Key Interview Question:**
> How do you prevent double charges during retries and failures?

---

## 2. Transaction & Ledger Design (Fintech Core)
- Double-entry bookkeeping (debit/credit invariants)
- Ledger vs balance model (append-only ledger preferred)
- Immutability (no updates, only compensating entries)
- Reconciliation systems (internal vs external)

**Key Concepts:**
- Atomicity across accounts
- Auditability (who, when, why)
- Time-based correctness

---

## 3. API & Workflow Design
- Idempotent APIs (idempotency keys)
- State machines (payment lifecycle)
  - initiated → authorized → captured → settled
- Async workflows (webhooks, event-driven systems)
- Delivery semantics
  - At-least-once + idempotency (practical approach)

---

## 4. Data Layer Expertise
- Strong relational DB knowledge (PostgreSQL/MySQL)
  - ACID transactions
  - Isolation levels (SERIALIZABLE, READ COMMITTED)
- Indexing & query performance
- Schema design for financial data

**Advanced (Expected for Senior Roles):**
- Event sourcing
- CQRS

---

## 5. Security & Compliance Awareness
- PCI-DSS basics
- Encryption
  - At rest
  - In transit
- PII handling
- Authentication/Authorization (OAuth, JWT)

**Must Know:**
- Avoid sensitive data in logs
- Tokenization basics

---

## 6. Fault Tolerance & Reliability
- Exactly-once effect (via idempotency + deduplication)
- Safe retries
- Dead-letter queues
- Monitoring & alerting

---

## 7. Event-Driven Architecture
- Kafka / pub-sub systems
- Event versioning
- Replay capability
- Handling out-of-order events

---

## 8. Observability & Debugging
- Structured logging
- Distributed tracing
- Audit trails

---

## 9. Product Thinking (Critical for Senior Roles)
- Balance correctness vs latency
- Understand customer impact
- Trade-offs (speed vs safety)

**Example:**
> Should checkout prioritize speed or strict consistency?

---

## 10. Must-Be-Able-To-Design Systems
- Payment processing system (Stripe/PayPal-like)
- Wallet system
- Ledger system
- Refund & dispute system
- Reconciliation system
- Fraud detection / rate limiting

---

## Expectations
- Ask strong clarifying questions
- Define invariants
  - "No money creation or loss"
- Design for failures first
- Use precise terminology

---

### additional

Idempotency keys for financial operations
[Stripe Doc](https://docs.stripe.com/billing/revenue-recovery/recovery-analytics)

[GraphQL](https://relay.dev/docs/)

Audit trails

PII encryption

Rate limiting partner APIs

Retry with exponential backoff

Circuit breakers for bank integrations

SLA monitoring
