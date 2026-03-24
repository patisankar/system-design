## General mistakes

https://www.educative.io/interview-prep/system-design/system-design-interview-trap-why-engineers-fail-and-succeed

1. Inadequate understanding of distributed system fundamentals  

  Strong candidates reason through pressure points. They explain how cache eviction storms, aggressive retries, or misconfigured health checks can amplify load.
  
   Consistency, availability, and partition tolerance (CAP)

   Replication, quorum choices, and sharding strategies with their failure modes

   Strong vs. eventual consistency

   SQL vs. NoSQL trade-offs
     
2. Treating building blocks as opaque primitives

  Databases: Indexing strategies, replication lag, and connection pooling.

  Caches: Eviction policies, cache penetration, and the thundering herd problem.

  Load balancing: Traffic shaping algorithms and sticky sessions.

  Queues: Backpressure, dead-letter queues, and async processing.

3. Rushing to design without clarifying requirements

   Strong candidates ask diagnostic questions: Did user count increase? Are database latencies rising? Is the cache hit rate dropping?

   Responding quickly often signals impatience. Real engineering requires narrowing down the problem before concluding.

   Key takeaway: Never design without explicitly defining three categories of requirements:

      Functional requirements: What the system does (e.g., posting content, viewing feeds).
      
      Non-functional constraints: The numbers that shape the design (DAU, QPS, P99 latency targets).
      
      Out of scope: What is intentionally excluded to keep the design focused.

4. Weak trade-off articulation

    Every decision represents a trade-off. You must balance latency against throughput or consistency against availability. Merely stating a tool "scales" is not a justification.
