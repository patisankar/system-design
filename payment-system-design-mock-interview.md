# Payment System Design Mock Interview: Questions and Answers

Note: The source text contained only the interviewer's side. Answers are reconstructed from the interviewer's paraphrases. Items marked "Not stated" had no answer in the text.

## 1. Requirements and Scope

**Q1. What scale are we talking about (thousands, millions, more), and are users consumers, merchants, or both?**
A: Tens of millions of users, with both consumers and merchants.

**Q2. Any other requirements to clarify?**
A: High availability. Payments must work at any time.

**Q3. What other functional or non-functional requirements matter?**
A: Durability and consistency of transactions.

**Q4. Anything on security, scalability, or supported payment methods?**
A: Secure client-to-server communication.

**Q5. Support multiple payment methods (cards, bank transfers, wallets) or just one?**
A: Multiple methods are in scope.

**Q6. Anything else to confirm (regulatory, mobile/web, auditability)?**
A: Not stated.

## 2. High-Level Design and Flow

**Q7. Sketch the high-level architecture and walk through a typical payment.**
A: The request goes through the load balancer to the gateway. The gateway processes the payment and stores the outcome in the transaction history DB.

**Q8. What roles do the "cache" and "payment types" blocks play?**
A: The cache speeds up retrieval of a user's transaction history and reduces DB load. How "payment types" integrates with the gateway: Not stated.

**Q9. Any other major components needed for core payment processing?**
A: Not stated.

## 3. Security and APIs

**Q10. Where do you handle authentication and secure connections?**
A: Mutual TLS secures connections, and OAuth handles authentication and authorization.

**Q11. Are these checks at the gateway or elsewhere?**
A: Browser-to-gateway traffic uses mTLS, and the gateway manages OAuth.

**Q12. Which API style fits best (REST, gRPC, other), and why?**
A: External clients use GraphQL or REST. Internal service-to-service calls use gRPC for efficiency.

## 4. Database and Consistency

**Q13. Which database for transaction history, and why?**
A: A relational database such as Postgres, with an archiving strategy for long-term retention.

**Q14. What schema and key entities?**
A: Tables for transactions (amount, payment method, user reference), users, and payment methods.

**Q15. Store balances as a field or compute on the fly?**
A: Persist balances and update them as transactions are processed.

**Q16. How do you keep balances consistent and prevent races when two payments arrive at once?**
A: Use locks so only one transaction updates a user's balance at a time.

**Q17. Lock at the application level, the database level, or both?**
A: Row-level locks in the database.

**Q18. What are the trade-offs as activity grows?**
A: Locking increases contention and latency under high load.

## 5. Scaling

**Q19. How do you mitigate those scalability challenges?**
A: Shard the transactions table by user.

**Q20. How do you choose a shard key, and how do you migrate users between shards?**
A: Use the user ID, possibly with some randomness. Migrate by grouping users together.

**Q21. What if a shard becomes hot because of a few very active users?**
A: Dynamically split or redistribute heavy users across more shards.

## 6. Caching

**Q22. What caching strategy and invalidation approach for transaction history?**
A: Set a time-to-live (TTL) on cache entries so they expire and refresh.

**Q23. What problems arise if a user expects to see a new transaction immediately?**
A: Use a write-through cache, updating the cache and the DB together on writes.

**Q24. What are the downsides of write-through?**
A: It adds extra load to the database.

**Q25. Would you consider write-back or lazy invalidation instead?**
A: Not answered directly. The follow-up suggests the candidate saw potential in write-back to reduce immediate DB writes, at some durability risk.

**Q26. With write-back, how do you avoid data loss if a cache node fails before syncing to the DB?**
A: Replicate cache data across multiple nodes.

**Q27. Rely on in-memory replication alone, or add persistent storage?**
A: Combine in-memory replication with persistent storage for stronger guarantees.

## 7. External Processors and Banks

**Q28. How do you design interactions with external processors or banks for reliability and security?**
A: Not stated. The candidate struggled with this one.

**Q29. How do you make sure a bank request isn't lost or duplicated during a network issue?**
A: Use idempotency so duplicate requests don't cause double charges.

**Q30. How do you implement idempotency for external payment calls?**
A: Generate an idempotency key per transaction and process it only once per key.

**Q31. Where do you store and check the keys?**
A: A dedicated table tracking idempotency keys and their processing status.

**Q32. If a downstream bank is slow or intermittently down, how do you handle retries without blocking the main flow?**
A: Use latency budgets and circuit breakers to stay responsive and keep fallback options.

**Q33. If a transaction still fails after retries, what do you tell the user, and how do you keep data consistent?**
A: Record the failure in the transaction history. Proactively notify merchants by webhook or email.

## 8. Surge Handling

**Q34. How does the design handle a sudden spike in payment volume?**
A: Scale out servers and allocate resources to the critical paths.

**Q35. What bottlenecks could still limit throughput under extreme load?**
A: Not clearly answered. The interviewer moved on to rate limiting.

**Q36. Where would you place rate limiting, and per user, per merchant, or globally?**
A: Apply limits globally and at the merchant level, covering both overall load and targeted abuse.

## 9. Regional Failure

**Q37. If the primary database region goes down, how does the system keep processing and stay consistent?**
A: Promote a replica to primary if the master fails.

**Q38. How do you handle in-flight or partially processed transactions at failover?**
A: Not answered in the text.

---

## Suggested Answers for the Gaps

- **Q28 (external processors):** Use tokenized or hosted integrations so raw card data stays out of scope. Use mTLS and signed requests to the processor. Add idempotency keys, timeouts, retries with backoff, and full request and response audit logs.
- **Q35 (bottlenecks):** The single-writer database, hot shards, lock contention on hot accounts, and the external processors' own rate limits.
- **Q38 (failover in-flight):** Model each payment as a state machine (created, pending, authorized, captured, failed) with durable idempotency keys. After failover, run a reconciliation job against processor and bank records. Resume or reverse any transactions stuck in a pending state, and use synchronous or semi-synchronous replication to minimize lost writes.
