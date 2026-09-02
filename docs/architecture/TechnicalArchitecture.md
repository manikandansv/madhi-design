# Madhi — Holistic Technical Architecture

This document covers the complete system across all six phases. Phase-specific detail (types, function signatures, API contracts) lives in the LLD. This is the architectural decision record — what we use, where, and why.

---

## 1. What We Are Building

A product-aware wrapper around a foundation model, connected to a React frontend through a layered backend. The value is not the model — it is what context we inject, what we do with the output, and where we draw the line between AI and deterministic logic.

Three AI modes, used by task type:

| Mode | Used when | Example |
|------|-----------|---------|
| Deterministic | It's math or a rule lookup | Feasibility formula, tax gap calculation, variance |
| Single-call AI | Bounded extraction or one-shot judgment | Goal parsing, milestone suggestion, allocation narration |
| Agentic | Next step depends on previous result | Goal clarification loop, PDF statement parsing, alert investigation |

---

## 1.5 Architecture Direction — DDD + Event-Driven Design, with Akka as a later runtime choice

Madhi is best modeled as a domain-first system with explicit commands and domain events. The product has long-running workflows, AI-assisted interpretation, approval transitions, and future event processing, so the design should emphasize clear business boundaries and observable state change before adding an actor runtime.

### Recommended architectural split

- Domain layer: aggregate roots, value objects, commands, domain events, and business rules
- Application layer: workflow orchestration, policy checks, and coordination between domain logic and infrastructure
- Infrastructure layer: persistence, AI providers, caching, event transport, and monitoring
- Runtime option: Akka actors only when the workflow becomes genuinely concurrent, supervision-heavy, or failure-isolated enough to justify actor-based orchestration

### Core domain model

| Bounded context | Aggregate / root | Examples of commands | Examples of events |
|----------------|------------------|----------------------|--------------------|
| Intake | UserPlan | SubmitIntake, AddGoal, ParseGoals | IntakeReceived, GoalsParsed |
| Planning | GoalSet / MilestoneSet | GenerateMilestones, ApproveMilestone | MilestonesGenerated, MilestoneApproved |
| Approval | ApprovalWorkflow | SubmitDecision, SkipMilestone | DecisionRecorded, PlanReady |
| Monitoring | AlertStream | OnVarianceDetected, InvestigateAlert | AlertRaised, AlertResolved |

### Why DDD and event-driven design are the real foundation

- The business has meaningful state transitions: submit intake → parse goals → generate milestones → approve or skip → persist plan.
- AI calls are slow and failure-prone, but they should not leak across the whole application as ad hoc service logic.
- Approval and plan transitions are domain actions, not merely HTTP controller behaviors.
- Future statement parsing, alert investigation, and variance processing fit naturally into event-driven processing.

### When Akka becomes justified

Akka is not the default starting point. It becomes a strong choice when workflow complexity, failure isolation, retries, and concurrency exceed what a clean domain + async event model can efficiently handle. In other words:

- Start with DDD + events + application orchestration
- Add Akka only when the workflow becomes actor-heavy or supervision-heavy
- Keep the domain model stable even if the execution runtime changes later

### Runtime layout

- Spring Boot remains the ingress layer for REST APIs, validation, and security.
- The domain model owns the business logic and transitions.
- Asynchronous events coordinate internal steps and future integrations.
- Kafka or Redis Streams can be introduced later for durable async handoff between workflow stages.
- Akka is optional infrastructure for orchestration, timeout handling, and replay-safe workflow execution when complexity justifies it.

This is not a rejection of Spring Boot or Akka. It is a deliberate recommendation to keep the architectural center on the domain, and only use Akka where it earns its place in the runtime.

---

## 2. System Layers — All Phases

