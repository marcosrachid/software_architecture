## Architecture & System Design Q&A

### 1. SOLID & Clean Code

**Q1. What practical benefits do SOLID principles bring to a real-world codebase?**  
**A.** SOLID reduces coupling, makes changes cheaper, and improves testability. In practice, following SOLID means smaller, focused classes, fewer side effects when changing behavior, and the ability to test logic without heavy infrastructure. Interviewers expect you to explain design decisions (how you split responsibilities, how you extend behavior) rather than just reciting the acronym.

**Q2. How would you recognize a violation of the Single Responsibility Principle (SRP) in a typical Spring service, and how would you fix it?**  
**A.** A typical SRP violation is a huge `UserService` or `OrderService` that validates requests, talks to the DB, sends emails, calls external APIs, and publishes events. This makes the class hard to test and easy to break. The fix is to extract separate components (e.g. `EmailSender`, `FraudChecker`, `PaymentProcessor`) and inject them, so each has a single clear reason to change.

**Q3. What is cyclomatic complexity, and how do you keep it under control?**  
**A.** Cyclomatic complexity measures the number of independent logical paths (driven by `if`, `for`, `while`, `switch`, etc.). High complexity makes code hard to test and prone to bugs. To keep it under control, you extract smaller methods, apply patterns such as Strategy or Chain of Responsibility, and replace long `if`/`switch` chains with polymorphism or handler maps.

**Q4. How do you apply “Clean Code” principles during incremental refactoring in a legacy system?**  
**A.** Use small, safe steps with tests. Introduce new abstractions (e.g. a `PriceCalculator`) next to existing code, migrate callers gradually, and separate refactoring-only commits from functional changes. The goal is to improve readability and design without changing behavior in large, risky chunks.

---

### 2. ACID, BASE, and Distributed Consistency

**Q5. What does ACID guarantee, and what does it _not_ guarantee?**  
**A.** ACID guarantees Atomicity, Consistency, Isolation, and Durability for transactions within a database. It focuses on correctness of data and transactional behavior, not on availability or scalability. An ACID-compliant DB can still be unavailable or non-scalable if not architected properly.

**Q6. In a microservices environment using Kafka, where do ACID and BASE typically apply?**  
**A.** ACID usually applies within a single service’s local database transaction (e.g. writing business data and an outbox record atomically). BASE applies across services and events: Kafka consumers process events at different times, projections and caches are eventually consistent, and cross-service workflows often use sagas and compensating actions rather than global transactions.

**Q7. How would you scale reads and writes in an ACID database while preserving strong consistency where needed?**  
**A.** For reads, use read replicas, caching (Redis/Memcached), and possibly sharding by tenant/region/key. For writes, first optimize and scale up (indexes, query tuning, shorter transactions), then apply sharding so ACID holds per shard and cross-shard transactions are rare. For global strong consistency at large scale, use distributed SQL/NewSQL (e.g. Spanner-like systems) and accept higher latency and stricter CAP trade-offs.

**Q8. Explain sharding with a concrete example in a SaaS system.**  
**A.** In a multi-tenant SaaS, you might choose `tenant_id` as shard key and create several physical databases (`users_shard_0`..`users_shard_3`). Requests resolve the `tenant_id`, compute `shard_index = hash(tenant_id) % 4`, and route to the correct DB. Each shard is ACID on its own; global operations across tenants (like reporting) are aggregated from multiple shards rather than done in a single cross-shard transaction.

---

### 3. CAP Theorem and Database Choices

**Q9. Under the CAP theorem, what is the main trade-off systems face during network partitions?**  
**A.** During network partitions, a distributed system must choose between consistency (all nodes see the same data or errors) and availability (every request receives a response). You cannot have both full availability and strong global consistency in the presence of partitions; you lean either towards CP (strong consistency, limited availability) or AP (high availability, eventual consistency).

**Q10. Give examples of CA, CP, and AP systems and explain why they fit each category.**  
**A.**

