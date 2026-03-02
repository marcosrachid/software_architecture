# software_architecture

Fundamentals of Software Architecture and Engineering

---

## 1. SOLID (Applied)

### 1.1 Applied view

- SOLID reduces **coupling**, makes **change** cheaper, and improves **testability**.
- In interviews, focus on **design decisions**: why you split components, how you extend without breaking, how you test.

### 1.2 Real-world violations

- **SRP (Single Responsibility Principle) violation**
  - A `UserService` that validates requests, talks to the DB, sends emails, and publishes events.
  - Effects: hard-to-test code, small changes break many things, frequent merge conflicts.
- **OCP (Open/Closed Principle) violation**
  - A `PaymentService` full of `if (type == "CREDIT_CARD") ... else if (type == "PIX") ...`.
  - Every new payment type requires changing the same class in multiple places.
- **LSP(Liskov Substitution Principle) / ISP( Interface Segregation) / DIP(Dependency Inversion Principle
  ) violations**
  - LSP: subclasses that override a method only to throw `UnsupportedOperationException`.
  - ISP: huge interfaces (`UserOperations`) where clients need only a few methods.
  - DIP: concrete classes doing `new MySqlUserRepository()` instead of depending on an interface.

### 1.3 SOLID in microservices

- **SRP at service level**
  - Each microservice focuses on **one bounded context** (`Billing`, `Catalog`, `Shipping`).
  - Avoid “god services” that span multiple domains.
- **OCP for evolution**
  - New flows or rules are added as **new handlers / event consumers**, not by inflating a central monolith service.
- **DIP between services**
  - Communication via **stable contracts** (APIs, versioned events) rather than shared concrete types.
  - Versioned schemas (Avro, JSON Schema) to keep consumers decoupled.

### 1.4 SRP in Spring services

- **Typical layering**
  - `Controller`: basic validation + delegating to services.
  - `Service`: business logic.
  - `Repository`: persistence.
- **Violation signals**
  - A huge `@Service` with many responsibilities (HTTP calls, DB, queues, emails).
- **Fix**
  - Extract smaller services (`EmailSender`, `PaymentProcessor`, `FraudChecker`) and inject them.

### 1.5 OCP with Strategy / Factory

- **Strategy**
  - `PaymentStrategy` interface (`pay(order)`), implementations: `CreditCardPayment`, `PixPayment`, etc.
  - The client receives/uses the right strategy without `if`-based type checks.
- **Factory**
  - `NotificationFactory` centralizes creation of `EmailNotification`, `SmsNotification` etc.
- **Practical rule**
  - OCP works when adding new behavior usually means **creating a new class/file**, not editing many existing ones.

---

## 2. Clean Code (Practical)

### 2.1 Naming

- **Clarity > brevity**: `calculateDiscount` > `calcDisc`.
- Names communicate **intent**, not implementation detail: `findActiveUsers` > `getUsers`.
- Avoid generic names: `data`, `value`, `manager`, `util`.

### 2.2 Cohesion

- A class/method should have **one clear reason to change**.
- Low cohesion signals:
  - A method description full of “and”: “validate order **and** save **and** send email **and** publish event”.
  - A class with methods that don’t belong together conceptually.

### 2.3 Cyclomatic complexity

- Number of independent logical paths (`if`, `for`, `while`, `switch`).
- High complexity → hard to test, easy to hide bugs.
- How to reduce:
  - Extract smaller methods.
  - Use Strategy / Chain of Responsibility for complex rules.
  - Replace long `if` chains with handler maps or polymorphism.

### 2.4 Incremental refactoring

- **Small steps + tests**.
- Introduce a new abstraction (e.g. `PriceCalculator`), migrate callers gradually.
- Separate **refactoring-only** changes from **functional** changes (different commits).

---

## 3. ACID

### 3.1 Where ACID really matters

- **Finance**: debits/credits, ledgers, reconciliation.
- **Strong consistency**: user permissions, critical inventory, reservations (seats, stock).
- Cases where tables must be **kept in sync at all times**.

### 3.2 ACID vs BASE

- **ACID**
  - Atomicity, Consistency, Isolation, Durability.
  - Strong transactions with immediate consistency (typical RDBMS).
  - **Focuses on correctness of data and transactions, not on availability or scalability**.
- **BASE**
  - Basic Available, Soft state, Eventual consistency.
  - Focus on availability/scale, accepting temporary inconsistency.

### 3.3 Scaling ACID databases (reads vs writes)