```
┌─────────────────────────────────────────────────────────┐
│                  Browser (React + TypeScript)            │
│  Intake form · Approval queue · Plan view · Dashboard   │
└──────────────────────────┬──────────────────────────────┘
                           │ HTTPS / WebSocket
                           ▼
┌─────────────────────────────────────────────────────────┐
│         Spring Boot API Layer (HTTP ingress + auth)     │
│  REST endpoints · validation · security · adapters       │
└──────────────────────────┬──────────────────────────────┘
                           │ commands/messages
                           ▼
┌─────────────────────────────────────────────────────────┐
│                Akka Domain Runtime (Reactive core)        │
│  IntakeWorkflowActor · GoalParserActor                  │
│  MilestoneEngineActor · ApprovalActor · EventRouter     │
│  Supervision · retries · timeout handling · state       │
└──────────────────────────┬──────────────────────────────┘
                           │ domain events / async jobs
          ┌────────────────┼──────────────────┐
          ▼                ▼                  ▼
   ┌─────────────┐  ┌─────────────┐  ┌────────────────────┐
   │ AI Providers│  │ Data + Cache│  │ Event / Queue      │
   │ Claude/OAI  │  │ Postgres /  │  │ Redis Streams /    │
   │ + prompts   │  │ Mongo / Redis│  │ Kafka (future)     │
   └─────────────┘  └─────────────┘  └────────────────────┘
```

---

## 3. Frontend — React + TypeScript

**Consistent across all phases.** What changes per phase is which views exist, not the technology.

| Phase | Views added |
|-------|------------|
| 1 | Intake form (7 steps), Approval queue, Plan view |
| 2 | Feasibility view, Path & tradeoffs panel |
| 3 | Allocation guidance view, Tax gap panel |
| 4 | Statement upload, Transaction list |
| 5 | Dashboard, Variance timeline, Alert feed |
| 6 | Adaptive milestone re-review flow |

Build tool: **Vite**. State: **React Context** (Phase 1–3) → evaluate Zustand if cross-view state becomes complex in Phase 5. Styling: **CSS modules**. No runtime CSS framework.

---

## 4. Backend Services — Java + Spring Boot

Thin and explicit. Java is the core team competency — the backend is on home ground. Spring Boot's ecosystem (spring-web, spring-kafka, spring-data-jpa) maps directly to each phase's backend needs. Kafka Streams — the Phase 5 stream processing library — is Java-native, making this the natural fit end-to-end. The backend never grows into a monolith — if a component becomes heavy (e.g. statement parsing), it moves to a separate Spring service or Kafka consumer worker, not a larger handler.

### Route inventory by phase

| Phase | Routes added |
|-------|-------------|
| 1 | `POST /parse-goal`, `POST /suggest-milestones` |
| 2 | `POST /compute-feasibility`, `POST /suggest-alternatives`, `POST /solve-goal-constraints` |
| 3 | `POST /compute-allocation`, `GET /tax-gap`, `POST /assess-risk-profile` |
| 4 | `POST /upload-statement` (with input validation gate — size, format, readability — before any processing), `GET /transactions`, `POST /categorize` |
| 5 | `GET /alerts`, `GET /variance`, `POST /dismiss-alert` |
| 6 | `POST /recompute-milestones` |

### Background job queue — Phase 4+

PDF parsing is too slow for a synchronous HTTP response. When a statement is uploaded:
1. Input validation (synchronous, in-request): file size, format, readability checked. Rejected files return immediately with a plain explanation — no job enqueued.
2. PII masking (synchronous, in-request): account numbers, customer names, IFSC codes masked server-side before the file is stored or any job is created. The stored file and the job payload contain only masked content.
3. Masked file stored → job enqueued in **Redis Streams**
4. Worker picks up job → runs PDF parsing agent → validates extracted totals against statement summary → writes result to MongoDB
5. Client polls or receives WebSocket notification when done

Redis Streams is used here rather than a separate message broker (RabbitMQ, SQS) because Redis is already in the stack for caching and rate limiting — no new infrastructure for Phase 4.

---

## 5. AI Wrapper

### Component inventory

```
server/src/main/java/com/madhi/ai/
├── PromptBuilder.java          # Assembles system + context + user input
├── OutputParser.java           # Schema validation — throws on malformed output
├── AgentRunner.java            # Multi-turn loop (Phase 4+, interface defined in Phase 1)
├── provider/
│   ├── AIProvider.java         # interface: complete() + runAgent()
│   ├── ClaudeProvider.java
│   └── OpenAIProvider.java
├── agents/                     # Task-specific agent definitions (Phase 4+)
│   ├── StatementParsingAgent.java
│   └── AlertInvestigationAgent.java
└── (resources) context/
    └── indian-finance.md       # Domain context injected into every prompt
```

