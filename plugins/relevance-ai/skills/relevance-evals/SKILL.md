---
name: relevance-evals
description: Manages evaluations and monitoring for agents and workforces (multi-agent systems) in Relevance AI — creating test cases, defining checks, configuring tool simulations, running evaluations, analysing results, setting up performance dashboards that monitor production traffic, and configuring alarms that notify when quality degrades. Use when testing agent or workforce behaviour, setting up automated testing, reviewing eval results, monitoring production agents for failures, or setting up alarms and notification channels for regressions.
---

# Evals & Monitoring Skill

Skill for managing pre-deploy testing and post-deploy monitoring of agents and workforces in Relevance AI.

## Overview

Evals work as two complementary surfaces:

- **Test sets + scenarios + checks** — pre-deploy testing. Scripted scenarios you replay against an agent or workforce, scored by checks. Think "integration tests for agents."
- **Performance dashboards** — post-deploy monitoring. Live production traffic is sampled and scored by checks. Think "Datadog / CloudWatch for agents."

## When to use what

| You want to…                                                                       | Use…                                                                              |
| ---------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| Test agent behaviour before publishing (regression suite, edge cases, "what if…")  | **Test sets + scenarios + checks** — pre-deploy unit/integration tests            |
| Monitor live production runs continuously, catch regressions in real conversations | **Performance dashboards** — sampling-based observability                         |
| Investigate "what's failing in production right now?"                              | List performance dashboards → list dashboard runs (failed only) → group reasons   |
| Check that a new or edited check agrees with real past tasks before trusting it    | **Historical scoring** — `relevance_run_evaluation` with `task_ids` + `check_ids` |

## Topics — load on demand

This file covers what nearly every evals task needs: the concepts, the core tools and the pre-deploy workflow. Everything else is in a topic of this skill: load it by key (`relevance-evals/<topic>`; the file is `<topic>.md` next to this one) when the task reaches it, not up front. When a prompt or another skill cites one of this skill's sections by name, load the topic that lists it here.

- **`relevance-evals/design`** — before proposing tests or monitoring, choosing between a test set and a performance dashboard, or showing the user a check: "Choosing the surface", "Eval Design Philosophy", "Recommending checks to the person who owns the agent".
- **`relevance-evals/authoring`** — before writing or changing any scenario or check, and whenever the user asks to calibrate or tune one: the "Grounding in real runs", "Authoring checks" and "Calibrating a check against historical tasks" runbooks, historical scoring, "Proposing Test Cases to Users".
- **`relevance-evals/checks`** — when drafting a `check_config`: "Writing Effective Checks", "Grading against examples", "Check Types", "Asserting tool arguments" (`argument_matchers`), "Custom tool checks" (`custom_tool` evaluators and their inputs), "Custom agent checks".
- **`relevance-evals/evaluator-agents`** — before creating, picking or previewing an evaluator agent for a `custom_agent` check: when it beats `llm_judge` / `custom_tool`, the create → attach tools → preview → publish → attach runbook, how its verdicts and errors read.
- **`relevance-evals/simulation`** — before running scenarios on a resource with tools, and after its tool surface changes: "Tool Simulation Config" (what to simulate, the config shape, levels, clearing, keeping coverage).
- **`relevance-evals/workforces`** — whenever `resource_type` is `workforce`: node-scoped simulation and checks, asserting sub-agent calls, a worked example.
- **`relevance-evals/monitoring`** — performance dashboards on live traffic: how they work, "Post-Deploy Workflow", the dashboard tools and URLs, "Investigating recent failures", alarms, dashboards that return no data.
- **`relevance-evals/pre-publish`** — "Pre-publish checks": gating publishes on test sets ("Test and Publish"), polling a gated publish, and whenever the user asks how to keep bad changes from going live.
- **`relevance-evals/verifying-a-fix`** — the "Verifying a fix" runbook: after editing an agent or workforce to fix a diagnosed failure, before reporting the fix as done.

## Concepts

