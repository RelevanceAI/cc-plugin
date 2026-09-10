---
title: Current model picks
description: Dated list of which model id to use for which task shape, with the reasoning effort to set alongside it.
---

# Current model picks

**Checked: 2026-08-11.**

This is the part of the model guidance that goes out of date. Everything in [SKILL.md](SKILL.md) stays true as models change; the ids below do not.

**If that date is more than a few months old**, treat the table as a starting point rather than an answer: rank the models `relevance_list_llm_models` returns using the properties in [SKILL.md](SKILL.md), prefer newer versions within a family, and tell the user the model guidance is overdue for a refresh.

## Which model for which task

Start at **Use**, then move along the row — step up when the task is complex or the cost of being wrong is high, step down when it is simple, high-volume, or cheap to be wrong. Confirm cost against `credits` in `relevance_list_llm_models` before telling a user something is cheaper.

| Task shape                                              | Use                         | Step up — complex or critical   | Step down — simple or high-volume | Why this one                                                                                                                                                                                                               |
| ------------------------------------------------------- | --------------------------- | ------------------------------- | --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Long-horizon agent — many tools, long unattended runs   | `anthropic-claude-sonnet-5` | `anthropic-claude-opus-5`       | `openai-gpt-5.6-terra`            | Errors compound across a long tool loop, and a large window absorbs a long run without truncation. Nothing checks the output before it lands, so this shape steps up readily.                                              |
| Customer-facing chat, human watching it stream          | `anthropic-claude-sonnet-5` | `anthropic-claude-opus-5`       | `anthropic-claude-haiku-4-5`      | Latency is felt as time-to-first-token, so set a low effort explicitly — see below, and do not inherit this model's default.                                                                                               |
| High-volume classification or extraction in a tool step | `openai-gpt-5.6-luna`       | `openai-gpt-5.6-terra`          | `openai-gpt-5-nano`               | Single-shot and schema-bounded, so failures are caught immediately by the next step. This is the cheapest place in a system to be wrong.                                                                                   |
| Long documents, whole-corpus analysis                   | `anthropic-claude-sonnet-5` | `anthropic-claude-opus-5`       | `google-gemini-3.5-flash`         | Retrieve and chunk regardless — recall degrades well before any model's window limit.                                                                                                                                      |
| Images, PDFs, audio, video                              | `google-gemini-3.6-flash`   | `google-gemini-3.1-pro-preview` | `google-gemini-3.5-flash-lite`    | Broadest native multimodal support. The flash tier handles ordinary extraction; step up for dense diagrams, poor scans, or anywhere a misread propagates downstream. Confirm `supported_input_media` covers the file type. |
| Code generation — transformation code, tool scripts     | `anthropic-claude-sonnet-5` | `anthropic-claude-opus-5`       | `openai-gpt-5.6-terra`            | Step up when the code runs unattended or writes to a real system — broken code fails loudly, subtly wrong code does not.                                                                                                   |
| Strict structured JSON at scale                         | `openai-gpt-5.6-terra`      | `openai-gpt-5.6-sol`            | `openai-gpt-5.6-luna`             | The constraint is schema honouring, not intelligence. Keep the schema flat where you can — deeply recursive schemas and string-length bounds are honoured inconsistently across vendors.                                   |
| Research or analysis, multi-source synthesis            | `anthropic-claude-sonnet-5` | `anthropic-claude-opus-5`       | `openai-gpt-5.6-terra`            | The failure to guard against is a confident fabrication nobody catches, so favour a current top-tier model and keep sources in context.                                                                                    |
| Routing, triage, `llm_condition`                        | `openai-gpt-5.6-luna`       | `openai-gpt-5.6-terra`          | `openai-gpt-5-nano`               | One call, flat output, and a wrong route fails loudly at the next step.                                                                                                                                                    |
| LLM-as-judge, eval scoring                              | `openai-gpt-5.6-terra`      | `openai-gpt-5.6-sol`            | `openai-gpt-5.6-luna`             | Must be independent of the model under test — never judge a model with itself. A broken judge fails in the direction of passing, so it looks like a good score.                                                            |
| Vision or computer use, screenshot in the loop          | `google-gemini-3.6-flash`   | `google-gemini-3.1-pro-preview` | `google-gemini-3.5-flash-lite`    | Cost here is dominated by image input, not text, so the text price is the wrong number to optimise.                                                                                                                        |

Two step-ups need a fallback set alongside them. `anthropic-claude-opus-5` needs a paid plan or the org's own key, and the platform names the exact plan when it blocks the call. `google-gemini-3.1-pro-preview` is a preview id and can be withdrawn at short notice — fall back to the flash tier.

Phone and meeting agents are not in this table — they pick from a restricted model list. Follow the phone-agent guidance in the agents skill instead.

## Set the effort explicitly

Take the valid values and the current default from the model's own `reasoning_capability` in `relevance_list_llm_models` — this file deliberately does not restate them, and a value borrowed from a sibling model can be invalid. What to pick:

- `anthropic-claude-sonnet-5`, `anthropic-claude-opus-5` — `low` or `medium` for chat and routine tool loops; `high` only for genuinely hard reasoning.
- Anything else on an effort ladder (`openai-gpt-5.6-*`, `openai-gpt-5-nano`, `google-gemini-3.*`) — `low` or `medium` once there is a tool loop. The bottom of that model's ladder is only right for single-shot latency-critical calls.
- `anthropic-claude-haiku-4-5` — takes a token budget, not an effort level. Give it a budget if it is in a tool loop.

`anthropic-claude-haiku-4-5` has the smallest context window of the models above. Pick it for speed and cost, not for long context.

Nothing here restates a model's own configuration — context windows, effort ladders, defaults and plan gating come from `relevance_list_llm_models` and the platform's error messages, which are authoritative. Everything above is judgement about which model suits which task at the checked date.