### Provider interface

```java
public interface AIProvider {
    Object complete(String systemPrompt, String userMessage, Map<String, Object> schema)
        throws AIProviderException;
    AgentResult runAgent(String systemPrompt, List<AgentTool> tools, String initialMessage)
        throws AIProviderException;
}
```

`runAgent` is stubbed in Phase 1 (not called), implemented in Phase 4 when `StatementParsingAgent` is built. Nothing else in the codebase changes.

### Model selection

| Task | Model | Reason |
|------|-------|--------|
| Goal parsing | Claude Haiku | Bounded extraction; speed matters |
| Milestone suggestion | Claude Sonnet | Judgment + narrative quality |
| Priority weight extraction (multi-goal solver) | Claude Haiku | Bounded extraction from a fuzzy priority statement — same class as goal parsing |
| Solver narration (multi-goal output) | Claude Haiku | Short narrative explaining solver decision; cost-sensitive |
| Risk profile narration (allocation) | Claude Haiku | Short narrative explaining why allocation follows from profile; cost-sensitive |
| Risk questionnaire follow-up (contradiction flagging) | Claude Haiku | Targeted single-call; reuses goal-clarification pattern |
| PDF statement parsing agent | Claude Sonnet | Multi-step reasoning |
| Alert investigation agent | Claude Sonnet | Multi-step reasoning |
| Allocation narration | Claude Haiku | Short narrative; cost-sensitive |

### Evolution path

| Phase | AI capability added |
|-------|---------------------|
| 1 | Prompt engineering + structured output. indian-finance.md is the entire domain layer. |
| 1 | Agentic goal clarification (2-turn, client-side) |
| 2 | Priority weight extraction (Haiku) — converts fuzzy priority statement into structured weights for the constrained-optimization solver. Solver itself is deterministic; AI is not in the critical path. Solver narration (Haiku) — explains what the solver decided in plain language. Infeasibility narration (Haiku) — when no feasible split exists, explains tradeoff options without choosing one. |
| 3 | Risk questionnaire follow-up (Haiku) — optional open-ended question for edge cases the fixed questionnaire doesn't cover; reuses Phase 1 goal-clarification-loop, no new mechanism. Contradiction flagging (Haiku) — detects stated contradictions between questionnaire score and goal phrasing, surfaces to user, does not override. Risk profile narration (Haiku) — explains why the resulting allocation follows from the specific risk profile. |
| 3 | RAG (portfolio extension): tax-gap narration retrieves from a maintained corpus of Indian tax rule text and Finance Act circulars at prompt time. Tax arithmetic is deterministic and unchanged — only the explanatory narration layer is grounded this way. EPF/PPF/NPS contribution limits injected via `indian-finance.md` (static) until rule-change frequency warrants RAG retrieval for those too. |
| 3 | LLM-as-judge (portfolio extension, evaluation only): a separate model call scores generated narration — feasibility explanations, allocation rationale — against a rubric (accuracy to numbers, clarity, tone for stated experience level). Not in the product path; accumulates quality signals in the evaluation harness. |
| 4 | Input validation and PII masking run server-side before any AI call: file size/format/readability checks, account number and name masking — deterministic, no model involved. AgentRunner activated for PDF parsing — multi-step extraction pipeline. Output validation (totals reconciliation against statement summary) is deterministic arithmetic in the final pipeline step. |
| 5 | Alert investigation agent. Approved/rejected milestone data starts accumulating as labelled training set. |
| 5+ | Fine-tune base model on accumulated labelled decisions. Same ProviderClient interface — new implementation. |
| Long term | Distill fine-tuned outputs into a smaller domain-specific model. Lower cost, lower latency. |

---

## 6. Data Layer

This is the most consequential architectural decision in the product. The wrong choice doesn't matter in Phase 1 (localStorage). It matters from Phase 4 onward.

### Technology decision matrix

