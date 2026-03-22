# Hercules SWE AI Agent Interview Grind Guide (Deep Version)

This version assumes:
- You already know modern webdev (TypeScript, React/Next.js, Supabase-style backend workflows, OAuth, API calling).
- You do NOT yet have deep systems/infra/agent-evals depth for this JD.

Goal: bridge you from strong product engineer to credible AI-agent systems engineer fast.

---

## First: Honest Answer About Resource Order

Your question was: Is the previous order correct?

Honest answer: not fully.

The previous order was decent as a topic list, but suboptimal for learning velocity in one day. It mixed foundations and advanced topics too early. For your current background, the better order is:

1. Agent architecture mental model (what system are we even building?)
2. API design + data modeling + reliability (core backend architecture)
3. Containers + monorepo + runtime/dev workflow (execution environment)
4. Data stores: Postgres -> Redis -> ClickHouse (operational role separation)
5. Context engineering (RAG, retrieval, compaction, caching)
6. Evals (offline + online, regression discipline)
7. Observability (Otel) + cloud deployment patterns (Cloudflare/AWS)
8. Stack specifics (Hono, Drizzle, Terraform/CDK)
9. Agent harnesses (Codex, Claude Code, Cursor, Copilot) and Skills

Why this order is better:
- You already know app dev. You need architecture primitives and production constraints first.
- Retrieval/caching/evals only click when API + data + system boundaries are clear.
- Harness comparison is most useful after you understand agent system internals.

---

## 1) AI Agent Product Architecture (for something like Hercules)

Hercules-like systems are not “just chatbots.” They are distributed product systems with AI decision loops.

### Core runtime loop

At high level:
1. User request arrives (often ambiguous, long, multi-turn).
2. System builds context (history + memory + retrieved docs/code + tool schema).
3. Model plans and executes through tool calls/subagents.
4. Results are validated (tests/lints/checks/policies/evals).
5. State is stored (conversation, artifacts, traces, outcomes).
6. System iterates or asks for approval.

### Real architecture boundaries you should speak to

1. Client/editor boundary
- Web app/editor where users define intent and review outputs.
- Streaming UX, state sync, optimistic updates.

2. Orchestration API boundary
- Request normalization, authz, rate limits, tenant isolation.
- Session lifecycle + model routing decisions.

3. Agent runtime boundary
- Tool execution, subagent orchestration, retries, cancellation, timeouts.
- Context assembly and compaction.

4. Retrieval boundary
- Ingestion/chunking/indexing.
- Query rewriting + hybrid ranking + filters.

5. Execution sandbox boundary
- Code exec/build/test in isolated envs.
- Strong security controls and approval flows.

6. Telemetry/eval boundary
- Traces, metrics, logs, quality scores.
- Offline benchmark suites + online monitors.

### Best resources

- https://martinfowler.com/articles/microservices.html
- https://martinfowler.com/bliki/MonolithFirst.html
- https://developers.cloudflare.com/agents/

Use this interview line:
“I’d start with a modular monolith and strict internal boundaries, then split services only where operational pressure proves it.”

---

## 2) API Design (what “design” actually means in senior contexts)

You asked what API design even means beyond RESTfulness.

REST style is one part. API design is contract engineering under change.

### What senior API design includes

1. Domain contract clarity
- Resource boundaries map business concepts, not DB tables.
- Request/response types align with product workflows.

2. Evolution safety
- Additive changes by default.
- Explicit deprecation/migration process.
- Versioning strategy chosen deliberately (URI/header/media type).

3. Operational behavior
- Idempotency on mutation endpoints where retries are likely.
- Async job endpoints for long-running actions.
- Clear failure semantics + machine-readable errors.

4. Performance shape
- Pagination/filter/sort/projections.
- Avoid N+1 and chatty endpoint patterns.
- Caching strategy and invalidation policy.

5. Observability and governance
- Correlation IDs, trace propagation.
- SLO-aware latency/error instrumentation.
- AuthN/AuthZ and tenant isolation patterns.

### Resource

- https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design

If asked “what makes an API good?” give this:
“Stable contracts, explicit semantics under failure, and predictable evolution without breaking downstream clients.”

---

## 3) Monorepos (what they are and what they are not)

A monorepo is a source-control strategy, not an architecture style.

- You can run microservices in a monorepo.
- You can run a monolith in a polyrepo.

### Why teams like Hercules often prefer monorepos

1. Atomic cross-cutting changes
- Update shared types + backend contract + frontend usage in one PR.

2. Contract consistency
- Shared schema/types reduce drift and integration bugs.

3. Build graph optimization
- Affected-only test/build runs via dependency graph.

4. Platform standardization
- Unified lint/test/release and internal tooling.

