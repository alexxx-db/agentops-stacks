# The Complete AgentOps Project Pipeline (Big Book Ch. 3)

A systematic path from conception to continuous improvement. **95% of AI pilots fail** (2025 MIT
study); the bridge is **measurement capability** — building the evaluation that earns trust from
users and leadership. Projects **iterate between phases**; that's expected, not a failure.

Use this as the project roadmap. Each phase has an objective and cross-references to the deeper
material. Route hands-on steps to the skills named at the end.

---

## Phase 1 — Project Conception & Team Formation

- **1.1 Business use case identification** — define a focused, measurable problem agents can
  solve. Avoid overly broad use cases or problems better solved with deterministic workflows.
  (→ anti-patterns.md)
- **1.2 Cross-functional team assembly** — leadership stakeholders + technical developers +
  domain SMEs with real expertise in the agent's domain. (→ stakeholder-management.md)
- **1.3 Stakeholder alignment & governance** — align on deliverables, success metrics, and
  project governance; educate leadership on AI limitations; plan the communication cadence.

## Phase 2 — Data Preprocessing & Indexing

- **2.1 Data & agent architecture planning** — design pipelines optimized for **AI consumption**,
  not traditional analytics. Emphasis shifts to unstructured data, vector indexes, embedding
  generation, tool-calling, and LLM-based extraction (vs. classic ETL).
- **2.2 Production data infrastructure** — ACID transactions, quality monitoring, failure
  handling, compliance frameworks.
- → Hands-on: `vector-search-ops` (RAG index + retriever), `uc-functions-ops` (tool
  registration), Lakeflow Declarative Pipelines, and Unity Catalog.

## Phase 3 — Agent Architecture Design

- **3.1 Architecture pattern selection** — pick the right complexity and framework:
  DIY/custom vs. code-first frameworks vs. low-code platforms. **Avoid over-engineering with
  supervisor/multi-agent patterns when deterministic chains suffice.** (→ deployment-patterns.md)
- **3.2 Cross-functional evaluation planning** — define what "good" looks like *with* domain
  experts and end users. You cannot calibrate a judge if the success metric isn't clear.
  (→ devops-principles.md Feedback + Continuous Learning; evaluation.md)
- **3.3 Guardrails & safety** — input validation, output monitoring, tool access controls,
  operational limits. (→ governance-and-access-control.md)

## Phase 4 — Agent Development & Evaluation Framework

- **4.1 Agent development workflow** — treat development as first-class **alongside**
  evaluation, with eval signals driving what gets built next.
- **4.2 The testing pyramid for AI** — start with manual SME review to learn quality patterns,
  then scale through automated LLM judges and programmatic checks; do both offline testing and
  online production monitoring. (→ evaluation.md)
- **4.3 Cost-effective evaluation** — LLM-based evals cost money (unlike unit tests). Use
  token-efficient strategies, field-based vs. trace-based evaluation, and smart sampling.
  (→ evaluation.md)
- → Hands-on: `agentops-lifecycle` (Steps 4–5 build the eval gate + dataset), backed by MLflow
  GenAI evaluation and MLflow Tracing.

## Phase 5 — Feedback Collection & Monitoring

- **5.1 Multi-source feedback integration** — end-user feedback + SME validation + LLM-judge
  assessments + system metrics; both structured pre-production reviews and real-time production
  collection. Automate via LLM-judge alignment to SME feedback + prompt optimization.
- **5.2 Production telemetry & observability** — operational metrics (latency, cost), quality
  metrics (eval scores, satisfaction), business impact (ROI, efficiency); data protection +
  audit trails. **Watch for runaway spend:** agent costs *multiply* (every sub-agent, retry,
  and guardrail adds cost — one request can become 4–6 LLM calls). The **Unity AI Gateway** is
  the fix: route all traffic through one control plane, autolog token/latency/model/cost per
  call, attribute across the span hierarchy, and enforce budgets (ALERT webhook at a threshold,
  REJECT over a hard cap). (→ governance-and-access-control.md)
- → Hands-on: `agentops-lifecycle` (Step 10 wires production monitoring + the feedback loop),
  backed by MLflow's metrics-query, trace-retrieval, and trace-analysis APIs.

## Phase 6 — Iterative Development & Optimization

- **6.1 Continuous learning feedback loop** — systematic cycles of collect → analyze →
  hypothesize → targeted improvement → validate → deploy. Focus on prompt engineering, tool
  selection, architecture tuning, cost optimization. (→ devops-principles.md Continuous Learning)
- **6.2 Trust building through transparency** — regular demos, metric transparency, honest
  failure analysis, user education; turn evaluation data into reusable organizational assets.

## Phase 7 — Scaling & Governance

- **7.1 Enterprise scaling & risk management** — expand pilots enterprise-wide: cost
  management, quality consistency, organizational change, infrastructure scaling.
  Getting one agent to prod is a milestone; running **dozens across business units** is a
  different discipline. As agents proliferate you hit a governance problem — you can't answer
  who owns what, what data they touch, or what drives cost. Stand up an **AI governance board**
  spanning engineering, security, legal/risk, and business owners: centralized standards with
  federated execution, human-in-the-loop for high-risk decisions, regulation tracking, and
  C-suite sponsorship so governance **enables** teams rather than blocking them.
  (→ governance-and-access-control.md)

---

## Delivering strategic business value (the payoff)

The pipeline turns experimental pilots into strategic assets by letting organizations:
accelerate time-to-value via repeatable processes; minimize risk by addressing failure modes
proactively; scale with consistent quality; generate measurable ROI; and build AI maturity that
compounds across projects.