| Technology | Role in Madhi | Phase introduced |
|-----------|--------------|-----------------|
| **localStorage** | Phase1Plan — in-browser only | Phase 1 |
| **PostgreSQL** | Core relational data: users, plans, goals, milestones | Phase 4 |
| **MongoDB** | Document store: raw statement files, parsed transaction documents, alert event log | Phase 4 |
| **Redis** | Session cache, AI response cache, rate limiting, lightweight job queue (Phase 3 only) | Phase 3 |
| **Vector DB** (pgvector / Pinecone) | RAG knowledge base: tax rules, EPF/PPF/NPS content, semantic retrieval | Phase 3 |
| **Kafka** | Durable event streaming: transaction events fan out to DB, search, and alert consumers | Phase 4 |
| **Kafka Streams** | Real-time transaction categorization pipeline; running variance as events arrive | Phase 5 |
| **Elasticsearch** | Full-text + faceted transaction search | Phase 5 |
| **Cassandra** | High-volume time-series transaction events — only if scale demands it | Phase 5+ (conditional) |

---

### PostgreSQL — Core relational store

**What lives here:**
- `users` — account, profile metadata
- `phase_plans` — plan per user per phase, versioned
- `goals` — structured goal objects, FK to plan
- `milestones` — confirmed + skipped, FK to plan, source preserved
- `approval_events` — audit trail of every approve/skip action
- `financial_accounts` — user-declared accounts from statement uploads

**Why PostgreSQL:**
Goals link to milestones link to plans — this is relational data with referential integrity requirements. Phase 2's feasibility math joins goals × milestones × income capacity. Phase 3's tax gap joins income × deductions × investments. These are joins. A document store handles them poorly.

**Why not MySQL:**
PostgreSQL's `pgvector` extension means the vector store can run inside Postgres in Phase 3, avoiding a separate vector DB service until scale demands it. One fewer moving part.

---

### MongoDB — Document store

**What lives here:**
- `statement_documents` — raw uploaded files (path reference + metadata)
- `parsed_transactions` — extracted transaction rows from CSV/PDF. Schema varies by bank format — some have merchant codes, some don't; some have UPI reference IDs, some don't. Forcing this into a fixed relational schema wastes migration effort every time a new bank format is added.
- `alert_events` — each alert is a timestamped event with a variable payload (spending alert payload differs from income-change alert payload). Document structure fits better than a wide relational table.
- `intake_snapshots` — in-progress form state saved mid-session (before plan is confirmed)

**Why MongoDB and not PostgreSQL for these:**
Statement parsing output is variable-schema by design — no two banks structure their CSV the same way. A MongoDB document stores the raw parsed output as-is; downstream categorization normalizes what it needs. Forcing variable-schema data into PostgreSQL means either JSONB columns (which lose the query benefits) or constant schema migration as new bank formats are added.

**Why not Cassandra for this:**
Cassandra's operational complexity (tuning consistency levels, compaction, cluster management) is not justified for document storage at Madhi's scale. MongoDB handles the read/write patterns here with far less ops overhead.

---

### Redis — Cache and queue layer

**What lives here:**

| Use | Data | TTL |
|-----|------|-----|
| AI response cache | Hash of (profile + raw_text) → StructuredGoal | 24h — same input should return same parse |
| Rate limiting | Per-user AI call count | 1 min rolling window |
| Session | JWT + session state | 24h |
| Job queue | Statement parsing jobs (Redis Streams) | Until consumed |
| Real-time notification state | Alert IDs pending delivery | Until dismissed |

**Why Redis:**
Already in the stack for caching. Redis Streams is a lightweight, persistent queue sufficient for statement parsing job throughput. Adding RabbitMQ or SQS for Phase 4 would be a new infrastructure dependency for marginal benefit at Madhi's scale.

**AI response caching — why it matters:**
A user re-submitting an intake form mid-session, or a mobile user retrying after a dropped connection, should not burn an AI API call for a parse result already computed. Hashing the input and caching the response is a cheap, effective deduplication.

---

### Vector DB — RAG knowledge base

**What lives here:**
- Indian tax rules (80C/80D/Section 24b limits, old vs. new regime conditions)
- EPF/PPF/NPS rules (contribution limits, withdrawal conditions, employer matching)
- SEBI category definitions for MF buckets
- IRDAI guidelines relevant to health/life insurance

