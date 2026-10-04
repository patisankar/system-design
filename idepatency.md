Idempotency prevents the same billing operation from creating multiple charges.

I use a stable key based on the subscription and billing period, enforce uniqueness in the database, and pass the same key to the payment provider. 

If the service crashes and the payment result is unknown, reconciliation checks the provider before retrying. 

This prevents duplicate charges while allowing incomplete work to recover automatically.
```
Concurrent requests → database uniqueness constraint
Crash after provider success → provider status lookup
Retry days later → same billing-period idempotency key
```
