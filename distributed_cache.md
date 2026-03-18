# overview
critical building blocks of cache (cache client, key-to-node hashing, eviction internals, invalidation guarantees)
## skils
Problem Scoping and Requirements Gathering
Core Architectural Design
Analytical and Strategic Skills
Data Model and Database Design
System Optimization
Effective Communication and Articulation

### functional requirement
Main users are internal and end-user applications, 2millios per hour

### Non functional
We need high availablility and stron consistency , with strict security, low latency is high prioirty

### technical
Client → App → Cache → DB.

proposing both a global cache and a local cache per server, with background jobs updating the global cache.

So you’re trading off strong consistency for eventual consistency between global and local caches.

Given you mentioned strong consistency earlier, how would you handle cases where different servers see stale data in their local caches? Would that be acceptable for your use case?
You’d use TTLs to limit staleness in local caches.

How would you handle cache invalidation or updates when there’s a write to the underlying database? What ensures that caches don’t serve outdated data after a write?

So after a write, you’d refresh the relevant cache entries.

Would you propagate that refresh to just the global cache, or do you have a mechanism to notify and update all local caches as well?

Got it—writes trigger an update to the global cache, which then reflects the new value.

How do local caches become aware of this update to avoid serving stale data, especially before their TTL expires?

So, when the global cache updates, it pushes the change out to all local caches.

What mechanism would you use for this fanout—are you imagining a pub/sub model, direct notifications, or something else?

You’d use a pub/sub model so local caches subscribe to updates from the global cache.

What technology or system would you consider for implementing this pub/sub—something like Redis Pub/Sub, Kafka, or another approach?

You’d use replicas for high availability.

How would you handle consistency between the Redis primary and its replicas, especially with strong consistency as a requirement?

So you’d use a leader-slave (primary-replica) setup for Redis.

Are you comfortable with the potential for eventual consistency between replicas, or do you require synchronous replication to meet your strong consistency goal?

You’d opt for synchronous replication to maintain strong consistency.

Let’s talk about sharding—how would you distribute data across multiple Redis nodes to scale as your user base grows?

ou’d use a load balancer to distribute requests across Redis nodes.

How would the load balancer determine which Redis node to route a particular cache key to, so that reads and writes for the same key always go to the same node?

So you’d use sticky sessions to ensure requests for the same key are routed consistently.

Could you elaborate on how sticky sessions would map cache keys to nodes in a scalable way, especially as nodes are added or removed?

Consistent hashing would help with flexible key-to-node mapping as your cluster scales.

Let’s focus on security for a moment—how would you secure data in transit and at rest within your distributed cache system?

You’d use public/private key encryption for securing data.

On the API side, what kind of interface would you expose for apps to interact with the cache—would you opt for REST, gRPC, or another approach? Why?

So the cache is tightly integrated with the server, and there isn’t a separate API layer exposed to clients.

Let’s talk about the data model: what kind of data structures would you store in the cache, and how would you handle serialization or deserialization?

You’d use hash-based structures, with serialization and deserialization handled as needed.

Thinking about cache eviction: which policy would you choose (e.g., LRU, LFU), and what’s your reasoning for that choice?

You’d go with LRU to evict the least recently used items.

How would you tune the cache size and TTL values to balance memory usage and cache hit ratio?

So you’d set TTL per entry and allow dynamic adjustment of cache size.

If you notice certain keys are accessed much more frequently than others—hot keys—how would your design handle potential bottlenecks or load spikes on those cache entries?

You’d consider LFU for hot keys to better manage frequently accessed data.

How would you detect and route requests for these hot keys differently—would you shard them, replicate them, or use another approach?

Thanks for updating the diagram to show the primary cache, replicas, and their relationship.

Where in this architecture would you place the sharding logic and the pub/sub mechanism for cache update notifications?

### Feedback
# System Design Feedback Summary

| Section | Rating | Positives | Areas To Improve |
|--------|--------|-----------|------------------|
| **Problem Identification** | Average | Identified low latency as a top priority. Mentioned high availability, consistency, and security. | Functional requirements were not clearly listed (GET/SET + eviction). Non-functional requirements were not structured. Introduced conflicting consistency requirements (strong vs eventual). Missing capacity planning such as QPS, item size, memory estimates, and hit rate targets. |
| **High-Level Design** | Below Average | Proposed replication and eventually mentioned consistent hashing. Suggested a pub/sub model (e.g., Redis) for propagating updates. | Architecture incomplete for distributed cache. Missing cache client/library on app servers, consistent hashing ring for node selection, and defined eviction subsystem per node. Started with load balancer and sticky sessions instead of consistent hashing. Local + global cache introduced coherence complexity without defined invalidation guarantees. Diagram implied cache directly querying DB instead of cache-aside pattern. |
| **Detailed Design** | Unsatisfactory | Mentioned TTL-based expiration. Recognized need for serialization/deserialization and hash-based storage. | Missing internal data structure details (hash map + doubly linked list for LRU, frequency buckets for LFU). Eviction policy inconsistent (LRU then LFU without explanation). Hot-key mitigation incorrect ("shard them"). Consistency model vague (no write-through/write-back definition). Security explanation weak (no TLS/mTLS, ACLs, auth, encryption). API/interface not defined (protocol, retries, client behavior). |
| **Trade-offs & Improvements** | Below Average | Attempted to address HA via replication and staleness via TTL. Recognized scaling needs consistent hashing. | Trade-offs not deeply analyzed. Flipped between strong consistency, eventual consistency, and synchronous replicas without discussing latency/availability impacts. Pub/sub design lacked discussion of message loss, replay, consumer lag, or reconciliation. Failure modes not discussed (node loss, resharding, cache stampede, cold start, replica lag). |