### Real risks

1. Tooling immaturity -> slow CI.
2. Weak ownership boundaries -> accidental coupling.
3. Over-sharing libraries -> hidden dependencies.

### Resource

- https://nx.dev/docs/concepts/decisions/why-monorepos

---

## 4) Containers + Dev Environments + Git Flow

You had a key question: “How does git flow differ in containers?”

Short answer: conceptually, almost not at all.

### What containerization changes

1. Deterministic runtime
- Your Node, package manager, CLIs, system libs are reproducible.

2. Team onboarding
- Faster ramp-up, fewer local-env snowflakes.

3. Security/isolation
- Reduced blast radius for tooling and build/test execution.

4. Deployment parity
- Better dev-prod consistency when images align.

### What usually stays the same

1. Branch/commit/push workflow.
2. Repo structure and PR process.
3. Most git commands.

### Where people get tripped up

1. Mounted files + permissions.
2. Credential forwarding (SSH, HTTPS tokens).
3. Line endings (Windows + WSL + container).

### Resources

- https://docs.docker.com/get-started/docker-overview/
- https://docs.docker.com/get-started/introduction/build-and-push-first-image/
- https://code.visualstudio.com/docs/devcontainers/containers
- https://containers.dev/

---

## 5) Data Stores by Job: Postgres vs Redis vs ClickHouse

This separation is interview-critical.

### Postgres (system of record, OLTP)

Use when:
- You need transactions, relational integrity, joins, constraints.
- Product-critical canonical state.

Resource:
- https://www.postgresql.org/docs/current/tutorial.html

### Redis (speed layer, state acceleration)

Use when:
- Hot-path cache, session/token, rate limiters, queues/pubsub.
- Fast ephemeral state where durability constraints are different.

Resource:
- https://redis.io/docs/latest/develop/get-started/
- https://redis.io/docs/latest/commands
- https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/

Senior talking points:
- Single-threaded command path means predictable fast ops.
- Bad key design or expensive commands can tank latency.
- TTL/eviction strategy is architecture, not tuning trivia.

### ClickHouse (analytical OLAP)

Use when:
- Large-scale aggregations and analytics over huge event data.
- Near-real-time dashboards and experimentation analysis.

Resource:
- https://clickhouse.com/docs/en/intro

Senior talking points:
- Columnar storage optimizes scans/aggregates, not transactional row updates.
- Keep OLTP and OLAP concerns separated.

---

## 6) Context Engineering (RAG, Retrieval, Compaction, Prompt Caching)

This is a core JD differentiator.

### 6.1 RAG and retrieval

RAG = retrieval-augmented generation: pull relevant external knowledge and feed model grounded context.

#### Advanced concerns that matter in production

1. Ingestion quality
- Parsing, cleaning, chunking, overlap choices.

2. Retrieval quality
- Query rewriting, hybrid lexical+semantic ranking, score thresholds.

3. Precision controls
- Metadata filters (tenant, project, recency, permissions).

4. Grounding and trust
- Citation strategy, source spans, provenance logging.

5. Cost and freshness
- Reindex policies, expiration, stale-context controls.

Resources:
- https://developers.openai.com/api/docs/guides/retrieval
- https://developers.openai.com/api/docs/guides/embeddings
- https://redis.io/docs/latest/develop/get-started/rag/

### 6.2 Prompt caching

Prompt caching reduces repeated prefill compute when prompt prefix is identical.

#### What matters practically

1. Prefix discipline
- Keep static system/tool/schema/context first.
- Put per-request dynamic content near the end.

2. Hit-rate engineering
- Normalize invariant prompt sections.
- Track cache metrics and tune structure.

3. Correctness understanding
- Caching changes latency/cost profile, not model semantics.

Resources:
- https://developers.openai.com/api/docs/guides/prompt-caching
- https://platform.claude.com/docs/en/docs/build-with-claude/prompt-caching

### 6.3 Compaction/summarization and subagents

As threads grow, naive full-history context is expensive and noisy.

Good pattern:
1. Keep short-term verbatim window.
2. Keep durable summaries for older turns.
3. Keep retrieval-backed facts externalized.
4. Use subagents to isolate complex subtasks and limit context bleed.

Resources:
- https://developers.openai.com/api/docs/guides/compaction
- https://developers.openai.com/api/docs/guides/conversation-state
- https://developers.openai.com/api/docs/guides/tools

---

## 7) Evals (the discipline most candidates miss)

Evals answer: “Is this system better, or just different?”

### Offline evals

- Curated test sets, repeatable, versioned.
- Used for regression detection and controlled improvements.

### Online evals

- Live traffic quality monitoring.
- Catching real-world drift, edge-case failures, and UX regressions.

