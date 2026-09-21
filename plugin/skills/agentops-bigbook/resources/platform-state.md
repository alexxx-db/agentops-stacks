# Platform Notes — Time-Sensitive Supplement

> **This file is NOT from the book.** *The Big Book of AgentOps* is a point-in-time document
> (cover date Jul 10, 2026). This file supplements it with fast-moving Databricks platform facts
> — serving endpoints and managed-product lifecycles — that affect *how* you implement the
> book's patterns.
>
> Deprecation dates, endpoint names, and product lifecycles change quickly. **Verify every
> specific date, endpoint, or API against current official Databricks documentation before
> relying on it.** When this file and the live docs disagree, the docs win.

---

## 1. The multi-agent supervisor: pattern ✅ vs. managed product ⚠

The single most important reconciliation between the book and current reality.

- The book's **Pattern 2 / Pattern 4** and the telco example use a **supervisor agent
  orchestrating sub-agents**. **The architectural *pattern* is valid and current.**
- **What shifts is the *managed* tooling that implements it.** Managed "supervisor" and
  multi-agent offerings have moving lifecycles — for example, the Beta **Supervisor API** is
  deprecated (per Databricks documentation, with an end-of-life in late 2026), and custom agents
  on Databricks Apps are the recommended replacement. Before depending on any managed
  supervisor / multi-agent product, check its current lifecycle status in the official docs.
- **The durable multi-agent path: a custom orchestrator (e.g. LangGraph) deployed on Databricks
  Apps**, with the AI Gateway in front and MLflow tracing throughout. Build the supervisor
  yourself rather than depending on a managed API whose lifecycle you don't control.

**Code-first client** for calling model serving through the gateway:
```python
from databricks_openai import DatabricksOpenAI
client = DatabricksOpenAI(use_ai_gateway=True)   # ships in the databricks-openai package
```

→ Managed option: **Agent Bricks**. → Custom path: **Databricks Model Serving** (ResponsesAgent)
served via **Databricks Apps**.

---

## 2. Model serving endpoints

- Foundation-model endpoints are versioned and are periodically deprecated and replaced. **Pin an
  explicit model version, version that choice like any other component, and migrate proactively**
  when a replacement is recommended — don't let a silent default shift break behavior.
- Prefer the latest capable model for new agents. Confirm the current endpoint list and any
  deprecation dates in-workspace or in the official docs before relying on them.
- Route calls through the **AI Gateway** so switching models is a config change, not a code change.

---

## 3. MLflow evaluation surface (current APIs to prefer)

These back the book's Feedback loop (open-source MLflow GenAI):
- `mlflow.genai.evaluate()` — offline and online scorer runs.
- `make_judge` — unified interface for judge-based evaluation.
- **Judge alignment** — continuously align judges to domain-expert (SME) feedback.
- **Automated prompt optimization** — run it right after an alignment job so aligned judges drive
  prompt improvements.
- Built-in scorers: Correctness, Guidelines, Safety, RetrievalGroundedness.

---

## 4. AgentOps Stacks

The book references **AgentOps Stacks** — pre-built scaffolding and deployment pathways that
implement the four deployment patterns as a recommended starting point for a new project. **This
repo is that scaffolding**; use the `agentops-stacks` skill to scaffold and `agentops-lifecycle`
to build and ship.
