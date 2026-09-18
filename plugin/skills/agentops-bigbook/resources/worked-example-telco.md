# Worked Example: Telco Customer-Support Agent (Big Book Ch. 5)

A concrete, end-to-end reference for the six planning activities. Use it as a template you can
map onto any new use case. The six activities:

1. **Map the current human workflow in detail.**
2. **Translate it into a technical architecture** — what's a tool-calling sub-agent vs. a
   branching-but-linear workflow; each agent's tools; required structured/unstructured data;
   how each sub-agent/sub-workflow is evaluated.
3. **List observability requirements** per stakeholder (C-level → business users → developers).
4. **Design tracing & logging** for those needs (start with `mlflow.autolog()` defaults, then
   customize).
5. **Map access controls** for each agent and end user.
6. **Identify reusable components** and build abstractions accordingly.

> Run these as a **design sprint** — a time-boxed, cross-functional workshop (à la Google
> Ventures) that compresses months of planning into an intensive session to align on design,
> identify data sources, and plan eval suites *before* committing to complex implementation.

Why an agent (not a linear LLM workflow) fits: a billing question about an unexpectedly high
bill requires **iterative** planning → action → evaluating the action's output → deciding the
next step. That loop is what justifies an agentic solution.

---

## Activity 1–2: Human workflow → technical architecture

Human support agents handle four broad query areas: **accounts, billing, products, technical
support** (a single call can span several). A human classifies these effortlessly; agents can't
be trusted to sprawl, so **give each agent a defined set of responsibilities and tools**.
Designing for modularity lets you construct prompts, tools, and evals cleanly.

**Decision:** one **sub-agent per query category** + a **supervisor agent** that routes.
**Sub-agents do NOT communicate with each other** — this simplifies the setup and reduces
sources of indeterminism. (This is Book Pattern 2; implement the supervisor as a *custom*
orchestrator — see `platform-state.md`.)

| Agent | Domain | Responsibilities |
|---|---|---|
| **Supervisor agent** | Workflow orchestration | Routes queries to specialized agents, generates responses, classifies query intent, runs sentiment analysis, extracts attributes for routing |
| **Account agent** | Customer profile mgmt | Customer profile + subscription queries |
| **Billing agent** | Financial | Billing, payment, and usage queries |
| **Product agent** | Product info | Plans, devices, promotions |
| **Tech support agent** | Technical assistance | Troubleshooting + technical support |

**Routing is a structured-output problem → evaluate it programmatically** (accuracy / F1 /
confusion matrix), per-agent and end-to-end. Not an LLM judge. (See `evaluation.md`.)

### Tools & data for the billing agent (the sample workflow)

Map the tools each agent gets. Structured data sources (bronze layer):

| Table | Summary |
|---|---|
| `…bronze.customers` | Profile: `customer_id`, segment, location, registration_date, status, contact prefs, scoring (loyalty_tier, churn_risk_score, customer_value_score, satisfaction_score) |
| `…bronze.subscriptions` | Links customers→plans/devices: `subscription_id`, `customer_id`, `plan_id`, `device_id`, `promo_id`, dates, contract terms, charges, status |
| `…bronze.plans` | Plan catalog: `plan_id`, name, type, pricing, data limits, feature flags, contract reqs |
| `…bronze.devices` | Device catalog: `device_id`, name, manufacturer, type, pricing, specs, colors, availability |
| `…bronze.promotions` | Offers: `promo_id`, name, discount details, date ranges, active status |
| `…bronze.billing` | Billing: `billing_id`, customer/subscription refs, dates, charge breakdowns (base/additional/tax), payment info, status |
| `…bronze.usage` | Usage: `usage_id`, `subscription_id`, daily usage (data_usage_mb, voice_minutes, sms_count), billing cycle |

Unstructured sources:

| Table | Content |
|---|---|
| `…bronze.knowledge_base` | Markdown: FAQs, policies, guides, procedures |
| `…bronze.support_tickets` | Free-text issue descriptions + agent resolution details |

---

## Activity 3–4: Observability & tracing by persona

Build one trace + feedback foundation (MLflow Tracing + Assessments + Feedback API), then expose
per-persona views: **Business SME** (conversation UI + trace side-panel + feedback → assessments
+ label-schema config + summary report; served via Databricks Apps / Databricks One so they need
no workspace login), **Executive sponsor** (ROI dashboard from Delta), **Customer** (Databricks
App: preset cards, cited sources, like/dislike tied to their trace via `mlflow.log_feedback()`).
Full detail in `evaluation.md` → "Observability by persona."

