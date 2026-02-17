# Design LangGraph
*Graph-Based Agent Runtime · 75 min*

## Phase 01: Clarify the Problem & Scope *(5–7 min)*

> **Say:** "We're designing LangGraph — a low-level, graph-based agent runtime for building stateful, multi-step LLM applications. LangGraph was born out of the LangChain ecosystem as a complete reboot, focused on production-readiness over ease-of-getting-started. The central insight: agents are fundamentally different from traditional software because LLMs are slow, non-deterministic, and open-ended. LangGraph's answer is to structure agent computation into discrete steps modeled as a cyclic graph, enabling checkpointing, human-in-the-loop, streaming, and safe parallelism — all without abstracting away developer control."

### Questions I'd Ask

- **Is this a framework or a platform?** *→ Both. LangGraph is the open-source runtime (MIT license). LangGraph Platform is the deployment layer with managed infrastructure, task queues, and APIs. We're designing the core runtime — the execution engine that powers both.*
- **How does it relate to LangChain?** *→ LangGraph is an extension of LangChain but can function independently. It was a deliberate reboot: LangChain was easy to start but hard to customize. LangGraph prioritizes customization and production-readiness. Nodes can be plain Python functions or LangChain Runnables.*
- **What's the execution model?** *→ Cyclic graph based on the Pregel/BSP (Bulk Synchronous Parallel) algorithm. Nodes are functions that read/write state. Edges (including conditional edges) define control flow. Execution proceeds in "super-steps" — each super-step runs eligible nodes in parallel, then applies updates deterministically. This enables cycles (loops), safe parallelism, and checkpointing.*
- **Why not use an existing framework?** *→ DAG frameworks (Airflow etc.) can't handle cycles. Durable execution engines (Temporal etc.) lack streaming, add inter-step latency, and degrade as history grows. LLM agents need both loops and low-latency streaming — no existing framework offered this combination.*
- **What LLMs does it support?** *→ LLM-agnostic. Integrates with any LangChain-compatible model (OpenAI, Anthropic, Google, Ollama, etc.). Different nodes in the same graph can use different models.*
- **Who's using it in production?** *→ LinkedIn, Uber, Klarna, Elastic, and thousands of others. The framework has processed billions of agent steps across enterprise deployments.*

### Agreed Scope

| In Scope | Out of Scope |
| --- | --- |
| StateGraph: nodes, edges, conditional routing | The LLM inference engine itself |
| Pregel execution algorithm (BSP-inspired) | LangChain core library internals |
| State management: channels, reducers, schemas | LangSmith (tracing/monitoring SaaS) |
| Checkpointing: persistence, threads, time-travel | Specific tool implementations |
| Human-in-the-loop: interrupt, resume, edit state | Multi-tenancy, billing, marketplace |
| Streaming: token-level, node-level, custom events | LangGraph Studio (visual debugger UI) |
| LangGraph Platform: task queue, deployment | Fine-tuning or training LLMs |

### Core Use Cases

- **UC1: ReAct agent with tools** — A single agent node calls an LLM, which decides whether to invoke a tool or produce a final answer. Conditional edge routes back to the agent (loop) or to END. Checkpointing saves state at each step so failures don't lose progress.
- **UC2: Multi-agent supervisor** — A supervisor node routes work to specialist agent sub-graphs (researcher, coder, analyst). Each specialist is itself a LangGraph with its own state. The supervisor collects outputs, evaluates quality, and either delegates more work or produces a final result.
- **UC3: Human-in-the-loop approval** — An agent drafts an email. Graph pauses at an interrupt point. Human reviews the draft, edits it, and resumes. The checkpoint preserves full state during the pause — even across server restarts. This is the key UX pattern for production agents.
- **UC4: Long-running research pipeline** — A multi-step pipeline: gather sources → analyze → synthesize → review. Each step may take minutes. Task queue handles execution asynchronously. On failure at step 3, checkpointing enables resuming from step 3 without re-running steps 1-2.
- **UC5: Chatbot with memory** — A conversational agent where each message appends to a persistent thread. Thread-level checkpointing maintains conversation history across sessions. Cross-thread Store enables long-term memory (user preferences, facts) shared across conversations.

