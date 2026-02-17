# Design A2A
*Agent-to-Agent Protocol · 75 min*

## Phase 01: Clarify the Problem & Scope *(5–7 min)*

> **Say:** "We're designing A2A — the Agent-to-Agent protocol — an open standard by Google (now Linux Foundation) that enables AI agents built on different frameworks to discover each other, negotiate interaction modalities, delegate tasks, and collaborate — all without exposing their internal state, tools, or memory. If MCP is 'USB-C for connecting models to tools,' A2A is 'the internet for agents to talk to each other.' MCP = agent-to-tool. A2A = agent-to-agent."

### Questions I'd Ask

- **What's the relationship to MCP?** *→ Complementary. MCP connects an agent to its tools (databases, APIs). A2A connects agents to each other. An agent might use MCP to query a database, then use A2A to delegate a subtask to a specialist agent. They live at different layers of the stack.*
- **Are agents opaque to each other?** *→ Yes — this is a core design principle. Agent A doesn't know Agent B's internal framework, LLM, memory, or tools. They interact only through the A2A protocol. This preserves IP and allows heterogeneous systems to collaborate.*
- **Synchronous or async interactions?** *→ Both. Quick tasks complete synchronously in one request-response. Long-running tasks use a lifecycle (submitted → working → completed) with streaming updates via SSE or push notifications via webhooks.*
- **How do agents find each other?** *→ Agent Cards: a JSON file at /.well-known/agent.json describing the agent's name, skills, endpoint, and auth requirements. Like a business card for AI agents.*
- **Protocol bindings?** *→ Three bindings: JSON-RPC over HTTP, gRPC, and plain HTTP/REST. The data model is defined in protobuf — bindings are generated from it. Protocol-agnostic data layer + concrete bindings.*
- **Enterprise-grade?** *→ Yes. OAuth 2.0, OpenID Connect, signed agent cards, task-scoped tokens, audit trails. Backed by 150+ partners including Google, SAP, Salesforce, ServiceNow.*

### Agreed Scope

| In Scope | Out of Scope |
| --- | --- |
| Agent Card: discovery, capabilities, skills | Internal agent implementation (LLM, memory, tools) |
| Task lifecycle: submit → working → completed/failed | Agent orchestration frameworks (ADK, LangGraph internals) |
| Message exchange: text, files, structured data (Parts) | Shared agent memory or state synchronization |
| Streaming (SSE) + push notifications (webhooks) | Agent training or fine-tuning |
| Auth: OAuth 2.0, OpenID, API keys, signed cards | Billing, marketplace, agent monetization |
| Multi-binding: JSON-RPC, gRPC, HTTP/REST | In-process agent communication (same runtime) |

### Core Use Cases

- **UC1: Task delegation** — A purchasing concierge agent discovers a pizza seller agent via its Agent Card, sends a message ("Order a large pepperoni"), and receives task updates as the seller agent processes the order. The agents may be built on entirely different frameworks (ADK vs CrewAI).
- **UC2: Multi-turn collaboration** — A travel agent delegates hotel booking to a hotel agent. The hotel agent asks a clarifying question ("Which dates?"). The travel agent relays the answer. Back-and-forth until the task completes. This requires stateful task tracking with the `input-required` state.
- **UC3: Long-running async task** — A research agent submits a deep analysis request to a data agent. The data agent takes 10 minutes. The research agent subscribes via SSE for streaming progress updates, or registers a webhook for push notification on completion.
- **UC4: Agent discovery** — An enterprise orchestrator fetches Agent Cards from `/.well-known/agent.json` for all registered agents. It matches the user's request to the agent with the best-matching skill. No hardcoded agent registry — just standard HTTP discovery.
- **UC5: Multimodal exchange** — A design agent sends an image artifact (PNG) to a QA agent for visual regression testing. The QA agent responds with a structured JSON report. A2A's Part system supports text, files, images, and structured data in a single message.

### Non-Functional Requirements

