# FireFoundry Core Concepts & Glossary (AgentSDK)

> A concise, opinionated reference you can drop next to your setup guide. It explains what the moving pieces are, why they exist, and when to use each—plus an A→Z glossary.

---

## Who this is for

- Engineers building with **FireFoundry** using the **AgentSDK** (TypeScript-first)
- Readers who already have the platform installed (see your separate Setup & Installation guide)

## How to use this page

- **New to the stack?** Read the **Mental model** and **Core building blocks**.
- **Already building?** Jump to **Decision guides** and the **Glossary**.

---

## Mental model (the 20-second tour)

FireFoundry separates **state** and **behavior**, then gives you orchestration primitives to scale both.

```mermaid
flowchart LR
  subgraph Data & State
    E[Entities\n(typed state + methods)]
    G[(Entity Graph / Storage)]
  end
  subgraph Reasoning & Behavior
    B[Bot\n(LLM-driven behavior)]
    P[Prompts & Tool Specs]
    D[Dispatch Table\n(runtime tool bindings)]
  end
  subgraph Orchestration
    R[Runnable Entity\n(typed invocable unit)]
    W[Waitable\n(bidirectional streaming + yield)]
    O[Workflows\n(fan-out/fan-in, retries)]
  end

  E <--> G
  B -->|uses| P
  P -->|declares| D
  R -->|invokes| B
  R -->|reads/writes| E
  W -->|interactive run| B
  O -->|coordinates| R
```

**In practice**: You model your business objects as **Entities** (typed with Zod), express behavior in **Bots** (LLM + prompts + tools), wire them via **Runnables** and **Waitables**, and coordinate at the workflow layer when you need parallelism, retries, and supervision.

---

## Core building blocks

### 1) Entities

**What**: Typed, persisted domain objects (schemas + methods).\
**Why**: Keep facts/state separate from LLM reasoning; enable idempotent workflows.\
**Use when**: You need durable state, history, or graph relationships.\
**Gotchas**: Be explicit about IDs and idempotency in methods called from workflows.

### 2) Bots

**What**: Reasoning units that run prompts/tools against a model pool.\
**Why**: Centralize LLM behavior behind a typed interface.\
**Use when**: You want reliable schema-validated outputs and tool calling.\
**Gotchas**: Keep prompts small, deterministic; prefer tool calls for high-precision tasks.

### 3) Prompts & PromptGroups

**What**: Structured authoring for instructions, examples, and I/O contracts.\
**Why**: Make behavior explicit and testable; encourage reuse across Bots.\
**Use when**: Multiple tasks share patterns (classification, extraction, planning).\
**Gotchas**: Overfitting to examples; drift between prompt tool specs and runtime bindings.

### 4) Tool Specs & Dispatch Tables

**What**: Tool specs live in prompts (LLM-facing); **dispatch tables** bind those specs to code (runtime).\
**Why**: Separate **interface** (what the LLM can call) from **implementation** (what actually runs).\
**Gotchas**: Keep names/args identical; validate inputs; throttle side-effects.

### 5) Runnable Entities

**What**: Typed invocations that compose Bots + Entities.\
**Why**: Package a unit of work that can be scheduled, retried, and observed.\
**Use when**: A step needs its own retries/metrics or is reused across workflows.

### 6) Waitables

**What**: Interactive, bidirectional runs that support **yield** and **sendMessage**.\
**Why**: Great for chat-style UX, human-in-the-loop, or progressive plans.\
**Use when**: You need mid-flight updates, confirmations, or tool gating.\
**Gotchas**: Always consume the iterator; define timeouts and cancelation semantics.

### 7) Workflows & Orchestration

**What**: Higher-level coordination (fan-out/fan-in, retries, cancellation).\
**Why**: Make reliability/latency tunable; keep business logic readable.\
**Use when**: Multi-step plans, parallel branches, backoffs, or compensations.

### 8) Model Pools