### Non-Functional Requirements

- **Low abstraction, maximum control:** Developers write regular Python functions as nodes. No magic. No black-box cognitive architectures. The framework provides building blocks (checkpointing, streaming, interrupt) that you opt into — they don't get in your way until you reach for them.
- **Deterministic parallelism:** When multiple nodes can run concurrently, the framework must guarantee that the final state is independent of execution order and timing. Variability in outputs must come from the LLM, never from the framework.
- **Streaming-first:** Every agent must support real-time streaming of intermediate results — token-by-token LLM output, node completions, custom events. End users expect responsiveness despite LLM latency.
- **Durable execution:** Checkpoints must survive process restarts. An agent paused for human review on Monday must be resumable on Tuesday from a different server. Persistence backends: in-memory (dev), SQLite (lightweight), PostgreSQL (production).
- **Scalable performance:** Framework overhead must be negligible. Execution time must scale gracefully with graph size (more nodes, more steps, larger state). No performance cliffs as agents grow in complexity.
- **Ecosystem integration:** Must work with LangChain tools, prompts, and models. Must integrate with LangSmith for tracing. Must support MCP (Model Context Protocol) for external tool servers.

> **Tip:** The key design philosophy from the LangGraph team: "We aimed to find the right abstraction for AI agents, and decided that was little to no abstraction at all." Instead of imposing a cognitive architecture (like role-playing agents or fixed workflows), LangGraph provides low-level primitives — graph structure, state management, checkpointing — and lets developers compose them however they need. This is what makes LangGraph uniquely flexible but also harder to learn than higher-level frameworks like CrewAI.

## Phase 02: Back-of-the-Envelope Estimation *(3–5 min)*

| Metric | Value | Detail |
| --- | --- | --- |
| Nodes per Graph | 3–50+ | Simple ReAct: 2-3 nodes. Multi-agent supervisor: 10-20 nodes. Complex pipeline with sub-graphs: 50+. |
| Super-Steps per Run | 5–100+ | Each loop iteration = 1 super-step. ReAct agent with 5 tool calls = ~10 super-steps. Research pipeline: 50-100+. |
| Checkpoint Size | 10 KB – 10 MB | State dict serialized per super-step. Chat history grows linearly. Postgres supports up to 1GB per field. Typical: 50-500 KB. |
| Agent Run Duration | 1s – 60 min+ | Simple chatbot: 1-5s. Multi-step research: 5-30 min. Long-running with human pauses: hours or days. |
| Checkpoint Writes per Run | 5–100+ | 1 checkpoint per super-step. 10 super-steps = 10 checkpoints. Must be fast (<5ms overhead) to not impact latency. |
| Production Scale | Billions of steps | LinkedIn, Uber, Klarna in production. Horizontally-scaling servers + task queues via LangGraph Platform. |

> **Decision:** **Key insight #1: LLM latency dominates, framework overhead must be invisible.** A single LLM call takes 0.5–5 seconds. If checkpointing adds 5ms per super-step and there are 20 super-steps, that's 100ms total — invisible against 20+ seconds of LLM time. But if checkpointing added 500ms per step, it would add 10 seconds — unacceptable. The framework must be so fast that developers never think about its overhead. LangGraph targets sub-millisecond per super-step for the execution algorithm itself.

> **Decision:** **Key insight #2: Agents are getting longer, not shorter.** With test-time compute and chain-of-thought, agents are taking more steps, running longer, and building larger state. The framework's performance must scale gracefully with: (1) number of steps (no degradation as history grows — unlike Temporal), (2) state size (efficient serialization), and (3) number of concurrent nodes (parallel execution without data races).