- **Opacity:** Agents are black boxes to each other. No sharing of internal prompts, memory, tools, or LLM choice. The protocol only exposes inputs (messages) and outputs (artifacts). This protects intellectual property and enables heterogeneous agent ecosystems.
- **Framework-agnostic:** An agent built with Google ADK can talk to one built with LangGraph, CrewAI, Semantic Kernel, or bare Python. The protocol is the contract, not the implementation.
- **Task-oriented:** Every interaction is structured as a Task with a lifecycle. Not free-form messaging — tasks have IDs, states, history, and artifacts. This enables tracking, auditing, and resumption.
- **Multi-modal content negotiation:** Agents declare what content types they accept and produce (text/plain, application/json, image/png). The client agent checks the server's capabilities before sending. No sending an image to a text-only agent.
- **Enterprise security:** OAuth 2.0, OpenID Connect, signed Agent Cards (cryptographic proof of identity), task-scoped short-lived tokens. Zero ambient authority.
- **Backward compatible:** Agent Cards include a `protocolVersion`. Older clients gracefully degrade when encountering newer servers. Unknown fields are ignored, not errors.

> **Tip:** The fundamental insight: MCP and A2A occupy different layers. MCP is the "nervous system" connecting an agent to its own tools and data (function calling). A2A is the "social network" connecting agents to other agents for collaboration. An enterprise stack uses both: MCP internally (agent ↔ tools), A2A externally (agent ↔ agent). A2A agents are opaque peers; MCP servers are transparent tool providers. This complementary layering is the key architectural distinction.

## Phase 02: Back-of-the-Envelope Estimation *(3–5 min)*

> **Say:** "A2A is a protocol, not a centralized service. But let's estimate the interaction patterns to understand what the protocol must handle."

| Metric | Value | Detail |
| --- | --- | --- |
| A2A-Enabled Agents (Ecosystem) | ~10K+ | Growing: 150+ partners. Every enterprise will have 5-50 internal agents communicating via A2A. |
| Tasks per Agent per Day | 100–10K | Enterprise orchestrator delegating to specialist agents. Varies wildly by use case. |
| Messages per Task | 1–20 | Simple: 1 request + 1 response. Multi-turn: 10-20 back-and-forth messages. |
| Task Duration | 100ms – 30 min | Instant: lookup agent. Long-running: data analysis, report generation, order fulfillment. |
| Message Size | 1 KB – 10 MB | Text: ~1KB. Structured JSON: 10-100KB. File artifacts (images, PDFs): up to MBs. |
| Agent Card Size | 2–10 KB | JSON: name, description, endpoint, skills[], auth, capabilities. Cached by clients. |

> **Decision:** **Key insight #1:** A2A is decentralized — there's no central broker. Every A2A server is a standalone HTTP endpoint. The scalability challenge belongs to each agent operator, not to a central infrastructure. A popular agent (like a Salesforce CRM agent) must handle its own traffic, exactly like a popular web API.

> **Decision:** **Key insight #2:** Task duration is bimodal. Quick tasks (lookups, simple questions) complete in under a second — synchronous HTTP is fine. Long-running tasks (research, data processing) take minutes — these need async patterns: SSE streaming for real-time updates, or webhooks for fire-and-forget notification. The protocol must handle both gracefully with the same task abstraction.

> **Decision:** **Key insight #3:** Discovery is the cold-start problem. Before agents can collaborate, they must find each other. Agent Cards at `/.well-known/agent.json` provide a pull-based discovery mechanism. But in an enterprise with 50 internal agents, some form of registry or catalog becomes necessary — the protocol defines the card format but not the registry.

## Phase 03: High-Level Design *(8–12 min)*

> **Say:** "A2A has a three-layer architecture: a protocol-agnostic data model (Task, Message, Part, Artifact), abstract operations (send message, get task, cancel), and concrete protocol bindings (JSON-RPC, gRPC, HTTP). The key participants are: Client agents (initiators) and Server agents (executors). Everything is task-oriented — agents don't have free-form conversations; they collaborate on tasks with defined lifecycles."

### Key Architecture Decisions