**What**: Named model configurations (e.g., `firebrand_completion_default`).\
**Why**: Decouple code from vendor/models; enable routing and fallbacks.\
**Use when**: You need consistent performance/cost SLAs across environments.\
**Gotchas**: Document pool names centrally; add per-task overrides sparingly.

### 9) Working Memory

**What**: **Persistent**, per-agent/per-thread store for attachments and artifacts (e.g., user- or system-provided files, tool outputs, plans, code/data snippets, internal monologue artifacts).

**Why**: Manage context economically—selectively surface items into prompts only when needed without burning tokens every turn.

**Use when**: You want to persist non-canonical artifacts that enrich future interactions (retrieved docs, generated notes/code, partial plans, intermediate results).

**Gotchas**: Persistent but **not** canonical business state—treat as an attachable scratchpad. Promote/reflect to **Entities** when an artifact becomes part of the source of truth. Establish retention/PII policies and quotas.

### 10) Types, Schemas & Helpers Types, Schemas & Helpers

- **Zod schemas** ensure outputs and tool args are validated.
- **Schema metadata** (e.g., `withSchemaMetadata`) enriches docs/UX/tooltips.
- **Type Helpers**:
  - **PTH** = PromptTypeHelper (typed prompt I/O & tool specs)
  - **BTH** = BotTypeHelper (typed bot inputs/outputs)
  - **RETH** = RunnableEntityTypeHelper (typed invocations)

---

## Glossary (A → Z)

**Ad hoc Tool Call**  
A tool invocation surfaced dynamically during a Bot run (declared in the prompt, bound at runtime via the dispatch table). Validate args; consider allowlists/rate limits.

**Bot**  
A reasoning unit configured with prompts, tool specs, and a model pool. Produces schema-validated outputs (via Zod). See also **BTH**.

**BotRequest / BotResponse / BotTry**  
Lifecycle structs capturing inputs, outputs, and attempts for a Bot invocation; useful for logging, retries, and analytics.

**BTH (BotTypeHelper)**  
Type helper for defining typed inputs/outputs for Bots.

**Cancellation Token**  
Signal propagated through workflows/waitables to abort work safely; pair with compensations for side effects.

**Dispatch Table**  
Runtime binding from LLM-declared tool names → concrete functions. Must match names/args in the prompt’s tool specs.

**Entity**  
Typed, persisted domain object with methods. Forms the durable source of truth and graph of your app’s state.

**Entity Graph**  
The network of Entities and relations; stored/persisted and accessed by Bots/Workflows.

**Model Pool**  
A named configuration selecting a family of models/providers and policies (fallbacks, rate limits). Keeps code vendor-agnostic.

**Orchestration**  
Coordinating multi-step work: sequencing, parallelism, retries, backoffs, and cancellation.

**PTH (PromptTypeHelper)**  
Type helper to define prompt inputs/outputs and declare tool specs in a typed way.

**Prompt / PromptGroup**  
Structured authoring constructs for instructions, examples, and I/O contracts; commonly grouped for reuse across Bots.

**RETH (RunnableEntityTypeHelper)**  
Type helper for defining typed invocations around Bots/Entities.

**Retries & Backoff**  
Controlled re-execution of idempotent steps after transient failures; pair with jitter and limits.

**Runnable Entity**  
A typed, invocable unit that composes Bots and Entities. Designed for reuse, observation, and independent retries within workflows.

**Tool Spec**  
LLM-facing description of a callable function (name, parameters, description). Implemented at runtime via the **Dispatch Table**.

**Waitable**  
An interactive, iterator-based run supporting `yield` and `sendMessage` for mid-flight questions, approvals, and tool-gating; ideal for chat/HITL UX.

**Working Memory**  
**Persistent**, attachable memory for artifacts (attachments, tool outputs, internal plans/code/data) associated with an agent/thread. Not auto-included in prompts; selectively loaded into context. Promote or sync to **Entities** when artifacts become canonical.

**Zod / Schema Metadata**  
Schema system used to validate inputs/outputs and tool arguments. `withSchemaMetadata` adds human-readable annotations for docs/UX.