# System Design Playbook

*Principal Engineer -- 75 min*

---

## Phase 1: Clarify the Problem & Scope

*5--7 min*

> **Goal:** Collapse ambiguity. Demonstrate product thinking. Negotiate scope before touching the whiteboard.

- **Restate the problem** in your own words -- confirm your understanding before designing anything.
- **Identify 2--3 core use cases** -- ask "what must this system absolutely do?" Prioritize ruthlessly.
- **Users & scale** -- Who are the users? DAU/MAU? Growth trajectory?
- **Access patterns** -- Read-heavy or write-heavy? Bursty or steady? Real-time or batch?
- **Non-functional requirements** -- Latency (p50/p99)? Consistency vs. availability? Durability? Compliance?
- **Define scope explicitly** -- "I'll focus on X and Y, treating Z as out of scope -- sound good?"
- **Write requirements on the Excalidraw canvas** -- this becomes your contract for the rest of the interview.

> **Tip:** If the interviewer says "it's up to you" -- make a decision, state the assumption, and move on. Don't stall.

---

## Phase 2: Back-of-the-Envelope Estimation

*3--5 min*

> **Goal:** Establish quantitative foundation that justifies architecture choices. Reason from numbers, not vibes.

- **Traffic** -- DAU x avg requests/user / 86,400 = QPS. *Separate reads & writes. Apply peak multiplier (2--5x).*
- **Storage** -- Avg object size x new records/day x retention. *Distinguish hot vs. cold data.*
- **Bandwidth** -- Write QPS x payload size (in). Read QPS x response size (out).
- **Memory (caching)** -- 80/20 rule: cache 20% of daily data. *Cache size = 0.2 x daily reads x avg size.*
- **Highlight the numbers that drive decisions** -- "At 50K reads/sec, we need a cache. At 2TB/day, we need partitioning early."

> **Tip:** Round aggressively: 86,400 ~ 100K. 1M seconds ~ 12 days. Write key numbers on the canvas -- you'll reference them later.

---

## Phase 3: High-Level Design

*8--12 min*

> **Goal:** Sketch end-to-end architecture on Excalidraw. Establish the skeleton you'll flesh out in the deep dive.

- **Start at the client, work inward** -- Client -> CDN/LB -> API Gateway -> Services -> Data Stores.
- **Draw read path and write path separately** -- narrate each as you draw.
- **Label each service** with its responsibility (e.g., "User Service," "Feed Service").
- **Show data stores** with what they hold and rough tech choice. *Don't over-commit yet.*
- **Show async pathways** -- queues, event buses, consumers.
- **Show external dependencies** -- third-party APIs, CDNs, DNS.

> **Tip:** **Narrate as you draw.** Don't sketch silently for 5 minutes. Color-code: one color for reads, another for writes. Leave space around components you'll deep-dive.

> **Say:** "Does this high-level flow make sense before I go deeper?"

---

## Phase 4: Deep Dive into Key Components

*25--30 min*

> **Goal:** Go deep on 3--4 components. Show principal-level depth in data modeling, storage, scaling, and tradeoffs.

> **Say:** "I think the most interesting challenges are X and Y -- shall I start there?"

### For each component, cover these dimensions

#### Data Model & Schema

- Core entities & relationships
- Primary access patterns
- Schema evolution over time
- Sketch key tables/collections

#### Storage Technology

- Why this tech over alternatives?
- Frame as tradeoff: gain X, lose Y
- Match to access pattern
- Name what you considered & rejected

#### Partitioning & Sharding

- Partition key choice & rationale
- Hot partition mitigation
- Re-partitioning strategy
- Interaction with access patterns

#### Caching Strategy

- Which layer(s) benefit?
- Policy: aside / through / behind
- Invalidation: TTL / event / version
- Thundering herd mitigation

#### Async Processing

- What's async and why?
- Delivery: at-least-once / exactly-once
- Failures, retries, dead-letter queues
- Consumer idempotency

#### API Design

- Key endpoints & methods
- Pagination (cursor vs. offset)
- Rate limiting & auth
- Versioning approach

> **Tip:** **Lead with "why" before "what."** Don't say "I'd use Kafka." Say "We need durable, replayable event delivery with multiple consumers, so a log-based broker fits -- the tradeoff is operational complexity."

---

## Phase 5: Cross-Cutting Concerns

*10--12 min*

> **Goal:** Show systems maturity. You think about how the system lives and dies in production, not just how it looks on a diagram.

#### Fault Tolerance