| Requirement | Decision | Why (and what was rejected) | Consistency |
| --- | --- | --- | --- |
| Agent-to-agent communication (not agent-to-tool) | Task-oriented protocol (not tool-call protocol) | Agents are opaque peers, not transparent function providers. A "task" abstraction captures multi-turn, long-running, multi-modal collaboration. MCP's tool-call model treats the server as a function to invoke — A2A treats it as an autonomous collaborator. | — |
| Agents on different frameworks must interoperate | Protobuf-first data model + multiple bindings | Single authoritative protobuf definition generates JSON-RPC, gRPC, and REST bindings. Protobuf ensures type safety across languages. A JSON-only spec would lack the rigor for gRPC and typed SDK generation. | — |
| Discover agent capabilities before delegating | Agent Cards at /.well-known/agent.json | Standard HTTP discovery (like .well-known/openid-configuration). No central registry needed — each agent self-describes. A central directory would be a single point of failure and a governance bottleneck. | AP |
| Tasks range from instant to 30+ minutes | Sync (blocking) + SSE stream + webhook push | One-shot: `message/send` blocks until done. Streaming: `message/stream` returns SSE events. Async push: webhook URL in task config. Three patterns cover the full duration spectrum. Polling alone would waste bandwidth; SSE alone doesn't work for fire-and-forget. | Eventual |
| Rich data: text, JSON, images, files | Part-based content model with MIME types | Each message contains Parts (like MIME multipart). Each Part has a content type: text/plain, application/json, image/png, etc. Agents negotiate supported types via Agent Card. Text-only would exclude visual and structured data critical for enterprise workflows. | — |
| Cross-vendor auth in enterprise environments | OpenAPI-aligned security schemes | Agent Card declares supported auth: OAuth 2.0, OpenID Connect, API keys. Aligns with OpenAPI spec — enterprise infra already handles these. Custom auth would require new infrastructure at every org. | CP |

### Architecture: Client-Server Agent Model

```mermaid
graph TD
```

### Flow 1: Task Delegation (the core A2A interaction)

```mermaid
graph TD
```

### Flow 2: Long-Running Task with Streaming

```mermaid
graph TD
```

## Phase 04: Deep Dives *(25–30 min)*

### Deep Dive 1: Agent Cards & Discovery (~8 min)

> **Goal:** **Agent Cards are the foundation of A2A interoperability.** Every A2A server publishes a JSON document describing who it is, what it can do, how to authenticate, and where to reach it. Clients fetch these cards to decide which agent to delegate to. Like DNS + API docs in one file.

```sql
// Agent Card: GET https://hotel-agent.example/.well-known/agent.json
{
  "name": "Hotel Booking Agent",
  "description": "Finds and books hotels based on location, dates, and budget",
  "url": "https://hotel-agent.example/a2a",
  "protocolVersion": "0.3",
  "provider": {"organization": "TravelCorp Inc."},
  "capabilities": {
    "streaming": true,       // supports message/stream
    "pushNotifications": true, // can push via webhook
    "stateTransitionHistory": true // preserves full task history
  },
  "skills": [
    {
      "id": "book_hotel",
      "name": "Book Hotel",
      "description": "Search and book hotels matching criteria",
      "tags": ["travel", "booking", "hotels"],
      "examples": ["Book a hotel in NYC under $200/night for March 5-8"]
    }
  ],
  "defaultInputModes": ["text/plain", "application/json"],
  "defaultOutputModes": ["text/plain", "application/json", "application/pdf"],
  "authentication": {
    "schemes": [
      {"scheme": "OAuth2", "flows": {"authorizationCode": {
        "authorizationUrl": "https://auth.travelcorp.com/authorize",
        "tokenUrl": "https://auth.travelcorp.com/token",
        "scopes": {"booking:write": "Create bookings", "booking:read": "View bookings"}
      }}}
    ]
  }
}
```

> **Decision:** **Why /.well-known/agent.json instead of a central registry?** Decentralized discovery: each agent self-publishes its card at a well-known URL (same pattern as `/.well-known/openid-configuration`). No single registry to operate, no governance bottleneck, no single point of failure. The tradeoff: discovery requires knowing the agent's base URL first. In practice, enterprises maintain an internal catalog or directory that lists known agent URLs. The A2A protocol standardizes the card format and location — the "how do you find the URL in the first place" is left to the deployment environment (DNS, service mesh, marketplace).

