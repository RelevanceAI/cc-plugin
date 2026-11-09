---
name: relevance-evals/design
description: Choosing between a test set and a live dashboard, sampling, suite design, and explaining checks to the owner.
---

# Designing Evals and Monitoring

## Choosing the surface

Most resources want both, doing different jobs. Say which one you're proposing and why in one line before you build either, and explain the difference the way the user would.

**The framing that lands:** an agent is a very capable **new teammate** — the capability is there, but the judgement that comes from experience isn't yet. When it gets something wrong, you **teach** it: fix its instructions, and keep a test case so a later change can't quietly undo the fix.

- **Offline test sets are the first couple of weeks**, when you check everything a new teammate does. Every mistake anyone catches becomes a test case here, so the set grows out of real mistakes rather than imagination.
- **Live monitoring is after they're onboarded.** You stop re-reading everything and check the handful of things that matter most, and whether the process was followed. Not every bug needs a live check — most belong in the offline set.

|         | Test set (offline)                                          | Performance dashboard (live)                                  |
| ------- | ----------------------------------------------------------- | ------------------------------------------------------------- |
| The job | Prove it does the job before launch, and after every change | Catch regressions and drift in real runs                      |
| Size    | Larger — the criteria that matter, plus the ways they break | A **subset of the same checks** — what you must be told about |
| Model   | The strongest that fits the judgement                       | A cheaper one where it still agrees with you                  |
| Cost    | Higher, but bounded and one-off                             | Scales with volume, so sampling bounds it                     |

**Criticality sets how much to cover.** Ask what happens the day this check fails. If someone should be woken up — money moves, a customer sees it, it can't be undone — score essentially every run, and set up an alarm on it when asked. If it is something the user would only review periodically, 10–20% is enough to see a trend.

**Volume sets the number.** Sampling exists to bound cost when there is a lot of traffic; a resource that does not run much has nothing to bound, so score all of it. A once-a-weekday agent sampled at 1% is scored about twice a year, which is not monitoring. This is the same reason alarms need a meaningful `min_sample_size` on low-traffic resources and sit in `insufficient_data` without one. When the resource has real history, derive the rate from it rather than from a rule of thumb.

**Cost, accurately.** Checks that share a model are graded together in one judge call and share that cost, so the number of checks you attach matters far less than the number of runs you score — the sample rate is the lever, not the check count. Keep one criterion per check: that is what makes a failing trend readable, and an alarm can only watch one check or the average across all of them, so a check bundling five criteria cannot get an alarm per criterion.

**Keep the live set a subset of the offline set**, and promote anything that actually breaks in production back into the offline set.

**An unattended resource wants the dashboard first.** A scheduled agent, a webhook consumer or anything else with no one reading its output has no human to notice a bad run, and — if it rarely changes — little for a publish gate to catch. The reverse holds for something a person drives and edits often. Be precise about what a dashboard does when you offer one: it scores runs and records the result, and **an alarm is the separate part that tells someone**. Until one exists, say the user would have to go and look.

## Earning checks from real runs

A new agent or workforce is demonstrated once on the owner's own example: a direct run of the draft when it only reads and replies, otherwise one scenario with a stand-in for every step that writes, run against the draft. A step writes when it sends or posts anything, or adds, changes or deletes data anywhere, a knowledge table included, since it may hold live data. Judge a step by what it does, not its name: a find-or-create step writes, and so can a search sent as a POST. A step you can't classify counts as one that writes. Either way, show its real output by quoting the first few lines of its reply rather than describing it. Always say, in their terms, which 2–3 situations that run didn't exercise; a stand-in shows what it would send, not that it arrives.

A run that found nothing proves only the empty path. Read what its reads returned. If they came back empty where the owner's data should hold items, read the same source once more broadly, fix the filter and rerun. If items were read but none qualified, say so with both counts. If their data has no qualifying case today, say so and run it once on a labelled stand-in (under an instruction not to build tests, ask first). Never loosen what they asked for to get a match, and never call an empty run working.