### Evaluation dimensions

1. Task correctness.
2. Latency + cost.
3. Safety/policy compliance.
4. Reliability under tool/use-case variance.
5. User-rated outcome quality.

### Resources

- https://developers.openai.com/api/docs/guides/evals
- https://developers.openai.com/api/docs/guides/evaluation-best-practices
- https://developers.openai.com/api/docs/guides/graders
- https://docs.langchain.com/langsmith/evaluation

Interview line:
“We treat prompts and agent policies like code: benchmarked, versioned, and regression-tested.”

---

## 8) Observability + Reliability for Agent Systems

If you cannot observe it, you cannot improve it.

### What to instrument

1. Request lifecycle traces (user request -> retrieval -> tool calls -> response).
2. Tool failure rates and latency distributions.
3. Cache hit/miss and context token economics.
4. Eval outcomes over time by cohort/model/version.

### OpenTelemetry role

- Vendor-neutral telemetry standard for traces/metrics/logs.
- Lets you move or combine backends without rewiring instrumentation.

Resource:
- https://opentelemetry.io/docs/what-is-opentelemetry/

---

## 9) Cloud + Infra (Cloudflare, AWS, Terraform, CDK)

### Cloudflare

Why it matters here:
- Edge execution, low latency, globally distributed workloads.
- Strong fit for API/agent endpoints and lightweight orchestration.

Resources:
- https://developers.cloudflare.com/workers/
- https://developers.cloudflare.com/agents/
- https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API

Note: Service Workers (browser) and Cloudflare Workers (edge runtime) are related by concept (event-driven runtime + request interception mindset) but are different technologies.

### IaC: Terraform and CDK

Terraform:
- Declarative infra state management across providers.
- Great for platform-standardized provisioning.

CDK:
- Define AWS infra in code with language-level abstractions and constructs.

Resources:
- https://developer.hashicorp.com/terraform/docs
- https://github.com/aws/aws-cdk

---

## 10) Framework/ORM relevance in this stack (Hono + Drizzle)

### Hono

- Lightweight, fast, edge-friendly web framework.
- Good for Cloudflare Workers style deployments and API-first services.

Resource:
- https://hono.dev/docs/

### Drizzle ORM

- SQL-forward TypeScript ORM with strong typing and explicit schemas.
- Useful when you want type-safe DB access without hiding SQL realities.

Resource:
- https://orm.drizzle.team/docs/overview

---

## 11) Agent Harnesses: How to Compare Codex / Claude Code / Cursor / Copilot

Don’t compare by “which feels smartest.” Compare by architecture and operating model.

### Evaluation rubric

1. Session model
- How context persists, resumes, and transfers.

2. Tooling model
- Tool APIs, approvals, sandboxing, policy hooks.

3. Execution model
- Local vs background vs cloud agents, parallelism controls.

4. Reliability model
- Retry behavior, self-correction, test-run integration.

5. Governance model
- Instruction hierarchy, memory controls, auditability.

### Official docs

- https://developers.openai.com/codex
- https://code.claude.com/docs/en/overview
- https://cursor.com/docs
- https://docs.github.com/en/copilot
- https://code.visualstudio.com/docs/copilot/overview

---

## 12) Skills (important, and yes this can matter a lot)

You asked to include Skills, and you’re right.

### What a skill is

A skill is a reusable capability package (instructions + assets/scripts) an agent can invoke for a known workflow.

Typical components:
1. `SKILL.md` manifest/instructions.
2. Optional helper scripts/templates/examples.
3. Versioning and distribution mechanism.

### Why skills are high leverage

1. Encodes team-specific best practices once.
2. Reduces prompt entropy and ad-hoc behavior.
3. Improves consistency across engineers and agent runs.

### Why skills are risky

1. They can influence tool execution and data handling.
2. Poorly scoped skills can cause overreach or unsafe actions.
3. Need explicit approval for high-impact operations.

Resource:
- https://developers.openai.com/api/docs/guides/tools-skills

---

## 13) Reordered 1-Day Plan (optimized for you)

This is the practical sequence I recommend now.

### Block A (2h): Architecture + APIs first

1. Read:
- https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design
- https://martinfowler.com/bliki/MonolithFirst.html

2. Output:
- One architecture sketch (boundaries listed above).
- One API contract checklist (idempotency, async jobs, versioning, tracing headers).

### Block B (1.5h): Monorepo + container execution model

1. Read:
- https://nx.dev/docs/concepts/decisions/why-monorepos
- https://code.visualstudio.com/docs/devcontainers/containers

2. Output:
- Explain monorepo value and risks in 90 seconds.
- Explain containerized git workflow and credential model in 90 seconds.