**Why:**
Tax rules change (Finance Act amendments). EPF rates change. Keeping this in `indian-finance.md` works in Phase 1 (static file, versioned). By Phase 3 when the tax gap feature runs, the file gets large enough that injecting the whole thing into every prompt is expensive. RAG retrieves only the relevant sections (e.g., for a salaried user on the new regime, retrieve only new-regime deduction rules — not the full 80C/80D tree).

**Technology choice — pgvector first, Pinecone later:**
Phase 3 starts with PostgreSQL's `pgvector` extension. Same database, no new service. If retrieval latency or vector count grows beyond what pgvector handles efficiently (~1M vectors), migrate to Pinecone or Weaviate. That migration touches only the RAG retrieval layer — nothing else.

---

### Elasticsearch — Transaction search

**What lives here:**
Indexed transaction records from MongoDB, synced via a simple change stream listener.

**What it enables:**
- Full-text search: "show me all Swiggy orders last quarter"
- Faceted filtering: category + amount range + date range + merchant
- Aggregations: total spend per category per month (for the dashboard)

**Why not PostgreSQL for this:**
Full-text search across transaction descriptions with fuzzy matching (merchant names are inconsistently formatted in bank statements) is where Elasticsearch's inverted index and analyzer pipeline genuinely outperform a LIKE query. PostgreSQL's `tsvector` works for simple cases but breaks down on unstructured merchant name matching.

**Why not Elasticsearch for primary storage:**
Elasticsearch is a search index, not a source of truth. MongoDB is the source; Elasticsearch is the search layer on top of it. Writes go to MongoDB first, then propagate to Elasticsearch. This separation means data is never lost if the Elasticsearch index needs to be rebuilt.

---

### Cassandra — Conditional

**What it handles well:**
High-volume, time-series write throughput. If Madhi reaches a scale where hundreds of thousands of users are each streaming transaction events in real time, Cassandra's distributed write performance outperforms PostgreSQL + MongoDB combined.

**Why we don't start with it:**
Cassandra requires explicit data modeling around query patterns at design time — there are no flexible queries after the fact. It has significant operational complexity (tuning consistency, managing compaction, cluster topology). At Madhi's scale through Phase 6, PostgreSQL with proper partitioning (PARTITION BY RANGE on `created_at` for transaction tables) handles the write volume without any of this complexity.

**Decision:** Do not introduce Cassandra until PostgreSQL transaction table partitioning is measurably insufficient. This is a Phase 5+ revisit, not a Phase 1–4 concern.

---

## 7. Streaming Platform

### Why a streaming platform is needed

Phase 4 introduces parsed transactions as a stream of events. Each transaction needs to reach multiple consumers:

1. **MongoDB writer** — persist the raw document
2. **Elasticsearch indexer** — make it searchable
3. **Alert engine** (Phase 5) — run variance detection
4. **Categorization pipeline** (Phase 5) — classify merchant/category

Delivering the same event to four consumers via direct HTTP calls creates tight coupling — every consumer must be available when the event is produced, and adding a fifth consumer means changing the producer. A streaming platform decouples producers from consumers entirely.

Redis Streams (used in Phase 4 for the statement parsing job queue) is sufficient for a single consumer. Once multiple consumers need the same event stream, Kafka is the right upgrade.

---

### Kafka — Event backbone (Phase 4+)

**Topics:**

| Topic | Producer | Consumers |
|-------|----------|-----------|
| `transactions.parsed` | Statement parsing worker | MongoDB writer, Elasticsearch indexer, alert engine |
| `statements.uploaded` | API route handler | Statement parsing worker |
| `alerts.fired` | Alert engine | Notification service, alert event log (MongoDB) |
| `milestones.approved` | Approval queue (backend sync) | Plan store, audit log, (future: ML training data sink) |
| `ai.calls` | AI Wrapper | Usage monitoring, cost tracking |

**Why Kafka over alternatives:**

| Alternative | Why not |
|-------------|---------|
| Redis Streams | Single consumer group; lacks Kafka's replay, partition, and multi-consumer-group model. Used in Phase 4 for the job queue; Kafka added in the same phase once multiple consumers need the transaction stream. |
| RabbitMQ | Message queue (point-to-point or pub/sub), not a log. No replay. Once a message is consumed, it's gone. Kafka retains the event log — essential for replaying transactions if the Elasticsearch index needs rebuilding. |
| AWS SQS/SNS | Vendor lock-in. Kafka (or managed Confluent / MSK) is portable. |

