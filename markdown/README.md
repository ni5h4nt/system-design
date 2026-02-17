# System Design Interview Prep

28 full system designs, a 6-phase playbook, and 143 interview Q&A pairs. Each design is a 75-minute walkthrough with architecture diagrams, deep dives, failure analysis, and the hardest questions interviewers ask.

| | |
|---|---|
| **27** System Designs | **6** Categories |
| **84** Deep Dives | **137** Interview Q&As |

---

## The 6-Phase Playbook

The framework behind every design: Clarify, Estimate, High-Level, Deep Dives, Cross-Cutting, Wrap-Up. Read this first.

> [system-design-playbook.md](system-design-playbook.md)

## Common Concepts Q&A

36 foundational questions every interviewer asks: SQL vs NoSQL, caching, sharding, CAP theorem, idempotency, circuit breakers, consistency patterns, and more.

> [common-concepts.md](common-concepts.md)

---

## Marketplaces & Consumer Platforms

| Design | Description | Core Tension |
|--------|-------------|--------------|
| [Uber](design-uber.md) | Real-time ride-hailing: geo-spatial matching, dynamic pricing, ETA prediction, and location tracking at 500K writes/sec. | 500K location writes/sec — in-memory spatial index with matching speed vs. trip completion reliability |
| [Facebook](design-facebook.md) | Social graph + news feed: fan-out strategies, ranked feed generation, and the celebrity problem at 2B+ users. | 17:1 read:write ratio, celebrity fan-out — hybrid push/pull feed architecture |
| [Amazon](design-amazon.md) | E-commerce at scale: product catalog, cart, checkout, multi-warehouse inventory, and the browse-vs-buy consistency split. | Browsing speed vs. buying correctness — two-level inventory (soft reserve + hard commit) |
| [YouTube](design-youtube.md) | Video platform: upload transcoding pipeline, adaptive bitrate streaming, CDN architecture, and recommendation engine. | Petabytes of storage + 1000:1 watch:upload ratio — cost-optimized streaming with CDN edge caching |
| [Weather App](design-weather.md) | Weather platform: multi-source data ingestion, forecast blending & precomputation, CDN-first serving at 200K QPS, and life-safety severe weather alerts. | 100,000:1 read:write ratio with extreme cacheability — precompute + CDN-first serving + separate alert push path |
| [Netflix](design-netflix.md) | Global video streaming: Control Plane (AWS) vs. Data Plane (Open Connect CDN), per-title shot-based encoding with VMAF, ISP-embedded OCAs, and ML-driven personalization. | Browse latency (sub-200ms APIs) vs. stream quality (73 Tbps zero-rebuffer) — Control/Data plane separation with proactive CDN caching |

## Aggregation Platforms

| Design | Description | Core Tension |
|--------|-------------|--------------|
| [Expedia](design-expedia.md) | Travel booking aggregator: scatter-gather search across 100+ suppliers, pricing cache, and multi-supplier booking sagas. | Search speed (<3s) vs. inventory freshness — two-phase: cached search + real-time booking |
| [Zillow](design-zillow.md) | Real estate data platform: 110M property graph, Zestimate ML pipeline, geo-spatial search, and Premier Agent marketplace. | Every home (110M) vs. just listings (2M) — comprehensive property graph + entity resolution |
| [Google Search](design-google-search.md) | Web search engine: crawler, inverted index, PageRank, query serving, and ranking at billions of pages and 100K QPS. | Index freshness vs. serving speed — offline crawl/index plane + online query serving plane |

## Financial Systems

| Design | Description | Core Tension |
|--------|-------------|--------------|
| [Robinhood](design-robinhood.md) | Retail brokerage: order execution, real-time market data streaming, portfolio management with consumer UX + institutional correctness. | Consumer traffic patterns + institutional correctness — separated read/write paths |
| [Bank of America](design-bofa.md) | Core banking: double-entry ledger, card authorization, check deposit, ACH/wire transfers, fraud detection with zero financial errors. | Real-time + batch coexistence — snapshot isolation with reconciliation as immune system |
| [Stripe](design-stripe.md) | Payment processing: card network integration, double-entry ledger, PCI-scoped card vault, idempotent API design, fraud detection, and T+2 settlement pipeline. | Speed (<1s checkout) vs. correctness (every cent must balance) — sync auth + async settlement + append-only ledger |

## Infrastructure & Observability