---

## Activity 2 (eval) & test-suite design

Note test-suite design is distinct from tracing/logging (activity 4): here we answer *how each
sub-agent/sub-workflow is evaluated for quality*. Some evals are obvious up front (the billing
agent needs a customer-spend tool → write a test that the tool works — a **top-down** eval). Some
failure modes only appear after **bottom-up** trace analysis (e.g. a SQL-gen step mixing up
product acronyms). Build a **thin slice** first, define expected inputs/outputs per substep,
prioritize **structured/JSON/boolean outputs** so F1 applies, and use **domain-aligned LLM
judges** for unstructured fields. During dev, the "quality" judge should be someone who
understands **both** the business use case and the technical side (PM, scrum master, embedded
data scientist, or technically-savvy business user). Full mechanics in `evaluation.md`.

---

## Activity 5: Access control (two levels)

- **Agent/tool-level (least privilege):** billing sub-agent = read spend/invoices, no writes;
  account-mgmt = write contact prefs behind a confirmation step; tech-support = read
  diagnostics, not billing; escalation = open a ticket, not close it.
- **End-user-level:** the authenticated customer's identity flows through every query
  (on-behalf-of-user passthrough, or a trusted non-LLM customer-ID filter — never from model
  output).

Both enforced in Unity Catalog; audit trail lands in a UC system table. Full detail in
`governance-and-access-control.md`.

---

## Activity 6: Future planning — reusable shared abstractions

Enterprises have long use-case backlogs (dozens–hundreds of GenAI use cases across many teams).
**Identify reusable components and build abstractions** — they standardize development, kill
duplicate work, and ensure every team conforms to best practices (especially security, privacy,
evals). Examples (available in the Databricks Field Solutions GitHub repo — see `sources.md`):

**1. Base agent class** — standard interfaces all agents implement: a uniform way to specify
Unity Catalog assets (a `UCConfig`), and easy init by name (which loads the right config file).
Can also hold class methods for machine-to-machine auth via the Databricks Workspace Client.
```python
class BaseAgent(ResponsesAgent, abc.ABC):
    """Base agent class all agents inherit from."""
    _config_cache: dict[str, AgentConfig] = {}
    def __init__(
        self,
        agent_type: str,
        llm_endpoint: Optional[str] = None,
        tools: Optional[list[dict]] = None,               # UC function tools
        vector_search_tools: Optional[dict[str, Any]] = None,  # name -> VectorSearchRetrieverTool
        system_prompt: Optional[str] = None,
        config_dir: Optional[Path | str] = None,
        inject_tool_args: Optional[list[str]] = None,     # extra args injected from custom_inputs
        disable_tools: Optional[list[str]] = None,        # simple or full UC function names
        uc_config: Optional[UCConfig] = None,
    ):
        """Initialize base agent from a config file."""
```

**2. Base class for evaluation scorers** — an opinionated wrapper over MLflow's unified
`make_judge` API so teams don't burn cycles trialing eval APIs (see `evaluation.md` for the
`BaseScorer` sketch).

**3. An MLflow CLI tool for common tasks** — wrap recurring MLflow tasks (creating labeling
sessions, generating eval reports) in a CLI. Twofold benefit: developers invoke standardized
tools instead of writing their own scripts, **and the same tools can be handed to AI coding
agents (e.g. Claude) to speed development.** Example — a Claude command that manages MLflow
Review App label schemas:
```
/label-schemas                       # show all schemas (type, options, instructions)
/label-schemas add quality rating from 1 to 5
/label-schemas add helpfulness categorical with options: Very Helpful, Helpful, Not Helpful
/label-schemas add feedback text field for comments
```
When available, it references experiment summaries (`experiments/[experiment_id]_summary.md`) to
suggest schemas tailored to the agent's capabilities.

→ Deploying this stack in agentops-stacks: `agentops-stacks` scaffolds it, `add-agent` /
`add-supervisor` compose the sub-agents + router, `agentops-lifecycle` drives eval + promotion,
and `vector-search-ops` / `uc-functions-ops` / `lakebase-ops` operate the components. Underlying:
Model Serving (ResponsesAgent), Apps (UIs), DABs (package + promote), MLflow GenAI evaluation.
