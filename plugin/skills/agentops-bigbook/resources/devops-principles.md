# DevOps Principles for Agent Operations (Big Book Ch. 4)

DevOps isn't new, but it must be **updated for AgentOps**. Three classic principles — **Flow,
Feedback, Continuous Learning** — remain foundational, adapted for the realities of generative
AI. For the hands-on eval mechanics referenced throughout, see `evaluation.md`.

Why GenAI breaks naive DevOps:
- **Nondeterministic behavior** — the same prompt can yield different outputs, so traditional
  pass/fail testing is insufficient.
- **Unstructured interfaces** — natural language / images / audio need different eval methods
  than structured-data APIs.
- **Context-dependent quality** — output quality depends on context, user intent, and domain
  knowledge that's hard to encode in a fixed test suite.
- **Expensive evaluation** — LLM-based evals consume tokens and cost money; test-suite design
  must be strategic.

---

## Principle 1 — Flow

Reduce the time to deploy changes to production **while improving** reliability and quality —
don't trade speed against reliability.

### Build a Golden Evaluation Dataset (the enabling step)

Unlike traditional software, you can't define all tests up front. GenAI needs a preliminary
step: **analyze traces of prior requests/responses to identify common failure modes.** Only
after failure modes are identified and categorized can you define automated checks for them.

**The Dangerous Default** ❌ — one tester runs five or six queries in a chat UI and eyeballs
whether each looks acceptable. This captures no error-frequency metrics across a broad input
distribution and classifies no failure types.

**The Scientific Approach** ✅ — apply the scientific method iteratively:

1. **Categorize failure modes** as fast as possible using **~100 traces**:
   - *In production already:* sample production traces; ensure they're diverse across input
     categories, conversation lengths, and tool-call types.
   - *Still in development:* work with an SME to define inputs covering the range of scenarios
     a workflow/sub-agent should handle. Because LLM calls are nondeterministic, **run each
     input 3–4 times** to measure response consistency.
2. **Create failure reports** using MLflow or notebooks with visualizations.
3. **Calibrate automated LLM-judge metrics** from step-2 insights + domain-expert consultation.
   Output = a set of scorers (e.g. MLflow GenAI scorers) that scale trace evaluation in dev and
   prod.
4. **Run scorers offline and online** — offline: pass an MLflow Dataset / Pandas DataFrame to
   `mlflow.genai.evaluate()`; online: collect traces into an OLAP store (Delta tables), then
   batch-evaluate a sample of production traces with the **same** scorers.
5. **Surface aggregated metrics in dashboards** to watch trends; have PMs/business SMEs
   periodically review and refine the suites against **new** failure modes seen in production.

> Tip: start from failure reports that existing frameworks give you. The MLflow Evaluation UI
> lets you filter traces that fail a specific assessment; surface those (with the judge's
> rationale) in a notebook, SQL dashboard, or Databricks App.

### Evaluations are a Revenue Generator, not a Cost Center

Reframe evals from "necessary evil / compliance checkbox" to a **strategic investment** that
de-risks AI initiatives, accelerates reliable deployment, and drives durable competitive
advantage. This reframing is often the key to getting eval work funded.

---

## Principle 2 — Feedback

GenAI blends software and ML, so developers must collect **both** operational telemetry (latency,
cost, errors — second nature to software engineers) **and** model-quality signals (precision,
recall — familiar to ML practitioners) **and** new GenAI feedback forms: **SME feedback, LLM-judge
outputs, and rules-based checks.**

Evaluate every feedback signal for:
- **Consistency** — SMEs may score the same output differently; establish inter-rater
  reliability with clear rubrics + regular calibration.
- **Representativeness** — end-user ratings suffer selection bias (only the very happy or very
  angry rate).
- **Alignment** — LLM judges must be continuously validated against human judgment; a judge
  that agrees with *itself* but diverges from SMEs gives false confidence.
- **Actionability** — "this response is bad" isn't actionable; categorize/structure feedback to
  reveal whether issues are retrieval, reasoning, tone, or factual.

### The Feedback Flow (three phases)

**Pre-production:**
- SMEs manually review and label data in labeling sessions.
- Developers analyze traces to find and fix quality issues.
- An evaluation dataset is built (inputs, outputs, metrics).
- Automated evaluation suites test the application.

**Production:**
- Deploy with tracing enabled (beta first, then full production).
- A real-time monitoring dashboard tracks performance.
- Users give feedback in-app via a feedback API.
- Production traces are continuously evaluated.

**Continuous improvement (the loop that compounds):**
- Evaluation results inform the next iteration. Offline eval uses the development dataset;
  online feedback comes from prod users + automated monitoring.
- Selected traces + human reviews are **added back** to the eval dataset.
- **Judge alignment** runs after each round of SME feedback so judges track SMEs as closely as
  possible.
- Aligned judges become **scorers for prompt optimization** to directly improve the agent. In
  MLflow, automate this by running `optimize_prompts()` immediately after an `align()` job.

This enables an **SME-driven development loop**: SME feedback triggers workflows that improve
monitoring (via aligned judges) and directly improve agent quality (via prompt optimization).
Initial cross-functional alignment on scope and expected outputs is critical; after that,
automation can drive continuous improvement. Inject manual gates where you want them (e.g.
re-run eval on a held-out dataset to confirm aligned judges generalize).

