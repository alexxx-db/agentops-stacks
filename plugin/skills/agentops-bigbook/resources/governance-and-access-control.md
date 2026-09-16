# Governance, Access Control & Cost (Big Book Ch. 3.7, 5.3.5)

Agentic systems need governance at three layers: **who each agent/tool can act as** (access
control), **what it costs** (the AI Gateway), and **who owns the whole portfolio** (the AI
governance board). Enforce all of it in the platform, not in prompt logic.

---

## The two-level permission model (non-negotiable)

In an agentic system, permissions apply at **two distinct levels**. Getting this wrong is a
security incident, not just a bug — it can expose sensitive data to unauthorized viewers.

### 1. Agent- and tool-level permissions (least privilege)
Each sub-agent operates with the **least privilege it needs**. Telco example:
- **Billing sub-agent:** read access to spend/invoice tables; **no** write to account settings
  or plan configs.
- **Account-management sub-agent:** write to update contact preferences — **gated behind an
  explicit confirmation step**.
- **Technical-support sub-agent:** read network/device diagnostics; **not** billing data.
- **Escalation path:** permission to *open* a human-review ticket, but **not** to resolve/close
  it autonomously.

Scoping tools this narrowly **contains the blast radius**: even if the LLM is manipulated into
calling a tool inappropriately, the tool itself can't act outside its granted scope.

### 2. End-user-level permissions (identity flows through)
Agent-level scoping is necessary but **not sufficient**. One deployed agent serves many users,
so the **end user's identity and permission scope must flow through every asset accessed** — a
user must only ever see what they're authorized for. Two patterns:
- **Identity passthrough (on-behalf-of-user)** — the agent queries **as** the end user, so their
  existing row- and column-level access policies apply automatically. Prefer this.
- **Explicit parameterization** — where passthrough isn't available, inject the authenticated
  user/customer ID as a **trusted, non-LLM-controlled filter** on every query. **Never take the
  identity filter from the model's output** — it can be manipulated (prompt injection).

### Enforce controls centrally (on Databricks)
Express **both** levels through **Unity Catalog**:
- **Agent-level scope** = **grants** on the tables, functions, MCP servers, and tools each agent
  can access. On Databricks Apps, this is defined in the app config that maps which UC assets
  the app's **service principal** can reach.
- **End-user scope** = row/column-level security enforced via **on-behalf-of-user identity
  passthrough**.

Centralizing in the **governance layer** (not agent/prompt logic) keeps permissions **auditable,
consistent across agents, and resistant to prompt-injection** attempts to talk the agent out of
them. Every tool call and data access is captured in the **audit trail** (a Unity Catalog system
table), which feeds observability + compliance reporting.

→ Hands-on: `uc-functions-ops` (tool EXECUTE grants + registration), Unity Catalog (grants,
on-behalf-of-user, audit system tables), and Databricks Apps (app service-principal config).

---

## Cost control via the Unity AI Gateway

**Agent costs multiply, they don't add.** Every sub-agent, retry, and guardrail check adds cost
on top — one user request can become **4–6 LLM calls** behind the scenes. The recurring failure
mode is discovering runaway spend *only after the invoice arrives*.

The **Unity AI Gateway** (OSS: MLflow AI Gateway) is the fix and is **mandatory in every
deployment pattern**. It is a centralized control plane for every LLM call:
- **Routes all traffic through one point** (also enables model routing + provider management).
- **Autologs traces** capturing token counts, latency, model, and cost per call.
- **Attributes spend across the full span hierarchy** (orchestrator → sub-agent → synthesis →
  guardrail) so you pinpoint which component drives cost **before** optimizing the wrong thing.
- **Enforces budget policies directly:**
  - **ALERT** — fires a webhook (e.g. a Slack notification) at a threshold.
  - **REJECT** — blocks requests outright over a hard cap.

Cost management stops being reactive firefighting and becomes something you tune over time.
(See Kyra Wulffert's "Preventing Runaway Agent Costs with MLflow AI Gateway" in `sources.md`.)

---

## Guardrails & safety (Phase 3.3)

Establish, at minimum: **input validation, output monitoring, tool access controls, and
operational limits.** In regulated industries, a common reference pattern is to route **every**
request and response through a **guardrail step that filters PII** from LLM outputs. Combine
with the least-privilege tool scoping above.

---

## The AI governance board (Phase 7 — scaling)

Getting one agent to prod is a milestone; running **dozens across business units** is a
different discipline. As agents proliferate, you can no longer answer basic questions: how many
agents are running, who owns them, what data they touch, what drives cost. You need an **AI
governance board**.

Unlike the old world where compliance, IT, and security operated separately from engineering and
business owners, an AI governance board **only succeeds when every team prioritizes governance.**
It must:
- **Span** engineering, security, legal/risk, and business owners.
- Set **centralized standards with federated execution.**
- Keep a **human in the loop for high-risk decisions.**
- **Track evolving regulation.**
- Carry **C-suite sponsorship** so governance **enables** teams rather than blocking them.

Pair this with the RACI matrix and the monthly governance review in `stakeholder-management.md`.
