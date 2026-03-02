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

### 4.3 Concrete examples (CA, CP, AP)

- **CA (Consistent + Available, not partition-tolerant)**
  - Example: a single-node relational database (one Postgres/MySQL instance) behind an application server.
  - Why CA:
    - As long as the single node is up, every request sees the same data (**consistency**) and the system responds (**availability**).
    - There is no meaningful tolerance to network partitions between replicas, because there are no replicas; if the node or its network link fails, the system is simply down (no **P**).
- **CP (Consistent + Partition-tolerant)**
  - Example: a strongly consistent metadata store like ZooKeeper/etcd used for leader election and configuration.
  - Why CP:
    - In the presence of a network partition, only the **majority quorum** is allowed to accept reads/writes; minority sides reject requests or become read-only.
    - The system preserves a single, correct view of data across nodes (**consistency**) and continues operating on at least one side of the partition (**P**), while sacrificing availability on the others.
- **AP (Available + Partition-tolerant)**
  - Example: a Dynamo-style key-value store or Cassandra cluster serving user profile or feed data.
  - Why AP:
    - During a network partition, all reachable nodes continue to accept reads and writes (**availability**), even if different partitions temporarily diverge.
    - The system tolerates partitions (**P**) by allowing replicas to diverge and later reconciling conflicts (e.g., last-write-wins, vector clocks), so it does not guarantee strict global consistency at all times.

### 4.4 Kafka and CAP

- Kafka is usually viewed as **AP** with strong per-partition ordering.
- Guarantees depend on `acks`, `min.insync.replicas`, and ISR configuration.
- Inside a partition, ordering is strong; consumers may be at different offsets.

### 4.5 NoSQL choices (CP vs AP)

- **CP** (HBase, etcd, strong-consistency modes of MongoDB)
  - Strong consistency, less availability during partitions.
- **AP** (Cassandra, Riak, Dynamo-style)
  - High availability, async replication, conflict resolution later.
- Many databases let you tune **consistency per operation** (strong/eventual).

### 4.6 Business perspective – when to prefer CA, CP, or AP

- **CA – Consistent + Available (no real partition tolerance)**
  - Typical business: smaller or more centralized systems (e.g. on-prem ERP, a single-hospital medical record system) where everything lives in one primary database or site.
  - Why it fits:
    - The organization mainly cares that data is **correct and available** while the local infrastructure is healthy.
    - It accepts that if the main site or DB goes down, the whole system is down until someone fixes it (no strong requirement for geo-distributed partition tolerance).
- **CP – Consistent + Partition-tolerant**
  - Typical business: domains where **wrong data is worse than being offline**:
    - Bank balances, limits, and payments.
    - Government registries (property records, identity).
    - Cluster configuration / leader election / feature flags that must never contradict each other.
  - Why it fits:
    - In a failure or partition, the system may return errors or become partially unavailable,
      but it preserves a **single, correct view of the truth**.
    - Business stance: “better to temporarily block operations than to commit or show wrong state.”
- **AP – Available + Partition-tolerant**
  - Typical business: domains where **user experience and continuity** matter more than perfectly up-to-date data:
    - Social network feeds, likes, comments.
    - Real-time-ish analytics dashboards and counters.
    - Recommendation systems (“trending”, “most viewed”, “you may also like”).
  - Why it fits:
    - Users prefer to see **something that works, even if slightly stale**, over error pages or timeouts.
    - The business accepts temporary inconsistencies (a like that appears later, a counter that is off by a bit) as long as the system stays responsive and data converges over time.

---

## 5. Hexagonal Architecture

Hexagonal Architecture (a.k.a. **Ports and Adapters**) is an architectural style that puts the **business core at the center** and treats everything else (web, database, messaging, CLI, external APIs) as **replaceable adapters** around it.  
The goal is to make the domain:

- **Independent of infrastructure and frameworks** (you can change DB, transport, or UI with minimal impact).
- **Easy to test** (core use cases can be tested with plain unit tests, without HTTP, Kafka, or real DBs).
- **Stable over time** (business rules change, but infrastructure details can evolve in separate layers).

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

### 5.5 Example – Hexagonal Architecture in an order service