- **Reads**
  - **Read replicas**: primary node handles writes; replicas replay the transaction log and serve read-only queries.
  - **Caching** (Redis/Memcached): keep hot data in memory; DB remains the source of truth and ACID boundary.
  - **Sharding for reads**: partition data by tenant/region/key, each shard being its own ACID database.
- **Writes**
  - First, **scale up and optimize**: proper indexing, shorter transactions, batching, query tuning.
  - Then, **sharding**: distribute writes across partitions; ACID is usually guaranteed **per shard**, and cross-shard transactions are avoided or very rare.
  - For global strong consistency at scale, use **distributed SQL / NewSQL** systems (Spanner-like, Raft/Paxos-based), accepting higher latency and occasional unavailability under partitions (CAP trade-offs).

#### 3.3.1 Example – read replicas for an OLTP system

- **Context**
  - E-commerce application with far more reads (browsing products, listing orders) than writes.
  - A single relational DB is close to CPU/IO limits mainly due to `SELECT`s.
- **Topology**
  - One **primary** node (`orders_primary`) that receives all writes (`INSERT/UPDATE/DELETE`) and can also serve critical reads.
  - Two **read replica** nodes (`orders_replica_1`, `orders_replica_2`) that consume the **transaction log** from the primary and apply changes transactionally.
- **How the application uses it**
  - The driver or repository layer distinguishes:
    - **Critical reads** (for example, fetching the latest payment state) → go to the **primary**.
    - **Non-critical / listing reads** (for example, listing user orders, loading product catalogs) → go to one of the **replicas** via load balancing.
  - Each replica is a full **ACID** database; replication can be:
    - **Synchronous**: a commit only succeeds once at least one replica has applied it; reads can be strongly consistent but with higher latency.
    - **Asynchronous**: the primary commits first; replicas may lag slightly; reads from replicas are **eventually consistent**.
- **Trade-offs**
  - Benefit: horizontal scaling of **reads** without changing the data model; the primary carries less load.
  - Cost: you must decide where it is acceptable to read slightly stale data (with async replication) and handle primary/replica failover logic.

#### 3.3.2 Example – user-based sharding in a SaaS

- **Context**
  - Multi-tenant SaaS with many users per customer (tenant).
  - A single database can no longer handle the write/read volume.
- **Sharding decision**
  - Choose `tenant_id` as the **shard key**.
  - Create 4 physical shards: `users_shard_0`, `users_shard_1`, `users_shard_2`, `users_shard_3`.
  - Routing rule: `shard_index = hash(tenant_id) % 4`.
- **How the application uses it**
  - For an authenticated request, the application resolves the `tenant_id`.
  - The persistence layer (for example, `UserRepository`) computes the `shard_index` and opens a connection only to the corresponding shard.
  - Within a shard, operations like “create user”, “update profile”, and “reset password” run in **normal ACID transactions** (per shard).
- **Trade-offs**
  - Benefit: user reads and writes are spread across 4 databases, reducing contention on a single node.
  - Cost: operations that need data from multiple tenants (for example, global reporting) no longer fit in a single ACID transaction; they are done via separate queries to each shard + aggregation in the application or analytical pipelines.

### 3.4 Impact on Kafka + microservices

- Events / Kafka are naturally more **BASE**:
  - Consumers read at different times.
  - Views/caches are **eventually consistent**.
- **Outbox pattern**
  - Use a local ACID transaction for business data + outbox table.
  - A separate process reads the outbox and publishes to Kafka.
- **Distributed reads**
  - Queries joining multiple microservices are not ACID.
  - Use **sagas**, compensating actions, and dedicated **read models**.

---

## 4. CAP Theorem

### 4.1 Concept

- Under network partitions, a system must lean towards:
  - **C (Consistency)**: all nodes see the same data (or errors).
  - **A (Availability)**: the system always responds, possibly with stale data.
- The key is **behavior under partitions**, not under normal operation.

### 4.2 Real trade-offs

- **CP**
  - Prioritizes consistency, may reject/deny requests during partitions.
  - Suitable where **errors are better than wrong data** (e.g. balances).
- **AP**
  - Keeps responding, accepting temporary divergence across nodes.
  - Suitable for social feeds, analytics, counters.

### 4.3 Kafka and CAP

- Kafka is usually viewed as **AP** with strong per-partition ordering.
- Guarantees depend on `acks`, `min.insync.replicas`, and ISR configuration.
- Inside a partition, ordering is strong; consumers may be at different offsets.

### 4.4 NoSQL choices (CP vs AP)

- **CP** (HBase, etcd, strong-consistency modes of MongoDB)
  - Strong consistency, less availability during partitions.
