# Sources & Further Reading

## The book itself

**The Big Book of AgentOps** — *A comprehensive guide to deploying quality agentic applications
safely and reliably.* Databricks. Cover date **Jul 10, 2026**.

**Authors:** Pavithra Rao, Jeanne Choo, Kyra Wulffert, Alex Baur.

- Blog announcement: https://www.databricks.com/blog/announcing-databricks-big-book-agentops
- Published PDF: https://www.databricks.com/sites/default/files/2026-08/2026-07-eb-the-big-book-of-agent-ops-final.pdf

### Chapter map (for citing)
1. Evolving Trends (what is an agent; AI system architectures; MLOps→LLMOps→AgentOps; **anti-patterns**)
2. Deployment Architecture Patterns (the 4 patterns; choosing; common components; AgentOps Stacks)
3. The Complete AgentOps Project Pipeline (7 phases)
4. DevOps Principles (Flow; Feedback; Continuous Learning)
5. Prioritizing High-Leverage Activities (6 planning activities; the **telco worked example**)
6. Stakeholder Management (stakeholder map; RACI; cadence; success metrics)

---

## Referenced within the book

- **Kyra Wulffert**, "Preventing Runaway Agent Costs with MLflow AI Gateway" — the end-to-end
  worked example for AI-Gateway budget control (book §3.5).
- **Databricks Field Solutions GitHub repository** — home of the reusable shared abstractions
  (BaseAgent, BaseScorer, MLflow CLI) from book §5.3.6.
- **The DevOps Handbook** — origin of the Flow / Feedback / Continuous Learning principles the
  book adapts for AgentOps.
- **Shreya Shankar & Hamel Husain**, *AI Evals* course — the **"Three Gulfs"** framework (book §4.2.4).
- **Shankar et al.**, *"Who Validates the Validators?"* — how eval criteria emerge from aligning
  automated evaluations with human judgment.
- **Google Ventures design sprints** — the time-boxed cross-functional workshop format the book
  recommends for the six planning activities (book §5.1).
- **2025 MIT study** — the "95% of AI pilots fail" figure that motivates measurement capability
  (book §3).

---

## Databricks platform documentation

For current product state (endpoints, APIs, deprecations), always prefer the official Databricks
documentation over this summary — product capabilities and lifecycles change over time.

The Databricks capabilities this playbook maps onto: **MLflow** (evaluation, tracing, feedback,
judge alignment, prompt optimization), the **AI Gateway**, **Unity Catalog**, **Databricks Asset
Bundles**, **Databricks Model Serving**, **Databricks Apps**, **Agent Bricks**, **Vector Search**,
**Genie**, and **Lakebase**.