- CA: A single-node relational database (one Postgres/MySQL instance) is consistent and available as long as it is up, but not partition-tolerant because there are no replicas to partition.
- CP: Systems like ZooKeeper/etcd keep a single correct view of state using majority quorum; minority partitions reject or limit requests, favoring consistency and partition tolerance over availability.
- AP: Dynamo-style key-value stores or Cassandra remain available on all reachable nodes during partitions, accepting temporary divergence and reconciling later—prioritizing availability and partition tolerance over strong consistency.

**Q11. From a business perspective, when would you choose CP vs AP?**  
**A.** Choose CP when wrong data is worse than downtime—e.g. bank balances, payment status, government registries, critical configuration. It is better to return errors than to show or accept inconsistent state. Choose AP when user experience and continuity matter more than perfectly up-to-date data—e.g. social feeds, counters, analytics dashboards, recommendations—where slight staleness is acceptable if the system remains responsive.

---

### 4. Hexagonal Architecture & DDD

**Q12. What problem does Hexagonal Architecture solve in a typical service?**  
**A.** It keeps business logic at the center, decoupled from infrastructure and frameworks. By separating ports (interfaces representing use cases and external dependencies) from adapters (concrete implementations like REST controllers, DB repositories, Kafka clients), it makes the service easier to test, easier to evolve (e.g. change DB or protocol), and safer to integrate with external systems.

**Q13. In the example order service, what would be typical ports and adapters?**  
**A.** Ports (in the domain): `OrderRepository`, `InventoryPort`, `PaymentPort`, `EventPublisherPort`. Inbound adapters: REST controller for `POST /orders`, Kafka consumer for `PaymentConfirmed` events, potentially a CLI or batch job. Outbound adapters: a JPA-based `OrderRepository`, an HTTP client implementing `InventoryPort`, a payment gateway client implementing `PaymentPort`, and a Kafka publisher implementing `EventPublisherPort`.

**Q14. How does DDD relate to microservice boundaries?**  
**A.** DDD defines **Bounded Contexts**—coherent areas of the domain with their own models and language (e.g. `Billing`, `Catalog`, `Shipping`). These contexts are natural candidates for microservice boundaries: each microservice ideally owns a single bounded context (or a small number of closely related ones), keeping models focused and avoiding shared, ambiguous “god models” across the system.

**Q15. When is DDD overkill, and when is it essential?**  
**A.** DDD is essential in complex, evolving domains with rich business rules—fintech, logistics, healthcare, marketplaces, large B2B SaaS—where naive CRUD designs quickly become unmanageable. It is usually overkill for very simple CRUD systems, admin panels, or small internal tools with limited logic, where the overhead of aggregates, bounded contexts, and ubiquitous language does not pay off.

---

### 5. REST, GraphQL, and API Design

**Q16. What does idempotency mean in HTTP APIs and why is it important?**  
**A.** Idempotency means that repeating the same request leads to the same final state. For HTTP, `GET`, `PUT`, and `DELETE` should be idempotent, while `POST` usually is not. In distributed systems, idempotency is critical to safely handle retries and at-least-once delivery; you often use idempotency keys, optimistic concurrency (ETags), or natural business keys to avoid creating duplicates.

**Q17. When would you choose GraphQL over REST, and when would you avoid it?**  
**A.** Choose GraphQL when clients have diverse and evolving data needs, over/under-fetching is a real pain, and you need a single, strongly-typed entry point that can federate multiple backends. Avoid GraphQL for very simple APIs with predictable calls and strong HTTP caching, or when the team cannot yet handle the extra complexity around schema design, gateway performance, and security.

**Q18. What common performance issue does GraphQL introduce, and how can you mitigate it?**  
**A.** The classic issue is the N+1 problem—one query to fetch a list and then one query per list item. You mitigate it with batching (e.g. DataLoader), resolvers that support bulk fetching by IDs, and resolver-level caching so repeated access to the same data doesn’t trigger repeated backend calls.

---

### 6. System Design & Resilience