- **Context**
  - Simple e-commerce **Order Service** responsible for:
    - Creating orders.
    - Calculating totals and discounts.
    - Reserving stock and charging payments via external systems.
- **Domain (core)**
  - Entities/aggregates:
    - `Order`, `OrderItem`, `Customer`, `Money`.
  - Domain services / use cases:
    - `PlaceOrderService` (or `PlaceOrderUseCase`).
    - `ConfirmPaymentService`.
  - Ports (interfaces owned by the domain):
    - `OrderRepository` (save/load orders).
    - `InventoryPort` (reserve/release stock).
    - `PaymentPort` (charge/refund).
    - `EventPublisherPort` (publish domain events like `OrderPlaced`).
- **Inbound adapters (driving the domain)**
  - REST controller:
    - `POST /orders` → maps HTTP request to a `PlaceOrderCommand` and calls `PlaceOrderService`.
  - Kafka consumer:
    - Listens to `PaymentConfirmed` events, maps payload to a domain DTO, and calls `ConfirmPaymentService`.
  - CLI or batch job:
    - A command-line tool that triggers retries or reconciliation by calling domain ports.
- **Outbound adapters (driven by the domain)**
  - Database adapter:
    - `JpaOrderRepository` implements `OrderRepository` using JPA/JDBC.
  - Inventory adapter:
    - `HttpInventoryClient` implements `InventoryPort`, calling an external inventory microservice over HTTP.
  - Payment adapter:
    - `StripePaymentClient` (or similar) implements `PaymentPort`, talking to a third-party payment gateway.
  - Messaging adapter:
    - `KafkaEventPublisher` implements `EventPublisherPort`, converting domain events to Kafka messages.
- **Why this is hexagonal**
  - The **domain layer** only knows about **ports and pure domain types**; it has no imports from HTTP, Kafka, JPA, or vendor SDKs.
  - Replacing infrastructure is a matter of swapping adapters:
    - Change from REST to gRPC → replace inbound adapters, keep domain and ports.
    - Change from Kafka to another broker → replace `EventPublisherPort` implementation.
    - Change DB technology → implement `OrderRepository` with a new adapter.
  - Testing becomes easier:
    - Unit tests instantiate `PlaceOrderService` with **in-memory implementations** of the ports (fakes/mocks), without any real network or database.

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

### 9.2.1 Launch strategy, deciders, and managed lists

- **Goal of a large-scale launch**
  - The main objective is not “deploy the code”, but **control risk** when exposing new behavior to real users.
  - Treat launches as an incremental experiment: start with a tiny blast radius, then gradually increase exposure while watching metrics.
- **Deciders / feature flag service**
  - Central service that answers questions like: `isEnabled("new_checkout_flow", userId, context)`.
  - Supports:
    - Percentage-based rollout (1%, 5%, 10%, 50%, 100% of traffic).
    - Targeting by segment (internal users, specific tenants, premium plans, regions).
    - Fast kill switches for problematic features.
  - Key benefit: **deploy != release** — code can be shipped dark and only turned on later via configuration.
- **Managed lists**
  - Centrally managed allow/deny/target lists (often via admin UI) used together with flags:
    - `allowlist` of beta customers or internal testers.
    - `denylist` of sensitive tenants that must be excluded from experiments.
  - Enable operations/business to change **who** is included in a rollout without redeploying.
- **Typical rollout pattern**
  - Phase 0: dark launch — code deployed, flag off; only unit/integration tests hit the path.
  - Phase 1: enable for internal staff or a small managed allowlist of friendly customers.
  - Phase 2: percentage-based rollout (1% → 5% → 10% → 50% → 100%), guarded by SLOs and alarms.
  - Phase 3: make the new path the default; eventually remove old code and the temporary flags.
- **Operational guardrails**
  - Dashboards comparing “old vs new path” for:
    - Error rate, latency, resource usage.
    - Business metrics (conversion, drop-off, revenue impact).
  - Clear runbooks for rollback:
    - First, flip the flag off (fast mitigation).
    - If necessary, roll back the deployment.

### 9.3 Advanced distributed consistency

- **Saga pattern**
  - Orchestration vs choreography, with compensating actions.
