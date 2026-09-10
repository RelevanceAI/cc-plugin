---
name: relevance-evals
description: Manages evaluations and monitoring for agents and workforces (multi-agent systems) in Relevance AI — creating test cases, defining checks, configuring tool simulations, running evaluations, analysing results, setting up performance dashboards that monitor production traffic, and configuring alerts that notify when quality degrades. Use when testing agent or workforce behaviour, setting up automated testing, reviewing eval results, monitoring production agents for failures, or setting up alerts and notification channels for regressions.
---

# Evals & Monitoring Skill

Skill for managing pre-deploy testing and post-deploy monitoring of agents and workforces in Relevance AI.

## Overview

Evals work as two complementary surfaces:

- **Test sets + scenarios + checks** — pre-deploy testing. Scripted scenarios you replay against an agent or workforce, scored by checks. Think "integration tests for agents."
- **Performance dashboards** — post-deploy monitoring. Live production traffic is sampled and scored by checks. Think "Datadog / CloudWatch for agents."

## When to use what

| You want to…                                                                       | Use…                                                                            |
| ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| Test agent behaviour before publishing (regression suite, edge cases, "what if…")  | **Test sets + scenarios + checks** — pre-deploy unit/integration tests          |
| Monitor live production runs continuously, catch regressions in real conversations | **Performance dashboards** — sampling-based observability                       |
| Investigate "what's failing in production right now?"                              | List performance dashboards → list dashboard runs (failed only) → group reasons |

## Concepts