**Kafka's replay capability is essential for Madhi:** if the Elasticsearch transaction index is corrupted or needs to be rebuilt for a new schema, the raw transaction events in Kafka can be replayed in full. Without Kafka, rebuilding the index means re-parsing every uploaded statement — expensive and fragile.

---

### Kafka Streams — Stream processing (Phase 5)

Kafka Streams is a Java library that runs inside the application process — no separate cluster. It consumes from Kafka topics, applies processing logic, and writes results back to Kafka or directly to a store.

**Pipelines in Madhi:**

**Transaction categorization pipeline:**
```
transactions.parsed (Kafka topic)
  → Kafka Streams: lookup merchant in category table
  → if no match: emit to transactions.uncategorized
  → if match: emit to transactions.categorized
  → MongoDB: write categorized transaction
  → Elasticsearch: index categorized transaction

transactions.uncategorized (Kafka topic)
  → AI categorization worker (single-call AI)
  → transactions.categorized
```

**Variance detection pipeline:**
```
transactions.categorized (Kafka topic)
  → Kafka Streams: windowed aggregation (30-day rolling sum per category)
  → compare against plan's expected monthly spend per category
  → if deviation > threshold: emit to alerts.fired
```

The windowed aggregation is the key capability — Kafka Streams maintains a rolling 30-day sum per category per user without requiring a database query on every new transaction. The state store is embedded in the Kafka Streams instance.

---

### Apache Flink — Not used

Flink provides stateful distributed stream processing with more complex event-time semantics than Kafka Streams. It is appropriate for:
- Complex pattern detection across multiple event streams
- Exactly-once processing at very high throughput across a distributed cluster
- Real-time ML feature computation at scale

None of these apply to Madhi through Phase 6. Kafka Streams handles variance detection, categorization, and aggregation with far less operational overhead. Flink would be over-engineering the stream processing layer.

**Decision:** Do not introduce Flink. Kafka Streams is the ceiling for Madhi's stream processing needs through Phase 6. Revisit only if Flink-specific capabilities (complex event patterns, sub-second ML inference across streams) become genuinely needed.

---

### Apache Spark — Not used

Spark is a batch / micro-batch processing framework for large-scale data jobs — typically tens of millions to billions of records. It is appropriate for:
- Large-scale ETL across data lakes
- Distributed ML model training (Spark MLlib)
- Historical analysis of massive datasets

Madhi's transaction volumes (hundreds per user per month, even at 100K users = ~50M transactions/month total) are well within what PostgreSQL + Kafka Streams handle without distributed computing. Spark's operational complexity — cluster management, shuffle tuning, driver/executor resource sizing — is not justified.

For ML model fine-tuning (Phase 5+), the training dataset is Madhi's labelled approval/rejection decisions — at most millions of records, not billions. A standard ML training job (Python, PyTorch, cloud GPU) is sufficient. Spark is not the tool.

**Decision:** Do not introduce Spark. If ML training at scale becomes necessary, evaluate a managed training service (SageMaker, Vertex AI) before reaching for Spark.

---

### Streaming platform by phase

| Phase | What's added | Why now |
|-------|-------------|---------|
| 4 | Redis Streams (PDF parsing job queue, single consumer) + Kafka (transaction event fan-out). Topics: `statements.uploaded`, `transactions.parsed` | Statement parsing is async — needs a job queue. Parsed transactions fan out to MongoDB, ES, and alert engine; Redis Streams can't serve multiple consumer groups, so Kafka is added in the same phase. |
| 5 | Kafka Streams — categorization pipeline + variance detection | Real-time aggregation over transaction stream; can't do rolling windows efficiently without stream processing. |
| 5+ | `milestones.approved` topic → ML training data sink | Start accumulating labelled data for future fine-tuning via a dedicated consumer. |

---

### Data layer by phase — introduction sequence