> **Decision:** **Key insight #3: Human time is the real latency.** When an agent pauses for human approval, the wait might be seconds, hours, or days. The checkpoint must be fully durable — surviving server restarts, deployments, and even machine migrations. This pushes toward external persistence (Postgres) over in-memory state, and full serialization of all execution context.

## Phase 03: High-Level Design *(8–12 min)*

> **Say:** "LangGraph models agent computation as a cyclic graph executed via a Pregel-inspired algorithm. State flows through channels. Nodes are regular Python functions that read and write state. Edges — including conditional edges — define control flow. The runtime executes in super-steps: each step runs eligible nodes in parallel with isolated state copies, then applies updates deterministically. Checkpoints are saved after every super-step, enabling persistence, time-travel, and human-in-the-loop."

### Key Architecture Decisions

| Requirement | Decision | Why (and what was rejected) | Consistency |
| --- | --- | --- | --- |
| Agents need loops (cycles) | Cyclic graph (not DAG) | ReAct agents loop: call LLM → use tool → check → repeat. DAG frameworks (Airflow) can't handle this. LangGraph uses a Pregel/BSP-based algorithm that natively supports cycles, with a configurable max iterations limit (recursion_limit) as a safety valve. | — |
| Safe parallelism without data races | Pregel/BSP execution model | When multiple nodes run in parallel, each gets an isolated copy of state. Updates are applied in deterministic order after all finish. This guarantees that execution order never influences output — variability comes only from LLMs. Rejected: shared mutable state (race conditions), locks (deadlocks, latency). | Strong |
| Durable state across failures & pauses | Checkpoint after every super-step | Serialized state snapshots saved to pluggable backends (Memory, SQLite, Postgres, Redis). Enables: resume from failure, human-in-the-loop pauses, time-travel debugging. Trade-off: adds write latency per step, but it's <5ms with Postgres and negligible vs. LLM latency. | Eventual |
| Minimal abstraction, maximum control | Nodes are plain Python functions | No role-playing agents, no task definitions, no crew abstractions. A node is `def f(state) → state`. The developer decides what each node does. This makes LangGraph a runtime, not a cognitive architecture. Rejected: opinionated agent patterns (too rigid for production diversity). | — |
| Real-time user feedback despite latency | First-class streaming at every level | Token-level streaming from LLMs, node-level streaming (events when nodes start/finish), and custom events emitted from within nodes. The graph structure makes streaming natural — each node completion is a streamable event. Zero-overhead when not used. | — |
| Runtime independent of SDK | Separate PregelLoop runtime from developer APIs | Two public APIs: StateGraph (declarative) and Functional API (imperative). Both compile to the same internal PregelLoop runtime. This lets the team evolve APIs and runtime independently. Enables deprecating old APIs (e.g., the original Graph API) without touching the runtime. | — |

### System Architecture

```mermaid
graph TD
```

### Flow: ReAct Agent Execution

```mermaid
graph TD
```

## Phase 04: Deep Dives *(25–30 min)*

> **Goal:** **The four deepest architectural questions:** (1) How does the Pregel runtime work? (2) How do state channels and reducers prevent conflicts? (3) How does checkpointing enable durability and time-travel? (4) How does human-in-the-loop interrupt/resume work without losing state?

### Deep Dive 1: Pregel Runtime — The Execution Engine (~8 min)

> **Goal:** **The core algorithm.** LangGraph's runtime is called PregelLoop, inspired by Google's Pregel system for large-scale graph processing. The key idea: structure agent computation into discrete "super-steps" with deterministic concurrency, enabling checkpointing, streaming, and safe parallelism as emergent properties of the execution model.

