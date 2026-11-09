---
name: relevance-evals/pre-publish
description: The publish gate — test sets with thresholds that every publish runs against the draft before it goes live.
---

# Pre-Publish Checks

## Pre-publish checks (publish gate / "Test and Publish")

An agent or workforce can have **one pre-publish check config**: 1–10 test sets, each with a `threshold_score` (minimum pass rate, 0–100) and a `block_on_failure` flag. When a config exists, `relevance_publish_agent` / `relevance_publish_workforce` run every configured test set against the current draft instead of publishing directly, and **automatically publish the draft to live when every blocking test set meets its threshold** — there is no separate publish step afterwards. A below-threshold non-blocking test set is reported but never blocks. This is the native "block bad changes from going live" mechanism.

Workflow:

1. `relevance_get_pre_publish_status` — read the current config (if any) and recent check runs.
2. If no config exists, build test sets + scenarios first (see the Pre-Deploy Workflow in [SKILL.md](SKILL.md#pre-deploy-workflow-test-sets--scenarios--checks)), then `relevance_upsert_pre_publish_config` with the complete test-set list.
3. Publish as normal (`relevance_publish_agent` / `relevance_publish_workforce`) — the gate runs automatically and the response includes a `check_id`. Pass a concise `version_name` and `version_description` so a successful publish leaves a useful version-history trail. `skip_checks: true` publishes without running the gate. Pass it when the user asks to skip the checks, or when you already ran every configured test set against this same draft, every blocking one met its threshold, and neither the draft nor the test sets (their scenarios, checks and tool simulation) have changed since — re-running them would bill the user twice for scoring identical bytes. If anything changed after that run, let the checks run.
4. `relevance_poll_pre_publish_check` with the returned `check_id` (long-polls ~50s per call; re-call until `is_terminal` is true).
5. Report the `outcome` and per-test-set scores, and state clearly whether the draft was published.

Work you start can outlast your turn, and every check on it is a tool call, so a turn that keeps waiting can end with no reply at all. Check on a run first: one check waits about its wait_seconds, and most runs finish within it. If it is still running, estimate when it will finish from its progress (an eval batch's completed_runs of total_runs) or from how long the same work took recently. If that is more than about 10 minutes away, or you have already waited about 10 minutes, stop checking: tell the user what is still running, give its link and when you expect it to finish, and offer to check back then if you can schedule one. If they would rather you keep waiting, keep waiting, but reply with what you have before your turn can run out.

Failure modes:

- **"already running" (409)**: a check is already in flight for this resource — don't re-publish; find its `check_id` with `relevance_get_pre_publish_status` and poll it.
- **"draft version changed" (409)**: the draft was saved again between that save and the trigger — re-publish from the latest draft.

**Surface this proactively** whenever a user asks how to gate publishes on tests (e.g. "how do I block bad agent changes from going live?"). The answer is: turn the resource's test sets into pre-publish checks with `relevance_upsert_pre_publish_config` — every publish then enforces the gate automatically.

## Proposing a publish check

A publish check costs the owner time on every publish: a check typically takes about 3 minutes, sometimes 10. Propose one in plain words with that cost, and set it up only once they agree — never configure one they haven't heard about. For example:

> "Because it emails customers on its own, I'd re-run these four examples before every change you publish. Each publish would take about 3 minutes longer, occasionally 10, and you'd hear straight away if a change broke the refund rule. Say 'no checks' if you'd rather not."

Set it up non-blocking (`block_on_failure: false`) unless they ask for blocking; for a blocking check, add that the change won't go live until it's fixed.

- **Recommend it** for a high-stakes build ([Earning checks from real runs](design.md#earning-checks-from-real-runs), rule 3) that will keep being edited.
- **Don't recommend it** when a person reviews every output or the resource rarely changes; live scoring catches drift better ([Choosing the surface](design.md#choosing-the-surface)).
- If they decline, record it and don't offer again.

## Pre-Publish Check Tools (publish gate)

See [Pre-publish checks](#pre-publish-checks-publish-gate--test-and-publish) for the workflow and failure modes. The gate itself runs through the plain publish tools (`relevance_publish_agent` / `relevance_publish_workforce`), which trigger the checks automatically when a config exists and return a `check_id`.

| Tool                                  | Description                                                                                                                                             |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `relevance_get_pre_publish_status`    | The publish-gate config (or `config: null`) plus recent check runs with per-test-set scores and publish outcomes                                        |
| `relevance_upsert_pre_publish_config` | Create or replace the gate: 1–10 test sets, each with `threshold_score` + `block_on_failure`. **Full replace**                                          |
| `relevance_delete_pre_publish_config` | Remove the gate — publishes are no longer checked afterwards                                                                                            |
| `relevance_poll_pre_publish_check`    | Long-poll a check by `check_id` (returned by a gated publish) until `is_terminal`; terminal responses include an `outcome` stating whether it published |