> **Decision:** **Skills vs. tools: why the distinction?** MCP servers expose tools (functions). A2A servers expose skills (capabilities). The difference: a skill is a high-level description of what the agent CAN DO, not a function signature. "Book hotel" is a skill — internally it might involve 20 MCP tool calls, LLM reasoning, and database queries. The client doesn't see any of that. Skills are for human and LLM understanding ("which agent can help with hotel booking?"), not for programmatic invocation. This opacity is intentional: the agent's internal implementation is its IP.

### Deep Dive 2: Task Lifecycle & Messaging (~10 min)

> **Goal:** **Tasks are the central abstraction.** Every A2A interaction is a Task with an ID, a state machine, a message history, and output artifacts. This is what makes A2A fundamentally different from simple request-response protocols — tasks can be long-running, multi-turn, pausable, and resumable.

| State | Meaning | Transitions To |
| --- | --- | --- |
| submitted | Task received, queued for processing | working, rejected |
| working | Agent actively processing the task | input-required, completed, failed, canceled |
| input-required | Agent needs more info from the client | working (after client sends more messages) |
| completed | Task finished successfully. Artifacts available. | (terminal) |
| failed | Task failed. Error details in messages. | (terminal) |
| canceled | Task canceled by client via tasks/cancel. | (terminal) |

> **Decision:** **Why task-oriented instead of free-form messaging?** Tasks provide: (1) Auditability: every task has an ID, a start time, an end state, and a complete message history. Enterprise compliance requires this. (2) Resumability: if a connection drops, the client can call `tasks/get` with the task ID to retrieve the current state. Free-form messaging loses this on disconnect. (3) Orchestration: a client agent managing 5 concurrent tasks to 5 different server agents needs structured tracking. Task IDs + states enable this. (4) Billing: task completion events are natural billing anchors. The tradeoff: more structured than simple chat. But A2A agents aren't chatting — they're collaborating on work.

> **Tip:** **Messages vs. Artifacts:** Messages are the conversation (back-and-forth between client and server during task execution). Artifacts are the deliverables (the output of a completed task). A research agent's messages might say "I'm analyzing the data… here's a preview…" while its artifact is the final PDF report. Messages are mutable (the conversation evolves). Artifacts are immutable (the deliverable is fixed once produced). This distinction matters for caching, auditing, and downstream consumption.

```sql
// Message structure (Part-based, multimodal)
Message
{
  "role": "user" | "agent",
  "messageId": "msg-uuid",
  "parts": [
    {"type": "text", "text": "Here's the sales data for Q3"},
    {"type": "file", "mimeType": "text/csv",
     "uri": "https://storage.example/q3-sales.csv"},
    {"type": "data", "mimeType": "application/json",
     "data": {"total_revenue": 4200000, "region": "APAC"}}
  ]
}

// Task structure (returned by server)
Task
{
  "id": "task-42",
  "state": "completed",
  "messages": [                     // conversation history
    {role: "user", parts: [...]},
    {role: "agent", parts: [...]}
  ],
  "artifacts": [                    // immutable outputs
    {name: "Q3 Report",
     parts: [{type: "file", mimeType: "application/pdf",
              uri: "https://...report.pdf"}]}
  ]
}
```

### Deep Dive 3: Streaming & Push Notifications (~5 min)

> **Goal:** **Three interaction patterns for three use cases.** Blocking (sync) for instant tasks, SSE streaming for real-time progress, webhooks for async fire-and-forget. The server's Agent Card declares which it supports.

| Pattern | Method | When | How |
| --- | --- | --- | --- |
| Blocking (sync) | `message/send` | Quick tasks (<5s). Simple Q&A, lookups. | HTTP POST. Response body contains the final task state + artifacts. Client blocks until done. |
| Streaming (SSE) | `message/stream` | Long tasks with progress. Research, analysis. | HTTP POST returns SSE event stream. Server sends task_status and task_artifact events as they occur. Stream closes on completion. |
| Push (webhook) | `message/send` + pushNotification config | Very long tasks. Client can't hold connection. | Client provides a webhook URL in the task config. Server POSTs status updates to the webhook. Fully async — client's HTTP connection closes immediately. |