- Single node failure -> what happens?
- AZ failure -> what happens?
- Downstream timeout -> what happens?
- Circuit breakers, bulkheads, retries w/ backoff
- Graceful degradation

#### Consistency Model

- Where eventual? Where strong? Why?
- Conflict resolution strategy
- Stale reads: cache vs. DB

#### Scalability Bottlenecks

- What breaks at 10x? At 100x?
- Walk each component: compute, storage, network
- Hardest thing to scale?

#### Security

- AuthN/AuthZ (OAuth, JWT, RBAC)
- Encryption at rest & in transit
- Input validation, rate limiting
- PII handling

#### Observability

- Golden signals: latency, traffic, errors, saturation
- Structured logging
- Distributed tracing
- Alerting without fatigue

#### Operational Readiness

- Deployment strategy (blue/green, canary)
- Runbooks & on-call
- Capacity planning
- Chaos engineering

> **Tip:** **Pick the 2--3 most relevant.** Proactively raising failure scenarios -- don't wait for the interviewer to ask -- is one of the strongest principal-level signals.

---

## Phase 6: Wrap-Up & Evolution

*3--5 min*

> **Goal:** Summarize, show strategic thinking, demonstrate you see the system's future.

- **Summarize in 2--3 sentences** -- the architecture, key decisions, and why they fit the requirements.
- **Acknowledge deferred work** -- "I intentionally kept X simple -- in production I'd add Y."
- **System evolution** -- What changes at 10x or 100x? What's in v2 vs. v1?
- **Operational investments** -- Better observability, chaos engineering, automated scaling.
- **Tech debt & migration paths** -- "The denormalized schema works now, but we'd need CDC for analytics later."

> **Tip:** This is your closing argument. Don't introduce new components. Tie a bow on what you already discussed.

---

# Technology Decision Reference

---

## Data Storage

*Decide by: access pattern, consistency needs, schema flexibility, scale, query complexity*

| Option | Reach For It When... | Watch Out For... |
|---|---|---|
| Relational | ACID transactions, complex joins, strong consistency, stable schema | Horizontal write scaling is hard |
| Document | Flexible schema, denormalized reads, key-document access | No joins -- pay with data duplication |
| Wide-Column | Massive write throughput, time-series, range scans | Query patterns must be known upfront |
| Key-Value | Ultra-low latency lookups by key. Sessions, caches, flags | Zero query flexibility beyond the key |
| Graph | Relationships ARE the data -- social graphs, fraud detection | Niche. Don't use just for foreign keys |
| Search | Full-text search, faceted filtering, log analytics | Not a primary store -- always pair with source of truth |
| Time-Series | Metrics, IoT, event logs. Append-heavy, time-range queries | Limited non-time access patterns |
| Object Store | Blobs: images, videos, files, backups. Cheap & durable | High latency. Not for transactional data |

---

## Communication Patterns

*Decide by: who talks to whom, latency tolerance, coupling, payload shape, need for response*

| Option | Reach For It When... | Watch Out For... |
|---|---|---|
| REST | Request-response, CRUD, public APIs, broad compatibility | Over-fetching / under-fetching, chatty |
| GraphQL | Flexible queries, multiple frontends with different data needs | Server complexity, N+1, harder caching |
| gRPC | Service-to-service, latency/bandwidth matters, streaming, typed contracts | Not browser-friendly, binary = harder debug |
| Queue | Point-to-point async, task offloading, write buffering | Not great for fan-out, consumed once |
| Event Stream | Fan-out, multiple consumers, replay, high throughput, event-driven | Ops complexity, ordering per partition only |
| WebSocket | Real-time bidirectional -- chat, collab editing, live updates | Stateful connections complicate scaling |
| SSE | Server->client push, one-directional. Live feeds, notifications | One-way only, less control than WS |

**Use Sync (REST / gRPC)** -- Caller needs the result to continue

**Use Async (Queue / Events)** -- Work can be deferred, retried, or fan-out to multiple consumers

---

## Compute & Deployment

*Decide by: team size, ops maturity, scaling granularity, deployment frequency, cost*

| Option | Reach For It When... | Watch Out For... |
|---|---|---|
| Monolith | Early stage, small team, simple domain. Don't prematurely distribute | Can't scale individual components |
| Microservices | Independent teams, independent deploy, different scaling profiles | Distributed systems complexity everywhere |
| Serverless | Bursty traffic, event-driven glue, low ops overhead | Cold starts, time limits, vendor lock-in |
| Containers + K8s | Many services needing orchestration, discovery, auto-scaling | Significant ops overhead. Overkill for simple systems |