- **AP** (Cassandra, Riak, Dynamo-style)
  - High availability, async replication, conflict resolution later.
- Many databases let you tune **consistency per operation** (strong/eventual).

---

## 5. Hexagonal Architecture

### 5.1 Ports & Adapters

- **Port**: interface representing a use case or gateway (`OrderService`, `PaymentPort`).
- **Adapter**: concrete implementation of a port (REST controller, Kafka consumer/producer, DB repository, CLI).
- Goal: keep the **domain independent** of DB, messaging, frameworks.

### 5.2 Where the domain lives

- Core: **entities, aggregates, domain services, ports (interfaces)**.
- Domain depends only on **interfaces**, not concrete infrastructure classes.
- Technical concerns (Spring, JPA, Kafka, HTTP clients) live in **infra/adapter** layers.

### 5.3 Integration with Kafka

- **Publishing**
  - Domain defines an `OrderCreatedEventPublisher` port.
  - Kafka adapter implements that port using a Kafka client.
- **Consuming**
  - Kafka listener (adapter) receives events, maps them to commands/DTOs.
  - It calls an application/domain port (`ProcessPaymentPort`, `ApplyDiscountPort`), keeping the core clean.

### 5.4 Avoiding framework dependence

- Avoid framework annotations on domain entities where possible.
- Define repository interfaces in the domain; implement them in infra (Spring Data, etc.).
- Keep domain as **plain objects**, not coupled to Spring/JPA/Kafka APIs.

---

## 6. REST Design Best Practices

### 6.1 Idempotency

- Repeating the same request yields the **same final state**.
- HTTP:
  - `GET`, `PUT`, `DELETE` should be idempotent; `POST` usually is not.
- In distributed systems:
  - Use **idempotency keys**.
  - Optimistic concurrency (`ETag`, `If-Match`).
  - Natural business keys to avoid duplicate entity creation.

### 6.2 API versioning

- Avoid breaking changes; when needed:
  - Path versioning: `/v1/orders`, `/v2/orders`.
  - Or header-based versioning for more advanced setups.
- Keep old versions active during migration windows.

### 6.3 Error handling

- Use proper HTTP codes: `400`, `401`, `403`, `404`, `409`, `422`, `500`.
- Standard error body:
  - `code`, `message`, `details`, `traceId`.
- Don’t expose stack traces or sensitive details; log them with `traceId`.

### 6.4 HATEOAS (concept)

- Responses include links to **related actions** on the current resource.
- Example: `GET /orders/123` with links to `cancel`, `pay`, `items`.
- In practice, many teams use a light form (key links only).

---

## 7. GraphQL

### 7.1 Federation (conceptual)

- The schema is split into multiple **subgraphs**, one per domain/team.
- A central **supergraph** combines them and exposes a unified endpoint.
- Teams own their subgraph schemas independently.

### 7.2 Schema stitching

- Combining several independent GraphQL schemas into one.
- Implemented in a gateway that imports and merges schemas, resolving conflicts.

### 7.3 GraphQL gateway

- Receives client queries, performs **query planning**.
- Splits into sub-queries per service, executes, and aggregates results.
- Can enforce auth, rate limiting, and caching.

### 7.4 Common issues (N+1)

- N+1: one query to get a list, then one query per list item.
- Mitigation:
  - **DataLoader** and batched fetching.
  - Backend endpoints optimized for bulk by IDs.
  - Local resolver-level caching.

### 7.5 When NOT to use GraphQL

- Very simple APIs with predictable calls and strong HTTP caching.
- Teams not ready to handle extra complexity (schema, gateway, performance, security).
- Cases where overfetching/underfetching is not a real pain.

---

## 8. System Design – Fundamentals

### 8.1 Vertical vs horizontal scaling

- **Vertical**
  - More CPU/RAM for a single machine.
  - Simple, but limited and often requires downtime.
- **Horizontal**
  - Add more instances.
  - Requires statelessness, load balancing, and often data partitioning.

### 8.2 Load balancer

- Distributes requests across instances.
- Strategies: round-robin, least connections, user/session hashing.
- Responsibilities: health checks, failover, path/host routing, TLS termination.

### 8.3 Stateless services

- No local session state between requests.
- State is stored in:
  - Tokens (e.g. JWT).
  - Shared cache (Redis).
  - Databases.
- Enables horizontal scaling and safe rolling deployments.

### 8.4 Caching (Redis, patterns)

- **Cache-aside (lazy loading)**
  - Read from cache; on miss, read DB, write to cache, return.
- **Write-through / write-behind**
  - Write-through: write to cache and DB synchronously.
  - Write-behind: write to cache and persist to DB asynchronously.