### Block C (2h): Data store role separation

1. Read:
- https://www.postgresql.org/docs/current/tutorial.html
- https://redis.io/docs/latest/develop/get-started/
- https://clickhouse.com/docs/en/intro

2. Output:
- Decision matrix: what goes into Postgres vs Redis vs ClickHouse.

### Block D (2.5h): Context engineering (core AI depth)

1. Read:
- https://developers.openai.com/api/docs/guides/retrieval
- https://developers.openai.com/api/docs/guides/prompt-caching
- https://platform.claude.com/docs/en/docs/build-with-claude/prompt-caching

2. Output:
- Design a retrieval pipeline (ingest -> index -> search -> grounded response).
- Define prompt structure optimized for cache hit rates.

### Block E (1.5h): Evals + observability

1. Read:
- https://developers.openai.com/api/docs/guides/evals
- https://developers.openai.com/api/docs/guides/graders
- https://opentelemetry.io/docs/what-is-opentelemetry/

2. Output:
- Offline eval suite proposal.
- Online eval + telemetry dashboard KPI list.

### Block F (1.5h): Stack + harnesses + skills

1. Read:
- https://developers.cloudflare.com/workers/
- https://developers.cloudflare.com/agents/
- https://hono.dev/docs/
- https://orm.drizzle.team/docs/overview
- https://developer.hashicorp.com/terraform/docs
- https://github.com/aws/aws-cdk
- https://developers.openai.com/codex
- https://code.claude.com/docs/en/overview
- https://cursor.com/docs
- https://docs.github.com/en/copilot
- https://developers.openai.com/api/docs/guides/tools-skills

2. Output:
- 5-minute “how I’d build and operate Hercules-like agent infra” verbal walkthrough.

---

## 14) Interview Conversion: how to sound senior, fast

### Use this structure in answers

1. Clarify goal + constraints.
2. Propose architecture boundaries.
3. Call out tradeoffs and failure modes.
4. Define metrics and validation plan.
5. Explain rollout strategy and fallback path.

### Phrases that help (and are true)

1. “I optimize for evolvability under change, not just clean diagrams.”
2. “I separate system-of-record storage from speed layer and analytics layer.”
3. “I treat retrieval quality and eval discipline as product features.”
4. “I use observability to close the loop between architecture intent and runtime behavior.”

---

## 15) Quick Reference Links (preserved, curated)

### Architecture/API/Monorepo
- https://martinfowler.com/articles/microservices.html
- https://martinfowler.com/bliki/MonolithFirst.html
- https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design
- https://nx.dev/docs/concepts/decisions/why-monorepos

### Containers + Dev workflow
- https://docs.docker.com/get-started/docker-overview/
- https://docs.docker.com/get-started/introduction/build-and-push-first-image/
- https://code.visualstudio.com/docs/devcontainers/containers
- https://containers.dev/

### Data stores
- https://www.postgresql.org/docs/current/tutorial.html
- https://redis.io/docs/latest/develop/get-started/
- https://redis.io/docs/latest/commands
- https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/
- https://clickhouse.com/docs/en/intro

### Context engineering
- https://developers.openai.com/api/docs/guides/retrieval
- https://developers.openai.com/api/docs/guides/embeddings
- https://redis.io/docs/latest/develop/get-started/rag/
- https://developers.openai.com/api/docs/guides/prompt-caching
- https://platform.claude.com/docs/en/docs/build-with-claude/prompt-caching
- https://developers.openai.com/api/docs/guides/tools
- https://developers.openai.com/api/docs/guides/compaction
- https://developers.openai.com/api/docs/guides/conversation-state

### Evals + observability
- https://developers.openai.com/api/docs/guides/evals
- https://developers.openai.com/api/docs/guides/evaluation-best-practices
- https://developers.openai.com/api/docs/guides/graders
- https://docs.langchain.com/langsmith/evaluation
- https://opentelemetry.io/docs/what-is-opentelemetry/

### Stack-specific + harnesses
- https://developers.cloudflare.com/workers/
- https://developers.cloudflare.com/agents/
- https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API
- https://developer.hashicorp.com/terraform/docs
- https://github.com/aws/aws-cdk
- https://hono.dev/docs/
- https://orm.drizzle.team/docs/overview
- https://developers.openai.com/codex
- https://code.claude.com/docs/en/overview
- https://cursor.com/docs
- https://docs.github.com/en/copilot
- https://code.visualstudio.com/docs/copilot/overview
- https://developers.openai.com/api/docs/guides/tools-skills

---

If you want next, I can create a second file with:
- likely founder interview questions,
- your ideal answer skeletons,
- and a 2-hour mock drill script based on this exact guide.