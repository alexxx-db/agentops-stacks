# Stakeholder Management (Big Book Ch. 6)

AgentOps changes how organizations plan, deliver, and operate software. Because agentic systems
are **nondeterministic and evolve quickly**, success depends on proactive stakeholder
management: clear ownership, short feedback loops, transparent metrics, and explicit decision
records. Even "standard" reliability metrics behave differently (a reasoning LLM can have highly
variable latency per query) — the metric names are unchanged, but *how you evaluate them* should
change.

> This level of management isn't needed for every use case, but it's **critical for complex,
> enterprise use cases** with a high quality bar and many cross-functional stakeholders.

---

## 1. Stakeholder map

The full superset of potential stakeholders. In smaller orgs one person may hold several roles,
and not all are relevant to every use case. Identify the **minimum viable set** and the value
each expects.

| Stakeholder | Primary concerns | Success signals |
|---|---|---|
| **Executive sponsor** | Strategic fit, risk, ROI, reputation | Clear ROI narrative, governance in place, predictable spend |
| **Product manager** | Problem–solution fit, adoption, user satisfaction | Task success rising, active users, fewer escalations |
| **AI engineer** | Output quality, latency, cost, reliability | Stable prompts/tools, improving eval scores, low incident rate |
| **Data engineer** | Data availability, quality, governance | Reliable pipelines, documented lineage, privacy preserved |
| **Platform engineer** | Scalability, SLAs, cost control, provider mgmt | SLOs met, budget adherence, safe rollout procedures |
| **Software engineer** | Integration UX, APIs, error handling | Clean contracts, graceful fallbacks, low UI regressions |
| **SME (domain expert)** | Domain accuracy, compliance, usefulness | High SME agreement, fewer critical misinterpretations |
| **Legal/compliance** | Regulatory exposure, auditability | Guardrails, audit trails, documented approvals |
| **Security/privacy** | Data leakage, supply-chain risk, access control | Passing red-team tests, no secret exposure, auditable access |
| **Finance/FinOps** | Cost predictability, unit economics | Cost per interaction trending down, budgets met |

---

## 2. Lightweight RACI for key decisions

**R** = Responsible (does the work), **A** = Accountable (final decision), **C** = Consulted,
**I** = Informed. Keep it short; revisit monthly.

| Decision | R | A | C | I |
|---|---|---|---|---|
| Use case selection & scope | Product manager | Executive sponsor | SME, AI engineer | All |
| Project resourcing | Executive sponsor | Product manager | All | All |
| Data access / PII handling | Data engineer w/ compliance | — | Security, product manager | Executive sponsor |
| Architecture & development | AI engineer | Product manager | Security, platform engineer, data engineer | Legal/compliance |
| Evaluation & guardrails | AI engineer, SME | Product manager | Legal/compliance, executive sponsor | Platform engineer |
| Budget / FinOps guardrails | Platform engineer | Finance/FinOps | Product manager, executive sponsor | AI engineer |
| Deployment & rollout plan | Platform engineer | Product manager | Software engineer | All |
| Incident response & escalation | Platform engineer | Executive sponsor | AI engineer, product manager | Security |
| Prompt/tooling change mgmt | AI engineer | Product manager | SME, platform engineer | Legal/compliance |
| Governance reviews / audits | Legal/compliance | Executive sponsor | Platform engineer, product manager | All |

---

## 3. Communication cadence

Cadence follows the product stage. **Early:** team formation, cross-functional metric
definition, SME labeling/feedback for fast iteration — **share early and often** (waiting for
perfection before showing SMEs badly slows iteration). **Late:** operations, cost optimization,
production monitoring, roadmap management.

### Early-stage (development, pre-production)
| Touchpoint | Duration | Objective | Owner | Key artifacts |
|---|---|---|---|---|
| Initial project kickoff | 30–45 min | Align on objectives, identify stakeholders, define agent metrics | Product manager | Project plan, model metrics, stakeholder map, RACI |
| Product demo (biweekly/monthly) | 30–45 min | Demo prototypes, review eval metrics, collect SME feedback | Product manager | Working prototype, top regressions, prioritized roadmap |
| Build review (twice-weekly / async) | 15–20 min | Plan/confirm experiments, track eval runs + acceptance thresholds | AI engineer w/ PM | Experiment log, eval dashboard snapshot, change requests |
| SME review sessions (as needed) | 60–90 min | Refine rubrics, review edge cases, validate domain correctness | PM w/ SME | Golden-set updates, judge-calibration notes, acceptance criteria (in MLflow) |
| Ops dry run (before first release) | 60 min | Create/validate runbooks, harden metrics, align deployment plan | Platform engineer | Runbook check, alert-test results, rollback plan, CI/CD architecture |

### Late-stage (production, scaling)
| Touchpoint | Duration | Objective | Owner | Key artifacts |
|---|---|---|---|---|
| Daily ops triage | 15 min | Scan for incidents, cost spikes, degraded evals, provider issues | Platform engineer | Open incidents, SLO dashboard, mitigation status |
| Weekly product/business review | 30–45 min | Track adoption + ROI, prioritize against metric movements | Product manager | Adoption funnel, task-success trend, cost per interaction, top regressions, roadmap |
| Monthly governance review | 45–60 min | Review risks, policy changes, audit items; confirm compliance | Legal/compliance w/ exec sponsor | Decision-log summary, change summaries, audit-trail exports |
| As-needed incident channel | — | Coordinate P1/P2 incidents + external comms | Platform engineer | Incident timeline, stakeholder updates, RCA |
| Quarterly roadmap/FinOps review | 45–60 min | Evaluate unit economics + model/provider mix, review ops plan, market trends | PM w/ finance + platform | Cost-per-interaction trends, vendor-mix report, savings backlog, ops-architecture changes |

---

## 4. Success metrics & dashboards

Defining **quality metrics** is often the hardest part of stakeholder communication — define
them **cross-functionally and up front**. Manual stakeholder review matters, but it's an
anti-pattern when one influential stakeholder's subjective feedback disproportionately drives the
roadmap. **Operational metrics** mirror traditional software (latency, success count) but can
have much higher variance given the pace of model releases and pricing changes — monitor
continuously for spikes/regressions/optimization.

**Track (mix depends on customer-facing vs. internal):**
- **Quality:** task success rate, SME agreement rate, harmful-output rate, judge-score trends,
  hallucination rate.
- **Operations:** P95 latency, error rate, incident MTTR, prompt/tool change-failure rate.
- **Cost:** cost per interaction, token input/output mix, cache-hit rate, vendor/model mix.
- **Adoption/business:** active users, retention, time saved, CSAT/NPS, revenue or cost
  avoidance.

**Recommended views:**
- **Executive:** ROI narrative, risk trend, budget vs. actual spend.
- **Product/engineering:** experiment throughput, regression alerts, guardrail trips.
- **Ops:** SLO compliance, incident heatmap, provider health.