- **Test Sets** — Folders that group related test cases.
- **Test Cases (Scenarios)** — Simulated user interactions with expected outcomes. Every test case belongs to a test set.
- **Checks** — Reusable evaluation criteria (`name` + `check_config`). A check is a standalone object on the agent or workforce that scenarios and performance dashboards reference by `check_id`; see [Checks are a reusable library](#checks-are-a-reusable-library-reference-them-by-id). (Internally — in API responses and result records — the backend still calls these "rules", so eval-run results expose fields like `rule_results` and `rule_name`. They are the same thing.)
- **Performance Dashboards** — Production monitoring configs that bind an agent or workforce to a set of checks, a `sample_rate` (the chance a task is scored each time it settles into a status) and an optional `task_status` filter.
- **Eval Batches** — A grouped execution of evaluations against an agent or workforce, identified by `eval_batch_id`.
- **Eval Runs** — Each scenario/check execution inside a batch, identified by `eval_run_id`.
- **Pre-publish checks** — An optional publish gate: test sets that every publish runs against the draft, publishing only when they pass. See `relevance-evals/pre-publish`.

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
4. Poll for completion with `relevance_poll_eval_batch_result` (long-polls for ~50s per call). When `is_terminal` is true the batch is done; otherwise re-call with the same `eval_batch_id`, or hand the run back as [Step 4](#step-4-poll--read-results) says.
5. Results show pass/fail per check with the judge's reasoning.

## Eval URLs

`relevance_run_evaluation` returns a `url` field pointing directly to the eval run page. **Always share this URL with the user immediately after starting an evaluation.**

## Agents vs Workforces

Evals and dashboards work for both individual agents and workforces (multi-agent systems). The tools are the same — only `resource_type` and a few fields differ. For a workforce, load `relevance-evals/workforces` — its simulation and tool-usage checks are scoped per graph node, not per agent.

> **Not tools.** Evals and performance dashboards accept only `resource_type: agent | workforce` — you **cannot** attach an eval set or a performance dashboard to an individual tool. To monitor a tool, evaluate the agent or workforce that calls it. A tool's own health surfaces through **usage analytics** (execution counts, error rates, credits — see the `relevance-analytics` skill) and its run history, not through evals or live scoring.

## Tool ids

> **⚠️ Always identify a tool by its `action_id`, never its `chain_id`/`studio_id`.** Every place that references a tool by id — the `tool_configs` keys of a simulation config **and** the `tool_id` in a `tool_usage` check — must use the tool's **`action_id`** (a 16-char hex like `2a7f1ccf456b7266`). Get it from `relevance_get_agent` (each action in `actions_summary` includes its `action_id`) or `relevance_get_agent_tools` (its `action_id_map`). **A sub-agent is referenced the same way** — by its `action_id` from the node's toolset (see [workforces.md](workforces.md#tool-simulation-per-node)). A workforce **tool node** that runs as a next step has its tool's `action_id` in `tool_node_action_ids` from `relevance_get_workforce` (see [workforces.md](workforces.md#tool-nodes)).
>
> The `chain_id` / `studio_id` (a UUID like `1c60ddff-…`) that `relevance_get_agent` ("View Agent") returns is a **different id**, and it is NOT recognised at eval runtime. Use it by mistake and the failure is **silent**: a `tool_usage` check never matches (always reports 0 calls), and a tool you marked "simulated" actually **runs live** (real emails sent, real records changed). When in doubt, call `relevance_get_agent_tools` first and key everything by `action_id`.

---

## Dedicated Tools

Every tool requires `resource_type` (`agent` or `workforce`) and `resource_id`.

### Eval Check Tools (the reusable check library)

The checks that scenarios and dashboards reference — see [Checks are a reusable library](#checks-are-a-reusable-library-reference-them-by-id).

- `relevance_list_eval_checks` — List every check on the agent or workforce, with `check_id`, `name`, `check_config`, and `attached_to`
- `relevance_create_eval_check` — Create a check (`name` + `check_config`); returns its `check_id`
- `relevance_update_eval_check` — Update a check by `check_id`. **Affects every scenario/dashboard using it**
- `relevance_delete_eval_check` — Delete a check by `check_id`. Removes it from the library and detaches it everywhere

### Test Set Tools

Test sets are collections (like folders) that group related test cases. Every test case must belong to a test set — create a test set first, then add test cases to it.

- `relevance_create_eval_test_set` — Create a test set to group test cases
- `relevance_get_eval_test_set` — Get a specific test set by ID
- `relevance_list_eval_test_sets` — List all test sets for an agent or workforce
- `relevance_update_eval_test_set` — Update a test set (**patch semantics** — only send changed fields)
- `relevance_delete_eval_test_set` — Delete a test set (optionally with its test cases)

### Test Case (Scenario) Tools

Test cases define individual evaluation scenarios. Each test case has a prompt, a set of attached checks (by `check_id`), and optionally a tool simulation config.

- `relevance_create_eval_test_case` — Create a test case with scenario, `check_ids` (existing checks), and optional simulation config
- `relevance_get_eval_test_case` — Get a specific test case by ID
- `relevance_list_eval_test_cases` — List test cases within a test set
- `relevance_update_eval_test_case` — Update a test case's metadata (name, scenario) — **does not modify checks**
- `relevance_delete_eval_test_case` — Delete a test case

### Test Case Check Management (attach / detach existing checks)

Attach/detach checks on a test case (**`relevance_update_eval_test_case` can't change checks** — use these). Creating and editing the checks themselves lives in [the check library](#checks-are-a-reusable-library-reference-them-by-id).

- `relevance_add_eval_test_case_rule` — Attach one or more existing checks to a test case by `check_id` (`check_ids`)
- `relevance_remove_eval_test_case_rule` — Detach a check from a test case by `check_id` — unlinks here only; the check stays in the library

To see existing checks on a test case (including their `check_id`s), use `relevance_get_eval_test_case`; to see the whole agent or workforce's check library, use `relevance_list_eval_checks`.

### Test Case Simulation Config

- `relevance_set_eval_test_case_simulation_config` — Set or clear tool simulation config on a test case (full replacement, not patch)

### Evaluation Execution

- `relevance_run_evaluation` — Run an evaluation. Scenario mode: `test_set_id` or `scenario_ids`. Historical mode: `task_ids` + `check_ids` scores tasks that already ran. Defaults to the active version; pass `version_id` to pin one. For unpublished edits pass the resource's `draft_version_id`; a draft needs no publish.

### Batch & Run Tools

- `relevance_list_eval_batches` — List eval batches for an agent or workforce
- `relevance_get_eval_batch_summary` — Get batch summary with scores and check results (single-shot)
- `relevance_poll_eval_batch_result` — Long-poll a batch until it's terminal or the wait window elapses. Returns the summary plus `is_terminal`
- `relevance_list_eval_runs` — List eval runs inside a batch
- `relevance_cancel_eval_batch` — Cancel pending/running eval runs in a batch
- `relevance_delete_eval_batch` — Delete a batch and all its eval runs

The performance dashboard tools are in `relevance-evals/monitoring`, and the pre-publish (publish gate) tools in `relevance-evals/pre-publish`.

### Patch Semantics (Important)

The update tools (`relevance_update_eval_test_case`, `relevance_update_eval_test_set`, `relevance_update_performance_dashboard`) use **patch semantics**: omitted fields are preserved. Checks are managed separately — attach/detach with the per-surface tools, edit the check itself with `relevance_update_eval_check`.

---

## Pre-Deploy Workflow (Test Sets + Scenarios + Checks)

**IMPORTANT:** Follow this workflow in order. Each step depends on the previous one. Before Step 1, work through [Runbook: Authoring checks](authoring.md#runbook-authoring-checks-ground--ask--confirm--calibrate) up to saving — create nothing, not even the test set, before then. The exceptions are the scenarios [Earning checks from real runs](design.md#earning-checks-from-real-runs) saves without that confirmation: the one that demonstrates a newly built agent or workforce on the owner's own example when something it does writes, the one per stated rule and outside write that a high-stakes build adds, and the ones the owner said yes to after seeing the demonstration.

1. **Create a Test Set** — required; test cases must belong to a test set.
2. **Create Test Cases (Scenarios)** — first get or create the checks (`relevance_list_eval_checks` → `relevance_create_eval_check`), then create test cases that reference them by `check_ids`.
3. **Run Evaluation** — only when the user asked for a run or a result, or a new build or a fix you are verifying calls for one (see Step 3); trigger with `test_set_id` or `scenario_ids`.
4. **Poll & Read Results** — check batch status, get detailed eval run results.

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

Get the `check_id`s first (reuse or create — see [Checks are a reusable library](#checks-are-a-reusable-library-reference-them-by-id)). If the agent or workforce has triggers or real runs, follow [Runbook: Grounding in real runs](authoring.md#runbook-grounding-in-real-runs) before writing scenarios or checks. Before creating new checks, follow [Runbook: Authoring checks](authoring.md#runbook-authoring-checks-ground--ask--confirm--calibrate). Then call `relevance_create_eval_test_case` with those `check_ids` and the (required) `test_set_id`.

> **🛑 Before running, simulate side-effecting tools.** Default rule: **if a tool connects to an external service or modifies data, simulate it** (only leave read-only / pure-compute tools live). Set `tool_simulation_config` on the test set — **default ON, unless the user asks for a live run** — so the eval doesn't fire real actions (real webhooks, emails, writes). Simulated calls still count for `tool_usage` checks. See [simulation.md](simulation.md) (load it before the first run).
>
> Some scenarios get this filled in for you at creation time — the create call's `simulated_tools` field is the only confirmation. If it is absent, or `tools_running_live` names anything, set the config yourself.

#### Scenario Configuration

**`prompt`** (required, even with `first_message`) — Describes the persona and situation for the simulated user. The LLM generates realistic messages based on this. When nobody replies (`max_turns: 1`), one line naming the source is enough; where a person can reply, `prompt` is still their persona after the first message.

**`max_turns`** (default: 10, range: 1–50) — Controls how many back-and-forth exchanges happen before evaluation.

**`first_message`** — Sent to the agent verbatim as the opening message, bypassing the simulated user (max 10,000 characters). Use it when the user asks to pin the opening line ("start by saying X") instead of writing that into `prompt`, and for a trigger-fed resource to replay what the trigger sends — see [Runbook: Grounding in real runs](authoring.md#runbook-grounding-in-real-runs).

```
scenario: {
  prompt: "You are an impatient customer.",
  first_message: "I need help with my order #12345"
}
```

**`runs_per_scenario`** (default: 1, range: 1–10) — Run the same scenario multiple times to test consistency. AI responses vary — 3–5 catches flaky behavior.

To view test cases in a test set:

```
relevance_list_eval_test_cases(resource_type, resource_id, test_set_id: "<test_set_id>")
```

### Step 3: Run Evaluation

Saving a scenario is not a request to run it: a run replays the whole agent or workforce, which can take many minutes and costs credits. Unless the user asked for a run or a result, or a new build ([Earning checks from real runs](design.md#earning-checks-from-real-runs)) or a fix you are verifying ([Verifying a fix](verifying-a-fix.md)) calls for one, stop after Step 2: say in their terms what each scenario tests, not its field or tool names, link the agent or workforce, and offer to run them now, with roughly how long a run takes (judged from recent runs: for a workforce, `created_at` to `updated_at` across several recent tasks in `relevance_list_workforce_tasks`; for an agent, an earlier run's `active_execution_time_ms` from `relevance_get_agent_task_metadata`) and that it costs credits, on a question card if your harness has them.

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

> **Version pinning:** Omit `version_id` to evaluate the active (published) version — it's resolved automatically. Pass `version_id` to evaluate a specific past/draft version; to evaluate the **current draft**, skip the version list and pass the `draft_version_id` from the resource's get-tool summary. A draft needs no publish to be evaluated. An evaluator a check runs is the exception: a `custom_agent` evaluator runs only once published ([evaluator agents](evaluator-agents.md)), and a `custom_tool` evaluator runs its published version once it has one ([custom tool checks](checks.md#custom-tool-checks)), so publish edits to an already-published evaluator first. For **workforces** this is **topology-only**: it pins the graph wiring of that version, but nested agents still run their published version (their draft only if never published), so an edit to a nested agent is invisible to a workforce eval until that agent is published — verify it with the agent's own eval pinned to its draft (full historical reproduction isn't supported).

### Step 4: Poll & Read Results

Evaluations take from 30 seconds to many minutes. A scenario run the user asked for on an existing agent or workforce goes in the background when it is a workforce, or an agent whose recent runs took more than about a minute, judged as in Step 3 (a multi-turn scenario takes longer than one run): reply straight away with its link and when it should finish, and offer to check back then if your harness can schedule one; if it can't, keep checking as the block below says rather than handing back only a link. Every other run, including a new build's demonstration or tests, a fix you are verifying and scoring past tasks with `task_ids`, is checked in the turn: keep checking without asking the user while it should finish soon, and hand a long one back:

> Work you start can outlast your turn, and every check on it is a tool call, so a turn that keeps waiting can end with no reply at all. Check on a run first: one check waits about its wait_seconds, and most runs finish within it. If it is still running, estimate when it will finish from its progress (an eval batch's completed_runs of total_runs) or from how long the same work took recently. If that is more than about 10 minutes away, or you have already waited about 10 minutes, stop checking: tell the user what is still running, give its link and when you expect it to finish, and offer to check back then if you can schedule one. If they would rather you keep waiting, keep waiting, but reply with what you have before your turn can run out.

#### Polling loop

1. Call `relevance_poll_eval_batch_result` with the `eval_batch_id`. It waits up to `wait_seconds` (default 50) for the batch to finish, so one call covers many checks.
2. While `is_terminal` is false, note progress from `completed_runs` of `total_runs` (e.g. "Still running — 2/5 complete") and call again, within the limit above.
3. Once it is terminal (`completed`, `failed` or `cancelled`), read the results. `relevance_get_eval_batch_summary` returns the same counts without waiting.

The summary response includes: `{ status, tasks_evaluated, total_runs, completed_runs, failed_runs, cancelled_runs, summary_score, name, rules }`. `summary_score` is on a **0-1 scale** (1 = all checks passed).

#### Reading results

1. Call `relevance_list_eval_runs` with the `eval_batch_id` to enumerate eval runs.
2. Each completed eval run has `result.rule_results` (this is the actual field name; each entry is a check result) with `rule_name`, `passed`, and `reason` for each check.
3. Present results in a clear table: scenario name, pass/fail per check, and the LLM judge's reasoning for any failures.

---

## Common Issues

### "scenario_ids or test_set_id are required"

Provide `test_set_id` to run all scenarios in a test set, or `scenario_ids` with specific test case IDs.

### "Cannot provide tool_simulation_config when using test_set_id"

When using `test_set_id`, config comes from the test set or individual test cases. Remove `tool_simulation_config` from `relevance_run_evaluation`.

### Evaluation takes too long

Reduce `max_turns`, simplify prompts, or run fewer test cases at once.
