Idempotency prevents the same billing operation from creating multiple charges.

A daily billing job may execute more than once because of retries or crashes. I create one durable billing operation per subscription and billing period. A unique constraint prevents duplicate operations, and the same operation is reused across retries. This makes the job retryable while ensuring that the business billing period is charged only once.

[AWS Idempotency and retries](https://docs.aws.amazon.com/durable-execution/patterns/best-practices/idempotency/) → understand the principle

[AWS Well-Architected](https://docs.aws.amazon.com/wellarchitected/latest/framework/rel_prevent_interaction_failure_idempotent.html) → explain implementation and tradeoffs

[Retries](https://docs.stripe.com/billing/revenue-recovery/smart-retries)
