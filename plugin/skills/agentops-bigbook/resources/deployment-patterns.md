# Deployment Architecture Patterns (Big Book Ch. 2)

Four patterns for deploying agentic systems, progressing from simple to complex along **two
independent axes**:

- **Account / workspace isolation** — operational complexity and compliance strength.
- **Agent specialization** — coordination complexity and domain optimization.

> **Prime directive:** choose the **simplest architecture that meets your current needs**, then
> evolve incrementally as requirements change. Do not start at P4 because you might need it
> someday.

---

## Pattern 1 — Multi-Environment, Single Account, Single Agent

**Use case:** starting with agentic AI, or a single self-contained agent across dev/staging/prod
in one account. **Complexity: Low.**

- **Application:** one monolithic agent, one focused use case (e.g. customer support); simple
  progression *data prep → agent dev → automated eval → deployment*; one app per environment.
- **Config:** declarative YAML/JSON; one agent definition with three environment overlays
  (dev/staging/prod); serving endpoints per environment with consistent naming.
- **Infra:** single Unity Catalog metastore across environments; **enable Unity AI Gateway**;
  shared lakehouse with catalog/schema isolation per env; an MLflow tracking server per
  workspace; Delta Sharing for cross-env data; shared vector search / tools / UC models.
- **CI/CD:** single Git repo; automated dev → staging → prod promotion; testing gates = unit
  (dev) / integration (staging) / validation (prod); SME feedback loop in dev; batch inference
  in prod for offline eval.

## Pattern 2 — Multi-Environment, Single Account, Multi-Agent

**Use case:** composite systems with an orchestrator + specialized sub-agents in one account.
**Complexity: Medium** (orchestration up, but single account keeps access control simple).

- **Application:** clear separation of concerns; each agent deployed independently as a
  Databricks App (microservices style); a **supervisor agent orchestrates** interactions;
  supports both parallel execution and sequential chaining.
- **Config:** multiple config files; shared schemas enforced with **Pydantic**; per-agent
  environment overlays; interaction patterns defined in the orchestrator config.
- **Infra:** single metastore with **agent-specific schemas**; **Unity AI Gateway** for
  consistent access/policies/tracing/spend across models, agents, and tools; shared lakehouse
  with per-agent table isolation; per-agent deployment workflow; a centralized MLflow tracking
  server aggregating across agents; shared resource pools with per-agent indexes/versions.
- **CI/CD:** parallel per-agent workflows; independent deployment cadences; cross-agent
  integration testing in staging; per-agent gates (unit/integration/validation); coordinated
  promotion when agents have dependencies; batch inference for multi-agent eval.

> ⚠ **Before you build P2:** the multi-agent *pattern* is valid, but the recurring anti-pattern
> is reaching for it too early. Confirm a sequential chain genuinely won't do. And implement the
> supervisor as a **custom orchestrator** (see `platform-state.md`), not a deprecated managed API.

## Pattern 3 — Multi-Environment, Multi-Account/Workspace, Single Agent

**Use case:** enterprises needing strict environment isolation (separate AWS accounts or
Databricks workspaces per env), deploying one agent across that isolated infra.
**Complexity: Medium-high** (simple app logic, complex infra).

- **Application:** one agent across fully isolated account boundaries; each env is a sovereign
  deployment unit; agent code stays consistent while infrastructure varies; prod adds extra
  monitoring/feedback.
- **Config:** account-aware parameters; cross-account references via external IDs; **DABs are
  critical** for environment abstraction — `databricks.yml` holds account-specific targets; UC
  catalog/schema names parameterized per account; workspace URLs, service principals, and
  secrets injected at deploy time. Example:
  ```yaml
  targets:
    dev:
      workspace: { host: https://dev-workspace.cloud.databricks.com }
      resources: { jobs: { deployment_job: { parameters: [ { catalog: dev_catalog } ] } } }
    staging:
      workspace: { host: https://staging-workspace.cloud.databricks.com }
      resources: { jobs: { deployment_job: { parameters: [ { catalog: staging_catalog } ] } } }
  ```
- **Infra:** multiple metastores (one per account); **Unity AI Gateway** to standardize runtime
  governance and cost attribution while preserving model/provider choice; Delta Sharing becomes
  critical for cross-account data; separate lakehouse per env; duplicated MLflow servers and
  vector indexes; **PrivateLink** (or similar) for cross-account networking; each account has
  independent UC governance, cluster policies, secret scopes, service principals.