- **Test Sets** — Folders that group related test cases.
- **Test Cases (Scenarios)** — Simulated user interactions with expected outcomes. Every test case belongs to a test set.
- **Checks** — Reusable evaluation criteria (`name` + `check_config`). A check is a standalone object on the agent or workforce that scenarios and performance dashboards reference by `check_id`; see [Checks are a reusable library](#checks-are-a-reusable-library-reference-them-by-id). (Internally — in API responses and result records — the backend still calls these "rules", so eval-run results expose fields like `rule_results` and `rule_name`. They are the same thing.)
- **Performance Dashboards** — Production monitoring configs that bind an agent or workforce to a set of checks and a `sample_rate` (fraction of live conversations to evaluate).
- **Eval Batches** — A grouped execution of evaluations against an agent or workforce, identified by `eval_batch_id`.
- **Eval Runs** — Each scenario/check execution inside a batch, identified by `eval_run_id`.

### Checks are a reusable library (reference them by id)

A scenario or dashboard is scored by checks it references by `check_id`. The workflow is always:

1. **List** existing checks with `relevance_list_eval_checks`. Each has a `check_id`, `name`, `check_config`, and an `attached_to` list (the scenarios/dashboards using it; an empty `attached_to` means it's an orphan).
2. **Reference or create.** Reference an existing `check_id` if one already fits, otherwise create one with `relevance_create_eval_check` (you can create several in parallel).
3. **Attach** the `check_id`s when you create or update the scenario/dashboard.

Two things follow from checks being shared:

- **Editing a check applies everywhere** — `relevance_update_eval_check` changes it on every scenario and dashboard it's attached to; check `attached_to` first.
- **Detach ≠ delete** — detaching (`relevance_remove_eval_test_case_rule` / `relevance_remove_performance_dashboard_rule`) only unlinks the check from that one scenario/dashboard; `relevance_delete_eval_check` destroys it everywhere.

### How a scenario eval works

1. You call `relevance_run_evaluation` with a test set or specific scenarios. It returns an `eval_batch_id` and a URL — share the URL with the user immediately.
2. The system replays each scenario prompt against the agent/workforce, generating a real conversation or task.
3. An LLM judge (or deterministic check) scores the result against each attached check.
4. Poll for completion with `relevance_poll_eval_batch_result` (long-polls for ~50s per call). When `is_terminal` is true the batch is done; otherwise re-call with the same `eval_batch_id`.
5. Results show pass/fail per check with the judge's reasoning.

### How a performance dashboard works

1. The dashboard is enabled with `sample_rate ∈ (0, 1]` on an agent or workforce.
2. As production conversations complete, the system probabilistically samples them (per `sample_rate`).
3. Each sampled conversation is scored against the dashboard's checks by the LLM judge.
4. Scores feed the timeseries chart and the per-test table — surfaced via `relevance_get_performance_dashboard_timeseries` and `relevance_list_performance_dashboard_runs`.

### Pre-publish checks (publish gate / "Test and Publish")

An agent or workforce can have **one pre-publish check config**: 1–10 test sets, each with a `threshold_score` (minimum pass rate, 0–100) and a `block_on_failure` flag. When a config exists, `relevance_publish_agent` / `relevance_publish_workforce` run every configured test set against the current draft instead of publishing directly, and **automatically publish the draft to live when every blocking test set meets its threshold** — there is no separate publish step afterwards. A below-threshold non-blocking test set is reported but never blocks. This is the native "block bad changes from going live" mechanism.

Workflow:

1. `relevance_get_pre_publish_status` — read the current config (if any) and recent check runs.
2. If no config exists, build test sets + scenarios first (see the Pre-Deploy Workflow), then `relevance_upsert_pre_publish_config` with the complete test-set list.
3. Publish as normal (`relevance_publish_agent` / `relevance_publish_workforce`) — the gate runs automatically and the response includes a `check_id`. Pass a concise `version_name` and `version_description` so a successful publish leaves a useful version-history trail. `skip_checks: true` bypasses the gate — only pass it when the user explicitly asks to publish without running the checks.
4. `relevance_poll_pre_publish_check` with the returned `check_id` (long-polls ~50s per call; re-call until `is_terminal` is true).
5. Report the `outcome` and per-test-set scores, and state clearly whether the draft was published.

Failure modes:

- **"already running" (409)**: a check is already in flight for this resource — don't re-publish; find its `check_id` with `relevance_get_pre_publish_status` and poll it.
- **"draft version changed" (409)**: the draft was saved again between that save and the trigger — re-publish from the latest draft.

**Surface this proactively** whenever a user asks how to gate publishes on tests (e.g. "how do I block bad agent changes from going live?"). The answer is: turn the resource's test sets into pre-publish checks with `relevance_upsert_pre_publish_config` — every publish then enforces the gate automatically.

## Eval & Dashboard URLs

`relevance_run_evaluation` returns a `url` field pointing directly to the eval run page. **Always share this URL with the user immediately after starting an evaluation.**

`relevance_create_performance_dashboard`, `relevance_list_performance_dashboards`, `relevance_update_performance_dashboard`, and `relevance_get_performance_dashboard_timeseries` all return a dashboard `url`. The per-check mutators (`relevance_add_performance_dashboard_rule`, `relevance_remove_performance_dashboard_rule`) embed the URL in their text response. Share the dashboard URL with the user the same way you share eval-run URLs.

### Agents vs Workforces

Evals and dashboards work for both individual agents and workforces (multi-agent systems). The tools are the same — only `resource_type` and a few fields differ:

| Concept           | Agent                                    | Workforce                                                                                                        |
| ----------------- | ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `resource_type`   | `agent`                                  | `workforce`                                                                                                      |
| `resource_id`     | The agent ID                             | The workforce ID                                                                                                 |
| Execution unit    | Conversation                             | Workforce task                                                                                                   |
| What happens      | Creates a conversation from the scenario | Creates a workforce task from the scenario                                                                       |
| Tool simulation   | Direct tool overrides via `tool_configs` | **Per-node** overrides via `node_configs[node_id]` (scopes each graph node — and each sub-agent — independently) |
| Tool-usage checks | Scoped to the agent's conversation       | **Scope to a node** with `node_id` to assert that node's (or its sub-agent's) tool/sub-agent calls               |

> **Not tools.** Evals and performance dashboards accept only `resource_type: agent | workforce` — you **cannot** attach an eval set or a performance dashboard to an individual tool. To monitor a tool, evaluate the agent or workforce that calls it. A tool's own health surfaces through **usage analytics** (execution counts, error rates, credits — see the `relevance-analytics` skill) and its run history, not through evals or live scoring.

> **Workforces are node-scoped, not agent-scoped.** A workforce is a graph of **nodes**; the same agent can appear in more than one node. Both tool simulation and tool-usage checks key off the graph `node_id`, so each occurrence is configured independently — and a node's **sub-agents are callable like tools**, so you can simulate or assert them too. See [Tool Simulation Config](#tool-simulation-config) and [Check Types](#check-types). Discover a workforce's nodes with `relevance_get_workforce`, and a node's full toolset (its tools **and** sub-agents, with their `action_id`s) with `relevance_get_agent_tools` passing the node's `agent_id` + `workforce_context: { workforce_id, node_id }`.

---

## Dedicated Tools

Every tool requires `resource_type` (`agent` or `workforce`) and `resource_id`.

### Eval Check Tools (the reusable check library)

The checks that scenarios and dashboards reference — see [Checks are a reusable library](#checks-are-a-reusable-library-reference-them-by-id).

| Tool                          | Description                                                                                            |
| ----------------------------- | ------------------------------------------------------------------------------------------------------ |
| `relevance_list_eval_checks`  | List every check on the agent or workforce, with `check_id`, `name`, `check_config`, and `attached_to` |
| `relevance_create_eval_check` | Create a check (`name` + `check_config`); returns its `check_id`                                       |
| `relevance_update_eval_check` | Update a check by `check_id`. **Affects every scenario/dashboard using it**                            |
| `relevance_delete_eval_check` | Delete a check by `check_id`. Removes it from the library and detaches it everywhere                   |

### Test Set Tools

Test sets are collections (like folders) that group related test cases. Every test case must belong to a test set — create a test set first, then add test cases to it.

| Tool                             | Description                                                        |
| -------------------------------- | ------------------------------------------------------------------ |
| `relevance_create_eval_test_set` | Create a test set to group test cases                              |
| `relevance_get_eval_test_set`    | Get a specific test set by ID                                      |
| `relevance_list_eval_test_sets`  | List all test sets for an agent or workforce                       |
| `relevance_update_eval_test_set` | Update a test set (**patch semantics** — only send changed fields) |
| `relevance_delete_eval_test_set` | Delete a test set (optionally with its test cases)                 |

### Test Case (Scenario) Tools

Test cases define individual evaluation scenarios. Each test case has a prompt, a set of attached checks (by `check_id`), and optionally a tool simulation config.

| Tool                              | Description                                                                                     |
| --------------------------------- | ----------------------------------------------------------------------------------------------- |
| `relevance_create_eval_test_case` | Create a test case with scenario, `check_ids` (existing checks), and optional simulation config |
| `relevance_get_eval_test_case`    | Get a specific test case by ID                                                                  |
| `relevance_list_eval_test_cases`  | List test cases within a test set                                                               |
| `relevance_update_eval_test_case` | Update a test case's metadata (name, scenario) — **does not modify checks**                     |
| `relevance_delete_eval_test_case` | Delete a test case                                                                              |

### Test Case Check Management (attach / detach existing checks)

Attach/detach checks on a test case (**`relevance_update_eval_test_case` can't change checks** — use these). Creating and editing the checks themselves lives in [the check library](#checks-are-a-reusable-library-reference-them-by-id).

| Tool                                   | Description                                                                                       |
| -------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `relevance_add_eval_test_case_rule`    | Attach one or more existing checks to a test case by `check_id` (`check_ids`)                     |
| `relevance_remove_eval_test_case_rule` | Detach a check from a test case by `check_id` — unlinks here only; the check stays in the library |

To see existing checks on a test case (including their `check_id`s), use `relevance_get_eval_test_case`; to see the whole agent or workforce's check library, use `relevance_list_eval_checks`.

### Test Case Simulation Config

| Tool                                             | Description                                                                      |
| ------------------------------------------------ | -------------------------------------------------------------------------------- |
| `relevance_set_eval_test_case_simulation_config` | Set or clear tool simulation config on a test case (full replacement, not patch) |

### Evaluation Execution

| Tool                       | Description                                                                                                          |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `relevance_run_evaluation` | Run an evaluation using test scenarios. Defaults to the active version; pass `version_id` to pin a specific version. |

### Batch & Run Tools

| Tool                               | Description                                                                                              |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `relevance_list_eval_batches`      | List eval batches for an agent or workforce                                                              |
| `relevance_get_eval_batch_summary` | Get batch summary with scores and check results (single-shot)                                            |
| `relevance_poll_eval_batch_result` | Long-poll a batch until it's terminal or the wait window elapses. Returns the summary plus `is_terminal` |
| `relevance_list_eval_runs`         | List eval runs inside a batch                                                                            |
| `relevance_cancel_eval_batch`      | Cancel pending/running eval runs in a batch                                                              |
| `relevance_delete_eval_batch`      | Delete a batch and all its eval runs                                                                     |

### Performance Dashboard Tools

| Tool                                             | Description                                                                                            |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| `relevance_create_performance_dashboard`         | Create a dashboard from existing checks (`check_ids`, max 10), `sample_rate`, and `enabled`            |
| `relevance_list_performance_dashboards`          | List dashboards for an agent or workforce (returns each dashboard's checks, sample rate, enabled flag) |
| `relevance_update_performance_dashboard`         | Update dashboard **metadata** (name / sample_rate / enabled) using patch semantics                     |
| `relevance_delete_performance_dashboard`         | Delete a dashboard (loses historical runs; underlying checks are NOT deleted)                          |
| `relevance_get_performance_dashboard_timeseries` | Score timeseries for a dashboard between two dates                                                     |
| `relevance_list_performance_dashboard_runs`      | List eval runs from a dashboard, optionally filtered to a single check and/or failed-only              |

### Performance Dashboard Check Management (attach / detach existing checks)

Mirror of the test-case surface — attach/detach checks on a dashboard (**not** via `relevance_update_performance_dashboard`). Creating and editing the checks themselves lives in [the check library](#checks-are-a-reusable-library-reference-them-by-id).

| Tool                                          | Description                                                                                       |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `relevance_add_performance_dashboard_rule`    | Attach existing checks to a dashboard by `check_id` (`check_ids`); fails if it would exceed 10    |
| `relevance_remove_performance_dashboard_rule` | Detach a check from a dashboard by `check_id` — unlinks here only; a dashboard must keep ≥1 check |

### Pre-Publish Check Tools (publish gate)

See [Pre-publish checks](#pre-publish-checks-publish-gate--test-and-publish) for the workflow and failure modes. The gate itself runs through the plain publish tools (`relevance_publish_agent` / `relevance_publish_workforce`), which trigger the checks automatically when a config exists and return a `check_id`.

| Tool                                  | Description                                                                                                                                             |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `relevance_get_pre_publish_status`    | The publish-gate config (or `config: null`) plus recent check runs with per-test-set scores and publish outcomes                                        |
| `relevance_upsert_pre_publish_config` | Create or replace the gate: 1–10 test sets, each with `threshold_score` + `block_on_failure`. **Full replace**                                          |
| `relevance_delete_pre_publish_config` | Remove the gate — publishes are no longer checked afterwards                                                                                            |
| `relevance_poll_pre_publish_check`    | Long-poll a check by `check_id` (returned by a gated publish) until `is_terminal`; terminal responses include an `outcome` stating whether it published |

### Patch Semantics (Important)

The update tools (`relevance_update_eval_test_case`, `relevance_update_eval_test_set`, `relevance_update_performance_dashboard`) use **patch semantics**: omitted fields are preserved. Checks are managed separately — attach/detach with the per-surface tools, edit the check itself with `relevance_update_eval_check`.

---

## Pre-Deploy Workflow (Test Sets + Scenarios + Checks)

**IMPORTANT:** Follow this workflow in order. Each step depends on the previous one.

```
┌─────────────────────────────────────────────────────────────────┐
│  STEP 1: Create a Test Set                                      │
│  └─► Required — test cases must belong to a test set            │
└───────────────────────────┬─────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 2: Create Test Cases (Scenarios)                          │
│  └─► First get/create the checks (list_eval_checks →            │
│      create_eval_check), then create test cases that reference  │
│      them by check_ids                                          │
└───────────────────────────┬─────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 3: Run Evaluation                                         │
│  └─► Trigger with test_set_id or scenario_ids                   │
└───────────────────────────┬─────────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  STEP 4: Poll & Read Results                                    │
│  └─► Check batch status, get detailed eval run results          │
└─────────────────────────────────────────────────────────────────┘
```

### Step 1: Create a Test Set

```
relevance_create_eval_test_set(resource_type, resource_id, name: "My Test Suite")
→ { "test_set_id": "..." }  // Save this for Step 2
```

You can check existing test sets first:

```
relevance_list_eval_test_sets(resource_type, resource_id)
```

### Step 2: Create Test Cases with Checks

Get the `check_id`s first (reuse or create — see [Checks are a reusable library](#checks-are-a-reusable-library-reference-them-by-id)). Before creating new checks, follow [Runbook: Authoring checks](#runbook-authoring-checks-ground--interview). Then call `relevance_create_eval_test_case` with those `check_ids` and the (required) `test_set_id`.

> **🛑 Before running, simulate side-effecting tools.** Default rule: **if a tool connects to an external service or modifies data, simulate it** (only leave read-only / pure-compute tools live). Set `tool_simulation_config` on the test set — **default ON, unless the user asks for a live run** — so the eval doesn't fire real actions (real webhooks, emails, writes). Simulated calls still count for `tool_usage` checks. See [Tool Simulation Config](#tool-simulation-config).

#### Proposing Test Cases to Users

When proposing test cases to a user before saving them, every proposed case MUST include both:

- a **user role-play prompt** the test will replay (the words the simulated user will say to the agent), AND
- at least one **success criterion / check** the agent's response should satisfy (a concrete check, not just a description of what the test "covers").

A one-line "what this checks" description without these two parts is not approvable.

#### Scenario Configuration

**`prompt`** — Describes the persona and situation for the simulated user. The LLM generates realistic messages based on this.

**`max_turns`** (default: 10, range: 1–50) — Controls how many back-and-forth exchanges happen before evaluation.

**`first_message`** — When the user asks to pin the opening line ("start by saying X"), put it here instead of writing it into `prompt`. It is sent to the agent verbatim as the opening message.

```
scenario: {
  prompt: "You are an impatient customer.",
  first_message: "I need help with my order #12345"
}
```

**`runs_per_scenario`** (default: 1, range: 1–10) — Run the same scenario multiple times to test consistency. AI responses vary — 3–5 catches flaky behavior.

#### Simple example (LLM Judge checks)

First create the checks (or reuse existing `check_id`s from `relevance_list_eval_checks`):

```
relevance_create_eval_check(resource_type, resource_id,
  name: "Greets user", check_config: { type: "llm_judge", prompt: "The agent greets the user warmly" })
→ { "check_id": "<check_a>" }

relevance_create_eval_check(resource_type, resource_id,
  name: "Offers help", check_config: { type: "llm_judge", prompt: "The agent asks how it can help" })
→ { "check_id": "<check_b>" }
```

Then create the test case referencing them:

```
relevance_create_eval_test_case(
  resource_type, resource_id,
  test_set_id: "<test_set_id>",
  name: "Basic Greeting",
  scenario: { prompt: "Hello, I need help", max_turns: 5 },
  check_ids: ["<check_a>", "<check_b>"]
)
→ { "display_id": "..." }
```

#### Mixed check types example

Create the checks, then attach them:

```
relevance_create_eval_check(resource_type: "agent", resource_id: "<agent_id>",
  name: "Uses research tool",
  check_config: { type: "tool_usage", tool_id: "<tool_action_id>", position: "anywhere", operator: "at_least", count: 1 })
→ { "check_id": "<check_tool>" }
// ...and likewise for the llm_judge and string_contains checks → <check_details>, <check_name>

relevance_create_eval_test_case(
  resource_type: "agent", resource_id: "<agent_id>",
  test_set_id: "<test_set_id>",
  name: "Company Research Request",
  scenario: {
    prompt: "You are a sales rep who needs background on Acme Corp before a call.",
    max_turns: 5
  },
  check_ids: ["<check_tool>", "<check_details>", "<check_name>"]
)
```

#### Single-input execution example

When the agent just needs to execute a task with no back-and-forth, use `max_turns: 1`:

```
relevance_create_eval_test_case(
  resource_type: "agent", resource_id: "<agent_id>",
  test_set_id: "<test_set_id>",
  name: "Data enrichment task",
  scenario: {
    prompt: "You need company data for a sales report.",
    first_message: "Enrich the company profile for Acme Corp",
    max_turns: 1
  },
  check_ids: ["<check_calls_enrichment>", "<check_structured_data>"]
)
```

#### Workforce example (node-scoped simulation + sub-agent tool-usage)

For a workforce, first discover node ids and each node's toolset, then scope simulation and checks per node.

```
// 1. Discover nodes (each agent node exposes its node_id and agent_id)
relevance_get_workforce(workforce_id: "<workforce_id>")
→ nodes: [{ node_id: "<summarizer_node>", config: { entity_link: { agent_id: "<summarizer_agent_id>" } }, ... }, ...]

// 2. Discover the summarizer node's tools AND sub-agents (with action_ids)
relevance_get_agent_tools(agent_id: "<summarizer_agent_id>",
  workforce_context: { workforce_id: "<workforce_id>", node_id: "<summarizer_node>" })
→ tools: [
    { action_id: "<send_email_action_id>",  kind: "tool" },
    { action_id: "<web_search_action_id>",  kind: "tool" },
  ]

// 3. Simulate the summarizer node's side-effecting tool (test-set level, per node)
relevance_update_eval_test_set(
  resource_type: "workforce", resource_id: "<workforce_id>", test_set_id: "<test_set_id>",
  tool_simulation_config: {
    node_configs: {
      "<summarizer_node>": {
        tool_configs: { "<send_email_action_id>": { overrides: { default: {
          output_overrides_enabled: true,
          output_overrides: { simulate_output: { is_simulated: true, simulation_prompt: "Return a sent-email confirmation." } }
        } } } }
      }
    }
  }
)

// 4. Node-scoped check: the summarizer node used web_search at least once
relevance_create_eval_check(resource_type: "workforce", resource_id: "<workforce_id>",
  name: "Summarizer searched the web",
  check_config: { type: "tool_usage", tool_id: "<web_search_action_id>", node_id: "<summarizer_node>",
                  position: "anywhere", operator: "at_least", count: 1 })
→ { "check_id": "<check_search>" }
```

Remember the `action_id`-not-`chain_id` rule applies to **sub-agents** too: to assert "the orchestrator invoked the Summarizer sub-agent", use a `tool_usage` check on the orchestrator node with `tool_id` = the Summarizer's `action_id` (a `kind: "sub_agent"` entry from step 2).

To view test cases in a test set:

```
relevance_list_eval_test_cases(resource_type, resource_id, test_set_id: "<test_set_id>")
```

### Step 3: Run Evaluation

```
relevance_run_evaluation(
  resource_type, resource_id,
  evaluation_run_name: "My Eval Run",
  test_set_id: "<test_set_id>"
)
→ { "eval_batch_id": "...", "eval_run_ids": ["...", "..."] }
```

**Run specific scenarios (ad-hoc):**

```
relevance_run_evaluation(
  resource_type, resource_id,
  evaluation_run_name: "My Eval Run",
  scenario_ids: ["<test_case_id_1>", "<test_case_id_2>"]
)
```

**Evaluate a specific version (discover-then-run):**

```
relevance_list_agent_versions(agent_id)        // or relevance_list_workforce_versions(workforce_id)
→ pick the version_id you want to test

relevance_run_evaluation(
  resource_type, resource_id,
  evaluation_run_name: "Eval of v3",
  test_set_id: "<test_set_id>",
  version_id: "<version_id>"
)
```

> **Version pinning:** Omit `version_id` to evaluate the active (published) version — it's resolved automatically. Pass `version_id` to evaluate a specific past/draft version; to evaluate the **current draft**, skip the version list and pass the `draft_version_id` from the resource's get-tool summary. For **workforces** this is **topology-only**: it pins the graph wiring of that version, but nested agents/tools still run their latest version (full historical reproduction isn't supported).

### Step 4: Poll & Read Results

**IMPORTANT: Evaluations take time (30 seconds to several minutes). You MUST poll automatically — do NOT stop and ask the user. Keep polling until done.**

#### Polling loop

1. Call `relevance_get_eval_batch_summary` with the `eval_batch_id`.
2. Check the `status` field:
   - `queued` or `running` → **wait 10 seconds**, call again. Repeat until terminal.
   - `completed`, `failed`, or `cancelled` → proceed to reading results.
3. While polling, briefly note progress (e.g. "Still running — 2/5 complete") but do NOT ask the user whether to continue.

The summary response includes: `{ status, tasks_evaluated, total_runs, completed_runs, failed_runs, cancelled_runs, summary_score, name, rules }`. `summary_score` is on a **0-1 scale** (1 = all checks passed).

#### Reading results

1. Call `relevance_list_eval_runs` with the `eval_batch_id` to enumerate eval runs.
2. Each completed eval run has `result.rule_results` (this is the actual field name; each entry is a check result) with `rule_name`, `passed`, and `reason` for each check.
3. Present results in a clear table: scenario name, pass/fail per check, and the LLM judge's reasoning for any failures.

For a **workforce**, a node-scoped `tool_usage` check (one created with a `node_id`) is scored only against that node's conversation(s), so its `rule_results` entry is attributable to that node — name the checks accordingly (e.g. "Summarizer searched the web") so the table makes clear which node each tool/sub-agent assertion covers.

---

## Runbook: Verifying a fix

After editing an agent or workforce to fix a diagnosed failure, verify the draft before reporting the fix as done:

1. **Capture the draft id.** Re-fetch the resource; its summary carries `draft_version_id` — the version your edit lives on. An eval run without it evaluates the live, unfixed version.
2. **Find real coverage.** List test sets and their scenarios. Don't branch on the `quality.has_evals` flag alone — it is also true when only orphan checks or empty test sets exist.
3. **Pick the smallest run.** Prefer the specific `scenario_ids` that exercise the diagnosed failure over a whole test set — evals cost credits.
4. **No coverage? Create it, then run.** Order matters: check (`relevance_create_eval_check`) → test set → scenario (`relevance_create_eval_test_case` requires `test_set_id` + `check_ids`). Base the scenario on the failing task's input, stripped of secrets and personal data.
5. **Confirm side-effecting tools are simulated** (the simulation default above) before running.
6. **Run pinned to the draft** — `relevance_run_evaluation(..., scenario_ids, version_id: <draft_version_id>)` — share the returned URL, and poll to terminal. If the run outlasts the conversation turn, share the URL, say it's still running, and (if your harness can schedule a future message) offer a check-back.
7. **Classify the outcome — three ways, not two.**
   - Checks failed → the fix is **not confirmed**: report failures-first and iterate.
   - Run errored (credits, timeout, judge/platform failure) → verification is **inconclusive** — never report the fix as verified.
   - All relevant checks passed → confirm the current `draft_version_id` still matches the one you evaluated (an edit mid-run invalidates the result), then report verified.

Caveats: workforce `version_id` pins topology only — nested agents run their latest version, so verify a nested-agent fix through that agent's own evals. Tools have no evals; verify a tool fix with a draft tool run along the failing path.

---

## Post-Deploy Workflow (Performance Dashboards)

Performance dashboards monitor live production traffic by sampling conversations and scoring them with checks. Use these AFTER an agent or workforce is live.

> **Heads-up:** dashboards only score the **latest published version** of the agent/workforce. Creating a dashboard on an unpublished agent or workforce succeeds, but no runs will fire until it's published — especially worth calling out when you've just generated a new agent/workforce and haven't published it yet. If `relevance_create_performance_dashboard` returns a `publish_hint` field, surface it to the user verbatim.

### Setup

```
┌──────────────────────────────────────────────────────────────────┐
│  1. Create or reuse the checks (max 10)                          │
│     └─► relevance_list_eval_checks / relevance_create_eval_check │
│  2. Create the dashboard from those check_ids                    │
│     └─► relevance_create_performance_dashboard with              │
│         check_ids: [...], sample_rate, enabled                   │
└──────────────────────────────────────────────────────────────────┘
```

Example:

```
relevance_create_eval_check(resource_type: "agent", resource_id: "<agent_id>",
  name: "Greets user", check_config: { type: "llm_judge", prompt: "The agent greets the user" })
→ { "check_id": "<check_greet>" }
relevance_create_eval_check(resource_type: "agent", resource_id: "<agent_id>",
  name: "No raw errors leaked",
  check_config: { type: "llm_judge", prompt: "The reply contains no raw API error messages, stack traces, or internal IDs" })
→ { "check_id": "<check_no_errors>" }

relevance_create_performance_dashboard(
  resource_type: "agent", resource_id: "<agent_id>",
  name: "Production quality watch",
  check_ids: ["<check_greet>", "<check_no_errors>"],
  sample_rate: 0.1,             // evaluate 10% of conversations
  enabled: true
)
```

### Manage the dashboard's checks

`relevance_update_performance_dashboard` does NOT manage checks. Use:

- `relevance_add_performance_dashboard_rule(…, config_id, check_ids)` — attach; fails if it would exceed 10 checks.
- `relevance_remove_performance_dashboard_rule(…, config_id, check_id)` — detach; fails if it would leave zero checks (delete the dashboard instead).

To create or edit the checks themselves, use the [check library](#checks-are-a-reusable-library-reference-them-by-id) tools.

### Reading dashboards

- **Timeseries** for the chart view (window must be ≤ 31 days):

  ```
  relevance_get_performance_dashboard_timeseries(
    resource_type, resource_id,
    config_id: "<dashboard_id>",
    start_date: "2026-05-06T00:00:00Z",   // last 7 days
    end_date:   "2026-05-13T00:00:00Z",
    check_id:   "<optional check_id to scope to a single check>"
  )
  ```

- **Per-run table** — `start_date` and `end_date` are **required**:

  ```
  relevance_list_performance_dashboard_runs(
    resource_type, resource_id,
    config_id: "<dashboard_id>",
    start_date: "2026-05-06T00:00:00Z",   // last 7 days
    end_date:   "2026-05-13T00:00:00Z",
    failed_only: true,                    // optional
    check_ids:  ["<check_id>", ...]       // optional — scope failure filter to specific checks
  )
  ```

  When `failed_only=true` and `check_ids` is omitted, the tool automatically fetches the dashboard's full check set and filters to failures across all of them — so a single call covers the dashboard, no per-check loop needed.

### Runbook: "Investigating recent failures"

When the user asks "what's failing in production?" / "monitor my recent failures" / "any regressions" / "how's my agent doing":

1. `relevance_list_performance_dashboards(resource_type, resource_id)` — if none exist, offer to set one up. Otherwise pick the enabled dashboard(s). (If you already fetched the agent/workforce, its `quality` block — `has_performance_dashboard`, `performance_dashboard_active` — already tells you whether monitoring exists, so you can decide whether to offer setup without this call; you still need the list to pick _which_ enabled dashboard to pull runs from.)
2. For each dashboard, list recent failed runs in **one call**:
   ```
   relevance_list_performance_dashboard_runs(
     resource_type, resource_id,
     config_id: "<dashboard_id>",
     start_date: "<7 days ago, ISO 8601>",
     end_date:   "<now, ISO 8601>",
     failed_only: true,
     page_size: 20
   )
   ```
3. Group failures by check name (eval-run results expose this as `rule_name` per check — backend storage term). For each, surface 1-2 representative failed runs with the LLM judge's `reason` field.
4. Summarise crisply: sample window, the dominant failing check(s), and the dominant failure reason.
5. Suggest next steps: tighten the system prompt, attach a missing tool, or turn the failure into a regression test by adding a scenario (`relevance_create_eval_test_case`) that exercises the same case.

### Recommend a periodic check-up (only when unmonitored)

When a dashboard has an alert configured, degradation is reported automatically — no scheduled check-up is needed; don't offer one. Recommend a periodic re-check only for a resource with no monitoring/alerting yet, paired with the offer to set monitoring up: if your harness can schedule a future check (a wake-up message or reminder), offer to schedule it and run this runbook then; if it can't, tell the user when to come back and what to look at. The scheduling and notification mechanics themselves are harness-specific — this skill only recommends the check-up and defines what to review.

### Alerts — get told instead of remembering to look

Alerts notify email, Slack, or Microsoft Teams when one dashboard check or its overall score degrades. They attach to agents or workforces, never individual tools. Offer one after setting up a dashboard; use the Relevance app UI if alert tools are unavailable.

Before creating, list existing alerts and ask which channels to use. Never invent recipients or connection IDs. One channel entry per type; one email entry contains all recipients. A caller-scoped empty list does not prove no teammate created an alert.

- Thresholds use fractions: `0.8` means 80%.
- Results are bucketed; `2 of 3` breaching buckets is less noisy than `1 of 1`.
- Low-traffic resources need a meaningful minimum sample size.
- Use at least a 60-minute cooldown unless the user requests faster repeats.

An alert is inert when its dashboard is not scoring live traffic or its rule does not belong to that dashboard. State is `ok`, `alarm`, or `insufficient_data`; the latter usually means too little traffic or too high a sample minimum. Only current state and latest timestamps are retained, not firing history.

### Agent alerts (a separate thing from monitor alerts)

Agent alerts cover operational events without a dashboard. Use `relevance_get_agent(summary: false)`, preserve every existing `notifications_config.notifications` entry, then call `relevance_update_agent({patch: {notifications_config}})` and publish normally. The list replaces wholesale.

Each notification requires `id`, `enabled`, `trigger`, and `channels`. Supported trigger types are `status_change`, `tool_error`, `agent_error`, `agent_published`, and `agent_triggers_updated`. Preserve escalation-status notifications: removing one can strand human handoffs.

### When dashboards return no data

- `enabled: false` on the dashboard — re-enable.
- `sample_rate: 0` — bump up.
- No live traffic on the agent or workforce — verify it's actually being invoked in production.
- Date range too narrow / too old — widen it.

---

## Eval Design Philosophy

### The Purpose of Evals

When an eval runs:

1. The system triggers the agent or workforce with the test scenario prompt
2. The agent generates a real conversation, or the workforce executes a real task
3. An LLM judge evaluates the output against the defined checks
4. Results show whether the criteria were met

### Designing a Good Test Suite

| Principle              | Description                                                                        |
| ---------------------- | ---------------------------------------------------------------------------------- |
| **Full Coverage**      | Cover all major use-cases and functionalities the agent or workforce should handle |
| **Minimal Test Cases** | Keep the number of test cases small — each eval is expensive and time-consuming    |
| **Targeted Scenarios** | Each test case should focus on testing specific behaviors, not everything at once  |
| **Realistic Prompts**  | Use prompts that mirror how real users would interact with the agent or workforce  |
| **Specific Checks**    | Each test case has its own checks tailored to what that scenario should achieve    |

**DO:**

- Test one primary capability per test case
- Use realistic user language and requests
- Include edge cases (budget constraints, special requirements, errors)
- Write checks that are specific and measurable

**DON'T:**

- Create dozens of overlapping test cases
- Use artificial or overly formal prompts
- Write vague checks like "agent is helpful"

### Example: Flight Search Agent Test Suite (5 tests)

| Test Case                | Use Case Covered                   | Key Checks                                           |
| ------------------------ | ---------------------------------- | ---------------------------------------------------- |
| Basic One-Way Flight     | Core search functionality          | Uses tool correctly, presents options, shows details |
| Round-Trip International | Return dates, international routes | Handles both legs, correct airports                  |
| Business Class Request   | Travel class preferences           | Sets correct class, premium options                  |
| Family Travel            | Multi-passenger handling           | Correct passenger count, total pricing               |
| Budget-Conscious Search  | Price optimization                 | Prioritizes price, acknowledges flexibility          |

### Cost-Benefit Tradeoff

Each eval costs credits (agent execution + LLM judge). On production dashboards keep `sample_rate` low (0.05–0.2) for high-traffic agents — sampling is what keeps costs bounded.

---

## Tool Simulation Config

Tool simulation lets you override or simulate tool outputs during evaluation runs, so you can test agent/workforce logic without calling real external services. Applies to **pre-deploy scenario runs only** — performance dashboards score real production calls.

> **🛑 Default: simulate every tool that has a real-world side effect.** When you set up evals for an agent/workforce, **simulate any tool that does something real** — webhooks, email / Slack / SMS sends, CRM or database writes, payments, file uploads, anything irreversible — **unless the user explicitly says to run it live.** Set `tool_simulation_config` on the **test set** (so it covers every scenario) before running. If you don't, the eval triggers the agent for real and **fires those side effects on every run** (real webhooks sent, real records written).
>
> **How to decide (default to simulating):** look at the tool's name, description, and steps. If it **connects to an external service** (any API, webhook, integration, OAuth/API-key tool, 3rd-party app) **or modifies data** (creates / updates / deletes / sends / writes anything — to an external system or to Relevance knowledge/tables), **simulate it.** Only leave a tool live when it is clearly **read-only / pure-compute** — search, lookups, fetching public info, calculations, formatting — and even then, prefer simulating if you're unsure. When in doubt, simulate.
>
> This does **not** weaken your checks: a simulated tool call is still recorded, so a `tool_usage` check (e.g. "webhook fired ≥1×" / "fired exactly 0×") counts simulated calls exactly like live ones. You get the behaviour assertion **and** safety. Only leave a tool live when the user wants the eval to exercise the real integration.

> **⚠️ Always identify a tool by its `action_id`, never its `chain_id`/`studio_id`.** Every place that references a tool by id — the `tool_configs` keys below **and** the `tool_id` in a `tool_usage` check — must use the tool's **`action_id`** (a 16-char hex like `2a7f1ccf456b7266`). Get it from `relevance_get_agent` (each action in `actions_summary` includes its `action_id`) or `relevance_get_agent_tools` (its `action_id_map`). **A sub-agent is referenced the same way** — by its `action_id` from the node's toolset (see [For Workforces](#for-workforces)).
>
> The `chain_id` / `studio_id` (a UUID like `1c60ddff-…`) that `relevance_get_agent` ("View Agent") returns is a **different id**, and it is NOT recognised at eval runtime. Use it by mistake and the failure is **silent**: a `tool_usage` check never matches (always reports 0 calls), and a tool you marked "simulated" actually **runs live** (real emails sent, real records changed). When in doubt, call `relevance_get_agent_tools` first and key everything by `action_id`.

### What to Simulate (and What NOT to)

**Simulate these tools:**

- API integrations (search, weather, CRM lookup, company enrichment)
- External services (email sending, Slack posting, file uploads)
- Database operations (knowledge table reads/writes)
- Third-party tools (Clearbit, SendGrid, Salesforce)

**Don't create tools just to simulate them.** If you're tempted to create a tool whose only step is `prompt_completion` (an LLM call) so you can simulate its output in evals, that reasoning should live in the agent's system prompt instead.

**Exception:** Tools that combine API calls with LLM processing (e.g., scrape + extract) are legitimate — simulate the API call part, let the LLM processing run naturally.

### Config Levels (Precedence)

Tool simulation config can be set at three levels:

| Level              | Where it's set                                   | Applies to                                                                               | Overridden by                           |
| ------------------ | ------------------------------------------------ | ---------------------------------------------------------------------------------------- | --------------------------------------- |
| **Test set level** | `tool_simulation_config` on the test set         | All scenarios in the test set                                                            | Scenario-level config                   |
| **Scenario level** | `tool_simulation_config` on the test case        | That specific scenario only                                                              | Nothing (wins **per tool** — see below) |
| **Batch level**    | `tool_simulation_config` on the evaluate request | Ad-hoc runs with `scenario_ids` only (**returns 400** if `test_set_id` is also provided) | Scenario-level config                   |

These three levels decide **where** a config is attached. Orthogonally, **within** a workforce config, a per-node entry (`node_configs[node_id]`) always takes precedence over a legacy per-agent entry (`agent_configs[agent_id]`) for that node.

**Levels merge per tool, not wholesale.** A scenario-level config wins only for the tools it names; every other tool still inherits the test-set config.

#### Clearing an override

Pass `null` as the `tool_simulation_config` to `relevance_set_eval_test_case_simulation_config` (test case) or `relevance_update_eval_test_set` (test set) to remove that level's config; the level then inherits from above. `null` is the only clear value — omitting the field keeps the stored config, and `{}` is rejected.

Because levels merge per tool, clearing a test case's config restores the test set's simulation — it does not make tools run live. To pin one tool live while the test set simulates it, give it an explicit entry with `output_overrides_enabled: false` (not `simulate_output: { is_simulated: false }`, which does not disable simulation):

```typescript
{ tool_configs: { '<tool_action_id>': { overrides: { 'default': { output_overrides_enabled: false } } } } }
```

### For Agents

```typescript
{
  tool_configs: {
    '<tool_action_id>': {
      overrides: {
        'default': {
          output_overrides_enabled: true,
          output_overrides: {
            simulate_output: {
              is_simulated: true,
              simulation_prompt: 'Describe what the tool should return...',
              model: 'relevance-cost-optimized',
            },
          },
        },
      },
    },
  },
}
```

### For Workforces

A workforce is a graph of **nodes**, and the same agent can appear in more than one node — so scope simulation **per node** with `node_configs`, keyed by `node_id`. Each entry has the same shape as the agent config above. This is what lets a node's **sub-agents** (and a sub-agent's own tools) be simulated independently of any other occurrence of that agent.

```typescript
{
  node_configs: {
    '<node_id>': {
      // Same shape as the agent config above. Keys are action_ids of the
      // node's tools OR sub-agents (a sub-agent is callable like a tool).
      tool_configs: { '<tool_or_subagent_action_id>': { /* same as agent shape */ } },
    },
  },
}
```

**Discovering the ids** (do this before writing the config):

1. `relevance_get_workforce` → the graph, with a `node_id` for every node.
2. `relevance_get_agent_tools` with the node's `agent_id` + `workforce_context: { workforce_id, node_id }` → that node's **full** toolset: its directly-attached tools, graph-attached tools, **and** the sub-agents it can call. Each entry has an `action_id` and a `kind` of `"tool"`, `"sub_agent"` or `"mcp_tool"`. Use those `action_id`s as the `tool_configs` keys (and as `tool_id` in [tool-usage checks](#check-types)).

> **Why per-node, not per-agent?** If the same agent runs in two nodes, `node_configs` lets each one simulate (or run live) independently. To simulate a **sub-agent's own** downstream tools, configure them under that sub-agent's node — discover them by calling `relevance_get_agent_tools` with that node's id.
>
> `agent_configs[agent_id]` is still accepted as a back-compat fallback (applied to a node only when it has no `node_configs` entry), but prefer `node_configs` for new workforces.

---

## Runbook: Authoring checks (ground → interview)

Follow this whenever you create a check — or rewrite an existing one; a rewrite is re-authoring and goes through the same steps. Checks written one-shot from the resource's config alone are the main source of noisy monitoring later.

**1. Ground before you author.** If the resource has real traffic, read it first: recent tasks/conversations, recent errors, and existing eval or dashboard results. Derive checks from what actually goes wrong. For a brand-new resource with no traffic, compile checks from the agreed success criteria and treat them as provisional until real runs exist.

**2. Interview, briefly.** Ask at most 2–3 questions about unresolved business judgments in the user's use case, skipping anything the conversation has already answered. Ground each question in a concrete situation: a common task's required outcome, an ambiguity or policy boundary where several behaviors are reasonable, or the acceptable fallback when completion fails. Offer 2–4 plausible behaviors as choices — via structured choice prompts if your harness has them, otherwise in free text. Each answer should be specific enough to inform an observable pass/fail criterion; choose the check type only when drafting. Do not ask for facts you can read from the resource's configuration or recent runs. A skipped answer is an answer: proceed, state the assumption you made, and never argue the user back into the interview.

---

## Writing Effective Checks

Checks are evaluated by an LLM judge (for `llm_judge` type) or by deterministic logic (for the other types).

**Good Checks (Specific & Observable):**

```typescript
{ name: "Greeting",          check_config: { type: "llm_judge",        prompt: "The agent greets the user within the first message" } }
{ name: "No hallucination",  check_config: { type: "llm_judge",        prompt: "The agent does not make up information not present in the provided context" } }
{ name: "Mentions pricing",  check_config: { type: "string_contains",  value: "pricing" } }
```

**Bad Checks (Vague):**

```typescript
{ name: "Helpful",  check_config: { type: "llm_judge", prompt: "The agent is helpful" } }  // What makes it helpful?
{ name: "Good response", check_config: { type: "llm_judge", prompt: "The response is good" } }  // No criteria
```

### Check Types

| Type                | `check_config.type` | Key Fields                                                                                                                                                                                                                                                                                                                                                                                               | When to Use                                                                                                                                                              |
| ------------------- | ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **LLM Judge**       | `llm_judge`         | `prompt`, optional `model`, `truncation_strategy`                                                                                                                                                                                                                                                                                                                                                        | General-purpose: subjective quality, tone, completeness, reasoning. Use when criteria can't be reduced to exact string matching or tool call counting. Most common type. |
| **String Contains** | `string_contains`   | `value`                                                                                                                                                                                                                                                                                                                                                                                                  | Verify the agent's **last text** message contains a substring (case-insensitive). Earlier messages are not checked — use `llm_judge` for "mentions X anywhere".          |
| **String Equals**   | `string_equals`     | `value`                                                                                                                                                                                                                                                                                                                                                                                                  | The agent's **last text** message equals `value` exactly (case-insensitive, trimmed). For deterministic outputs only.                                                    |
| **Tool Usage**      | `tool_usage`        | `tool_id` (the **`action_id`** of a tool **or sub-agent** — see the ⚠️ note under Tool Simulation Config), `position` (`anywhere`/`first`/`last`), `operator` (`at_least`/`at_most`/`exactly`), `count`, optional `argument_matchers` (assert **what arguments** the tool was called with — see [Asserting tool arguments](#asserting-tool-arguments-argument_matchers)), and (workforce only) `node_id` | Verify a specific tool — or sub-agent — was (or wasn't) called, optionally with specific arguments.                                                                      |

**Workforce scoping (`node_id`).** For a workforce, a `tool_usage` check optionally takes a `node_id` (from `relevance_get_workforce`). It then counts calls within **that node's own conversation(s)** only — aggregated across the node's runs, deduped per conversation. This is the only way to assert a **sub-agent's own** tool calls: without `node_id`, a workforce check sees only the orchestrator's (main) conversation. Two common shapes:

- **Orchestrator invoked a sub-agent** → `tool_usage` on the orchestrator node (or no `node_id`), with `tool_id` = the **sub-agent's `action_id`** (sub-agents are called like tools, so the call lands in the orchestrator's conversation).
- **A sub-agent used one of its own tools** → `tool_usage` with `node_id` = the **sub-agent's node**, and `tool_id` = that tool's `action_id` (both from `relevance_get_agent_tools` with the node's `agent_id` + `workforce_context: { workforce_id, node_id }`).

### Asserting tool arguments (`argument_matchers`)

A `tool_usage` check can go beyond "was this tool called" and assert **what arguments it was called with**. Add an optional `argument_matchers` array to the `check_config`; each matcher targets one input via `argument_path` and tests its value.

**How matchers combine with `count`:**

- **Across matchers → AND.** An invocation "matches" only when it satisfies **every** matcher in the array.
- **Within one matcher's `expected_values` → OR.** Any one listed value satisfies that matcher.
- The `count` + `operator` you already set on the check now bound **how many invocations matched all the matchers** (instead of all calls). So with matchers:
  - `at_least 1` → "called **at least once with these arguments**".
  - `at_most 0` → "**never** called with these arguments".
  - `count`/`operator` are still required for `position: "anywhere"` (and unused for `first`/`last`).

**`argument_path`** is dot-separated and supports array indexing: `"city"`, `"filters.city"`, `"items[*].name"` (any element), `"items[0].name"`. If the path yields multiple values, the matcher passes when **any** of them matches.

**Match types:**

| `match_type` | Fields                                                                 | Matches when…                                                                                                                                                                                               |
| ------------ | ---------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `exact`      | `expected_values` (string / number / boolean array), `case_sensitive?` | the value **equals** one of `expected_values`. Strings compare **case-insensitively and trimmed by default**; set `case_sensitive: true` to preserve case (still trimmed). Numbers/booleans are unaffected. |
| `contains`   | `expected_values` (string array)                                       | the value, as text, **contains** any one of the substrings. **Always case-insensitive + trimmed.** String arguments only.                                                                                   |
| `not_empty`  | (just `argument_path`)                                                 | the value is **present and non-empty**. Works on any type.                                                                                                                                                  |
| `regex`      | `pattern` (string), `ignore_case?`                                     | the (string) value **matches** the JS regular expression. Set `ignore_case: true` for the `i` flag. String arguments only.                                                                                  |

**Validated when you save** (the check is rejected with a `400` — fix and retry, don't loop):

- `argument_path` must exist on the tool's input schema, and the argument's type must fit the matcher: `regex` and `contains` need a **string** argument; `exact` values must match the argument's primitive type and any `enum`.
- `regex` `pattern` must be a valid expression, ≤ 512 chars, and not risk catastrophic backtracking.
- Caps: ≤ 10 matchers per check, ≤ 100 `expected_values`, ≤ 8 KB per value (and in aggregate).
- On a **workforce**, a check with `argument_matchers` must also set `node_id` — matchers are validated against (and scoped to) that node's sub-agent tool.

**Example — "searched for the right city" (called at least once with `city: "Sydney"`):**

```
relevance_create_eval_check(resource_type: "agent", resource_id: "<agent_id>",
  name: "Searched Sydney",
  check_config: {
    type: "tool_usage", tool_id: "<search_action_id>",
    position: "anywhere", operator: "at_least", count: 1,
    argument_matchers: [
      { match_type: "exact", argument_path: "city", expected_values: ["Sydney"] }
    ]
  })
→ { "check_id": "<check_city>" }
```

**Example — "never emailed a competitor domain" (called zero times with a matching recipient):**

```
relevance_create_eval_check(resource_type: "agent", resource_id: "<agent_id>",
  name: "No competitor emails",
  check_config: {
    type: "tool_usage", tool_id: "<send_email_action_id>",
    position: "anywhere", operator: "at_most", count: 0,
    argument_matchers: [
      { match_type: "regex", argument_path: "to", pattern: "@(acme|globex)\\.com$", ignore_case: true }
    ]
  })
→ { "check_id": "<check_no_competitor>" }
```

**Workforce:** matchers compose with `node_id` — set both to assert a **sub-agent's** call arguments (e.g. "the Summarizer node called `web_search` with `query` containing 'pricing'").

## Common Issues

### "scenario_ids or test_set_id are required"

Provide `test_set_id` to run all scenarios in a test set, or `scenario_ids` with specific test case IDs.

### "Cannot provide tool_simulation_config when using test_set_id"

When using `test_set_id`, config comes from the test set or individual test cases. Remove `tool_simulation_config` from `relevance_run_evaluation`.

### Evaluation takes too long

Reduce `max_turns`, simplify prompts, or run fewer test cases at once.

### Checks are too vague — inconsistent results

Rewrite checks to be specific and observable. Include concrete criteria like "at least two", "within the first message", "mentions by name". A rewrite is re-authoring — follow [Runbook: Authoring checks](#runbook-authoring-checks-ground--interview).

### Performance dashboard returns no data

- Check `enabled: true` on the dashboard.
- Check `sample_rate > 0` — `0` disables evaluation entirely.
- Verify the agent or workforce has live production traffic in the time range.
- Widen the timeseries date range.

### "A performance dashboard must have at least one check"

`relevance_remove_performance_dashboard_rule` refuses to drain the last check. Either add a replacement first via `relevance_add_performance_dashboard_rule`, or delete the dashboard entirely via `relevance_delete_performance_dashboard`.