```sql
// The Pregel execution algorithm (simplified)
PregelLoop.run(input, config):
    channels = initialize_channels(graph.schema)    // State containers
channels[__start__].write(input)                // Write input to entry channel
for step in range(recursion_limit): // Max iterations safety
// 1. SELECT: find nodes whose subscribed channels have new data
eligible = []
        for node in graph.nodes:
            for ch in node.subscribed_channels:
                if ch.version > node.last_seen_version[ch]:
                    eligible.append(node)
                    break
        
        if not eligible:
            break                                       // No work → halt
// 2. EXECUTE: run eligible nodes in parallel with isolated state
        futures = []
        for node in eligible:
            state_copy = deep_copy(channels)         // Isolated snapshot
            futures.append(
                executor.submit(node.fn, state_copy)    // Parallel execution
            )
        
        results = await gather(futures)
        
        // 3. APPLY: merge updates deterministically
        for node, result in sorted(results):            // Deterministic order!
            for channel_name, value in result.items():
                channels[channel_name].update(value)   // Through reducer
                channels[channel_name].version += 1
            node.last_seen_version = current_versions()
        
        // 4. CHECKPOINT: save full state snapshot
        checkpointer.put(config, serialize(channels))
        
        // 5. STREAM: emit super-step events
        yield StreamEvent(step, eligible, channels)
    
    return channels[output_channel].value
```

> **Decision:** **Why Pregel/BSP and not simple sequential execution?** Sequential execution is simpler but can't parallelize independent nodes. Shared-memory parallelism risks data races. Pregel gives you the best of both: automatic parallelization when dependencies allow, but with the determinism guarantee that execution order never influences output. The trade-off: you must think in terms of "what channels does my node subscribe to" rather than "what runs after what." But this is precisely what enables features like checkpointing (the full state is available between super-steps) and streaming (each super-step boundary is a natural emission point).

> **Tip:** **Deterministic concurrency is the killer feature.** Imagine two nodes A and B both run in parallel and both append to `messages`. With naive parallelism, the order depends on which finishes first — non-deterministic! LangGraph applies updates in a fixed order (by node name or registration order), so the result is always the same regardless of timing. This is critical for debugging: if you see a bug, you can reproduce it exactly by replaying the same inputs.

### Deep Dive 2: State, Channels & Reducers (~7 min)

> **Goal:** **State is the data that flows through the graph.** It's defined by a typed schema (TypedDict or Pydantic). Channels are the containers for each field. Reducers define how concurrent updates to the same field are merged. This is the mechanism that makes parallel execution safe.

| Concept | What it is | Example |
| --- | --- | --- |
| State Schema | TypedDict or Pydantic model defining the shape of state | `class State(TypedDict): messages: list[BaseMessage]; summary: str` |
| Channel | Named container for a single state field. Has a current value and a version counter (monotonically increasing string) | `channels["messages"]` → holds the list of messages, version "3" |
| Reducer | Function that merges a new value into the existing channel value. Attached via `Annotated` type hints. | `messages: Annotated[list, add]` → new messages are appended, not replaced |
| No Reducer | Without a reducer, `update_state` overwrites the channel value directly | `summary: str` → last write wins |
| Channel Version | Bumped on every update. Used by the runtime to determine which nodes need to re-run (their subscribed channels have newer versions than they last saw) | Node "agent" last saw messages at v2, current is v3 → agent is eligible to run |

```sql
// State schema with reducers

from typing import Annotated
from operator import add

class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], add]    // Reducer: append new messages
next_agent: str                                     // No reducer: last write wins
tool_results: Annotated[dict, merge_dicts]        // Custom reducer: deep merge
iteration_count: Annotated[int, add]              // Reducer: sum increments
// What happens when two parallel nodes both write to "messages":
// Node A returns: {"messages": [msg_a]}
// Node B returns: {"messages": [msg_b]}
// Reducer `add` applies both: messages = old + [msg_a] + [msg_b]
// Order: deterministic (A before B, always), regardless of which finished first
```

> **Decision:** **Why channel versions instead of simple dirty flags?** Versions enable fine-grained scheduling. A node subscribes to specific channels. If only "summary" changes but the node only subscribes to "messages", it won't re-run — preventing unnecessary LLM calls. Versions also enable time-travel: you can inspect the exact state at any historical super-step by its version numbers.

### Deep Dive 3: Checkpointing & Persistence (~7 min)