- **Outbox pattern**
  - Persist business data + event in one local transaction, publish later.
    -- **Exactly-once semantics**
  - Often implemented as **at-least-once + idempotency** in practice.
    -- Staff-level explanation:
  - Why 2PC across microservices is a bad idea.
  - How to achieve safe eventual consistency.

### 9.4 API gateway vs BFF

- **API Gateway**
  - Central entry point for all external traffic into the microservice landscape.
  - Typical responsibilities: routing to services, TLS termination, rate limiting, authentication, logging, request/response transformation.
  - Should expose **generic, reusable APIs** that can be consumed by multiple clients (web, mobile, partners), and stay relatively stable.
- **BFF (Backend for Frontend)**
  - A thin backend **per client experience** (e.g. Web BFF, Mobile BFF) that speaks to the gateway and/or services.
  - Responsibilities: aggregating data from multiple services, shaping payloads to match each UI, hiding chatty interactions from the frontend.
  - Allows different clients to evolve at different speeds (mobile vs web) without forcing breaking changes on all consumers.
- **How they work together**
  - A common pattern: **clients → BFFs → API Gateway → microservices** (or BFFs calling services directly when simple).
  - The gateway focuses on cross-cutting concerns (authN/Z, throttling, observability), while BFFs focus on **client-specific UX and composition**.
- **Pitfalls**
  - Overloading the gateway with too much business logic turns it into a bottleneck and “mini-monolith”.
  - Too many BFFs with duplicated logic and no ownership boundaries can create fragmentation; treat BFFs as real services with clear teams and contracts.

### 9.5 Microservices anti-patterns

- **Distributed monolith**.
- **Chatty services** (too many synchronous calls).
- **Shared database** across services.
- **Premature over-engineering**.
- Classic interview: “When would you NOT use microservices?” (answer: when a modular monolith is enough).

### 9.6 Cloud & infrastructure

#### 9.6.1 Containers & orchestration

- Docker layering and immutable images.
- Health checks: liveness vs readiness.
- Rolling updates and HPA (Horizontal Pod Autoscaler).
- Kubernetes as the standard platform in many orgs.

#### 9.6.2 CI/CD

- Trunk-based development vs GitFlow.
- Blue/Green deployment.
- Canary releases.
- Feature flags.
- Tools: GitHub Actions, GitLab CI, Jenkins.

### 9.7 Performance & scalability (Staff level)

- **Thread pool tuning**.
- **Backpressure** (especially with Kafka / reactive streams).
- Basic **GC tuning** concepts.
- **Connection pool sizing**.
- Understanding **CPU-bound vs IO-bound** workloads.
- Typical question: how you investigate degradation under load (metrics, profiling, traces, bottlenecks).

### 9.8 Security

- **OAuth2 vs JWT** (flows, scopes, token types).
- Token expiration and refresh strategies.
- **mTLS** between services.
- **OWASP Top 10** for modern APIs.
- Distributed rate limiting as an extra protection layer.

### 9.9 Technical leadership

- Defining **technical standards** and guardrails.
- Handling **architectural disagreements** via explicit trade-offs.
- Process for adopting new tech (RFCs, experiments, controlled rollout).
- Using **ADR (Architecture Decision Records)** to document impact and context.
- Understanding the **organizational impact** of technical decisions.

### 9.10 Data modeling & messaging (Kafka)

- **Schema evolution**.
- **Backward, forward, and full** compatibility.
- Avro vs JSON (size, validation, schema registry).
- Consumer groups, offsets, and partitioning (keys, ordering, parallelism).

### 9.11 Additional GraphQL topics

- When NOT to use GraphQL:
  - Simple APIs with good HTTP caching and stable patterns.
  - Teams not ready for the added complexity.
- Overfetching/underfetching vs REST.
- Cacheability (caching by query/variables, gateway-level caching).
- **BFF (Backend for Frontend)** exposing GraphQL/REST tailored to each client.

### 9.12 Study priorities (Staff today)

1. Observability (logs, metrics, tracing).
2. Saga + Outbox (distributed consistency).
3. Microservice anti-patterns.
4. Kubernetes fundamentals (deployments, health checks, HPA).
5. Technical leadership (trade-offs, ADRs, standards).