> **Say:** Don't justify K8s unless asked. Focus on `what` needs to scale and `why`, not deployment tooling.

---

## Caching

*Decide by: what's slow, what's read-heavy, what can tolerate staleness*

| Layer | When | Invalidation |
|---|---|---|
| Client / Browser | Static assets, predictable-TTL API responses | TTL, cache-control headers |
| CDN | Static content, media, geo-distributed users | TTL, purge on deploy |
| App (Redis) | Hot reads, query results, sessions, aggregations | Cache-aside + TTL (default), event-driven |
| Local / In-proc | Config, feature flags, small lookups | Short TTL, risk of cross-instance inconsistency |

**Cache-Aside (most common)** -- Read cache -> miss -> read DB -> populate cache. Watch for thundering herd.

**Write-Through / Write-Behind** -- Through: write both, stronger consistency. Behind: write cache, async flush, risk data loss.

---

## Event-Driven Patterns

| Pattern | When | Tradeoff |
|---|---|---|
| Notification | "Something happened" -- lightweight trigger | Consumers may need callback for details |
| State Transfer | Include full state so consumers don't call back | Larger payloads, data duplication |
| Event Sourcing | Full audit trail, rebuild state by replaying events | Complexity, eventual consistency |
| CQRS | Read/write models have different shapes or scale needs | Two models to maintain, sync lag |
| Saga | Distributed transactions where 2PC isn't feasible | Complex failure handling, compensating txns |

---

## Consistency & Coordination

| Pattern | When | Cost |
|---|---|---|
| Strong (single leader) | Financial txns, inventory -- stale reads cause harm | Higher latency, write bottleneck |
| Eventual | Feeds, analytics, notifications -- staleness OK | Simpler to scale, possible stale reads |
| Read Replicas | Scale reads without sharding, tolerate lag | Stale reads from replicas |
| Multi-Leader | Multi-region writes | Conflict resolution is hard |
| Consensus | Leader election, distributed locks, config mgmt | Quorum latency. Not for hot-path data |

---

## Load Balancing & API Gateway

| Option | When | Note |
|---|---|---|
| L4 (TCP) | Raw throughput, simple round-robin, no content inspection | Fastest, least flexible |
| L7 (HTTP) | Path routing, header inspection, SSL termination, rate limiting | Most common choice |
| API Gateway | AuthN, rate limiting, request transform, multi-backend routing | Single entry point for clients |
| Service Mesh | mTLS, fine-grained traffic control, network-level observability | Only at significant microservice scale |

---

# Guides

---

## The 5-Step Decision Pattern

> **Goal:** Use this every time you hit a technology decision point. Steps 4 and 5 are what separate principal-level answers from mid-level ones.

- **1. State the requirement** -- "We need low-latency reads at 100K QPS with a simple key-based access pattern."
- **2. Name the decision criteria** -- "I'm optimizing for read latency and horizontal scalability over query flexibility."
- **3. Make the choice** -- "A key-value store like DynamoDB fits here."
- **4. Name the tradeoff** -- "We give up ad-hoc querying, but our access patterns are well-defined."
- **5. Mention what you rejected** -- "I considered Postgres with read replicas, but at this QPS with key lookups, a purpose-built KV store is more efficient."

---

## Handling Interviewer Pivots

- **Don't panic** -- Constraints aren't traps. Say: "Great constraint. Let me think about how that changes the design."
- **Trace the impact** -- Walk through which components are affected and which aren't.
- **Revise the diagram live** -- Cross out the old approach, draw the new one, narrate your reasoning.
- **Acknowledge uncertainty** -- "I haven't worked with this exact pattern, but my instinct is X because of Y."

---

## Meta Principles

**01 -- Think out loud.** The interviewer evaluates your reasoning, not your final diagram. Silence is your enemy.

**02 -- Drive the conversation.** At principal level, you lead. Don't wait for "what about caching?" -- bring it up.

**03 -- Stay grounded in requirements.** Every decision traces to Phase 1. Catch yourself gold-plating -> refocus.

**04 -- Name the tradeoffs.** "We gain X, give up Y, acceptable because Z." This is the principal-level sentence pattern.

**05 -- "It depends" is fine** -- but always follow with *what it depends on.*

**06 -- Excalidraw is a living doc.** Annotate decisions, write numbers, mark failure zones. It should tell the story alone.

**07 -- Manage your time.** If 10 min on one component with 2 more to go -- say so and move on.

**08 -- Treat interviewer as collaborator.** Ask preferences. Respond to cues. It's a design conversation, not a presentation.