**Q19. How do you decide between vertical and horizontal scaling for a service?**  
**A.** Vertical scaling (bigger machines) is simple but limited and often requires downtime; it is good for quick wins and smaller systems. Horizontal scaling (more instances) supports much larger growth but requires stateless services, load balancing, and often data partitioning. In practice, you scale vertically until diminishing returns and then design for horizontal scaling.

**Q20. What is the role of a load balancer in a modern architecture?**  
**A.** A load balancer distributes incoming traffic across instances, using strategies like round-robin, least connections, or consistent hashing by user/session. It handles health checks, failover, path/host routing, and TLS termination, improving both availability and scalability of stateless services.

**Q21. Why are stateless services recommended for horizontal scaling?**  
**A.** Stateless services do not store user/session state in memory between requests, so any instance can serve any request. State lives in tokens, shared caches (e.g. Redis), or databases. This makes it trivial to add/remove instances behind a load balancer and perform rolling deployments without losing user sessions.

**Q22. Explain the difference between circuit breakers and retries in resilient systems.**  
**A.** Retries attempt to recover from transient failures by trying again, often with exponential backoff. Circuit breakers protect systems from persistent failures: they stop sending requests to a failing dependency after a threshold, enter an “open” state, and only probe again occasionally in a “half-open” state. Together, they prevent cascading failures and reduce pressure on unhealthy services.

---

### 7. API Gateway, BFF, and Microservices Anti-Patterns

**Q23. What are the main responsibilities of an API Gateway in a microservices architecture?**  
**A.** An API Gateway is the central entry point for external traffic. It handles routing to services, TLS termination, authentication and authorization, rate limiting, logging, and request/response transformation. It should expose stable, generic APIs that can be reused by multiple clients (web, mobile, partners).

**Q24. How does a Backend for Frontend (BFF) complement an API Gateway?**  
**A.** A BFF is a thin backend dedicated to a specific client experience (e.g. web, mobile). It aggregates data from multiple services, shapes payloads to fit each UI, and hides chatty interactions from the frontend. The gateway focuses on cross-cutting concerns; BFFs focus on client-specific composition and UX needs.

**Q25. Name some common microservices anti-patterns and why they are problematic.**  
**A.**

- Distributed monolith: services are deployed separately but tightly coupled, making changes and deployments as painful as in a monolith—without its benefits.
- Chatty services: too many synchronous calls between services increase latency and fragility.
- Shared database: multiple services writing to the same DB break autonomy, coupling schemas and deployments.
- Premature microservices: splitting too early adds complexity and operational overhead without clear domain boundaries or benefits.

---

### 8. Launch Strategy, Feature Flags, and Observability

**Q26. Why is “deploy != release” an important concept in large-scale systems?**  
**A.** Separating deploy from release lets you ship code dark and then control exposure with feature flags/deciders. This enables gradual rollouts, quick mitigation (kill switches), and safer experimentation without tying business changes to deployment cycles.

**Q27. How do deciders (feature flag services) and managed lists help you launch features safely?**  
**A.** Deciders provide dynamic, runtime decisions about whether a feature is enabled for a given user/context, supporting percentage rollouts, segment targeting, and fast rollbacks. Managed lists (allow/deny/target lists) let ops and product teams choose exactly which tenants/users are included in or excluded from a rollout, without redeploying code.

**Q28. What observability practices are essential when rolling out a risky change?**  
**A.** You need structured logs, metrics tied to SLIs/SLOs (latency, error rates, saturation), distributed tracing with correlation IDs, and dashboards that compare “old vs new” paths. Alerts must be wired to these metrics, and runbooks should describe how to roll back or flip flags off quickly if degradation is detected.

---

### 9. Kafka, Messaging, and Exactly-Once Semantics

**Q29. Why is “exactly-once” delivery hard to achieve in distributed systems, and how is it approximated in practice with Kafka?**  
**A.** Exactly-once is hard because networks, brokers, and consumers can all fail or retry, making duplicates or lost messages possible. In practice with Kafka, you approximate exactly-once by combining at-least-once delivery with idempotent producers and consumers that de-duplicate based on keys or identifiers, plus transactional writes (e.g. outbox + idempotent processing).