> **Decision:** **Why SSE instead of WebSocket?** Same reasoning as MCP: SSE works through standard HTTP infrastructure (proxies, load balancers, CDNs) without special configuration. WebSocket requires persistent connection upgrading that enterprise proxies often break. SSE is server→client only, which is exactly what A2A streaming needs (the server pushes progress updates). The client sends messages via standard POST — no need for a bidirectional socket.

### Deep Dive 4: Auth & Trust Model (~7 min)

> **Goal:** **A2A agents operate across organizational boundaries.** A Salesforce agent talking to a ServiceNow agent — these are different companies, different trust domains. Auth must be enterprise-grade, standard, and auditable.

| Auth Mechanism | When | How It Works |
| --- | --- | --- |
| OAuth 2.0 | Cross-org agent communication | Client obtains access token via authorization code flow. Token scoped to specific skills. Short-lived (minutes). Standard enterprise SSO integration. |
| OpenID Connect | Identity verification | Server verifies the client agent's identity claim. "Is this really the TravelCorp orchestrator?" ID tokens prove identity; access tokens prove authorization. |
| API Keys | Internal/trusted agents | Simple bearer token for intra-org communication. Less secure but lower friction for trusted internal agents. |
| Signed Agent Cards | Agent identity integrity | Agent Card is cryptographically signed. Client verifies the signature to ensure the card hasn't been tampered with. Prevents "agent impersonation" — a malicious server pretending to be a legitimate one. |

> **Decision:** **Why signed Agent Cards?** Without signatures, a man-in-the-middle could modify an Agent Card to redirect the endpoint to a malicious server. Signed cards (introduced in v0.3) provide cryptographic proof: "This card was published by the organization that controls this domain." The client verifies the signature before trusting the card's endpoint, auth, and capabilities. This is analogous to HTTPS certificates — you verify the server's identity before trusting it. Signed Agent Cards extend this trust model to the agent discovery layer.

> **Decision:** **Task-scoped tokens.** A best practice: tokens should be scoped per task, with expiry measured in minutes. A hotel booking task gets a token that authorizes exactly that booking operation, expiring in 5 minutes. This follows zero-trust principles — no ambient authority, no long-lived credentials, no broad scopes. If a token leaks, the blast radius is one task, not the entire agent's permissions.

## Phase 05: Cross-Cutting Concerns *(10–12 min)*

### Failure Scenarios

| Scenario | Mitigation |
| --- | --- |
| Remote agent goes down mid-task | Client callstasks/getwith the task ID to check status. If the server is unreachable, client retries with exponential backoff. If server comes back, it may resume from its last state (if it persisted task state). If the task is lost, client receives an error and can re-submit. The task ID enables idempotent resumption. |
| SSE stream disconnects during long task | Client reconnects and callstasks/getto retrieve current state + history. The task'sstateTransitionHistorycapability ensures no updates are lost. Client can also re-subscribe viatasks/resubscribeto resume streaming from the current point. The task ID is the reconnection anchor. |
| Agent Card is stale or tampered with | Signed Agent Cards (v0.3): client verifies cryptographic signature before trusting. TTL/caching: Agent Cards should have a Cache-Control header. Clients periodically re-fetch to detect capability changes. Card versioning:protocolVersionfield enables compatibility checks. |
| Content type mismatch (client sends image to text-only agent) | Agent Card declaresdefaultInputModes(e.g., ["text/plain"]). Client checks before sending. If a mismatch occurs, server returns a JSON-RPC error: "Unsupported content type." This is content negotiation failure — caught at protocol level, not at the application level. |
| Circular delegation (Agent A delegates to B, B delegates back to A) | Not solved at the protocol level — this is an orchestration concern. Best practice: each task carries a delegation depth counter or a chain of agent IDs. Agents refuse tasks that would create cycles. The orchestrator agent is responsible for preventing loops in its delegation logic. |
| Malicious agent returns harmful content | The client agent (orchestrator) is responsible for validating server responses before presenting to the user. Content filtering, output validation, and human-in-the-loop approval for sensitive actions. A2A doesn't trust remote agent output blindly — the client mediates. This mirrors MCP's host-mediated security model. |

