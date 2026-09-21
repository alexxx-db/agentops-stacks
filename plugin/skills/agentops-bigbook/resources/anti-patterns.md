# AgentOps Anti-Patterns (Big Book Ch. 1.4 + Ch. 6)

These "seem reasonable in theory but create problems in practice." Naming the anti-pattern a
team has walked into is often the highest-leverage thing you can do. Each row: the problem, the
risk it creates, and what to do instead.

| Anti-pattern | Problem | Risk | Do instead |
|---|---|---|---|
| **Starting too broad** | Undefined scope, unclear success criteria (e.g. "a chatbot that answers every employee question") | Agents that are mediocre across all domains rather than excellent in one | Focus on a single, well-scoped use case |
| **Overcomplex architecture** | Supervisor / multi-agent when a simple sequential chain would suffice | Orchestration overhead, debugging complexity, infinite loops between agents | Prefer sequential chains over complex multi-agent orchestration |
| **ReAct loops for single-tool tasks** | Wrapping predetermined actions in a Reason-and-Act loop | Added latency + cost with no benefit when the next action is never in doubt | Use direct function calls for deterministic actions |
| **Overcomplicated tools** | LLM-powered tools (e.g. Text-to-SQL) where a parameterized query would work | Latency, unpredictability, security risk | Reserve LLM-based tool selection for genuinely complex mapping |
| **Unoptimized retrieval** | Accepting the first RAG implementation without measuring/improving it | Poor outputs from irrelevant or missing retrieved context | Evaluation-driven retrieval optimization |
| **Ungoverned tools** | Agents with unrestricted endpoint access and broad credentials | Security/compliance/reliability failures that cascade system-wide | A centralized, governed tool registry |
| **No rollback or versioning** | Behavior emerges from prompts + tools + routing logic + data — none versioned | Can't debug or recover from production failures | Version-control **every** agent component |
| **No systematic evaluation** | Manual spot-checks instead of structured test suites | Failure modes invisible in dev that surface at scale in prod | Standardize on programmatic evaluation suites |
| **No human in the loop (high-stakes)** | Agents autonomously executing consequential actions | Unacceptable risk when agents misread edge cases or novel situations | Mandate human-in-the-loop for high-stakes actions |
| **No cost controls** | No per-request budgets, rate limits, or cost tracking | Runaway costs from expensive tool calls, excessive LLM iterations, unbounded retrieval | Proactive budget monitoring + limits (at the AI Gateway) |

---

## The subtle one: stakeholder-driven metric distortion (Ch. 6.4)

Manual reviews by stakeholders are important, **but** it becomes an anti-pattern when
**subjective feedback from one influential stakeholder disproportionately drives the roadmap**.
Guard against it: define quality metrics **cross-functionally and up front**, communicate them
clearly, and measure against the shared, written criteria — not the loudest voice.

---

## How to use this in advising

1. Listen for the symptom (unreliable, expensive, stuck, "our multi-agent thing keeps looping").
2. Map it to the anti-pattern above.
3. Prescribe the "do instead," and point to the skill/resource that implements it:
   - versioning/rollback → Databricks Asset Bundles (DABs)
   - systematic eval → `evaluation.md` + MLflow GenAI evaluation
   - ungoverned tools / cost → `governance-and-access-control.md`
   - overcomplex architecture → `deployment-patterns.md` (drop a pattern; prefer a chain)
