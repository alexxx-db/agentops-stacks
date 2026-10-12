---
name: usage-tracking-ops
description: Set up and operate usage tracking / cost attribution for an AgentOps Stacks project — tag resources with project/environment/agent, attach a serverless usage policy, retrofit tags onto a project scaffolded without them, tag newly added compute, and query attributed usage in system.billing.usage. Triggers on "track usage", "cost attribution", "chargeback", "tag resources", "usage policy", "how much does my agent cost".
---

# usage-tracking-ops — Usage Tracking & Cost Attribution

Covers tagging an AgentOps Stacks project so its Databricks usage can be attributed to
the project, environment, and agent. Tags are for **tracking and monitoring**, not spend limits.
Say "serverless usage policy", never "budget policy". The bundle/API field is still named
`budget_policy_id`, and the CLI group is still `databricks account budget-policy`.

References: [Use tags to attribute and track usage](https://docs.databricks.com/aws/en/admin/account-settings/usage-detail-tags),
[Serverless usage policies](https://docs.databricks.com/aws/en/admin/usage/budget-policies),
[AI Search usage policies](https://docs.databricks.com/aws/en/ai-search/budget-policies).

## When to use

- "tag my agent's resources for cost attribution"
- "how much is this project costing?" / "usage by agent"
- "I scaffolded without tags, add them now"
- "set up a usage policy" / "attach a usage policy"
- "I added a job cluster / SQL warehouse / serving endpoint, how do I tag it?"
- deploy errors mentioning `budget_policy_id`

## Tag scheme

| Key | Value | Where |
|---|---|---|
| `project` | bundle name | the usage policy (serverless) and every classic/experiment resource |
| `environment` | `${bundle.target}` | the usage policy (serverless) and every classic/experiment resource |
| `agent` | agent name | per-agent MLflow experiments |

Serverless resources (Apps, Vector Search, Lakebase) get tags only from the usage policy.
Policies never tag classic compute. Per-agent app usage comes from `usage_metadata.app_name`.

Policy facts to keep in mind:
- A user with access to exactly one policy gets it applied to new resources automatically. With
  several and none chosen, the alphabetically first one is used. Set `budget_policy_id` explicitly.
- Granting access doesn't attach a policy to existing resources; redeploy with `budget_policy_id` set.
- Tag changes on a policy apply only to usage that starts after the change.
- A deleted policy's ID stays on the resource without error, but its usage is no longer tagged.
- Use `budget_policy_id`. Some resources also expose `usage_policy_id`, which is Private Preview.

## Key resources in the project

| File | Mechanism |
|---|---|
| `databricks.yml` | `variables.usage_policy_id`. Each app has `budget_policy_id: ${var.usage_policy_id}` |
| `resources/lakebase.yml` | `budget_policy_id` only (tags come from the policy) |
| `resources/vector_search.yml` | `budget_policy_id` on the VS endpoint (no direct tags). Name is shared across targets, so targets in one workspace share the endpoint and its policy. Index-level policies exist but aren't in the bundle schema; set them in the UI or with `update_index_budget_policy()` |
| `resources/data_pipeline.yml` | serverless pipeline: `tags` map plus `budget_policy_id` |
| `resources/uc_function_registration.yml` | Runs on serverless: billing tags come from `budget_policy_id`; the job `tags` map labels the job. A commented `job_clusters` example sets `custom_tags` for classic compute. An `existing_cluster_id` cluster must carry its own `custom_tags` |
| `resources/experiment.yml` | experiment `tags` list (not billed, but links traces) |
| `scripts/create_usage_policy.sh` | Creates one policy per target and wires it in (needs an account-level CLI profile) |
| `scripts/enable_usage_policy.py` | Writes policy IDs into `databricks.yml` and uncomments the `budget_policy_id` lines. Refuses if a target lacks an ID |
| `docs/usage-tracking.md` | Project-specific guide, including console steps for an admin |

Detect whether tagging is enabled: `grep -n "usage_policy_id:" databricks.yml`. If a
`budget_policy_id` line starts with `#`, no policy has been attached yet.

---

## Operations

### 1. Set up a serverless usage policy

Serverless usage (Apps, Vector Search, serverless jobs/pipelines, Lakebase) only carries custom
tags through a policy. **Never create one without the user's explicit confirmation.**

Creating a policy needs the **workspace admin** role or the **Serverless usage policy: Manager**
permission. Seeing every policy in the account also needs the account-level **Billing admin** role.

1. Ask whether the user is a workspace admin (or has the Manager permission).
   - **Yes, in the UI** → workspace: username → **Settings → Compute**, next to **Serverless usage
     policies** click **Manage → Create**. Name it `<project>-<target>`, add tags `project` and
     `environment`. Repeat for each target, then wire the IDs in (step 2).
   - **Yes, and they have an account-level CLI profile** (the CLI only has the account API) → run the script:
     ```bash
     databricks auth login --host <accounts-console-url> --account-id <account-id> -p my-account
     ./scripts/create_usage_policy.sh my-account dev staging prod
     ```
     If the project lacks the script (it was scaffolded with tags off), create the policies directly:
     ```bash
     databricks account budget-policy create -p my-account --json '{
       "policy": {"policy_name": "<project>-<target>",
                  "custom_tags": [{"key": "project", "value": "<project>"},
                                  {"key": "environment", "value": "<target>"}]}}'
     ```
   - **No** → give them the workspace UI steps above to forward to a workspace admin.
   - **They already have a policy** → the ID is on the policy's page (**Settings → Compute →
     Serverless usage policies → Manage**), or in `databricks account budget-policy list -p <account-profile>`.
     The policy **must carry `project` and `environment` tags**. Apps, Vector Search, and
     Lakebase take their tags only from the policy, so an untagged policy attributes nothing.
     Check the policy page, or an app's Settings page ("Selected policy has no tags").
2. Wire the IDs in. `create_usage_policy.sh` does this automatically. Otherwise run:
   ```bash
   python3 scripts/enable_usage_policy.py dev=<id> staging=<id> prod=<id>   # per target
   python3 scripts/enable_usage_policy.py --default <id>                     # one shared policy
   ```
   It sets `targets.<t>.variables.usage_policy_id` (or the default) and uncomments every
   `# budget_policy_id: ${var.usage_policy_id}` line in `databricks.yml` and `resources/*.yml`.
   If any target would be left without an ID, it exits **without changing anything**. Pass the
   missing targets and don't edit the files by hand to get around it.
3. If the project has no `scripts/enable_usage_policy.py` (it was scaffolded before the script
   existed), make the same edits by hand. Set `usage_policy_id` under every target (or the
   default) **before** uncommenting the `budget_policy_id` lines, so no target ends up with an empty ID.
4. **Grant access.** On each policy (**Settings → Compute → Serverless usage policies → Manage** →
   policy → **Permissions** → **Grant access**), grant the **User** role to the developers and to the
   staging/prod deploy service principals. Otherwise deploy fails.
5. `databricks bundle validate -t <target>`, then deploy.

### 2. Retrofit tags onto a project scaffolded with tags off

Add exactly what the template renders with `input_enable_cost_tags=yes`:

```yaml
# databricks.yml
variables:
  usage_policy_id:
    description: Serverless usage policy ID for usage tracking and cost attribution
    default: ""          # set once a policy exists
resources:
  apps:
    <agent>:
      # budget_policy_id: ${var.usage_policy_id}   # uncomment once the ID is set
```

```yaml
# resources/lakebase.yml (postgres_projects.memory)
      # budget_policy_id: ${var.usage_policy_id}

# resources/uc_function_registration.yml and resources/data_pipeline.yml
      # budget_policy_id: ${var.usage_policy_id}
      tags:
        project: <project>
        environment: ${bundle.target}

# Any job on classic compute: tag the cluster, since serverless policies don't cover it
      job_clusters:
        - job_cluster_key: main
          new_cluster:
            custom_tags:
              project: <project>
              environment: ${bundle.target}

# resources/vector_search.yml (vector_search_endpoints.vs_endpoint)
      # budget_policy_id: ${var.usage_policy_id}

# resources/experiment.yml and each per-agent experiment in databricks.yml
      tags:
        - key: project
          value: <project>
        - key: environment
          value: ${bundle.target}
        - key: agent
          value: <agent>
```

Never leave `budget_policy_id: ${var.usage_policy_id}` live while the variable is empty.
Keep it commented until a real ID is set, then wire it with operation 1 step 2. Copy
`scripts/enable_usage_policy.py` from a freshly rendered template if the project lacks it. Once tagging is retrofitted, `add_agent.py` and
`add_supervisor.py` detect the `usage_policy_id` variable and tag new agents automatically.
Then `databricks bundle validate -t dev`.

### 3. Tag newly added resources

| Resource | Add |
|---|---|
| Job cluster (`job_clusters[].new_cluster`) | `custom_tags: {project: <project>, environment: ${bundle.target}}` |
| Existing all-purpose cluster (`existing_cluster_id`) | the same `custom_tags` on the cluster itself; the job can't tag it |
| Serverless job (no cluster set) | `budget_policy_id: ${var.usage_policy_id}` for billing (don't rely on job tags for serverless), plus job `tags` to label the job |
| Serverless pipeline (`serverless: true`) | pipeline `tags` plus `budget_policy_id: ${var.usage_policy_id}`. Job-triggered pipelines don't inherit the job's policy, so set it on the pipeline |
| SQL warehouse | `tags: {custom_tags: [{key: project, value: <project>}, {key: environment, value: ${bundle.target}}]}` (not covered by usage policies) |
| Model serving endpoint | `tags: [{key: project, value: <project>}, ...]` plus `budget_policy_id: ${var.usage_policy_id}` |
| Lakebase project | `budget_policy_id: ${var.usage_policy_id}` only |
| Vector Search endpoint / App | `budget_policy_id: ${var.usage_policy_id}` only |

Limits: max 20 custom tags per resource. Keys and values may only contain letters, digits, and
`+ - = . , _ : @` (no spaces or `/`).

### 4. Query attributed usage

```sql
SELECT
  u.usage_date,
  u.custom_tags['environment']      AS environment,
  u.billing_origin_product           AS product,
  u.usage_metadata.app_name          AS app_name,
  SUM(u.usage_quantity)              AS dbus,
  SUM(u.usage_quantity * p.pricing.effective_list.default) AS est_list_cost
FROM system.billing.usage u
LEFT JOIN system.billing.list_prices p
  ON u.sku_name = p.sku_name
 AND u.cloud = p.cloud
 AND u.usage_start_time >= p.price_start_time
 AND (p.price_end_time IS NULL OR u.usage_end_time <= p.price_end_time)
WHERE u.custom_tags['project'] = '<project>'
  AND u.usage_date >= current_date() - INTERVAL 30 DAYS
GROUP BY ALL
ORDER BY u.usage_date DESC, est_list_cost DESC;
```

Per-agent usage: group by `usage_metadata.app_name` (App compute) and filter traces by the
experiment `agent` tag. Estimates use list price. Your negotiated rate may differ.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Deploy: permission denied on `budget_policy_id` | Deploying identity lacks **User** on the policy | Grant User on the policy to the user/SP (step 1.4) |
| Deploy: policy not found / invalid ID | Wrong account or a typo | Re-check `usage_policy_id` per target, then `databricks account budget-policy get <id>` |
| Usage stopped being tagged, deploys still succeed | The policy was deleted. Resources keep its ID, but it applies no tags | Create a new policy and wire its ID in (step 1) |
| Resource got a policy you didn't set | The deployer has access to policies, so one is applied by default (the only one, or the alphabetically first) | Set `budget_policy_id` explicitly |
| Serverless pipeline tags lag after a policy change | Pipeline is in development mode | Expected: up to 24 hours to propagate |
| App / Lakebase / VS rows have no tags, policy is attached | The policy itself has no tags | Add `project` and `environment` to the policy (Settings → Compute → Serverless usage policies) |
| Serverless rows have no `project` tag | `budget_policy_id` still commented, or the policy lacks the tags | Attach the policy (step 1). Tags aren't retroactive |
| No rows yet | Billing lag | `system.billing.usage` lags by a few hours |
| Tag rejected at deploy | Invalid characters (space, `/`) or more than 20 tags | Fix the key/value. Project and agent names are already safe |
| Job usage has no tags | Serverless: `budget_policy_id` still commented. Classic: the cluster lacks `custom_tags` (job tags don't reach an `existing_cluster_id` cluster) | Attach the policy, or add `custom_tags` to the job cluster or existing cluster |
| VS usage all under one `environment` | Targets share a workspace, so they share the one VS endpoint | Expected. Give each target its own endpoint name if you need the split |
| `environment` wrong for serverless usage | One shared policy across targets | Create one policy per target and override `usage_policy_id` per target |