### A2A vs. MCP — Complementary, Not Competing

| Dimension | MCP | A2A |
| --- | --- | --- |
| What it connects | Agent → Tools and data sources | Agent → Other agents |
| Server opacity | Transparent: server exposes tool schemas | Opaque: server's internals are hidden |
| Interaction model | Tool call: invoke function, get result | Task collaboration: multi-turn, stateful |
| State | Stateful session (capability negotiation) | Stateful tasks (lifecycle, history, artifacts) |
| Discovery | Manual config (no standard discovery) | Agent Cards at /.well-known/agent.json |
| Transport | STDIO (local) + Streamable HTTP (remote) | JSON-RPC + gRPC + HTTP/REST (remote only) |
| Auth | OAuth 2.1 for remote | OAuth 2.0, OpenID, API keys, signed cards |
| Content | Tool result (text, images) | Multimodal Parts (text, files, structured data, images) |
| Typical flow | Host → Client → Server → Tool → Response | Client Agent → Server Agent → Task → Artifacts |

### Monitoring & SLOs

> **Tip:** **Monitoring.** For A2A server operators: task completion rate, task latency by skill, error rate by client, concurrent tasks, SSE stream duration. For client agents: discovery latency (Agent Card fetch), task delegation success rate, end-to-end task duration, content negotiation failures. Key SLOs: task completion p95 <30s for synchronous, <5min for async; discovery (card fetch) p95 <500ms; streaming event delivery latency <1s. Alerting: task failure rate >10%, SSE disconnection rate >5%, auth failure rate >1% (credential issue), circular delegation detected.

## Phase 06: Wrap-Up & Evolution *(3–5 min)*

### What I'd Build Next

- **Agent marketplace & registry:** Google Cloud already launched an AI Agent Marketplace. Standardized A2A Agent Cards + signed identity + skill metadata enable a searchable catalog. Enterprises browse, evaluate, and connect agents like npm packages.
- **Dynamic UX negotiation:** An agent adds audio or video capability mid-conversation. "Let me show you a screen recording of the bug" — switches from text to video Part. Requires runtime capability upgrade negotiation within an active task.
- **Hierarchical multi-agent orchestration:** Agent A delegates to Agent B, which sub-delegates to C and D. Task dependency graphs with parallel execution and fan-out/fan-in. Requires formalized sub-task linking and progress aggregation.
- **QuerySkill():** Dynamically check if an agent can handle an unanticipated skill. "Can you translate this to Japanese?" without that skill being in the Agent Card. The agent evaluates at runtime and responds with confidence score.
- **Latency-aware agent selection:** Twilio's extension: agents broadcast latency metrics. The orchestrator routes to the most responsive agent. Enables SLA-driven agent selection and graceful degradation (play a filler prompt if all agents are slow).
- **Agent-to-agent trust chains:** Transitive trust: "I trust Agent A. Agent A vouches for Agent B. Therefore I conditionally trust Agent B." Formalized trust delegation with signed attestations. Critical for enterprise-scale agent meshes.

> **Say:** "A2A complements MCP by solving the agent-to-agent collaboration problem. Where MCP connects agents to tools (transparent, function-call model), A2A connects agents to other agents (opaque, task-oriented model). The key abstractions — Agent Cards for discovery, Tasks for stateful collaboration, Parts for multimodal content, and three interaction patterns (sync, streaming, push) — enable heterogeneous agents built on any framework to interoperate. Protobuf-first data model with JSON-RPC, gRPC, and REST bindings ensures the protocol works across any tech stack. Enterprise auth (OAuth 2.0, signed cards) makes it production-ready. The result: an 'internet of agents' where specialist AI systems collaborate across organizational boundaries to solve problems no single agent can."

## Phase 07: Interview Q&A *(Practice)*

**Q:** Why do we need A2A when we already have MCP? Can't agents just expose themselves as MCP servers?