- Redis uses:
  - Read cache, session store, distributed locks, simple queues, counters.
- Concerns: TTLs, invalidation, cache stampede.

### 8.5 Rate limiting

- Limit requests per IP/user/API key in a time window.
- Algorithms: token bucket, leaky bucket, fixed window, sliding window.
- Typically implemented at API gateways, reverse proxies, or middleware.

### 8.6 Circuit breaker

- Prevents cascading failures across services.
- States:
  - **Closed**: normal traffic.
  - **Open**: short-circuits calls after too many failures/timeouts.
  - **Half-open**: probes a few calls to see if the downstream has recovered.
- Benefits: protects resources and reduces perceived latency during outages.

---

## 9. Modern Architecture – Staff-Level Topics

### 9.1 Observability (essential)

- **Structured logs**.
- **Metrics**: SLIs, SLOs, SLAs.
- **Distributed tracing** and **Correlation IDs**.
- **OpenTelemetry** as a common instrumentation standard.
- Tools: Prometheus, Grafana, Jaeger, Datadog.
- Typical question: how you debug intermittent issues in a 30+ microservice system.

### 9.2 Advanced resilience

- Beyond circuit breakers:
  - **Bulkhead pattern** (resource isolation).
  - **Retry with exponential backoff**.
  - **Timeout budgets** per operation.
  - **Dead Letter Queues (DLQs)** for messages that repeatedly fail.
- Libraries: Resilience4j and cloud-native equivalents.

### 9.3 Advanced distributed consistency

- **Saga pattern**
  - Orchestration vs choreography, with compensating actions.
- **Outbox pattern**
  - Persist business data + event in one local transaction, publish later.
- **Exactly-once semantics**
  - Often implemented as **at-least-once + idempotency** in practice.
- Staff-level explanation:
  - Why 2PC across microservices is a bad idea.
  - How to achieve safe eventual consistency.

### 9.4 Microservices anti-patterns

- **Distributed monolith**.
- **Chatty services** (too many synchronous calls).
- **Shared database** across services.
- **Premature over-engineering**.
- Classic interview: “When would you NOT use microservices?” (answer: when a modular monolith is enough).

### 9.5 Cloud & infrastructure

#### 9.5.1 Containers & orchestration

- Docker layering and immutable images.
- Health checks: liveness vs readiness.
- Rolling updates and HPA (Horizontal Pod Autoscaler).
- Kubernetes as the standard platform in many orgs.

#### 9.5.2 CI/CD

- Trunk-based development vs GitFlow.
- Blue/Green deployment.
- Canary releases.
- Feature flags.
- Tools: GitHub Actions, GitLab CI, Jenkins.

### 9.6 Performance & scalability (Staff level)

- **Thread pool tuning**.
- **Backpressure** (especially with Kafka / reactive streams).
- Basic **GC tuning** concepts.
- **Connection pool sizing**.
- Understanding **CPU-bound vs IO-bound** workloads.
- Typical question: how you investigate degradation under load (metrics, profiling, traces, bottlenecks).

### 9.7 Security

- **OAuth2 vs JWT** (flows, scopes, token types).
- Token expiration and refresh strategies.
- **mTLS** between services.
- **OWASP Top 10** for modern APIs.
- Distributed rate limiting as an extra protection layer.

### 9.8 Technical leadership

- Defining **technical standards** and guardrails.
- Handling **architectural disagreements** via explicit trade-offs.
- Process for adopting new tech (RFCs, experiments, controlled rollout).
- Using **ADR (Architecture Decision Records)** to document impact and context.
- Understanding the **organizational impact** of technical decisions.

### 9.9 Data modeling & messaging (Kafka)

- **Schema evolution**.
- **Backward, forward, and full** compatibility.
- Avro vs JSON (size, validation, schema registry).
- Consumer groups, offsets, and partitioning (keys, ordering, parallelism).

### 9.10 Additional GraphQL topics

- When NOT to use GraphQL:
  - Simple APIs with good HTTP caching and stable patterns.
  - Teams not ready for the added complexity.
- Overfetching/underfetching vs REST.
- Cacheability (caching by query/variables, gateway-level caching).
- **BFF (Backend for Frontend)** exposing GraphQL/REST tailored to each client.

### 9.11 Study priorities (Staff today)

1. Observability (logs, metrics, tracing).
2. Saga + Outbox (distributed consistency).
3. Microservice anti-patterns.
4. Kubernetes fundamentals (deployments, health checks, HPA).
5. Technical leadership (trade-offs, ADRs, standards).