> **Goal:** **Checkpointing transforms a stateless graph into a durable, resumable computation.** After every super-step, the full state is serialized and saved. This enables: resume from failure, human-in-the-loop pauses, time-travel debugging, and memory across conversations (threads).

| Concept | Description | Implementation |
| --- | --- | --- |
| Checkpoint | Serialized snapshot of all channel values, versions, and metadata at a super-step | Dict: `{v, ts, id, channel_values, channel_versions, versions_seen, pending_sends}` |
| Thread | A unique execution context (like a conversation). Each thread has its own checkpoint history. | `config = {"configurable": {"thread_id": "user-123"}}` |
| Checkpointer | Pluggable persistence backend implementing `put`, `get_tuple`, `list` | InMemorySaver (dev), SqliteSaver (light), PostgresSaver (production) |
| Pending Writes | If node A succeeds but node B fails in the same super-step, A's writes are saved as "pending" so A doesn't re-run on resume | Stored alongside checkpoint. On resume, pending writes are applied and only failed nodes re-execute. |
| Time Travel | Load any historical checkpoint and fork execution from that point | `graph.get_state_history(config)` → list of all checkpoints. `graph.invoke(input, config_with_checkpoint_id)` → fork. |
| Store (KV) | Cross-thread key-value storage for long-term memory (user prefs, facts) | InMemoryStore (dev), PostgresStore / RedisStore (production). Accessed via namespaced keys. |

```sql
// Checkpoint lifecycle
Super-step 1:
  Node "agent" runs → writes messages
  Checkpoint #1 saved: {messages: [user, ai], versions: {messages: 2}}

Super-step 2:
  Node "tools" runs → writes tool results
  Checkpoint #2 saved: {messages: [user, ai, tool], versions: {messages: 3}}

💥 Process crashes
Resume:
  Checkpointer.get_tuple(thread_id) → Checkpoint #2
  PregelLoop restores channels from checkpoint
  Continues from super-step 3 (no re-running steps 1-2)

Time Travel:
  graph.update_state(config_for_checkpoint_1, {"messages": [edited_msg]})
  → Forks from Checkpoint #1 with modified state
  → Execution diverges from the original timeline
```

> **Decision:** **Checkpoint sizing and TTL.** Every super-step writes a checkpoint. A 50-step agent creates 50 checkpoints. At 100KB each = 5MB per run. For a chatbot with 100 conversations/day × 30 days = 15GB/month. PostgresSaver supports TTL (time-to-live) to auto-delete old checkpoints. Blob storage offloads large state objects. The 1GB practical limit per checkpoint means extremely large state objects (e.g., full document embeddings) should be stored externally and referenced by ID.

### Deep Dive 4: Human-in-the-Loop (~7 min)

> **Goal:** **Human-in-the-loop is the most important production pattern for AI agents.** Users need to approve actions, edit drafts, provide feedback, and steer agent behavior. LangGraph implements this via the `interrupt()` function, which pauses graph execution at any point, saves state via checkpointing, and resumes when the human responds — even from a different server.

```sql
// Human-in-the-loop: draft → review → send

from langgraph.types import interrupt, Command

def draft_email(state):
    draft = llm.invoke("Draft an email about: " + state["topic"])
    
    // Pause execution, present draft to human
human_response = interrupt(
{"draft": draft, "prompt": "Edit this draft or approve"}
)
// Execution resumes HERE when human responds
    if human_response["action"] == "approve":
        return {"email": draft, "status": "approved"}
    elif human_response["action"] == "edit":
        return {"email": human_response["edited_draft"], "status": "edited"}

// What happens under the hood:
// 1. interrupt() saves current state as checkpoint
// 2. Graph execution halts, returns interrupt payload to caller
// 3. ... time passes (seconds, hours, days) ...
// 4. Human calls graph.invoke(Command(resume=response), config)
// 5. Checkpointer loads saved state
// 6. interrupt() returns human's response
// 7. Node continues from where it paused
```