**A:** You could expose an agent as an MCP tool, but you'd lose three critical capabilities. (1) Opacity: MCP tools expose their full schema — input parameters, output format. An A2A agent is a black box: you send it a natural language request and it figures out how to fulfill it. You don't need to know its internal tool signatures. (2) Multi-turn collaboration: MCP is request-response (call a function, get a result). A2A supports stateful tasks where the server agent can ask for clarification, provide progress updates, and deliver artifacts incrementally. A hotel booking agent that asks "Which room type?" can't do that with a single MCP tool call. (3) Long-running tasks: MCP tool calls are expected to complete quickly. A2A tasks can run for minutes with SSE streaming. The fundamental mental model differs: MCP treats the server as a transparent tool. A2A treats it as an autonomous collaborator. Both are needed — use MCP for your own tools, A2A for other agents.

**Q:** How does A2A handle an agent needing clarification mid-task?

**A:** This is the `input-required` task state — one of A2A's most important design features. When a server agent needs more information, it transitions the task to `input-required` and includes a message explaining what it needs ("I found 3 hotels. Which one do you prefer?"). The client agent receives this state, processes it (maybe relays the question to the human user or makes a decision itself), and sends another message to the same task. The server receives the clarification and transitions back to `working`. This loop can repeat multiple times. The task ID is the anchor — all messages and state transitions are tracked against it. This is fundamentally different from MCP, where a tool call either succeeds or fails — there's no mechanism for the tool to ask a follow-up question. A2A's task lifecycle was explicitly designed for this kind of collaborative, conversational workflow.

**Q:** What stops a malicious agent from impersonating a trusted one?

**A:** Three layers of defense: (1) HTTPS: the Agent Card is fetched over TLS from the agent's domain. You trust the card because you trust the domain's TLS certificate. This prevents network-level MitM. (2) Signed Agent Cards (v0.3): the card itself is cryptographically signed. Even if someone copies the card to a different URL, the signature verification fails because it's bound to the original domain/key. This prevents card-level impersonation. (3) OAuth 2.0: even if an attacker tricks a client into connecting to the wrong server, the OAuth flow redirects through the legitimate authorization server — the attacker can't obtain valid tokens. In practice, enterprise deployments add a fourth layer: an internal agent registry (trusted catalog) that only lists verified agents. The client only communicates with agents in the registry. Unknown agents are rejected regardless of their card contents.

**Q:** Why protobuf as the normative data model instead of JSON Schema?

**A:** Protobuf gives three advantages over JSON Schema: (1) Multi-binding generation: a single .proto file generates JSON-RPC types, gRPC stubs, and REST types. JSON Schema can't generate gRPC bindings. (2) Type safety: protobuf has strict typing with required fields, enums, and oneof unions. JSON Schema validation is looser and more error-prone. (3) Canonical serialization: signed Agent Cards require deterministic serialization for signature verification. Protobuf has well-defined canonical encoding; JSON serialization is non-deterministic (key ordering varies). The tradeoff: protobuf is less human-readable than raw JSON. But developers interact with SDKs (Python, TypeScript), not raw proto. The proto file is for machines (code generation, validation), the JSON representation is for humans (debugging, Agent Cards). A2A publishes both — proto is normative, JSON is derived.

**Q:** How would you design a system where 50 enterprise agents need to discover and communicate with each other?

**A:** Layer 1: Each agent publishes its Agent Card at its own `/.well-known/agent.json` endpoint. This is the A2A standard. Layer 2: An internal Agent Registry (like a service catalog) crawls all agent endpoints, fetches their cards, and indexes them by skill, domain, and trust level. This isn't part of the A2A spec but is necessary for enterprise scale. Layer 3: An Orchestrator Agent queries the registry: "Which agent can handle hotel bookings?" The registry returns matching Agent Cards ranked by capability match and past performance. Layer 4: The orchestrator authenticates via the card's declared OAuth scheme and delegates the task. For auth at scale: a centralized identity provider (Okta, Azure AD) issues tokens. All 50 agents trust the same IdP. Agent-to-agent auth becomes "present a valid token from our IdP." For observability: distributed tracing (OpenTelemetry) across all A2A task delegations. Every task carries a trace ID. You can visualize the full delegation chain: user → orchestrator → hotel agent → payment agent.
