# Common Concepts Q&A

The foundational questions that come up in every system design interview. These aren't tied to a specific system — they test whether you understand the building blocks.

| Stat | Count |
|------|-------|
| **Topics** | 15 |
| **Q&A Pairs** | 36 |
| **Linked Designs** | 18 |

---

## 1. SQL vs NoSQL

*The single most common system design question. Not "which is better?" but "which access patterns does each serve?"*

| Dimension | SQL (PostgreSQL, MySQL) | NoSQL (DynamoDB, Cassandra, MongoDB) |
|-----------|------------------------|--------------------------------------|
| Data model | Relational tables with joins | Key-value, document, wide-column, graph |
| Schema | Schema-on-write (enforced upfront) | Schema-on-read (flexible) |
| Consistency | Strong (ACID transactions) | Tunable (eventual to strong) |
| Scale model | Vertical first, then shard manually | Horizontal by design (partition key) |
| Query flexibility | Ad-hoc queries, complex joins | Optimized for known access patterns |
| Best for | Financial records, user accounts, relational data | High-throughput writes, time-series, session data, carts |

`Uber -> Postgres for trips, Cassandra for location history` `Amazon -> Postgres for orders, DynamoDB for cart` `Robinhood -> CockroachDB for ledger (SQL + distributed)`

**Q:** When would you choose NoSQL over SQL?

**A:** When three conditions align: (1) the access pattern is simple and known — single-key lookups or range scans, not ad-hoc joins, (2) the write volume exceeds what a single SQL node can handle (>50K writes/sec sustained), and (3) you don't need multi-row transactions. Classic examples: Uber's driver location stream (append-only, 125K writes/sec, never updated — Cassandra), Amazon's shopping cart (key-value by user_id, spiky traffic, TTL expiration — DynamoDB), click event streams (append-only, partitioned by time — ClickHouse). The mistake people make: choosing NoSQL for "scale" when their actual volume is 500 writes/sec — well within PostgreSQL's capability — and then suffering from the lack of joins and transactions.

**Q:** Can SQL databases scale horizontally? When would you still choose them over NoSQL?

**A:** Yes, with caveats. Vitess (used by GitHub for MySQL), Citus (PostgreSQL), and CockroachDB all provide horizontal SQL. The difference: cross-shard queries and distributed transactions are possible but expensive — seconds instead of milliseconds. You choose SQL when: (1) the data is inherently relational (orders reference users, products, payments — foreign keys matter), (2) you need ACID transactions (financial ledgers, inventory decrements), (3) you need ad-hoc query flexibility (analytics, admin dashboards, debugging). The sweet spot for sharded SQL: shard by a key (user_id, org_id) so 95% of queries are single-shard, and accept that the 5% of cross-shard queries are slower. This is exactly what Robinhood does with CockroachDB — orders, positions, and balances are all sharded by user_id, so "show me my portfolio" is a single-shard query.

**Q:** What's the polyglot persistence pattern and when is it worth the complexity?

**A:** Polyglot persistence means using different databases for different data types within one system — each optimized for its access pattern. Uber is a textbook example: PostgreSQL for trips (ACID), Redis for live locations (geospatial, sub-ms), Cassandra for location history (append-only, high throughput), Kafka for event streaming. The complexity cost is real: more systems to operate, monitor, back up, and reason about. Data consistency across stores requires application-level coordination (sagas, eventual consistency). It's worth it when: (1) a single database CANNOT serve all access patterns at the required latency/throughput, (2) you have the operational maturity to manage multiple systems, (3) the data naturally separates (you don't need transactions across the stores). For a startup with <1M users, PostgreSQL for everything is almost always the right call. Polyglot persistence is a scaling optimization, not a starting point.

---

## 2. Caching Strategies

*Caching is the most impactful performance optimization — and the hardest to get right.*

| Pattern | How It Works | Best For | Risk |
|---------|-------------|----------|------|
| Cache-aside | App checks cache first, reads DB on miss, writes to cache | General purpose. Most common. | Stale reads after DB update |
| Write-through | App writes to cache AND DB simultaneously | Read-heavy with consistency needs | Write latency overhead |
| Write-behind | App writes to cache, async flush to DB | High write throughput | Data loss if cache fails before flush |
| Read-through | Cache itself fetches from DB on miss | CDN, managed cache layers | Cold start / thundering herd |

`Facebook -> Redis for feed cache (cache-aside, 500 posts per user)` `Bitly -> Redis for URL lookups (read-through, 95% hit rate)` `GitHub -> Memcached for query results (cache-aside, fragment caching)`

**Q:** How do you handle cache invalidation?