| Pattern | How | Example |
| --- | --- | --- |
| Approve/Reject | interrupt() before a sensitive action. Human approves → action executes. Human rejects → agent takes alternative path. | Agent wants to send an email. Interrupt with draft. Human approves or asks for revision. |
| Edit State | `graph.update_state(config, new_values)` modifies any checkpoint field. Resume picks up the edited state. | Agent planned 5 steps. Human edits step 3 to change the approach before execution continues. |
| Time Travel | Load a past checkpoint, fork from that point. The agent re-executes from the historical state with optionally modified inputs. | "That response was wrong. Go back to before the search and try a different query." |
| Multi-Turn | Interrupt multiple times in a single run. Each interrupt saves state. Each resume continues from exactly where it left off. | Research agent: interrupt after gathering sources (review?), interrupt after draft (edit?), interrupt before publish (approve?). |

> **Tip:** **Why interrupt() is better than callbacks.** Callbacks force you to split your logic across multiple functions — the "before" function and the "after" callback. With interrupt(), the logic stays in one function, making it easy to read and debug. Under the hood, interrupt() uses checkpointing to serialize the execution state, which means it works across server restarts. This is what "durable execution" means in LangGraph's context: your code can pause indefinitely and resume exactly where it left off.

## Phase 05: Cross-Cutting Concerns *(10–12 min)*

### Failure Scenarios

| Scenario | Mitigation |
| --- | --- |
| Agent enters infinite loop (cycles endlessly) | recursion_limit— configurable max super-steps (default varies). When hit, execution halts and returns the current state. This is the primary safety valve against runaway agents. The limit counts super-steps, not LLM calls, so a single node making multiple LLM calls still counts as one step. |
| Node fails mid-super-step (other parallel nodes succeeded) | Pending writes.Completed nodes' outputs are saved as pending checkpoint writes. On resume, only the failed node re-executes. This prevents wasting LLM calls re-running nodes that already succeeded. If the same node fails repeatedly, the developer can implement retry logic within the node or use error callbacks. |
| Process crashes mid-execution | Last checkpoint is the recovery point. Checkpoints are saved after each super-step. On restart, load the latest checkpoint for the thread and resume. At worst, you lose the current in-flight super-step's work. With LangGraph Platform's task queue, the queue manager detects the crash and re-enqueues the work. |
| State grows too large (long conversations) | Multiple strategies: (1) Useentrypoint.final(value=result, save=trimmed_state)to control what's checkpointed vs. returned. (2) Summarize old messages instead of keeping full history. (3) Use the Store for large data and reference by key. (4) PostgresSaver supports up to 1GB per field; for larger objects, use blob storage with lifecycle rules. |
| LLM API rate limit or timeout | Node-level retry with backoff. LangGraph doesn't impose framework-level rate limits (unlike CrewAI's max_rpm) — developers implement retries within nodes or via LangChain's built-in retry mechanisms. Checkpointing ensures progress is saved even if retries eventually fail. |
| Checkpoint database becomes bottleneck | PostgresSaver uses connection pooling. Checkpoint TTL auto-deletes old data. Blob storage offloads large objects. For extreme scale, LangGraph Platform handles checkpoint infrastructure automatically. MongoDB has a 16MB document limit per checkpoint — PostgreSQL is preferred for large states. |

### LangGraph vs. CrewAI — Architectural Comparison