Databricks surfaces this through **MLflow Tracing** (span-level capture), **MLflow Assessments**
(the UI for reviewing traces + attaching judgments), and the **MLflow Feedback API**
(`mlflow.log_feedback()`) for recording human + automated feedback against a trace.

### Feedback best practices

- **Start with boolean feedback** (thumbs up/down); once you see patterns via MLflow's search
  APIs, expand to numeric ratings or structured types.
- **Use source attribution** — set meaningful `source_id` on `AssessmentSource` objects; MLflow
  keeps full audit trails with timestamps + source.
- **Consistent naming** — standardize names like `user_satisfaction` / `quality_rating` across
  traces so search/aggregation is meaningful.
- **Combine programmatic + UI collection** — API for automated capture, UI for manual review;
  both integrate.
- **Link feedback to fresh traces** — collect immediately after generation while context is
  available, using MLflow's direct trace-feedback linkage.

### Feedback is iterative — analyze manually FIRST, then automate

Collecting feedback isn't enough; GenAI feedback must be **categorized and analyzed** before it
improves the app. Example: you start with a generic "relevance" judge (binary pass/fail +
rationale). An SME disagrees and corrects it via the labeling UI. You realize "relevance" is too
general and split it into sharper criteria (e.g. *solution appropriateness*, *context
alignment*), yielding a **custom judge that reflects the enterprise's real quality standards**.

The pattern — **manual SME review → discover evaluation criteria → automate them into LLM
judges / code checks** — is fundamental because: requirements emerge through exploration
(quality criteria are often unknown up front); domain expertise is essential (SMEs catch
nuances engineers miss); automation scales human judgment (judges apply agreed criteria
consistently). A common failure: teams build in isolation, trying to perfect the system before
showing SMEs, and discover fundamental misalignment too late. **Early, continuous SME
involvement** prevents wasted effort. (Cf. Shankar & Husain's "Three Gulfs" and "Who Validates
the Validators?" — see `sources.md`.)

---

## Principle 3 — Continuous Learning

GenAI moves fast: what was hard yesterday is easy tomorrow (e.g. post-training improvements in
instruction-following and tool use unlocked agents that weren't feasible a year earlier). So
stakeholders must **continuously update their mental model of what's buildable**. Organizations
rarely want *one* successful app — they want to repeatedly ship high-quality apps across
departments (some have hundreds of candidate use cases). Scale by **encoding hard-won knowledge
into reusable assets**:

### 1. Standardized frameworks
Make development function like an assembly line: faster time-to-market, higher quality, lower
risk. The book's example is **Databricks Asset Bundles** — a deployment unit that packages
notebooks/jobs/pipelines/models/libraries/config as: a *resource collection*, an
*infrastructure-as-code package* (YAML for compute, permissions, schedules, env settings), a
*versioned deployment artifact* (promote dev→staging→prod, deploy atomically), and a *dependency
container*. Lifecycle: `init` a template → edit config for targets → add source → `databricks
bundle validate` → `databricks bundle run`. Benefits: standardization across teams, version
control of YAML definitions, and deployment via external CI/CD (GitHub Actions, Azure DevOps).
→ Databricks Asset Bundles.

### 2. Reference architectures
Proven blueprints of design patterns + best practices. Example: in regulated industries, route
every request/response through a **guardrail step that filters PII** from LLM outputs. Reference
architectures create shared understanding of *how* and *why* systems are built, and capture
~**70–80% of architectural decisions up front**, leaving teams the 20–30% specific to their use
case. (These are the deployment patterns from Ch. 2 — see `deployment-patterns.md`.)

### 3. Industry- / enterprise-specific agentic design patterns
Blueprints that drill into how the **agents themselves** should be designed:

- **Collaborative metric development & LLM-judge calibration** — evaluating agentic AI requires
  synthesizing knowledge across technical teams (capabilities), business leaders (outcomes),
  and SMEs (domain quality). Without a unified view, teams chase ambiguous criteria. When a
  technical lead, a domain expert, and a PM review the same response they often disagree on
  "good" — and **surfacing that disagreement is the point**. Gather concrete output examples
  covering production scenarios, work through them cross-functionally, make the differing
  success definitions explicit, and usually develop **multiple** criteria rather than one
  "quality score." Then scale with judges trained on the shared understanding (judges handle
  routine evaluation and flag edge cases for humans).
- **Business stakeholder alignment via shared design patterns** — two worked examples:
  - **Insurance claims processing** — event-driven; a claim event triggers document review; a
    missing-doc event fires a webhook emailing the customer. Workflows must **pause and resume**,
    so engineers need a **state store** that checkpoints the paused step, and an agent
    sub-graph that can resume at specific points (not rerun from scratch). State depends on both
    local and global/parallel events (e.g. registering with a regulator). Shared design
    elements across claims departments: a common **state-store schema** and a common **JSON
    resume-event payload**. → state store: `lakebase-ops` (Lakebase checkpointer + memory).
  - **Financial services compliance** — regulators periodically release new rules; agents
    monitor regulatory updates, interpret policy changes, route ambiguous cases for human
    review, and maintain audit trails. Document + reuse these across compliance/audit use cases.

**The compounding effect:** teams that codify patterns, align judges with domain experts, and
reuse components across use cases move from one-off agents to a **reliable, ever-improving
portfolio** of agentic systems. Continuous learning isn't purely technical — collaborative
metric development keeps evaluation aligned with business intent, and educating business
stakeholders builds the shared understanding needed to act on feedback.