| Phase | What gets added | Why now |
|-------|----------------|---------|
| 1 | localStorage | No backend. Plan is device-local. Acceptable for Phase 1. |
| 3 | Redis (cache + rate limit), pgvector (RAG for tax rules) | AI calls increase; caching prevents redundant spend. RAG needed as tax content grows. |
| 4 | PostgreSQL (users, plans, goals, milestones), MongoDB (statements, transactions), Redis Streams (job queue) | File uploads need server-side storage. Plan must survive device changes. Transactions need persistence. |
| 5 | Elasticsearch (transaction search), Redis (alert state) | Dashboard aggregations and transaction search are the primary Phase 5 interactions. |
| 5+ | Cassandra (conditional) | Only if PostgreSQL transaction partitioning is measurably insufficient at observed scale. |

---

## 8. Infrastructure Topology

### Phase 1–3 (local / minimal cloud)

```
Developer machine / single cloud instance
├── Spring Boot (Java) — port 8080
├── React (Vite) — port 5173
├── Redis — port 6379
└── PostgreSQL + pgvector — port 5432
```

No container orchestration. `docker-compose` to run dependencies locally. Deploy on a single cloud VM (Fly.io, Railway, or Render) for Phase 3 preview.

### Phase 4–5 (production-ready)

```
Load Balancer (nginx / cloud LB)
├── Spring Boot API service (horizontal scale if needed)
├── Statement parsing worker (separate Spring Boot app — Kafka consumer)
├── PostgreSQL (managed — Supabase or RDS)
├── MongoDB Atlas (managed)
├── Redis Cloud (managed)
├── Elasticsearch (Elastic Cloud or OpenSearch Serverless)
└── Vector DB (pgvector on Postgres, or Pinecone if volume demands)
```

Managed services for the data layer — not self-hosted. The engineering investment should go into the product, not database operations.

### What doesn't change between phases

- The React frontend stays React.
- The Spring Boot backend stays Spring Boot (new routes, same framework).
- The AI wrapper interface (`AIProvider`) stays the same — new implementations, not a new interface.
- The `indian-finance.md` context file stays as the cheapest quality lever.

---

## 9. Phase-by-phase additions summary

| Phase | Frontend | Backend routes | AI mode | Data added | Streaming added |
|-------|----------|---------------|---------|-----------|----------------|
| 1 | Intake, Approval, Plan view | `/parse-goal`, `/suggest-milestones` | Single-call + agentic (2-turn) | localStorage | — |
| 2 | Feasibility view, Path panel, Multi-goal solver | `/compute-feasibility`, `/suggest-alternatives`, `/solve-goal-constraints` | Single-call (narration + priority weight extraction); solver is deterministic | — | — |
| 3 | Allocation view, Tax gap, Risk profile | `/compute-allocation`, `/tax-gap`, `/assess-risk-profile` | Single-call + RAG; risk questionnaire scoring is deterministic | Redis, pgvector | — |
| 4 | Statement upload, Transaction list | `/upload-statement`, `/transactions`, `/categorize` | Deterministic (input validation, PII masking, totals reconciliation) + Agentic (PDF pipeline) | PostgreSQL, MongoDB | Redis Streams (job queue) + Kafka (fan-out) |
| 5 | Dashboard, Alert feed, Variance | `/alerts`, `/variance` | Agentic (alert investigation) | Elasticsearch | Kafka Streams (categorization + variance) |
| 6 | Milestone re-review | `/recompute-milestones` | Reuses Phase 2 + 1 | — | — |

---

## 10. What this architecture is not

- Not a monolith. Every component is a small, replaceable Spring Boot service or worker.
- Not a custom AI model. The foundation model is a dependency, swappable via one env var.
- Not fine-tuned until there is labelled data from real usage to fine-tune on — that comes in Phase 5+.
- Not Cassandra from day one. PostgreSQL with partitioning is the right call until scale proves otherwise.
- Not Flink. Kafka Streams handles Madhi's stream processing needs — Flink's distributed cluster model is not justified.
- Not Spark. Madhi's transaction volumes and ML training needs don't require distributed batch computing. A managed training service is the right call if fine-tuning requires GPU at scale.
- Not Kubernetes from day one. Docker Compose locally, managed cloud services in production, container orchestration only when horizontal scaling is actually needed.