| Dimension | LangGraph | CrewAI |
| --- | --- | --- |
| Abstraction | Nodes + Edges + State Graph | Agents + Tasks + Crews + Flows |
| Philosophy | "No abstraction is the right abstraction." Low-level, maximum control. | Role-playing agents with natural language personas. Higher-level, opinionated. |
| Execution | Pregel/BSP algorithm with cyclic graphs, deterministic parallelism | Sequential / Hierarchical processes |
| State | Typed schema with channels, reducers, version tracking | Flow state (typed) + task-to-task context passing |
| Persistence | Built-in checkpointing (Postgres, SQLite, Redis) + KV Store | Memory system (short/long/entity) + SQLite |
| Human-in-loop | First-class `interrupt()` function with full state preservation | `human_input: true` on tasks |
| Debugging | "Debug the graph" — trace state transitions, edge conditions, checkpoints | "Debug the agent" — trace LLM reasoning, delegation chains |
| Learning curve | Higher: understand graph theory, channels, reducers, BSP model | Lower: define agents in YAML/Python, run |
| Flexibility | Maximum: any topology, any control flow, sub-graphs | Covers 90% of use cases simply |
| LangChain dep. | Ecosystem integration (optional but natural) | None (standalone) |
| Platform | LangGraph Platform: task queue, APIs, managed Postgres, LangSmith | CrewAI AMP: Studio, Control Plane, integrations |

### Streaming Architecture

#### Token Streaming [REAL-TIME]
- LLM tokens streamed as they're generated
- Zero buffering: token → user immediately
- `stream_mode="messages"`
- Works with any LangChain chat model

#### Node Events [STRUCTURED]
- Events when nodes start, finish, error
- Shows which node is active
- `stream_mode="updates"`
- Natural for progress indicators

#### State Values [SNAPSHOT]
- Full state emitted after each super-step
- Client sees cumulative state changes
- `stream_mode="values"`
- Best for UI state synchronization

#### Custom Events [FLEXIBLE]
- Developer emits arbitrary events from nodes
- Used for: progress bars, logs, metrics
- `stream_mode="custom"`
- Framework-agnostic payloads

### LangGraph Platform

#### Task Queue [INFRA]
- Decouples triggering from execution
- Retries with backoff on failure
- Horizontal scaling of workers
- Handles long-running background jobs

#### Assistants API [API]
- Templatize cognitive architectures
- Configure tools, prompts, models per assistant
- Version and deploy independently
- REST API for any client

#### Managed Persistence [STORAGE]
- Auto-managed Postgres checkpointer
- No manual checkpointer configuration
- TTL policies for data retention
- Blob storage for large state

#### Deployment Options [DEPLOY]
- LangGraph Cloud (managed SaaS)
- Self-hosted in your VPC
- 1-click deploy from LangSmith
- LangGraph Studio (visual debugger)

## Phase 06: Wrap-Up & Evolution *(3–5 min)*

> **Say:** "LangGraph is a graph-based agent runtime built on the Pregel/BSP execution model. The core architecture: typed state flows through channels with reducers. Nodes are plain Python functions that read/write state. Edges (including conditional) define control flow with full support for cycles. The runtime executes in super-steps with deterministic parallelism, checkpoints after each step, and streams intermediate results in real-time. This enables the six production features that agents need: parallelization, streaming, checkpointing, human-in-the-loop, task queues, and tracing. LangGraph Platform adds managed infrastructure for deployment at scale."

### Evolution & Future

| Phase | What Changed | Why |
| --- | --- | --- |
| Pre-LangGraph | LangChain Chains & Agents | Easy to start, but hard to customize and scale. Opinionated abstractions didn't fit production diversity. |
| LangGraph v0 (2024) | Graph API (no shared state) | First attempt at structured agents. Quickly deprecated — shared state was essential for real workflows. |
| LangGraph + StateGraph | Channels, reducers, Pregel runtime | The core architecture that shipped. Deterministic parallelism, checkpointing, streaming. Production-proven at LinkedIn, Uber. |
| Functional API | `@entrypoint` + `@task` decorators | Imperative alternative to StateGraph for developers who prefer writing sequential Python over declaring graphs. |
| LangGraph Platform | Managed deployment: task queue, APIs, persistence | The task queue was the one feature that required infrastructure beyond a Python library. |
| LangGraph 1.0 (alpha) | API stabilization, performance hardening | Production maturity. Signal that the API contract is stable for enterprise adoption. |
| Future: Distributed Execution | Nodes running on different machines | Agents are getting longer and more complex. Single-machine execution won't scale forever. Pregel/BSP was originally designed for distributed graph processing — LangGraph's architecture is naturally extensible to distributed execution. |