**Q30. What is the outbox pattern, and how does it help with consistency between a service’s database and Kafka?**  
**A.** The outbox pattern writes both business data and an “outbox” event row in the same local ACID transaction. A separate process reads the outbox table and publishes to Kafka. This avoids the classic “write DB, then fail before producing the event” problem and ensures that either both DB and event are persisted or neither is.

**Q31. How do consumer groups and partitions contribute to scalability and ordering in Kafka?**  
**A.** Partitions allow parallelism by letting multiple consumers in a group process different subsets of messages; each partition is processed by one consumer in the group. Ordering is guaranteed per partition, not globally. Consumer groups ensure that each message is processed by exactly one consumer instance for a given group.

---

### 10. Redis, Caching, and Rate Limiting

**Q32. Compare cache-aside and write-through caching strategies. When would you use each?**  
**A.** Cache-aside reads from cache first, then falls back to DB on a miss and writes the result to cache; writes go to DB and optionally invalidate or update cache. It is simple and widely used for read-heavy workloads. Write-through writes data to both cache and DB synchronously on every update, keeping cache always fresh but increasing write latency; it is useful when you need very fresh cached data and can afford the extra write cost.

**Q33. What is a cache stampede, and how can you mitigate it?**  
**A.** A cache stampede happens when a popular key expires and many requests simultaneously miss the cache and hit the database, causing a thundering herd. Mitigations include jittered TTLs, request coalescing (single-flight), background refreshes, and using “soft TTL” with stale-while-revalidate patterns.

**Q34. How does Redis help with rate limiting, and what algorithms are commonly used?**  
**A.** Redis can maintain counters and timestamps per IP/user/API key to enforce limits in time windows. Common algorithms are token bucket, leaky bucket, fixed window, and sliding window. Using Redis ensures the rate limiting state is shared across multiple API instances.

---

### 11. Kubernetes, Deployments, and CI/CD

**Q35. What is the difference between liveness and readiness probes in Kubernetes, and why do they matter?**  
**A.** Liveness probes determine if a container is still alive; if it fails, Kubernetes restarts the container. Readiness probes determine if a container is ready to receive traffic; if it fails, the pod is removed from service endpoints but not restarted. Together, they help avoid sending traffic to unhealthy instances and automatically recover from crashes or deadlocks.

**Q36. How do blue/green and canary deployments differ?**  
**A.** Blue/green uses two production environments (blue and green); you deploy to the idle one and then switch all traffic at once. Canary deployments gradually send a small percentage of traffic to the new version, increasing it over time while monitoring metrics. Blue/green simplifies rollback by switching back; canary reduces risk by exposing only a fraction of users to new code initially.

**Q37. Why is trunk-based development often favored in modern CI/CD pipelines?**  
**A.** Trunk-based development keeps one main integration branch with small, frequent merges. It reduces long-lived branches, merge conflicts, and integration pain. Combined with feature flags, it supports continuous delivery, where code is always in a releasable state and changes are smaller and easier to reason about.

---

### 12. Security, Tokens, and Zero Trust

**Q38. What are the key differences between OAuth2 and JWT in an API ecosystem?**  
**A.** OAuth2 is an authorization framework describing flows for obtaining tokens (e.g. authorization code, client credentials). JWT (JSON Web Token) is a token format that can carry claims and be signed. OAuth2 may use JWT as a token type, but they are not the same concept: OAuth2 = “how you obtain and use tokens”; JWT = “how a token is encoded and verified”.

**Q39. Why is token expiration important, and how do refresh tokens help?**  
**A.** Token expiration limits the impact of token leakage and forces periodic re-validation of user/session state. Refresh tokens allow clients to obtain new access tokens without re-authenticating the user, but they must be stored more securely and can be revoked to terminate sessions.