- **CI/CD:** Git branches map to accounts/envs; cross-account deploys need elevated perms;
  promotion includes account-switching logic; testing adapted for account boundaries;
  continuous deployment must handle cross-account IAM.

## Pattern 4 — Multi-Environment, Multi-Account, Multi-Agent

**Use case:** large enterprises with both agent composition and strict account isolation,
multiple teams, compliance requirements, large-scale prod. **Complexity: High** — needs a
dedicated MLOps/AgentOps team and mature governance.

- **Application:** full matrix (agents × accounts × environments); each agent independently
  versioned and deployed across account boundaries; prod adds post-deployment validation
  workflows, batch inference for large-scale eval, SME feedback loops, multi-agent routing.
- **Config:** hierarchical — base agent configs (per-agent YAML) + environment overlays
  (dev/staging/prod JSON) + account-specific overrides; validation via Pydantic + JSON Schema
  + pre-deployment validation jobs.
- **Infra:** multiple metastores with a complex sharing topology; **Unity AI Gateway as the
  unified runtime control layer across models, agents, tools, MCP servers, and harnesses**
  (centralized permissions, policies, observability, routing, cost); bidirectional Open Sharing;
  dedicated lakehouse per account; per-agent-per-account compute and vector indexes; MLflow
  servers federated across accounts; cross-account networking (PrivateLink prod↔staging, VPC
  peering / Transit Gateway for dev, centralized DNS).
- **CI/CD:** repo with multiple project dirs (M1/M2/M3); independent per-agent CI/CD;
  multi-stage promotion with cross-account gates; continuous deployment orchestrates
  cross-agent dependency resolution, account promotion sequences, and per-agent-per-account
  rollback.
- **Operational considerations:** observability spans per-agent-across-accounts,
  cross-agent interaction, account-level utilization, env-specific SLAs; cost attribution at
  agent level + env budgets + cross-account transfer costs; security = account-level audit
  logs + per-agent access policies + cross-account lineage.

---

## Choosing the right pattern

| Pattern | When to use | Team maturity | Primary challenges |
|---|---|---|---|
| Single account, single agent | Starting out, PoCs, single-purpose agents | Beginner | Initial setup, basic CI/CD |
| Single account, multi-agent | Composite agents, orchestration, microservices | Intermediate | Agent coordination, shared resources |
| Multi-account, single agent | Compliance, env isolation, enterprise governance | Intermediate/advanced | Cross-account networking, IAM complexity |
| Multi-account, multi-agent | Enterprise scale, many teams, complex systems | Advanced | Full operational overhead, needs a dedicated AgentOps team |

The P1→P4 progression *is* the organization's maturity journey. Advise teams to walk it, not
teleport to the end.

---

## Common pipeline components (shared by ALL patterns)

Regardless of complexity, every pattern shares these foundations:

- **Source control** — Git with branch strategies aligned to environments; PR review; commit
  triggers for CI/CD.
- **Testing strategy** — unit (dev) / integration + regression (staging) / validation (prod).
- **MLflow integration** — automated + human eval of quality; end-to-end tracing; production
  monitoring and feedback.
- **Lakehouse foundation** — Unity Catalog (AI asset registry + policy), Delta/Iceberg storage,
  vector search, AI tools & functions.
- **Runtime governance & control** — **Unity AI Gateway** (centralized governance across models,
  agents, tools), runtime permissions/policies/guardrails, usage observability + cost
  attribution + budgets/limits, model routing & provider management.
- **Deployment automation** — DABs as IaC, environment-specific parameter injection, automated
  rollback.
- **Feedback loops** — SME validation, continuous monitoring/eval, metrics-driven improvement.

## AgentOps Stacks

**AgentOps Stacks — this repo — is that scaffolding.** It provides pre-built pathways that
implement these patterns and remove much of the productionization complexity, and is the
recommended **starting point** for a new AgentOps project. Scaffold with the `agentops-stacks`
skill, then drive the build with `agentops-lifecycle`.

→ Hands-on: `agentops-stacks` scaffolds all of this (the DAB deployment unit, per-agent
Databricks Apps, and dev/staging/prod targets); `agentops-lifecycle` then drives it through
build → eval gate → promotion. Underlying: Databricks Asset Bundles, Model Serving, Apps, and
Unity Catalog (governance).
