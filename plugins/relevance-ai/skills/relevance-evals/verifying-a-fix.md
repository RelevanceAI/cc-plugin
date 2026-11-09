---
name: relevance-evals/verifying-a-fix
description: Verify a fix to an agent or workforce with an eval run pinned to the edited draft.
---

# Verifying a Fix

## Runbook: Verifying a fix

After editing an agent or workforce to fix a diagnosed failure, verify the draft before reporting the fix as done:

1. **Capture the draft id.** Re-fetch the resource; its summary carries `draft_version_id` — the version your edit lives on. An eval run without it evaluates the live, unfixed version.
2. **Find real coverage.** List test sets and their scenarios. Don't branch on the `quality.has_evals` flag alone — it is also true when only orphan checks or empty test sets exist.
3. **Pick the smallest run.** Prefer the specific `scenario_ids` that exercise the diagnosed failure over a whole test set — evals cost credits.
4. **No coverage? Create it, then run.** Order matters: check (`relevance_create_eval_check`) → test set → scenario (`relevance_create_eval_test_case` requires `test_set_id` + `check_ids`). Base the scenario on the failing task's input, with secrets and personal data swapped for realistic fakes — for a trigger-fed resource, replayed as its `first_message`. Calibrate a new check against finished runs first ([Authoring checks](authoring.md#runbook-authoring-checks-ground--ask--confirm--calibrate) step 4), the failing task among them.
5. **Confirm side-effecting tools are simulated** (the default in [simulation.md](simulation.md)) before running.
6. **Run pinned to the draft** — `relevance_run_evaluation(..., scenario_ids, version_id: <draft_version_id>)` — share the returned URL, and poll to terminal. Work you start can outlast your turn, and every check on it is a tool call, so a turn that keeps waiting can end with no reply at all. Check on a run first: one check waits about its wait_seconds, and most runs finish within it. If it is still running, estimate when it will finish from its progress (an eval batch's completed_runs of total_runs) or from how long the same work took recently. If that is more than about 10 minutes away, or you have already waited about 10 minutes, stop checking: tell the user what is still running, give its link and when you expect it to finish, and offer to check back then if you can schedule one. If they would rather you keep waiting, keep waiting, but reply with what you have before your turn can run out.
7. **Classify the outcome — three ways, not two.**
   - Checks failed → the fix is **not confirmed**: report failures-first, make at most one more fix and rerun, then report what still fails and ask before going further.
   - Run errored (credits, timeout, judge/platform failure) → verification is **inconclusive** — never report the fix as verified.
   - All relevant checks passed → confirm the current `draft_version_id` still matches the one you evaluated (an edit mid-run invalidates the result), then report verified.

Caveats: workforce `version_id` pins topology only — nested agents run their published version, so verify a nested-agent fix through that agent's own evals. Tools have no evals; verify a tool fix with a draft tool run along the failing path when that path only reads. When it writes, run the steps before the write with `relevance_run_transformation`. Never run something that writes just to try it, and that includes checking a fix: trigger the agent or workforce with `simulate: true`, naming in `live_action_ids` only tools that read, so every other tool returns a stand-in instead of running; or exercise the write in an eval that stands in for it; or run it only once the user has agreed to that specific run and named where it may write, such as a test recipient. After any run, a finished tool call whose `output_source` is missing or `tool-run` ran for real: say which of those sent or changed something, and where. A stand-in shows what it would send, not that it arrives, so call that write unproven.
