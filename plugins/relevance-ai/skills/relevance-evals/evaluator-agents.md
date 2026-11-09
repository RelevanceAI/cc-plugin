---
name: relevance-evals/evaluator-agents
description: Evaluator agents — agents that grade conversations as a `custom_agent` check. When to use one over `llm_judge` / `custom_tool`, how to create, preview, publish and attach one, and how its verdicts and errors read.
---

# Evaluator agents (`custom_agent` checks)

An evaluator agent is an agent whose only job is to grade one conversation at a time against a check. A `custom_agent` check **runs** it: for every graded conversation the eval system triggers the evaluator's **published** version, the evaluator investigates with its own tools, and its verdict becomes the check result. Unlike an `llm_judge` check (one prompt, one model call over the transcript) it can look things up — a CRM record, a knowledge base, a ticket — and unlike a `custom_tool` check it reasons across steps instead of running a fixed pipeline.

## What makes an agent an evaluator

- Its `runtime_engine` is `evaluator`. `relevance_list_agents` and `relevance_get_agent` show `runtime_engine` on any agent whose engine is not the default; a normal agent has no such field. An ordinary agent cannot be used as a check: create an evaluator rather than repurposing an existing agent.
- At run time the platform injects two tools: **Read evaluated conversation** (the graded transcript as JSON messages, paginated, `from_end` to see how it finished; on a workforce one `scope` per graph node) and **Submit evaluation verdict** (`passed` boolean + `reason`). They are injected, never stored — do not add them to `actions`, and do not reference them with `{{_actions.<id>}}` pills; plain prose is enough ("read the conversation", "submit your verdict").
- Its run is a normal, visible conversation on the evaluator agent, billed as that agent's own run; the eval batch shows that cost against the check.

### How a run ends

- The evaluator reads, investigates, then calls "Submit evaluation verdict" → `rule_results[]` gets `passed` and `reason`.
- If it finishes without submitting, it is nudged once and made to submit from what it found — so a verdict is almost always recorded.
- An evaluator **error** (not a fail) is recorded when the run hits an LLM error, stops on a tool approval or escalation, times out, or the agent is unpublished, deleted, not an evaluator, or the caller may not trigger it. Read `rule_results[].error` apart from real verdicts, as with any check.

## When to choose it

| The check needs…                                                                                  | Use                                                                    |
| ------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| A judgement that can be made from the transcript alone                                            | `llm_judge` — cheaper, no setup                                        |
| A deterministic comparison against values pulled out of the transcript                            | `custom_tool` — see [Custom tool checks](checks.md#custom-tool-checks) |
| Verification against a source of truth (did the record actually change? is the answer in the KB?) | **`custom_agent`** with the lookup tools attached                      |
| Multi-step investigation, or reading a specific workforce node's own transcript                   | **`custom_agent`**                                                     |

Prefer the simplest check that can be trusted: an evaluator agent is a second agent to maintain, and each graded conversation costs a full agent run.

## Runbook: creating an evaluator agent

Follow the [Authoring checks](authoring.md#runbook-authoring-checks-ground--ask--confirm--calibrate) runbook for the criterion itself; this runbook is the mechanics.

1. **Create it as an evaluator.** `relevance_create_agent` with `runtime_engine: "evaluator"`, a name like `Evaluator: <check name>`, and a `system_prompt` structured as:
   - **Check criteria** — what a passing conversation looks like and what fails it, concrete and observable (the same bar as [Writing Effective Checks](checks.md#writing-effective-checks)).
   - **How to investigate** — which tools to use to verify what the conversation claims, and what to read first.
   - **Rules** — e.g. fail when evidence is missing rather than assume, cite the message that decided it.
     Do not restate the mechanics (reading the transcript, submitting the verdict): the platform adds them. The evaluator must never ask the user anything — nobody is there to answer — and must never change anything outside its own run.
2. **Attach read-only tools** with `relevance_attach_tools_to_agent`, and set `action_behaviour: "never-ask"` on the agent and on every attached tool (`relevance_update_agent_action`). A tool that asks for approval stops the run and the check errors. Never attach tools that write.
3. **Preview it** with `relevance_trigger_agent`, pasting a transcript (or a description of one) as the `message`. A direct trigger is a **preview**: there is no attached conversation to read, so the evaluator grades the message text it was given, and it runs the **draft** by default. Read the verdict from the "Submit evaluation verdict" tool run in `relevance_list_agent_task_messages` (`relevance_poll_agent_result` truncates tool output). Iterate on the prompt until the verdicts and reasons are right.
4. **Publish** with `relevance_publish_agent` (approval card). Checks only ever run the published version — a draft edit changes nothing until it is published, and an evaluator with no published version cannot be attached.
5. **Attach the check**: `relevance_create_eval_check(resource_type, resource_id, name, check_config: { type: "custom_agent", agent_id: "<evaluator_agent_id>" })`, then calibrate it against finished tasks like any other check ([Calibrating a check against historical tasks](authoring.md#runbook-calibrating-a-check-against-historical-tasks)). The check can go on scenarios and performance dashboards alike.

To reuse one that exists, look for `runtime_engine: "evaluator"` in `relevance_list_agents` (search by name, e.g. "Evaluator:") and confirm with `relevance_get_agent` that it has an `active_version_id`.

## Constraints

- An agent cannot be its own evaluator; the check create/update call is rejected.
- Only run it as a check or by a direct trigger. Attached as a sub-agent, as a workforce node or behind a trigger it refuses to run ("This is an evaluator agent…") — nothing stops you attaching it, so check `runtime_engine` before wiring an agent anywhere.
- A `custom_agent` check on a performance dashboard scores live conversations with no triggering user, so the evaluator's permission is checked only when the check is saved.
- If `relevance_create_eval_check` / `relevance_update_eval_check` answers "Custom evaluator agent checks are not enabled for this deployment.", the feature is not enabled for this project: tell the user so, keep the evaluator agent if they want it for later, and cover the criterion with `llm_judge` or `custom_tool` for now.