Nobody can say what to test on an agent they haven't seen run, so don't ask whether they want tests before the demonstration. Once the owner has seen it run, one concrete proposal to turn what it showed, plus the situations it didn't exercise, into tests is not that question. It is optional, so it never comes ahead of an answer to something the owner asked, and it isn't presented as the recommended choice. Their yes is the confirmation, so don't ask for another, and the tests are run once and repaired at most once ([Runbook: Requested setup on a new build](authoring.md#runbook-requested-setup-on-a-new-build)). The rest is earned from real runs: after about 5–10 of them, go through a few with the owner and turn their verdicts into checks ([Runbook: First checks from real runs](authoring.md#runbook-first-checks-from-real-runs)). Say that as what will happen, not as a question; the owner can decline it. It never replaces the proposal above.

Decide what else it gets before launch in this order:

1. **A standing instruction not to build tests** overrides rules 3 and 4, the proposal after the demonstration and the later review, but not something the owner asks for in the conversation. Save nothing beyond the demonstration and propose no checks. If rule 3 would have applied, say once, as part of the hand-back rather than a question of its own, what a mistake would cost and that you'll test those rules if they say so. Schedule no later review.
2. **What the owner asked for, and only that**, in their own words or as a line they added to the plan: tests are scenarios, monitoring is a dashboard, a publish check is a publish check ([Runbook: Requested setup on a new build](authoring.md#runbook-requested-setup-on-a-new-build)).
3. **High-stakes**, judged by what a mistake would cost: a tool that writes outside the platform (email, CRM, payments, anything a customer sees) runs with no approval step, money moves, it can't be undone, or the user stated a rule that protects customers, money or compliance. Add one scenario per such rule and per outside write, and propose a publish check ([Proposing a publish check](pre-publish.md#proposing-a-publish-check)). When unsure, treat it as high-stakes. A style rule ("never use bullet points") is a check on the demonstration, not a reason for this. These scenarios are saved without the authoring runbook's confirmation step.
4. **Unattended** (a trigger or schedule feeds it and nobody reads each output): propose a dashboard beside the publishing decision and never create one unasked: checks that hold for any run (never one naming the example's own values), the sample rate you'd set for its risk and volume, that every scored run costs credits, and that it records scores but tells nobody without an alarm. The yes to publish accepts it; "no monitoring" declines it. Create it enabled after that yes, before the publish.

---

## Eval Design Philosophy

### The Purpose of Evals

When an eval runs:

1. The system triggers the agent or workforce with the test scenario prompt
2. The agent generates a real conversation, or the workforce executes a real task
3. An LLM judge evaluates the output against the defined checks
4. Results show whether the criteria were met

### Designing a Good Test Suite

| Principle              | Description                                                                       |
| ---------------------- | --------------------------------------------------------------------------------- |
| **Focused Coverage**   | Cover what the user cares about first, not every capability; say what's left out  |
| **Minimal Test Cases** | Keep the number of test cases small — each eval is expensive and time-consuming   |
| **Targeted Scenarios** | Each test case should focus on testing specific behaviors, not everything at once |
| **Realistic Prompts**  | Use prompts that mirror how real users would interact with the agent or workforce |
| **Specific Checks**    | Each test case has its own checks tailored to what that scenario should achieve   |

**DO:**

- Test one primary capability per test case
- Use realistic user language and requests
- Include edge cases where what the user cares about is most likely to break
- Write checks that are specific and measurable

**DON'T:**

- Add a test for every tool or capability just because it exists
- Create dozens of overlapping test cases
- Use artificial or overly formal prompts
- Write vague checks like "agent is helpful"

### Example: Flight Search Agent Test Suite (3 tests)

The user said wrong airports, dates or passenger counts hurt most.

| Test Case                | Use Case Covered                   | Key Checks                                     |
| ------------------------ | ---------------------------------- | ---------------------------------------------- |
| Basic One-Way Flight     | Core search functionality          | Uses tool correctly, correct airports and date |
| Round-Trip International | Return dates, international routes | Handles both legs, correct airports            |
| Family Travel            | Multi-passenger handling           | Correct passenger count, total pricing         |

Left out: travel class and budget preferences, which the user rated lower risk.

### Cost-Benefit Tradeoff

Each eval costs credits (agent execution + LLM judge). On a **high-traffic** production dashboard, sampling is what keeps that bounded — 10–20% is usually enough for anything that isn't critical. Coverage itself is a criticality decision, and low-traffic resources need more of it, not less: see [Choosing the surface](#choosing-the-surface).

### Recommending checks to the person who owns the agent

The people who own these resources know their business completely and this product not at all. A check they cannot restate in their own words is a check they cannot approve, tune, or trust when it fails at 6am.

**Lead with what it protects.** A check is a success criterion made executable, so name the consequence first and the mechanism second — "nobody gets sent a signal that was never in the data; it compares the numbers it reports against what the tool actually returned", not "an LLM-judged check on the final message". Never open with a field name, a check type, or a tool name: `uat_mode`, "a tool-usage check" and `relevance_run_evaluation` are what you read and call, not what you say. The same rule the diagnostics skill applies to errors applies here.

**Every check has to be able to tell someone something.** The test is not whether the expected value is constant — plenty of the best checks assert a constant. It is whether a person named that value as a rule, and would want to hear about it changing:

- **Worth checking:** a boundary the user stated — never email a customer without approval, only ever write to the sandbox account, never spend above this limit. The value is fixed, and that is exactly the point: a later edit can silently break it and nothing else would catch that.
- **Not worth checking:** configuration nobody called a rule, restated back. "Every call passes `uat_mode: true`" when `uat_mode` is a fixed tool setting the resource never chooses tests the configuration, not the behaviour.
- **Actively harmful:** a value that is _scheduled_ to change. If UAT mode is on "until we switch it off", a check asserting it passes until that day and then fails on every run forever.

**A quiet run is not a failed run — but it must say so.** A run that finishes having done nothing looks identical to a broken one in every status field, so the agent has to tell them apart: it should report the quiet day in those words rather than reporting success in the language it uses when it did work, or going silent. Don't write a check that fails whenever the output is empty — a chaser with no overdue invoices, or a sync with no changes, is a good run. Write the check against what the agent said about it. (Agent instructions should already carry the matching exit condition — see the agents skill's trigger guidance.)

**A declined approval is not a failed run.** When a write waits for a person's approval, the yes or no is theirs, not the agent's. Judge what the agent controls: it drafted what was asked, asked before writing, and after a no (the tool result reads "Action rejected by user.") it did not write, retry or work around it, and said it stopped. Never fail a run because the write didn't happen after a no, and leave a run still waiting on a decision out of calibration. To test both answers, set a scenario's `approval_handling` to `approve_all` or `reject_all`; the default, `auto`, lets the simulated user decide, so a verdict can change from run to run.

**What the criterion's shape tells you about the check to write:**

| The user says…                                    | Write…                                                                                                                                   |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| "It must never email a customer without approval" | A `tool_usage` check, `at_most` 0, with `argument_matchers`                                                                              |
| "It has to pick the right account"                | A `tool_usage` check with an argument matcher on the id                                                                                  |
| "It should sound like us"                         | An `llm_judge` check with their own good and bad examples pasted in — see [Grading against examples](checks.md#grading-against-examples) |
| "It has to actually find something"               | An `llm_judge` check on the result, not on the run finishing                                                                             |
| "It gets the numbers right"                       | An `llm_judge` check comparing what it reported against what the tool returned                                                           |
| "It has to match what's actually in the CRM / KB" | A `custom_agent` check — an evaluator agent with the lookup tools attached — see `relevance-evals/evaluator-agents`                      |

A "must never" is usually the highest-value check there is: it is the one a later change can silently undo. (Inside an engagement, this is the compile step — the `relevance-engagements` skill's verification file covers it in more depth.)

**Never characterise run health from `relevance_list_agent_tasks`.** It returns ids, titles, timestamps and `creation_surface` — no state and no error. Ten rows coming back is not ten runs succeeding. To say anything about how the runs are going, read state: `relevance_list_tasks` with `agent_id` (plus `states` or `view`) returns `state`, `has_active_failures` and `errored_at`; `relevance_list_workforce_tasks` takes a `status_filter`. If you have not read state, do not describe the runs.