> **Tip:** **The Pregel bet.** Choosing Pregel/BSP as the execution algorithm was the most consequential design decision. It was originally designed by Google for distributed graph processing on clusters of machines. LangGraph uses it for single-machine agent execution today, but the architecture naturally extends to distributed execution in the future — where different nodes run on different servers. The super-step synchronization model, channel-based state passing, and checkpoint-per-step design all work identically in a distributed setting. This is why performance scales gracefully: the algorithm was designed for planet-scale computation, and LangGraph uses it for agent-scale computation.

## Phase 07: Interview Q&A — Practice Questions *(Practice)*

> **Goal:** **Q: Why did LangGraph choose Pregel/BSP over simpler approaches?**

> **Say:** "Three reasons: (1) Deterministic parallelism — execution order never influences output. (2) Native support for cycles, which agents need for tool-calling loops. (3) Natural checkpointing boundaries — the super-step synchronization points are exactly where you want to save state. DAG frameworks can't handle cycles. Simple sequential execution can't parallelize. Shared-memory parallelism risks data races. Temporal-style durable execution degrades with long histories and adds inter-step latency. Pregel gives us all the features we need with none of these limitations."

> **Goal:** **Q: How does LangGraph handle the case where two parallel nodes both write to the same state field?**

> **Say:** "Through reducers. Each state field can have an associated reducer function — for example, the `add` operator for lists. When two parallel nodes both write to `messages`, the reducer appends both updates in a deterministic order (by node name). Without a reducer, last-write-wins in deterministic order. This is the key mechanism that makes parallel execution safe: developers declare their merge strategy upfront, and the runtime applies it consistently."

> **Goal:** **Q: What happens if a long-running agent needs to pause for human approval and the server restarts?**

> **Say:** "This is exactly what checkpointing is designed for. When `interrupt()` is called, the full graph state is serialized and persisted to the checkpointer (e.g., PostgreSQL). The graph halts and returns the interrupt payload. Hours later — even after server restarts, deployments, or machine migrations — the human's response triggers `graph.invoke(Command(resume=response), config)`. The checkpointer loads the saved state, and execution resumes from exactly where it paused. The node function literally continues from the line after `interrupt()`."

> **Goal:** **Q: LangGraph vs. CrewAI — when would you pick each?**

> **Say:** "CrewAI when you want speed-to-value with a multi-agent pattern: define roles, goals, backstories in YAML, and run. It's opinionated and covers the common 'team of specialized agents' pattern well. LangGraph when you need maximum control over the execution topology: custom graph structures, fine-grained state management, complex branching/looping, or production-grade durability (checkpointing, human-in-the-loop). LangGraph is lower-level — more flexible, but more effort. The trade-off is abstraction vs. control, and it depends on whether the standard patterns fit your use case."

> **Goal:** **Q: How does LangGraph's streaming work and why is it important?**

> **Say:** "LLM calls take seconds to minutes. Without streaming, users stare at a blank screen. LangGraph supports four streaming modes: token-level (real-time LLM output), node events (which step is running), state values (full state after each super-step), and custom events (developer-defined). The graph structure makes this natural: each super-step boundary is a streaming emission point, and each node can stream its internal LLM tokens. Multiple modes can be combined. LangGraph Platform exposes this via server-sent events (SSE) over HTTP."

> **Goal:** **Q: How would you scale LangGraph to handle thousands of concurrent agent executions?**

> **Say:** "Three layers: (1) LangGraph Platform's task queue decouples incoming requests from execution, enabling fair scheduling and retry. (2) Horizontally-scaling worker servers, each running graph executions independently. Checkpoints go to shared Postgres, so any worker can resume any thread. (3) Connection pooling for the checkpointer (PostgresSaver with psycopg_pool). Future: the Pregel architecture was designed for distributed processing, so eventually nodes themselves could execute on different machines — though single-machine execution handles most current workloads."