**A:** The famous "hardest problem in computer science." There are three approaches, each with tradeoffs: (1) TTL-based expiration — set a time-to-live and accept staleness within that window. Simplest to implement. Good when slight staleness is acceptable (product catalog: 5-minute TTL means a price change takes up to 5 minutes to reflect). (2) Event-driven invalidation — when the source data changes, publish an event that invalidates or updates the cache. More complex but near-real-time freshness. Facebook does this: when a user updates their profile, a Kafka event triggers invalidation of all cached versions. (3) Content-addressable URLs — make the cache key include a content hash (Amazon product images: cdn.example.com/images/{hash}.jpg). When content changes, the URL changes, so the old cache entry naturally expires while the new URL is cached fresh. The worst strategy: trying to invalidate everything immediately across a distributed cache — the coordination cost often exceeds the cost of just setting a short TTL.

**Q:** What's the thundering herd problem and how do you solve it?

**A:** When a popular cache entry expires, hundreds of concurrent requests simultaneously miss the cache and all hit the database — a "thundering herd." The database spikes, potentially cascading into failure. Solutions: (1) Lock-based recompute — the first request that misses acquires a lock (Redis SETNX) and recomputes while others wait or serve slightly stale data. Only one database query fires instead of hundreds. (2) Probabilistic early expiration — each request has a small random chance of refreshing the cache BEFORE it expires, spreading the recompute over time instead of concentrating it at expiration. (3) Background refresh — a separate process refreshes popular cache entries before they expire, so they never go cold. This is what Bitly does for popular URLs — a background job refreshes the top 1% of URLs every minute so they never expire from cache. (4) Request coalescing — if 100 identical cache misses arrive within 50ms, collapse them into a single database query and share the result. This is built into some CDNs and proxy layers.

**Q:** Redis vs Memcached — when would you choose each?

