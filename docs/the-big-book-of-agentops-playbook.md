# The Big Book of AgentOps — Operational Playbook

*A distilled, operational companion to Databricks' **The Big Book of AgentOps** — a guide to
deploying agentic applications that are reliable, observable, safe, scalable, and maintainable.*

> **Source & attribution.** This playbook condenses the publicly published Databricks eBook
> *The Big Book of AgentOps* (Pavithra Rao, Jeanne Choo, Kyra Wulffert, Alex Baur) into a single
> operational reference. It is a summary for practitioners, not a replacement — read the original
> for the full treatment.
> - Announcement: <https://www.databricks.com/blog/announcing-databricks-big-book-agentops>
> - eBook (PDF): <https://www.databricks.com/sites/default/files/2026-08/2026-07-eb-the-big-book-of-agent-ops-final.pdf>
>
> Product names and APIs referenced here (MLflow, Unity Catalog, the AI Gateway, Databricks Asset
> Bundles, Databricks Apps, Agent Bricks, Vector Search, Genie, Lakebase) are Databricks platform
> features; the evaluation APIs are part of open-source MLflow. Product capabilities and lifecycles
> change over time — verify specifics against current documentation before relying on them.

---

## Contents

1. [What AgentOps is (the mental model)](#1-what-agentops-is-the-mental-model)
2. [The principles (the non-negotiables)](#2-the-principles-the-non-negotiables)
3. [The AgentOps anti-patterns](#3-the-agentops-anti-patterns)
4. [Deployment architecture patterns](#4-deployment-architecture-patterns)
5. [The seven-phase project pipeline](#5-the-seven-phase-project-pipeline)
6. [DevOps principles for agents (Flow, Feedback, Continuous Learning)](#6-devops-principles-for-agents-flow-feedback-continuous-learning)
7. [Evaluation & observability](#7-evaluation--observability)
8. [Governance, access control & cost](#8-governance-access-control--cost)
9. [Stakeholder management](#9-stakeholder-management)
10. [Worked example: a telco customer-support agent](#10-worked-example-a-telco-customer-support-agent)
11. [Staying current (model & product lifecycle)](#11-staying-current-model--product-lifecycle)
12. [Sources & further reading](#12-sources--further-reading)

---

## 1. What AgentOps is (the mental model)

**An agent** perceives its environment, decides, and acts toward a goal. It is autonomous,
reactive, and proactive, and it can interact with other agents and with humans. That autonomy —
multi-step tool use against real systems — is exactly what makes an agent operationally harder to
run than a single LLM call.

**The lineage:**

- **MLOps** (2010s) — operationalizing data and models.
- **LLMOps** (early 2020s) — tuning, deploying, and managing LLMs at scale.
- **AgentOps** (2024+) — building, evaluating, deploying, governing, and monitoring autonomous
  systems that reason, plan, and take actions.

Agents add concerns the earlier disciplines didn't have to solve: permission boundaries,
multi-step tracing, cascading-failure prevention, retry and recovery, and orchestration that
doesn't loop.

**The six core AgentOps challenges** that everything else serves:

1. Leveraging enterprise-specific context
2. Reliability
3. Observability
4. Safety
5. Scalability
6. Maintainability

**The platform stack (as the book frames it on Databricks):**

- **MLflow is the core** — the unified layer for **evaluation** (`mlflow.genai.evaluate()` with
  LLM-judge and rules-based scorers), **observability** (MLflow Tracing: span-level capture), and
  **feedback/monitoring** in production (the Assessments UI and a feedback-logging API).
- **The AI Gateway** (in open source, the MLflow AI Gateway) is the single control plane every LLM
  call routes through — traffic management, budget policies, and per-component cost attribution.
  **It belongs in every pattern, not just the complex ones.**
- **Unity Catalog** is the AI-asset registry and the place where governance is enforced — grants on
  tables, functions, tools, and MCP servers; on-behalf-of-user identity passthrough; and audit via
  system tables.
- **Deployment bundles** (infrastructure-as-code: code + jobs + models + config, promoted
  dev → staging → prod and deployed atomically) are the standard versioned deployment unit.
- **Managed agent tooling and code-first agents on application hosting** are the two broad ways to
  build and serve.

---

## 2. The principles (the non-negotiables)

Six principles from the book, extended by the three classic DevOps principles. Treat them as
defaults you deviate from only with a stated reason.

1. **Start narrow, expand deliberately.** One well-scoped use case with clear success metrics beats
   a broad "answer everything" agent that is mediocre at all of it.
2. **Evaluation is a first-class citizen.** Build evaluation *alongside* the agent, never after.
   Evaluation is a revenue generator — it de-risks and accelerates delivery — not a cost center.
3. **Process guides automation.** Map the human workflow first, then automate deliberately. Start
   with manual expert review, discover the real quality criteria, *then* automate them into judges
   and checks.
4. **Version everything** — prompts, tools, **routing logic**, and data sources. Agent behavior
   emerges from all four; if they aren't versioned, you can't debug or roll back.
5. **Design for observability.** Every decision point and tool call must be traceable and
   attributable, cost included. If you can't see it, you can't operate it.
6. **Humans in the loop for high-stakes actions.** Consequential, irreversible actions get an
   explicit approval gate.

Extending the classic DevOps trio:

- **Flow** — deploy changes fast *and* reliably; don't trade speed for quality.
- **Feedback** — short, rich loops from experts, users, judges, and metrics.
- **Continuous learning** — codify hard-won lessons into reusable frameworks, reference
  architectures, and design patterns.

---

## 3. The AgentOps anti-patterns

These "seem reasonable in theory but create problems in practice." Naming the anti-pattern a team
has walked into is often the highest-leverage thing you can do.

| Anti-pattern | Problem | Risk | Do instead |
|---|---|---|---|
| **Starting too broad** | Undefined scope, unclear success criteria (e.g. "a chatbot that answers every question") | Agents mediocre across all domains rather than excellent in one | Focus on a single, well-scoped use case |
| **Overcomplex architecture** | Supervisor / multi-agent when a simple sequential chain would suffice | Orchestration overhead, debugging complexity, infinite loops between agents | Prefer sequential chains; reach for multi-agent only when the problem truly needs it |
| **ReAct loops for single-tool tasks** | Wrapping predetermined actions in a reason-and-act loop | Added latency and cost with no benefit when the next action is never in doubt | Use direct function calls for deterministic actions |
| **Overcomplicated tools** | LLM-powered tools (e.g. text-to-SQL) where a parameterized query would work | Latency, unpredictability, security risk | Reserve LLM-based tool selection for genuinely complex mapping |
| **Unoptimized retrieval** | Accepting the first RAG implementation without measuring or improving it | Poor outputs from irrelevant or missing context | Evaluation-driven retrieval optimization |
| **Ungoverned tools** | Agents with unrestricted endpoint access and broad credentials | Security, compliance, and reliability failures that cascade | A centralized, governed tool registry |
| **No rollback or versioning** | Behavior emerges from prompts + tools + routing + data — none versioned | Can't debug or recover from production failures | Version-control **every** agent component |
| **No systematic evaluation** | Manual spot-checks instead of structured test suites | Failure modes invisible in dev that surface at scale in prod | Standardize on programmatic evaluation suites |
| **No human in the loop (high-stakes)** | Agents autonomously executing consequential actions | Unacceptable risk when agents misread edge cases | Mandate human-in-the-loop for high-stakes actions |
| **No cost controls** | No per-request budgets, rate limits, or cost tracking | Runaway costs from expensive tool calls, excessive iterations, unbounded retrieval | Proactive budgets and limits at the AI Gateway |

**The subtle one — stakeholder-driven metric distortion.** Manual review by stakeholders matters,
but it becomes an anti-pattern when **subjective feedback from one influential stakeholder
disproportionately drives the roadmap**. Guard against it: define quality metrics
**cross-functionally and up front**, communicate them clearly, and measure against the shared,
written criteria — not the loudest voice.

---

## 4. Deployment architecture patterns

Complexity grows along **two independent axes**: **account/workspace isolation** (operational
complexity and compliance strength) and **agent specialization** (coordination complexity and
domain optimization).

> **Prime directive:** choose the **simplest architecture that meets your current needs**, then
> evolve incrementally. Don't start at the most complex pattern because you might need it someday.

### The four patterns

**Pattern 1 — Single account, single agent.** *Complexity: low.* One monolithic agent, one focused
use case, across dev/staging/prod in a single account. Declarative config with per-environment
overlays; one metastore; the AI Gateway enabled; catalog/schema isolation per environment; a single
Git repo with automated dev → staging → prod promotion; unit / integration / validation gates.

**Pattern 2 — Single account, multi-agent.** *Complexity: medium.* An orchestrator plus specialized
sub-agents in one account. Each agent deployed independently (microservices style); a **supervisor
agent** routes and coordinates; supports parallel execution and sequential chaining. Shared schemas
enforced with a validation library; the AI Gateway for consistent access, policies, tracing, and
spend across models, agents, and tools; a centralized tracking server aggregating across agents;
per-agent CI/CD with cross-agent integration testing in staging.

> The multi-agent *pattern* is valid — but the recurring anti-pattern is reaching for it too early.
> Confirm a sequential chain genuinely won't do, and implement the supervisor as **your own custom
> orchestrator** rather than depending on a managed product whose lifecycle you don't control
> (see [§11](#11-staying-current-model--product-lifecycle)).

**Pattern 3 — Multi-account, single agent.** *Complexity: medium-high (simple app logic, complex
infrastructure).* One agent deployed across strictly isolated accounts or workspaces (e.g. a
separate account per environment for compliance). Infrastructure-as-code bundles are critical for
environment abstraction — account-specific targets, parameterized catalog/schema names, and
secrets injected at deploy time. Multiple metastores; cross-account data sharing; cross-account
networking (e.g. PrivateLink); the AI Gateway to standardize runtime governance and cost
attribution while preserving model/provider choice.

**Pattern 4 — Multi-account, multi-agent.** *Complexity: high — needs a dedicated AgentOps/MLOps
team and mature governance.* The full matrix of agents × accounts × environments. Hierarchical
config (base agent configs + environment overlays + account overrides) with schema validation and
pre-deployment validation jobs; the AI Gateway as the unified runtime control layer across models,
agents, tools, MCP servers, and harnesses; federated tracking servers; multi-stage promotion with
cross-account gates and per-agent-per-account rollback. Observability, cost attribution, and
security all operate at agent, account, and environment levels simultaneously.

### Choosing the right pattern

| Pattern | When to use | Team maturity | Primary challenges |
|---|---|---|---|
| Single account, single agent | Starting out, PoCs, single-purpose agents | Beginner | Initial setup, basic CI/CD |
| Single account, multi-agent | Composite agents, orchestration, microservices | Intermediate | Agent coordination, shared resources |
| Multi-account, single agent | Compliance, environment isolation, enterprise governance | Intermediate–advanced | Cross-account networking, IAM complexity |
| Multi-account, multi-agent | Enterprise scale, many teams, complex systems | Advanced | Full operational overhead; a dedicated AgentOps team |

The 1 → 4 progression *is* an organization's maturity journey. Walk it; don't teleport to the end.

### Common pipeline components (shared by all patterns)

Regardless of complexity, every pattern shares the same foundations:

- **Source control** — Git with a branch strategy aligned to environments; PR review; CI/CD on commit.
- **Testing strategy** — unit (dev) / integration + regression (staging) / validation (prod).
- **MLflow integration** — automated and human evaluation of quality; end-to-end tracing;
  production monitoring and feedback.
- **Lakehouse foundation** — Unity Catalog as the AI-asset registry and policy layer; Delta/Iceberg
  storage; vector search; governed AI tools and functions.
- **Runtime governance & control** — the AI Gateway for centralized governance across models,
  agents, and tools; runtime permissions/guardrails; usage observability, cost attribution, and
  budgets; model routing and provider management.
- **Deployment automation** — infrastructure-as-code bundles, per-environment parameter injection,
  automated rollback.
- **Feedback loops** — expert validation, continuous monitoring/evaluation, metrics-driven
  improvement.

Databricks maintains reference scaffolding ("AgentOps Stacks") that implements these patterns as a
recommended starting point for a new project.

---

## 5. The seven-phase project pipeline

A systematic path from conception to continuous improvement. A widely cited figure holds that
**~95% of AI pilots fail**; the bridge is **measurement capability** — the evaluation that earns
trust from users and leadership. Projects **iterate between phases**; that's expected, not a failure.

1. **Project conception & team formation**
   - Identify a focused, measurable business use case; avoid overly broad problems or ones better
     solved with deterministic workflows.
   - Assemble a cross-functional team: leadership stakeholders + technical developers + domain
     experts with real expertise in the agent's domain.
   - Align on deliverables, success metrics, and governance; educate leadership on AI limitations;
     plan a communication cadence.

2. **Data preprocessing & indexing**
   - Design pipelines optimized for **AI consumption**, not classic analytics — unstructured data,
     vector indexes, embedding generation, tool-calling, LLM-based extraction.
   - Build production data infrastructure: ACID transactions, quality monitoring, failure handling,
     compliance.

3. **Agent architecture design**
   - Select the right complexity and framework (custom vs. code-first frameworks vs. low-code).
     **Avoid over-engineering with supervisor/multi-agent patterns when deterministic chains
     suffice.**
   - Plan evaluation cross-functionally — define what "good" looks like *with* domain experts and
     end users. You cannot calibrate a judge if the success metric isn't clear.
   - Establish guardrails and safety: input validation, output monitoring, tool access controls,
     operational limits.

4. **Agent development & evaluation framework**
   - Treat development as first-class **alongside** evaluation, with evaluation signals driving what
     gets built next.
   - Use the **testing pyramid for AI**: start with manual expert review to learn quality patterns,
     then scale through automated LLM judges and programmatic checks; do both offline testing and
     online production monitoring.
   - Design for **cost-effective evaluation** — LLM-based evaluation costs money (unlike unit
     tests). Use token-efficient strategies, field-based vs. trace-based evaluation, and smart
     sampling.

5. **Feedback collection & monitoring**
   - Integrate multi-source feedback: end-user feedback + expert validation + LLM-judge assessments
     + system metrics; both structured pre-production reviews and real-time production collection.
   - Instrument production telemetry and observability: operational metrics (latency, cost), quality
     metrics (eval scores, satisfaction), business impact (ROI, efficiency), plus data protection
     and audit trails. **Watch for runaway spend** — agent costs *multiply* (every sub-agent, retry,
     and guardrail adds cost; one request can become several LLM calls). The AI Gateway is the fix.

6. **Iterative development & optimization**
   - Run systematic cycles: collect → analyze → hypothesize → targeted improvement → validate →
     deploy. Focus on prompt engineering, tool selection, architecture tuning, and cost optimization.
   - Build trust through transparency: regular demos, metric transparency, honest failure analysis,
     user education. Turn evaluation data into reusable organizational assets.

7. **Scaling & governance**
   - Expand pilots enterprise-wide: cost management, quality consistency, organizational change,
     infrastructure scaling. Getting one agent to production is a milestone; running **dozens across
     business units** is a different discipline.
   - As agents proliferate you hit a governance problem — who owns what, what data they touch, what
     drives cost. Stand up an **AI governance board** (see [§8](#8-governance-access-control--cost)).

**The payoff.** The pipeline turns experimental pilots into strategic assets: faster time-to-value
through repeatable processes, lower risk by addressing failure modes proactively, consistent quality
at scale, measurable ROI, and AI maturity that compounds across projects.

---

## 6. DevOps principles for agents (Flow, Feedback, Continuous Learning)

DevOps isn't new, but it must be updated for AgentOps, because generative AI breaks naive DevOps:

- **Nondeterministic behavior** — the same prompt can yield different outputs, so traditional
  pass/fail testing is insufficient.
- **Unstructured interfaces** — natural language, images, and audio need different evaluation
  methods than structured-data APIs.
- **Context-dependent quality** — output quality depends on context, user intent, and domain
  knowledge that's hard to encode in a fixed test suite.
- **Expensive evaluation** — LLM-based evaluation consumes tokens and money, so test-suite design
  must be strategic.

### Principle 1 — Flow

Reduce the time to deploy changes to production **while improving** reliability and quality.

**Build a golden evaluation dataset (the enabling step).** Unlike traditional software, you can't
define all tests up front. GenAI needs a preliminary step: **analyze traces of prior
requests/responses to identify common failure modes.** Only after failure modes are identified and
categorized can you define automated checks for them.

- **The dangerous default** ❌ — one tester runs five or six queries in a chat UI and eyeballs
  whether each looks acceptable. This captures no error-frequency metrics across a broad input
  distribution and classifies no failure types.
- **The scientific approach** ✅ — apply the scientific method iteratively:
  1. **Categorize failure modes** quickly using **~100 traces.** In production, sample real traces
     (diverse across input categories, conversation lengths, and tool-call types). Still in
     development? Work with an expert to define inputs across the range of scenarios, and — because
     LLM calls are nondeterministic — **run each input 3–4 times** to measure consistency.
  2. **Create failure reports** with visualizations.
  3. **Calibrate automated LLM-judge metrics** from those insights plus domain-expert consultation.
     The output is a set of scorers that scale trace evaluation across dev and prod.
  4. **Run scorers offline and online** — offline against a dataset or dataframe; online by
     collecting production traces into an analytical store and batch-evaluating a sample with the
     **same** scorers.
  5. **Surface aggregated metrics in dashboards** and have PMs/business experts periodically review
     and refine the suites against **new** failure modes seen in production.

**Evaluations are a revenue generator, not a cost center.** Reframe evaluation from "necessary evil"
to a strategic investment that de-risks AI, accelerates reliable deployment, and builds durable
advantage. This reframing is often the key to getting evaluation work funded.

### Principle 2 — Feedback

GenAI blends software and ML, so teams must collect operational telemetry (latency, cost, errors),
model-quality signals (precision, recall), **and** new GenAI feedback forms: expert feedback,
LLM-judge outputs, and rules-based checks. Evaluate every feedback signal for:

- **Consistency** — experts may score the same output differently; establish inter-rater reliability
  with clear rubrics and regular calibration.
- **Representativeness** — end-user ratings suffer selection bias (only the very happy or very angry
  tend to rate).
- **Alignment** — LLM judges must be continuously validated against human judgment; a judge that
  agrees with *itself* but diverges from experts is false confidence.
- **Actionability** — "this response is bad" isn't actionable; structure feedback to reveal whether
  the issue is retrieval, reasoning, tone, or facts.

**The feedback flow (three phases):**

- **Pre-production** — experts label data in review sessions; developers analyze traces to find and
  fix quality issues; an evaluation dataset (inputs, outputs, metrics) is built; automated suites
  test the application.
- **Production** — deploy with tracing enabled (beta first, then full production); a real-time
  dashboard tracks performance; users give in-app feedback; production traces are continuously
  evaluated.
- **Continuous improvement (the loop that compounds)** — evaluation results inform the next
  iteration; selected traces and human reviews are **added back** to the evaluation dataset; judges
  are **re-aligned** after each round of expert feedback; aligned judges then become scorers that
  drive automated prompt optimization.

This enables an **expert-driven development loop**: expert feedback triggers workflows that improve
monitoring (via aligned judges) and directly improve agent quality (via prompt optimization).
Initial cross-functional alignment on scope and expected outputs is critical; after that, automation
can drive continuous improvement, with manual gates wherever you want them.

**Feedback best practices:**

- **Start with boolean feedback** (thumbs up/down); expand to numeric or structured ratings once you
  see patterns.
- **Attribute sources** — record who or what produced each judgment, with timestamps, for a full
  audit trail.
- **Name consistently** — standardize assessment names so search and aggregation stay meaningful.
- **Combine programmatic and UI collection** — APIs for automated capture, UI for manual review.
- **Link feedback to fresh traces** — collect immediately after generation while context is
  available.

**Analyze manually first, then automate.** Example: you start with a generic "relevance" judge. An
expert disagrees and corrects it in the labeling UI. You realize "relevance" is too general and
split it into sharper criteria (e.g. *solution appropriateness*, *context alignment*), yielding a
custom judge that reflects real quality standards. The pattern — **manual expert review → discover
criteria → automate them into judges/checks** — is fundamental because requirements emerge through
exploration, domain expertise catches nuances engineers miss, and automation scales human judgment.
A common failure is building in isolation and discovering fundamental misalignment too late. Early,
continuous expert involvement prevents wasted effort.

### Principle 3 — Continuous learning

GenAI moves fast: what was hard yesterday is easy tomorrow, so stakeholders must continuously update
their mental model of what's buildable. Organizations rarely want *one* successful app — they want
to repeatedly ship high-quality apps across departments. Scale by **encoding hard-won knowledge into
reusable assets**:

1. **Standardized frameworks** — make development function like an assembly line. Deployment bundles
   are the canonical example: a resource collection + an infrastructure-as-code package + a versioned
   deployment artifact + a dependency container, promoted dev → staging → prod and deployed via
   external CI/CD.
2. **Reference architectures** — proven blueprints of design patterns and best practices (e.g. in
   regulated industries, route every request/response through a guardrail step that filters PII).
   They create shared understanding of *how* and *why*, capturing ~70–80% of architectural decisions
   up front and leaving teams the 20–30% specific to their use case. (These are the deployment
   patterns from [§4](#4-deployment-architecture-patterns).)
3. **Industry- and enterprise-specific agentic design patterns** — blueprints for how the **agents
   themselves** should be designed. Two illustrations from the book:
   - **Insurance claims processing** — event-driven; workflows must **pause and resume**, so
     engineers need a **state store** that checkpoints a paused step and an agent sub-graph that can
     resume at specific points (not rerun from scratch). Shared design elements across departments:
     a common state-store schema and a common resume-event payload.
   - **Financial-services compliance** — agents monitor regulatory updates, interpret policy
     changes, route ambiguous cases for human review, and maintain audit trails; document and reuse
     these across compliance/audit use cases.

**The compounding effect:** teams that codify patterns, align judges with domain experts, and reuse
components move from one-off agents to a **reliable, ever-improving portfolio** of agentic systems.
Continuous learning isn't purely technical — collaborative metric development keeps evaluation
aligned with business intent, and educating business stakeholders builds the shared understanding
needed to act on feedback.

---

## 7. Evaluation & observability

The single most important AgentOps capability. **Core stance:** evaluation is a first-class citizen,
built alongside the agent, never bolted on afterward.

### The testing pyramid for AI

Build bottom-up:

1. **Manual expert review** — start here to *learn the quality patterns* and discover criteria.
2. **Automated LLM judges** — encode the discovered criteria so they apply consistently at scale.
3. **Programmatic / rules-based checks** — deterministic assertions (formats, guardrails, tool
   correctness).
4. **Online production monitoring** — the same scorers run against sampled production traces.

Do **both** offline testing (pre-deploy) and online monitoring (post-deploy), using the same scorers
so dev and prod quality are measured the same way.

### Two complementary ways to design a test suite

- **Top-down** — reason from the design about what *must* work and write those tests up front,
  without any traces. (The billing agent needs a tool to query customer spend, so write a test that
  the tool returns correct results.) You can also generate personas and scenarios and have an LLM
  synthesize sample queries.
- **Bottom-up** — some failure modes only appear after **error analysis of real traces** (e.g. an
  LLM SQL step mixing up product acronyms). Sample ~100 traces, categorize failures, then align
  LLM-judge outputs to expert feedback to create automated tests.

### Make quality measurable: structure the outputs

Quality is far easier to define when outputs are **structured JSON or binary true/false**, because
established metrics like **F1** apply. **Prioritize structured responses over free text** wherever
possible; for genuinely unstructured fields (transcripts, chat logs, notes), develop domain-aligned
LLM judges.

**Build a "thin slice" first** — a simplified version of the workflow, just complex enough to
visualize how data flows through it, rather than covering the entire business process at once. For
each substep, define expected inputs and outputs; those become concrete, runnable checks.

**Routing evaluation is programmatic, not LLM-judge.** A supervisor's routing decision is a
*structured output* (which sub-agent), so evaluate it with **accuracy / F1 / a confusion matrix**,
and gate **per-agent and end-to-end** — not with an LLM judge.

### The MLflow evaluation surface

The open-source MLflow GenAI APIs back this playbook's Flow and Feedback loops:

- **`mlflow.genai.evaluate()`** — run scorers over a dataset or dataframe (offline); the same
  scorers run online over sampled production traces.
- **Scorers** — built-in scorers (correctness, guidelines, safety, retrieval groundedness, …) plus
  custom scorers. A useful pattern is to wrap them in an opinionated base class so teams get running
  fast:

  ```python
  from abc import ABC, abstractmethod

  class BaseScorer(ABC):
      def __init__(self, name, sample_rate):
          self.name = name
          self.sample_rate = sample_rate      # sampling keeps online evaluation cheap

      @abstractmethod
      def get_judge_scorer(self):
          """Return an LLM-judge-based scorer for this criterion."""
          ...
  ```

- **A unified judge interface** for judge-based evaluation.
- **Judge alignment** — continuously tune judges to expert feedback so a judge tracks experts rather
  than agreeing only with itself.
- **Automated prompt optimization** — run it right after an alignment job so aligned judges directly
  drive prompt improvements. This is the automation payoff of the expert-driven loop.

### Observability by persona

Build one trace-and-feedback foundation (tracing + an assessments UI + a feedback API), then expose
per-persona views:

- **Business/domain expert** — a conversation UI with a trace side-panel and a feedback control that
  writes assessments, plus a summary report; served as an app so reviewers need no platform login.
- **Executive sponsor** — an ROI dashboard built from the stored metrics.
- **End user** — an app surface with preset prompts, cited sources, and a like/dislike control tied
  back to their specific trace.

### Cost-effective evaluation

LLM-based evaluation costs money. Use sampling for online scorers, prefer field-based over
trace-based evaluation where it suffices, and reserve the most expensive judges for the cases that
need them.

---

## 8. Governance, access control & cost

Agentic systems need governance at three layers: **who each agent/tool can act as** (access
control), **what it costs** (the AI Gateway), and **who owns the whole portfolio** (the governance
board). Enforce all of it in the platform, not in prompt logic.

### The two-level permission model (non-negotiable)

Permissions apply at **two distinct levels**. Getting this wrong is a security incident, not just a
bug — it can expose sensitive data to unauthorized viewers.

1. **Agent- and tool-level permissions (least privilege).** Each sub-agent operates with the least
   privilege it needs. In a telco example: the billing sub-agent reads spend/invoice tables but has
   **no** write access to account settings; the account-management sub-agent can write contact
   preferences but **only behind an explicit confirmation step**; the technical-support sub-agent
   reads diagnostics but **not** billing data; the escalation path can *open* a human-review ticket
   but **not** resolve it autonomously. Scoping tools this narrowly **contains the blast radius** —
   even if the LLM is manipulated into calling a tool inappropriately, the tool can't act outside its
   granted scope.
2. **End-user-level permissions (identity flows through).** One deployed agent serves many users, so
   the **end user's identity and permission scope must flow through every asset accessed.** Two
   patterns:
   - **Identity passthrough (on-behalf-of-user)** — the agent queries **as** the end user, so their
     existing row- and column-level policies apply automatically. Prefer this.
   - **Explicit parameterization** — where passthrough isn't available, inject the authenticated
     user/customer ID as a **trusted, non-LLM-controlled filter** on every query. **Never take the
     identity filter from the model's output** — it can be manipulated via prompt injection.

**Enforce controls centrally.** Express both levels through the catalog/governance layer: agent-level
scope as **grants** on the tables, functions, tools, and MCP servers each agent can reach (on an app
host, defined in the config that maps which assets the app's service principal can access); end-user
scope as row/column-level security via identity passthrough. Centralizing in the governance layer —
not in agent or prompt logic — keeps permissions **auditable, consistent across agents, and
resistant to prompt injection.** Every tool call and data access lands in an audit trail that feeds
observability and compliance reporting.

### Cost control via the AI Gateway

**Agent costs multiply, they don't add.** Every sub-agent, retry, and guardrail check adds cost on
top; one user request can become several LLM calls behind the scenes. The recurring failure mode is
discovering runaway spend *only after the invoice arrives.*

The **AI Gateway** is the fix and belongs in **every** deployment pattern. It is a centralized
control plane for every LLM call that:

- **Routes all traffic through one point** (also enabling model routing and provider management).
- **Autologs** token counts, latency, model, and cost per call.
- **Attributes spend across the full span hierarchy** (orchestrator → sub-agent → synthesis →
  guardrail), so you can pinpoint which component drives cost *before* optimizing the wrong thing.
- **Enforces budget policies** directly — an **alert** (e.g. a webhook/notification) at a threshold,
  and a **reject** that blocks requests over a hard cap.

Cost management stops being reactive firefighting and becomes something you tune over time.

### Guardrails & safety

Establish, at minimum: **input validation, output monitoring, tool access controls, and operational
limits.** In regulated industries, a common reference pattern routes **every** request and response
through a guardrail step that filters PII from LLM outputs. Combine that with the least-privilege
tool scoping above.

### The AI governance board (for scaling)

Running dozens of agents across business units is a different discipline from shipping one. As agents
proliferate you can no longer answer basic questions — how many are running, who owns them, what data
they touch, what drives cost. An AI governance board only succeeds when every team prioritizes
governance. It should:

- **Span** engineering, security, legal/risk, and business owners.
- Set **centralized standards with federated execution.**
- Keep a **human in the loop for high-risk decisions.**
- **Track evolving regulation.**
- Carry **executive sponsorship** so governance **enables** teams rather than blocking them.

---

## 9. Stakeholder management

Because agentic systems are nondeterministic and evolve quickly, success depends on proactive
stakeholder management: clear ownership, short feedback loops, transparent metrics, and explicit
decision records. This depth isn't needed for every use case, but it's critical for complex
enterprise use cases with a high quality bar and many cross-functional stakeholders.

### Stakeholder map

| Stakeholder | Primary concerns | Success signals |
|---|---|---|
| **Executive sponsor** | Strategic fit, risk, ROI, reputation | Clear ROI narrative, governance in place, predictable spend |
| **Product manager** | Problem–solution fit, adoption, satisfaction | Task success rising, active users, fewer escalations |
| **AI engineer** | Output quality, latency, cost, reliability | Stable prompts/tools, improving eval scores, low incident rate |
| **Data engineer** | Data availability, quality, governance | Reliable pipelines, documented lineage, privacy preserved |
| **Platform engineer** | Scalability, SLAs, cost control | SLOs met, budget adherence, safe rollout procedures |
| **Software engineer** | Integration UX, APIs, error handling | Clean contracts, graceful fallbacks, low UI regressions |
| **Domain expert (SME)** | Domain accuracy, compliance, usefulness | High expert agreement, fewer critical misinterpretations |
| **Legal/compliance** | Regulatory exposure, auditability | Guardrails, audit trails, documented approvals |
| **Security/privacy** | Data leakage, access control | Passing red-team tests, no secret exposure, auditable access |
| **Finance/FinOps** | Cost predictability, unit economics | Cost per interaction trending down, budgets met |

### A lightweight RACI for key decisions

**R** = Responsible, **A** = Accountable, **C** = Consulted, **I** = Informed. Keep it short; revisit
monthly.

| Decision | R | A | C | I |
|---|---|---|---|---|
| Use case selection & scope | Product manager | Executive sponsor | SME, AI engineer | All |
| Project resourcing | Executive sponsor | Product manager | All | All |
| Data access / PII handling | Data engineer w/ compliance | Security, product manager | Executive sponsor | All |
| Architecture & development | AI engineer | Product manager | Security, platform + data engineer | Legal/compliance |
| Evaluation & guardrails | AI engineer, SME | Product manager | Legal/compliance, exec sponsor | Platform engineer |
| Budget / FinOps guardrails | Platform engineer | Finance/FinOps | Product manager, exec sponsor | AI engineer |
| Deployment & rollout | Platform engineer | Product manager | Software engineer | All |
| Incident response & escalation | Platform engineer | Executive sponsor | AI engineer, product manager | Security |
| Prompt/tooling change mgmt | AI engineer | Product manager | SME, platform engineer | Legal/compliance |
| Governance reviews / audits | Legal/compliance | Executive sponsor | Platform engineer, product manager | All |

### Communication cadence

Cadence follows the product stage. **Early** (development, pre-production): kickoff to align on
objectives and metrics; biweekly/monthly demos to review evaluation metrics and collect expert
feedback; short twice-weekly build reviews; expert review sessions to refine rubrics; an ops dry run
before first release. **Late** (production, scaling): daily ops triage; a weekly product/business
review of adoption and ROI; a monthly governance review; an as-needed incident channel; a quarterly
roadmap/FinOps review of unit economics and model/provider mix.

### Success metrics & dashboards

Defining **quality metrics** is often the hardest part — define them **cross-functionally and up
front.** Track a mix appropriate to your context:

- **Quality** — task success rate, expert agreement rate, harmful-output rate, judge-score trends,
  hallucination rate.
- **Operations** — P95 latency, error rate, incident MTTR, prompt/tool change-failure rate.
- **Cost** — cost per interaction, input/output token mix, cache-hit rate, model/provider mix.
- **Adoption / business** — active users, retention, time saved, CSAT/NPS, revenue or cost
  avoidance.

Provide role-appropriate views: an **executive** view (ROI, risk trend, budget vs. actual), a
**product/engineering** view (experiment throughput, regression alerts, guardrail trips), and an
**ops** view (SLO compliance, incident heatmap, provider health).

---

## 10. Worked example: a telco customer-support agent

A concrete, end-to-end reference for six planning activities you can map onto any use case:

1. Map the current human workflow in detail.
2. Translate it into a technical architecture — what's a tool-calling sub-agent vs. a
   branching-but-linear workflow; each agent's tools; the structured/unstructured data required; how
   each part is evaluated.
3. List observability requirements per stakeholder (executive → business users → developers).
4. Design tracing and logging for those needs (start with autolog defaults, then customize).
5. Map access controls for each agent and end user.
6. Identify reusable components and build abstractions accordingly.

> Run these as a **design sprint** — a time-boxed, cross-functional workshop that compresses months
> of planning into an intensive session to align on design, identify data sources, and plan
> evaluation suites *before* committing to complex implementation.

**Why an agent (not a linear workflow)?** A billing question about an unexpectedly high bill requires
**iterative** planning → action → evaluating the result → deciding the next step. That loop is what
justifies an agentic solution.

### Activities 1–2: human workflow → architecture

Human agents handle four broad areas — **accounts, billing, products, technical support** — and a
single call can span several. A human classifies these effortlessly; agents can't be trusted to
sprawl, so **give each agent a defined set of responsibilities and tools.**

**Decision:** one **sub-agent per query category** plus a **supervisor agent** that routes.
**Sub-agents do not communicate with each other** — this simplifies the setup and reduces sources of
indeterminism. (This is the multi-agent pattern; implement the supervisor as your own custom
orchestrator — see [§11](#11-staying-current-model--product-lifecycle).)

| Agent | Domain | Responsibilities |
|---|---|---|
| **Supervisor** | Orchestration | Classifies intent, routes to specialized agents, runs sentiment analysis, extracts routing attributes, generates the final response |
| **Account** | Customer profile | Profile and subscription queries |
| **Billing** | Financial | Billing, payment, and usage queries |
| **Product** | Product info | Plans, devices, promotions |
| **Tech support** | Technical | Troubleshooting and technical assistance |

Routing is a structured-output problem, so **evaluate it programmatically** (accuracy / F1 /
confusion matrix), per-agent and end-to-end — not with an LLM judge.

The billing agent's tools map onto structured tables (customers, subscriptions, plans, devices,
promotions, billing, usage) and unstructured sources (a markdown knowledge base of FAQs and policies;
free-text support tickets).

### Activities 3–4: observability & tracing by persona

Build one trace-and-feedback foundation, then expose per-persona views (business expert, executive
sponsor, customer) as described in [§7](#7-evaluation--observability).

### Activity 2 (evaluation): test-suite design

Distinct from tracing: here you answer *how each sub-agent is evaluated for quality.* Some evaluations
are obvious up front (the billing agent needs a customer-spend tool → test that the tool works — a
top-down evaluation); some failure modes only appear after bottom-up trace analysis. Build a thin
slice first, define expected inputs/outputs per substep, prioritize structured/boolean outputs so F1
applies, and use domain-aligned judges for unstructured fields. During development, the "quality"
judge should be someone who understands **both** the business use case and the technical side.

### Activity 5: access control (two levels)

Agent/tool-level least privilege (billing reads spend/invoices, no writes; account-management writes
behind a confirmation step; tech-support reads diagnostics, not billing; escalation opens a ticket,
doesn't close it) **and** end-user-level identity passthrough (the authenticated customer's identity
flows through every query, or a trusted non-LLM customer-ID filter). Both enforced in the governance
layer; audit trail captured centrally. (See [§8](#8-governance-access-control--cost).)

### Activity 6: reusable shared abstractions

Enterprises have long use-case backlogs, so **identify reusable components and build abstractions** —
they standardize development, eliminate duplicate work, and ensure every team conforms to best
practices (especially security, privacy, and evaluation). Examples:

- **A base agent class** — standard interfaces every agent implements: a uniform way to declare which
  governed assets it uses, initialization by name (loading the right config), and shared helpers for
  machine-to-machine auth.
- **A base evaluation-scorer class** — an opinionated wrapper over the unified judge API so teams
  don't burn cycles trialing evaluation APIs (see the `BaseScorer` sketch in
  [§7](#7-evaluation--observability)).
- **A small CLI for common evaluation tasks** — wrap recurring tasks (creating labeling sessions,
  generating evaluation reports) in a CLI. The twofold benefit: developers invoke standardized tools
  instead of writing their own scripts, **and the same tools can be handed to AI coding agents** to
  speed development.

---

## 11. Staying current (model & product lifecycle)

Generative-AI platforms move quickly. Two durable lessons matter more than any specific product
state:

- **Version and pin your model choice, and plan for deprecations.** Foundation models are retired and
  replaced on a regular cadence. Pin an explicit model version, version that choice like any other
  component, and migrate proactively when a replacement is recommended — don't let a silent default
  shift break behavior. Route calls through the AI Gateway so a model swap is a config change, not a
  code change.
- **Prefer portable orchestration for multi-agent systems.** The multi-agent *pattern* (a supervisor
  coordinating specialized sub-agents) is sound. But managed "supervisor" products come and go; the
  durable path is to **build the supervisor yourself as a custom orchestrator** on standard
  application hosting, with the AI Gateway in front and tracing throughout. That keeps you in control
  of the lifecycle and avoids depending on a managed API you can't version.

Whenever a specific endpoint name, API, or deprecation date matters, **verify it against current
official documentation** rather than trusting any point-in-time summary (this one included).

---

## 12. Sources & further reading

**The book itself** — *The Big Book of AgentOps: A comprehensive guide to deploying quality agentic
applications safely and reliably* (Databricks). Authors: Pavithra Rao, Jeanne Choo, Kyra Wulffert,
Alex Baur.

- Announcement blog: <https://www.databricks.com/blog/announcing-databricks-big-book-agentops>
- eBook (PDF): <https://www.databricks.com/sites/default/files/2026-08/2026-07-eb-the-big-book-of-agent-ops-final.pdf>

**Chapter map (for citing the original):**

1. Evolving trends (what an agent is; AI system architectures; MLOps → LLMOps → AgentOps;
   anti-patterns)
2. Deployment architecture patterns (the four patterns; choosing; common components)
3. The complete AgentOps project pipeline (seven phases)
4. DevOps principles (Flow; Feedback; Continuous Learning)
5. Prioritizing high-leverage activities (the six planning activities; the telco worked example)
6. Stakeholder management (map; RACI; cadence; success metrics)

**External references cited by the book:**

- *The DevOps Handbook* — origin of the Flow / Feedback / Continuous Learning principles.
- Shreya Shankar & Hamel Husain, *AI Evals* — the "Three Gulfs" framework.
- Shankar et al., *"Who Validates the Validators?"* — how evaluation criteria emerge from aligning
  automated evaluations with human judgment.
- Google Ventures design sprints — the time-boxed cross-functional workshop format.
- The widely cited 2025 study behind the "~95% of AI pilots fail" figure that motivates measurement
  capability.

**Databricks platform documentation** — for current product state (endpoints, APIs, deprecations),
always prefer the official Databricks documentation over any distilled summary.