**Q40. What is mTLS and when would you use it between microservices?**  
**A.** mTLS (mutual TLS) is a variant of TLS where both client and server present certificates and authenticate each other. It is used between microservices to enforce strong, bidirectional authentication and encrypt traffic, which is especially important in zero-trust networks and when services communicate across less-trusted boundaries.

---

### 13. DDD, Bounded Contexts, and Integration

**Q41. How do you identify bounded contexts in a large legacy system?**  
**A.** Look for areas with distinct terminology, data models, and business rules (e.g. how “customer” or “order” is used differently in sales vs billing). Analyze organizational boundaries (teams, responsibilities) and existing code modules or databases. Bounded contexts appear where language and models diverge but need clear, explicit integration contracts.

**Q42. What are common integration patterns between bounded contexts or microservices?**  
**A.** Common patterns include synchronous APIs (REST/gRPC), asynchronous messaging and events (Kafka), anti-corruption layers to translate models, and dedicated read models or projections for cross-context queries. The key is to avoid sharing databases and instead define explicit contracts and ownership.

**Q43. How does an aggregate root help enforce invariants in a high-concurrency system?**  
**A.** The aggregate root is the only entry point for modifying related entities, ensuring all changes go through business rules and invariants. In high concurrency scenarios, you typically load and save the aggregate as a whole, using optimistic concurrency (versioning) or locking to prevent conflicting updates and maintain consistency.

---

### 14. Performance, Backpressure, and Resource Management

**Q44. What is backpressure, and why is it important in systems using Kafka or reactive streams?**  
**A.** Backpressure is the ability of a downstream consumer to signal that it cannot handle more data at the current rate. In Kafka or reactive streams, applying backpressure prevents fast producers from overwhelming slow consumers, which would otherwise cause queue buildup, latency spikes, and potential outages.

**Q45. How do you approach thread pool and connection pool sizing for a service?**  
**A.** Start by understanding whether workloads are CPU-bound or IO-bound. For IO-bound services, a somewhat higher number of threads may be acceptable; for CPU-bound work, thread counts should be close to CPU cores. For DB/HTTP connection pools, consider downstream capacity and timeouts; too few connections underutilize resources, too many can overload dependencies or increase contention. Measure, adjust based on metrics, and avoid tuning purely by guesswork.

**Q46. What metrics would you monitor first when investigating a performance regression in production?**  
**A.** Start with the “four golden signals”: latency, traffic, errors, and saturation. Look at per-endpoint latency, error rates, CPU/memory usage, connection pool utilization, GC activity, and downstream dependency metrics. Correlate changes (deploys, configuration changes) with metric shifts to narrow down root causes.

---

### 15. Miscellaneous Architecture & Leadership

**Q47. Why is a shared database across multiple microservices considered an anti-pattern?**  
**A.** It couples services at the data level, making schema changes risky and deployments tightly coordinated. Services can no longer evolve independently, and transaction boundaries become unclear. It undermines the whole purpose of microservices—independent ownership and deployment.

**Q48. What is an Architecture Decision Record (ADR), and why is it useful?**  
**A.** An ADR is a lightweight document capturing an important architectural decision, its context, options considered, the choice made, and its consequences. ADRs create a historical record of why things were done a certain way, making it easier for new team members to understand the system and for teams to revisit or revise decisions later.

**Q49. How do you handle architectural disagreements in a team?**  
**A.** Make trade-offs explicit, list options with pros/cons, and tie arguments back to requirements, constraints, and risk. Use structured processes like RFCs, ADRs, and design reviews. Seek alignment on goals, not personal preferences, and when in doubt, favor reversible decisions and experiments over long, theoretical debates.

**Q50. When would you _not_ choose microservices, even if they are fashionable?**  
**A.** You would avoid microservices when the domain and codebase are still small, team size is limited, and deployment/operational overhead would outweigh benefits. A modular monolith with clear boundaries is often better early on; microservices make sense when you have clear bounded contexts, scaling needs, team autonomy requirements, and enough maturity in operations and observability.