**A:** Memcached: pure key-value cache, multi-threaded, slightly faster for simple GET/SET at very high throughput. No persistence, no data structures. Choose it when: you only need caching (not queuing, not pub/sub), you want maximum simplicity, and you're caching serialized objects. GitHub uses Memcached for query result caching — dead simple, fast. Redis: data structures (lists, sets, sorted sets, geospatial, streams), persistence options, Lua scripting, pub/sub. Choose it when: you need more than GET/SET — Uber uses Redis GEORADIUS for spatial driver lookups, Facebook uses Redis lists for feed caches, Robinhood uses Redis for real-time market data with TTL. The honest answer for most systems: Redis. It does everything Memcached does (caching) plus much more. Memcached's only advantage is raw throughput on simple operations at very large scale (Facebook's scale, where they use both).

---

## 3. Sharding & Partitioning

*Splitting data across multiple database nodes. The key decision: what's the shard key?*

| Strategy | How | Pro | Con |
|----------|-----|-----|-----|
| Hash-based | hash(key) % N shards | Even distribution | Resharding is painful (consistent hashing helps) |
| Range-based | Key ranges (A-M -> shard 1) | Range queries efficient | Hot ranges (popular letters) |
| Geographic | Data lives in regional shards | Low latency per region | Cross-region queries are expensive |
| Tenant-based | One shard per customer / org | Perfect isolation | Operational overhead at scale |

`Uber -> shard by city_id (geographic)` `Robinhood -> shard by user_id (hash-based, blast radius isolation)` `Okta -> shard by org_id (tenant isolation)` `GitHub -> shard by repo_id via Vitess (hash-based)`

**Q:** How do you choose a shard key?

**A:** The shard key determines which data is co-located. The rule: choose the key that appears in 90%+ of your queries. For Uber, most queries are location-bounded ("find drivers near me in San Francisco"), so sharding by city_id means most queries hit a single shard. For Amazon, most queries are user-scoped ("my orders," "my cart"), so sharding by user_id co-locates all of a user's data. For GitHub, most queries are repo-scoped ("show this repo's files, issues, PRs"), so sharding by repo_id is natural. The mistake: choosing a shard key that forces scatter-gather queries. If you shard by user_id but your most common query is "find all orders for product X" (which spans all users), every query must fan out to all shards. That's when you need a secondary index (like Elasticsearch) for cross-shard queries rather than changing the shard key.

**Q:** What is consistent hashing and why does it matter?

**A:** With naive hash-based sharding (hash(key) % N), adding or removing a shard changes N, which remaps almost every key — requiring a massive data migration. Consistent hashing maps both keys and shards onto a ring (0 to 2^32). Each key maps to the nearest shard clockwise on the ring. When you add a shard, only keys between the new shard and its predecessor need to move — roughly 1/N of the data instead of nearly all of it. Virtual nodes (each physical shard gets multiple points on the ring) ensure even distribution. This is critical for: cache clusters (adding a Memcached node shouldn't invalidate 90% of cached data), DynamoDB-style systems (adding partitions during auto-scaling), and distributed hash tables. The key insight: consistent hashing makes scaling operations proportional to 1/N instead of N.

**Q:** How do you handle cross-shard queries?

**A:** You generally avoid them — that's the whole point of choosing the right shard key. But when they're unavoidable (admin dashboards, analytics, search), there are three approaches: (1) Scatter-gather — send the query to all shards, merge results. Works for aggregations (COUNT, SUM) but latency = slowest shard. Splunk's distributed search does exactly this. (2) Secondary indexes — maintain a separate, non-sharded index for cross-shard queries. Amazon uses Elasticsearch for "search all products" even though products are sharded by category in the primary DB. (3) Materialized views — pre-compute cross-shard aggregations into a dedicated table. Datadog pre-rolls metrics across tenants for internal dashboards. The key tradeoff: cross-shard queries are expensive by design. If you find yourself doing them frequently, either your shard key is wrong or you need a dedicated analytics path (OLAP) separate from your transactional path (OLTP).

---

## 4. Replication & Consensus

*Keeping copies of data in sync across multiple nodes for availability and durability.*

| Model | Mechanism | Consistency | Latency Impact |
|-------|-----------|-------------|----------------|
| Leader-follower | One writer, N readers replicate async | Eventual (reads may be stale) | Low write latency |
| Synchronous replication | Write confirmed only after N replicas ack | Strong | Higher write latency |
| Multi-leader | Multiple writers, conflict resolution | Eventual with conflicts | Low per-region |
| Consensus (Raft/Paxos) | Majority quorum agrees on each write | Linearizable | Highest (consensus rounds) |

`GitHub Spokes -> 3-replica synchronous for git repos` `CockroachDB -> Raft consensus (used by Robinhood)` `Cassandra -> Tunable: write to 2 of 3 replicas`

**Q:** What's the difference between replication and sharding?

**A:** They solve different problems and are often used together. Sharding splits data: shard 1 has users A-M, shard 2 has N-Z. Each shard holds different data. This increases write capacity (two shards handle 2x the writes). Replication copies data: replica 1 and replica 2 both hold all of shard 1's data. This increases read capacity (two replicas handle 2x the reads) and availability (if one dies, the other serves). In practice, you do both: shard your data for write throughput, then replicate each shard for read throughput and fault tolerance. GitHub: MySQL is sharded by repo_id (different repos on different shards), and each shard has 2 read replicas (same data copied for read scaling and failover).

**Q:** How does Raft consensus work in simple terms?

**A:** Raft elects a leader among N nodes (typically 3 or 5). All writes go through the leader. The leader sends the write to all followers and waits for a majority (2 of 3, or 3 of 5) to acknowledge before confirming the write to the client. This guarantees: even if a minority of nodes fail, the write is durable (a majority has it). If the leader dies, followers notice (no heartbeat) and hold an election — the follower with the most up-to-date log wins. The new leader continues serving writes within seconds. CockroachDB uses Raft per-range (chunk of data), so different ranges can have different leaders on different nodes, distributing write load. The tradeoff: every write requires a round-trip to a majority of nodes, adding 5-20ms latency. For Robinhood's ledger, this is acceptable because correctness matters more than speed.

---

## 5. CAP Theorem & Tradeoffs

*You can't have all three: Consistency, Availability, Partition tolerance. In distributed systems, partitions WILL happen — so you're choosing between C and A.*

> **Rule:** The real-world framing: CAP isn't a binary toggle. It's a spectrum. Most systems choose "tunable consistency" — strong consistency for critical paths (payments, inventory), eventual consistency for everything else (feeds, recommendations, analytics). The interview answer isn't "we're CP or AP" — it's "we're CP for these operations and AP for those, and here's why."

`Robinhood -> CP: ledger must be consistent (no phantom shares)` `Uber -> AP: slightly stale driver locations are fine` `Facebook -> AP for feed, CP for post creation` `BofA -> CP: $0.01 discrepancy triggers investigation`

**Q:** Give an example of choosing availability over consistency, and explain why.

**A:** Facebook's News Feed. When you load your feed, the Feed Service reads from Redis (precomputed feed cache). If the cache is slightly stale — say, a friend posted 30 seconds ago and it's not in your cache yet — Facebook still shows you the feed. The alternative (strong consistency) would be: refuse to show the feed until we've confirmed with every write replica that no new posts exist. If any replica is slow or unreachable (a network partition), the feed would show an error. Facebook chose: a slightly stale feed is infinitely better than no feed. Post CREATION, however, is consistent — when you post, it's durably written and acknowledged. You'd be upset if your post vanished. But seeing your friend's post 30 seconds late? You'd never notice. This is the pattern: availability for reads, consistency for writes, on the same system.

**Q:** When is strong consistency non-negotiable?

**A:** When inconsistency creates financial loss, safety risk, or regulatory violation. Three clear cases: (1) Financial transactions — Robinhood's ledger uses serializable isolation because showing a user they own 10 shares of AAPL when they actually own 5 is a regulatory violation. Bank of America's core ledger must balance to the penny. (2) Inventory decrements — Amazon's "buy" button uses compare-and-swap to ensure we never sell more than we have. Overselling means shipping products we don't have and refunding angry customers. (3) Authentication — Okta can't have a race condition where a deprovisioned user's session is still valid on some replicas. A fired employee accessing corporate systems is a security incident. The common thread: these are paths where "oops, we were inconsistent for 5 seconds" has real-world consequences beyond a bad user experience.

---

## 6. Microservices vs Monolith

*The architecture decision that's more about team organization than technology.*

**Q:** When should you start with a monolith vs microservices?

**A:** Almost always start monolith. The reason is organizational, not technical: with a small team (< 20 engineers), microservices add operational overhead (service discovery, distributed tracing, network failures, deployment coordination) without proportional benefit. A monolith lets a small team move fast — one repo, one deploy, one database, in-process function calls instead of network calls. You split into microservices when: (1) team size makes the monolith a coordination bottleneck (50+ engineers stepping on each other's code), (2) different components need independent scaling (the image processing pipeline needs 10x the compute of the API server), (3) different components have different deployment cadences (the payment team deploys weekly, the feed team deploys hourly). Amazon, Uber, and Netflix all started as monoliths and split as they grew. The goal isn't "microservices" — it's "independent deployability and scaling where it matters."

**Q:** How do you handle transactions across microservices?

**A:** You can't use traditional database transactions because each service has its own database. The pattern is a saga — a sequence of local transactions with compensating actions for rollback. Amazon's checkout is a saga: (1) Order Service: reserve inventory -> (2) Payment Service: charge card -> (3) Fulfillment Service: create shipment. If step 2 fails (card declined), the saga executes a compensating transaction: release the inventory reservation from step 1. Two saga implementations: choreography (services react to events — "inventory.reserved" triggers payment) and orchestration (a coordinator service directs the flow). Orchestration is easier to reason about and debug. Key requirement: every step must be idempotent (safe to retry) because network failures can cause duplicate messages. The tradeoff vs. monolith: in a monolith, this would be a single database transaction with automatic rollback. Sagas add complexity but enable independent scaling and deployment.

**Q:** What's the "distributed monolith" anti-pattern?

**A:** When you split into microservices but they're all tightly coupled — every request cascades through 10 services synchronously, a change in one service requires coordinated deployment of 5 others, and all services share a single database. You got all the complexity of microservices (network calls, distributed failures, separate deployments) with none of the benefits (independent scaling, independent deployment, fault isolation). The symptoms: you can't deploy one service without deploying three others, a single service failure cascades to all services, and your "microservices" meetings look exactly like your old monolith meetings. The fix: services must own their data (no shared databases), communication should be async where possible (events via Kafka, not synchronous HTTP chains), and each service must be deployable and testable independently. If your service can't function when another service is down (with degraded behavior, not an error), you haven't actually decomposed.

---

## 7. API Design: REST vs gRPC vs GraphQL

*Choosing the right API protocol for the right communication pattern.*

| Protocol | Best For | Tradeoff |
|----------|----------|----------|
| REST | Public APIs, CRUD operations, browser clients | Over-fetching, multiple round-trips for complex views |
| gRPC | Internal service-to-service, streaming, low-latency | Not browser-native, requires protobuf schema management |
| GraphQL | Mobile clients needing flexible queries, BFF pattern | Complexity, N+1 query problem, caching is harder |
| WebSocket | Real-time bidirectional (chat, live prices, locations) | Stateful connections, harder to load-balance |

**Q:** When would you choose gRPC over REST?

**A:** For internal service-to-service communication where latency matters. gRPC uses HTTP/2 (multiplexed connections, header compression) and Protocol Buffers (binary serialization, 5-10x smaller and faster than JSON). This matters when: services call each other thousands of times per second (Uber's Trip Service calling Location Service), you need bidirectional streaming (Datadog's agent streaming metrics), or you want contract enforcement (protobuf schemas catch breaking changes at compile time, not runtime). REST is still better for: public APIs (developers expect JSON and HTTP verbs), browser clients (native fetch/XHR support), and simple CRUD where raw performance isn't critical. Many systems use both: gRPC internally, REST at the edge. The API Gateway translates between them.

**Q:** How do you design a paginated API for large result sets?

**A:** Two approaches: offset-based and cursor-based. Offset-based (`?page=3&limit=20`) is simple but broken at scale — if new items are inserted while paginating, you'll either skip items or see duplicates. Also, `OFFSET 10000 LIMIT 20` in SQL scans and discards 10,000 rows. Cursor-based pagination (`?cursor=abc123&limit=20`) uses an opaque cursor (typically the last item's ID or timestamp) to fetch the next page. The query becomes `WHERE id > cursor_id ORDER BY id LIMIT 20`, which is an index scan regardless of how deep you are. The cursor is encoded (base64) so clients treat it as opaque. This is what every major API uses: Facebook's Graph API, GitHub's API v4, Slack's API. The tradeoff: cursor-based doesn't support "jump to page 5" — you must traverse sequentially. For most UIs (infinite scroll), this is fine.

---

## 8. Event-Driven Architecture & Message Queues

*Decoupling producers from consumers with async messaging. Kafka, SQS, RabbitMQ — different tools for different patterns.*

| System | Model | Best For |
|--------|-------|----------|
| Kafka | Durable event log, consumer groups, replay | Event sourcing, high throughput, stream processing |
| SQS | Simple queue, at-least-once, auto-scaling | Job queues, task distribution, decoupling |
| RabbitMQ | Broker with routing, exchanges, priorities | Complex routing, request-reply, priority queues |

**Q:** When should you use a message queue vs a synchronous API call?

**A:** Use async when: (1) the caller doesn't need the result immediately — Amazon's order confirmation doesn't need to wait for the fulfillment warehouse to acknowledge, (2) the downstream service is slower or unreliable — Expedia querying 10 travel suppliers synchronously would make the search take 15 seconds; instead, results are aggregated as they arrive, (3) you need to absorb traffic spikes — YouTube receives upload bursts after events, and the transcode queue absorbs the spike without scaling the transcoding fleet for peak. Use synchronous when: (1) the caller needs an immediate response — Robinhood's order placement needs to confirm the order was accepted before the user proceeds, (2) the operation is fast and reliable — reading from Redis cache is 1ms, no need to queue it, (3) error handling requires immediate feedback — a login attempt should tell the user right away if the password is wrong, not "we'll get back to you."

**Q:** What's the difference between a message queue and an event stream?

**A:** A message queue (SQS, RabbitMQ) is a pipe: a message is produced, consumed by one consumer, and deleted. It's fire-and-forget — the producer sends work to a pool of workers. If you add consumers, each message still goes to exactly one consumer (competing consumers pattern). An event stream (Kafka) is a log: an event is appended, and multiple independent consumers can read it at their own pace. Events are retained for a configurable period (days to weeks), enabling replay. Multiple consumer groups each get every event independently. Kafka is for: "something happened (order.placed), and multiple systems need to react independently (inventory, notifications, analytics)." SQS is for: "here's a job (transcode this video), exactly one worker should do it." The practical difference: if you delete your Kafka consumers and recreate them, they can replay all events from the beginning. With SQS, consumed messages are gone. This makes Kafka essential for event sourcing, audit trails, and rebuilding downstream state.

**Q:** How do you handle message ordering and exactly-once delivery?

**A:** Ordering: Kafka guarantees ordering within a partition. If you need all events for order #123 to be processed in sequence (placed -> filled -> settled), put them on the same partition by using order_id as the partition key. Cross-partition ordering doesn't exist — and you should design so you don't need it. Exactly-once delivery: it doesn't truly exist in distributed systems. What we have is "effectively exactly-once" through idempotent consumers. The producer sends at-least-once (retries on failure may duplicate), and the consumer deduplicates using a unique event ID. The consumer keeps track: "I've already processed event ABC, skip." Kafka Streams supports exactly-once semantics within the Kafka ecosystem (using transactional producers and consumer offsets), but as soon as you interact with external systems (a database), you're back to at-least-once + idempotent consumer. The practical pattern: (1) every event gets a UUID, (2) consumers check a dedup table before processing, (3) processing + dedup-table-insert happen in a single transaction.

---

## 9. Horizontal vs Vertical Scaling

*When to add more machines vs bigger machines.*

**Q:** When is vertical scaling the right choice?

**A:** When the workload is inherently single-threaded or hard to distribute, AND you haven't hit the hardware ceiling. Examples: (1) a primary database that handles 10K TPS — a bigger box with more RAM (128GB -> 512GB) and faster SSD means more data in buffer pool and fewer disk reads. Simpler than sharding. (2) Bank of America's mainframe — COBOL transaction processing that's been optimized for 40 years runs fastest on a single high-end processor. (3) A Redis instance — Redis is single-threaded by design; a faster CPU is more effective than adding nodes (until you need more memory than one box has). Vertical scaling fails when: you hit hardware limits (~256 cores, ~12TB RAM currently), you need fault tolerance (one big box = one big SPOF), or your workload is naturally parallelizable (web request handling, MapReduce, video transcoding). Most system design answers should mention vertical scaling as a Phase 1 approach: "we'd start with a beefy Postgres on a 64-core machine, which handles up to ~50K TPS. Beyond that, we'd shard."

**Q:** What does "stateless services" mean and why does it matter for scaling?

**A:** A stateless service stores no data between requests — every request contains everything needed to process it (auth token, parameters). State lives in external stores (database, Redis, S3). Why it matters: if a service is stateless, you can run 100 copies behind a load balancer and any copy can handle any request. Scaling is just "add more copies." No session affinity, no data migration, no coordination. If one copy crashes, requests just go to another. Contrast with a stateful service like a WebSocket server holding 500K persistent connections: you can't just add a server — existing connections are on specific servers. Scaling requires connection-aware routing. Most services in a well-designed system are stateless (API servers, business logic, transformations). The few stateful components (databases, caches, WebSocket servers) require special scaling strategies. The interview pattern: always design services as stateless and push state to dedicated stores.

---

## 10. Load Balancing

*Distributing traffic across multiple servers. The first line of defense for both scaling and availability.*

| Algorithm | How | Best For |
|-----------|-----|----------|
| Round-robin | Rotate through servers sequentially | Homogeneous servers, simple workloads |
| Least connections | Route to server with fewest active connections | Variable request duration (API + long-running) |
| Weighted | More traffic to beefier servers | Mixed hardware, canary deployments |
| Consistent hashing | Same key always routes to same server | Caching, sticky sessions, stateful services |
| Geographic / Anycast | Route to nearest data center | Global services (Cloudflare, Google) |

**Q:** What's the difference between L4 and L7 load balancing?

**A:** L4 (transport layer) load balancers route based on IP and port — they don't look at HTTP content. Fast (just forward packets), simple, works for any TCP/UDP protocol. AWS NLB is L4. L7 (application layer) load balancers inspect HTTP content: headers, URL path, cookies. They can route /api/* to the API cluster and /static/* to the CDN. They can do SSL termination, header injection, rate limiting, and authentication. AWS ALB and Nginx are L7. Use L4 when: you need raw throughput and low latency (market data feeds, game servers, database connections). Use L7 when: you need content-based routing, SSL termination, or request-level features. Most web architectures use L7 at the edge (customer-facing) and L4 internally between services (where routing is simpler and every millisecond of parsing overhead matters at high RPS).

**Q:** How do you avoid the load balancer becoming a single point of failure?

**A:** Multiple layers: (1) DNS round-robin to multiple LB instances — if one LB goes down, DNS routes to others. (2) Active-passive LB pairs — a standby LB takes over via VRRP/keepalived if the primary fails (sub-second failover). (3) Anycast — Cloudflare's approach: the same IP is announced from 330+ data centers. If one DC's LB fails, traffic naturally routes to the next nearest DC via BGP. (4) Cloud-managed LBs (AWS ALB/NLB) are inherently distributed — AWS runs them as a fleet of nodes across availability zones, so there's no single LB instance. The meta-answer: at the scale where a single load balancer is a concern, you're typically using a managed LB service or a distributed routing layer (like Envoy mesh) rather than a single Nginx box.

---

## 11. Content Delivery Networks

*Caching content at the edge, close to users. Typically reduces latency by 10-50x for static content.*

**Q:** What should and shouldn't go through a CDN?

**A:** CDN: static assets (images, CSS, JS, fonts), video segments (YouTube's HLS chunks), public API responses that are cacheable (product catalog pages with 5-minute TTL), downloadable files. NOT CDN: personalized content (your feed, your cart — unique per user, uncacheable), real-time data (live stock prices, chat messages), write operations (POST/PUT/DELETE), authenticated API responses with user-specific data. The edge case: semi-personalized content. Amazon's product page is 80% the same for everyone (product description, images, reviews) and 20% personalized (recommendations, "you might also like"). A smart CDN strategy caches the 80% at the edge and fetches the 20% from origin, assembling them via edge-side includes (ESI) or client-side JavaScript. This is the difference between a 50ms page load (80% cached) and a 500ms page load (100% from origin).

**Q:** How does a tiered CDN architecture work?

**A:** Without tiers: each edge PoP independently fetches from origin on a cache miss. If content is popular globally, 100 PoPs make 100 identical origin requests. Origin is overwhelmed. With tiers: misses at edge PoPs go to a regional "shield" (upper-tier cache) first. Only the shield fetches from origin. So 100 edge misses become 1 shield request to origin. Cloudflare's architecture uses this: 330 edge DCs -> ~30 regional shields -> origin. A viral video on YouTube: first view in Tokyo misses the Tokyo edge, hits the APAC shield, which fetches from origin and caches. Every subsequent Tokyo view (and Osaka, Seoul, Singapore views that hit the APAC shield) is served without touching origin. Origin load reduction: potentially 100x or more. The tradeoff: tiered caching adds one hop of latency on shield misses (edge -> shield -> origin instead of edge -> origin), but the cache hit rate improvement far outweighs this.

---

## 12. Rate Limiting & Backpressure

*Protecting your system from traffic it can't handle — whether from legitimate spikes or attacks.*

| Algorithm | How | Pro | Con |
|-----------|-----|-----|-----|
| Token bucket | Tokens refill at fixed rate, request consumes a token | Allows bursts up to bucket size | Burst can still overwhelm |
| Sliding window | Count requests in a rolling time window | Smooth, no burst edge cases | More memory per client |
| Leaky bucket | Requests enter a queue, processed at fixed rate | Smooth output rate guaranteed | Excess requests queued (latency) or dropped |

**Q:** How do you implement distributed rate limiting across multiple API servers?

**A:** The challenge: if you have 10 API servers each allowing 100 requests/second per user, a user can actually send 1000/sec by spreading across servers. Solution: centralized counter in Redis. Each API server increments a Redis counter keyed by user_id and checks the limit atomically using a Lua script: `INCR key; if count > limit then REJECT; EXPIRE key TTL`. Redis handles millions of these per second. The tradeoff: every request adds a Redis round-trip (~1ms). For ultra-low-latency paths, you can use local rate limiting (each server allows limit/N where N is server count) as a first pass, and Redis as a second pass. This is slightly inaccurate (a user could get limit * 1.1 in edge cases) but avoids the Redis hop for most requests. Robinhood uses centralized rate limiting (10 orders/sec per user) because the accuracy requirement is strict — algorithmic trading on a retail platform must be prevented.

**Q:** What's backpressure and how does it differ from rate limiting?

**A:** Rate limiting protects your system from external clients: "you can only send 100 requests/sec." Backpressure protects your system from internal overload: "the downstream service is slow, so I'll slow down too instead of overwhelming it." Example: Datadog's ingestion pipeline. If the metrics processing service is slow (maybe a noisy tenant), Kafka consumers slow their consumption rate — Kafka naturally buffers. Without backpressure, the consumers would keep pulling at full speed, accumulating in-memory buffered messages until they OOM. Backpressure mechanisms: (1) Kafka: consumers control pull rate, (2) gRPC: flow control built into HTTP/2, (3) reactive streams: subscriber signals demand to publisher, (4) circuit breakers: stop calling an overloaded service entirely. The key insight: in a pipeline (A -> B -> C), if C is slow, B must slow down too, and A must slow down after that. If any component doesn't respect backpressure, it becomes a buffer that eventually explodes.

---

## 13. Circuit Breakers & Resilience Patterns

*Preventing cascading failures when a dependency goes down.*

> **Tip:** The circuit breaker pattern has three states: CLOSED (normal, requests pass through), OPEN (dependency is down, requests fail fast without calling the dependency), HALF-OPEN (periodically let one request through to check if the dependency recovered). Like an electrical circuit breaker that trips to prevent damage.

**Q:** How do circuit breakers prevent cascading failures?

**A:** Without a circuit breaker: Service A calls Service B with a 30-second timeout. B is down, so every request from A hangs for 30 seconds. A's thread pool fills up. Now A can't handle ANY requests — even ones that don't need B. Service C, which depends on A, also starts failing. This is a cascade. With a circuit breaker: after A sees 5 consecutive failures to B, the breaker trips OPEN. Subsequent calls to B fail immediately (100ms instead of 30 seconds). A's thread pool stays healthy, and it can serve requests that don't need B. Every 30 seconds, the breaker goes HALF-OPEN and tries one request to B. If it succeeds, the breaker closes (B is back). If it fails, it stays open. Expedia uses this for supplier APIs: if Marriott's API is down, the circuit breaker trips and Expedia shows results from other suppliers instead of hanging the entire search. Graceful degradation over total failure.

**Q:** What other resilience patterns should every distributed system use?

**A:** Five essential patterns: (1) Retries with exponential backoff — retry failed requests at 1s, 2s, 4s, 8s intervals with jitter (random offset to avoid thundering herd). (2) Timeouts — every external call must have a timeout. No exceptions. An API call without a timeout is a thread pool leak waiting to happen. (3) Bulkheads — isolate failure domains. Separate thread pools for critical vs. non-critical dependencies. If the recommendation service thread pool exhausts, the checkout thread pool is unaffected. (4) Fallbacks — when a dependency fails, return a degraded but useful response. Amazon's product page with recommendation service down still shows the product, just without "customers also bought." (5) Idempotency — make operations safe to retry (see next topic). These patterns compose: a retry triggers inside a circuit breaker, with a timeout on each attempt, using a bulkheaded thread pool, with a fallback if all retries fail.

---

## 14. Idempotency

*Making operations safe to retry. Essential in distributed systems where network failures make duplicates inevitable.*

**Q:** What does idempotent mean and why is it critical?

**A:** An operation is idempotent if performing it multiple times has the same effect as performing it once. HTTP GET is naturally idempotent — reading the same resource 10 times doesn't change anything. HTTP POST (create) is NOT naturally idempotent — submitting a payment form twice might charge twice. In distributed systems, duplicates are inevitable: a client sends a request, the server processes it but the response is lost (network timeout), the client retries, and now the server has received the request twice. Without idempotency, the user is charged twice. The fix: idempotency keys. The client generates a unique ID (UUID) per operation and includes it in the request. The server checks: "have I seen this key before?" If yes, return the cached result. If no, process and store the key+result. Stripe's API uses this pattern — every payment request includes an Idempotency-Key header. Amazon's checkout uses client-generated order IDs. Robinhood's order placement uses ClOrdID. Without idempotency, every retry is a potential double-spend.

**Q:** How do you implement idempotency in a database?

**A:** Two approaches: (1) Dedup table: create a table `processed_requests(idempotency_key PRIMARY KEY, result, created_at)`. In a single transaction: check if the key exists (if yes, return stored result), process the request, insert the key with the result. The transaction ensures atomicity — you can't process without recording, and you can't record without processing. Clean up old entries with a TTL job (delete entries older than 24 hours). (2) Natural idempotency: design the operation itself to be idempotent. Instead of `UPDATE balance SET amount = amount - 100` (non-idempotent: running twice deducts 200), use `UPDATE balance SET amount = 900 WHERE amount = 1000` (idempotent: running twice, the second attempt finds amount=900, not 1000, and the WHERE clause doesn't match). This is a compare-and-swap pattern. Amazon's inventory decrement uses this: `UPDATE inventory SET count = count - 1 WHERE product_id = X AND count > 0 AND NOT EXISTS (SELECT 1 FROM processed_orders WHERE order_id = Y)`. The order_id check makes it idempotent.

---

## 15. Consistency Patterns

*Understanding the spectrum from "always correct" to "eventually correct" — and where each belongs.*

| Level | Guarantee | Cost | Example |
|-------|-----------|------|---------|
| Strong / Linearizable | Every read sees the most recent write | Highest latency, lowest throughput | Bank balance, inventory count |
| Sequential | All operations appear in a total order | High | Distributed locks, leader election |
| Causal | Causally related operations are ordered; concurrent ones aren't | Medium | Social media comments (reply after original) |
| Eventual | All replicas converge given enough time | Lowest latency, highest throughput | DNS, CDN cache, social feed |
| Read-your-writes | You always see your own writes (others may not) | Low overhead | Profile updates, post creation |

**Q:** What's "read-your-writes" consistency and when do you need it?

**A:** After you write data, your subsequent reads always see that write — even if other users haven't seen it yet. Example: you update your Facebook profile picture. You refresh the page and expect to see the new picture, not the old one. But your friend in another country might still see the old picture for 5 seconds (eventual consistency for them is fine). Without read-your-writes, you'd update your picture, refresh, see the OLD picture, panic, try again — terrible UX. Implementation: after a write, route subsequent reads from that user to the primary replica (which has the latest data) instead of a secondary replica (which may be stale). Or: after a write, include a timestamp in the user's session, and any read checks if the replica is at least that fresh — if not, read from primary. This is a targeted consistency boost: strong consistency for the user who wrote, eventual consistency for everyone else. Much cheaper than strong consistency for all reads.

**Q:** How do you handle conflicts in an eventually consistent system?

**A:** When two replicas accept conflicting writes before syncing, you need a conflict resolution strategy. Options: (1) Last-writer-wins (LWW): use timestamps, most recent write wins. Simple but lossy — an earlier write is silently discarded. DynamoDB uses this by default. Fine for overwrite-style data (user profile updates), bad for accumulator data (like counters). (2) Application-level merge: the application defines how to merge conflicts. Google Docs uses operational transformation to merge concurrent edits into a coherent document. Git uses three-way merge for code. (3) CRDTs (Conflict-free Replicated Data Types): data structures mathematically guaranteed to merge without conflicts. A G-Counter (grow-only counter) is a CRDT — each replica tracks its own increments, and the total is the sum across replicas. No conflicts possible. Used in distributed counters (view counts, like counts). (4) Read-repair: on read, if replicas disagree, the system repairs the inconsistency before returning. Cassandra does this with a configurable consistency level. The key insight: the conflict resolution strategy should match the data semantics. Profile updates: LWW is fine. Financial balances: conflicts are unacceptable, so use strong consistency instead.