| Design | Description | Core Tension |
|--------|-------------|--------------|
| [Cloudflare](design-cloudflare.md) | Edge network: anycast routing, CDN caching, DDoS mitigation, WAF, and Workers serverless — all with zero centralized hot path. | Zero centralized hot path — anycast + edge-local processing at 330+ PoPs |
| [Splunk](design-splunk.md) | Log analytics: schema-on-read indexing, time-bucketed inverted index, SPL query language, and 10 TB/day sustained ingestion. | Write-optimized ingestion vs. read-optimized search — time-bucketed inverted index |
| [Datadog](design-datadog.md) | Cloud monitoring: metrics, logs, and traces ingestion, time-series storage, dashboards, and alerting across millions of hosts. | Millions of data points/sec ingestion vs. sub-second dashboard queries — TSDB architecture |
| [Docker](design-docker.md) | Container platform: Linux namespaces + cgroups for process isolation, OverlayFS for layered copy-on-write storage, content-addressable registry for image distribution. | VM-level isolation vs. container-level speed — shared kernel with defense-in-depth security layers |
| [bit.ly](design-bitly.md) | URL shortener: short code generation, <10ms redirect hot path, click analytics pipeline, and 100:1 read:write ratio. | Redirect speed (<10ms) vs. analytics accuracy — fast redirect path + async analytics pipeline |

## Developer Tools

| Design | Description | Core Tension |
|--------|-------------|--------------|
| [GitHub](design-github.md) | Code hosting: distributed Git at centralized scale, Spokes replication, pull requests, Actions CI/CD, and 200M+ repositories. | Distributed VCS — centralized hosting — Spokes replication for strong consistency |
| [Claude Code](design-claude-code.md) | Agentic coding: autonomous code generation with permission classification, tool orchestration, and the autonomy-vs-safety tension. | Autonomy vs. safety — permission classification balances speed with trust |
| [CodeRabbit](design-coderabbit.md) | AI code review: model cascade, ephemeral sandbox execution, precision-over-recall philosophy for PR review at scale. | Context depth vs. review latency — model cascade + ephemeral sandboxes |
| [MCP](design-mcp.md) | Model Context Protocol: JSON-RPC 2.0 wire format, host-client-server architecture, STDIO/HTTP transports, capability negotiation, and OAuth 2.1 auth. | M x N integration problem — M+N via universal protocol with host-mediated security boundary |
| [A2A](design-a2a.md) | Agent-to-Agent Protocol: Agent Cards for discovery, task-oriented lifecycle, multimodal Parts, SSE streaming, and signed-card trust model for cross-org agent collaboration. | Opaque agent interop across frameworks — task lifecycle + Agent Cards + protobuf-first multi-binding design |
| [CrewAI](design-crewai.md) | Multi-agent orchestration: role-based agents with backstories, Crews for autonomous collaboration, Flows for deterministic control, and the autonomy-vs-control dial. | Agent autonomy vs. production determinism — Flows (deterministic backbone) + Crews (intelligence where it matters) |
| [LangGraph](design-langgraph.md) | Graph-based agent runtime: Pregel/BSP execution with cyclic graphs, channel-based state with reducers, checkpointing for durability, and first-class human-in-the-loop. | Low abstraction + maximum control — Pregel super-steps give deterministic parallelism, checkpointing, and streaming for free |

## Identity & Security

| Design | Description | Core Tension |
|--------|-------------|--------------|
| [Okta](design-okta.md) | Identity platform: SSO/SAML/OIDC federation, MFA orchestration, multi-tenant isolation, and the trust platform paradox. | Absolute security with maximum availability — SECURITY > AVAILABILITY > CONSISTENCY > LATENCY |

## AI Observability

| Design | Description | Core Tension |
|--------|-------------|--------------|
| [Prove AI](design-proveai.md) | GenAI observability: trace-to-metric conversion, agentic remediation engine, self-hosted data sovereignty, and case management for nondeterministic systems. | Observing nondeterminism vs. remediating it — two-phase: telemetry foundation + agentic remediation |
| [Mem0](design-mem0.md) | AI memory layer: LLM-based fact extraction, hybrid vector + graph datastore, memory consolidation (dedup/conflict resolution), and multi-tenant retrieval under 50ms. | High-quality extraction (expensive LLM calls) vs. low-latency retrieval — async write plane + sync read plane |
| [OpenFGA / Zanzibar](design-openfga.md) | Google's global authorization system: relationship-based access control (ReBAC) via tuples, graph traversal check resolution, zookie consistency protocol, Leopard indexing for nested groups, and multi-layer caching at 10M+ checks/sec. | Consistency vs. latency — zookies let apps specify freshness per-request, solving the "new enemy" problem while serving most checks from stale cache |

## GenAI Architecture

| Design | Description | Core Tension |
|--------|-------------|--------------|
| [Production GenAI Architecture](design-genai-arch.md) | Universal 5-layer blueprint: Intake, Generation, Evaluation, Memory & Learning, and Orchestration — with hybrid retrieval, GraphRAG, LLM-as-Judge, and adaptive agent orchestration. | Flexibility vs. complexity — future-proof abstraction layers with progressive implementation (ship Week 1, evolve forever) |

---

*Built for system design interview preparation — 29 designs x 75 minutes each*
