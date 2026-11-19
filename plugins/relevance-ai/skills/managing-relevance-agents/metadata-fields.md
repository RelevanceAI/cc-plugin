---
title: Agent Metadata Fields
description: Label every run of an agent with values such as a count, a yes/no flag or a category (the Memory tab's fields in the app), so each task shows them and tasks can be filtered by them. Load before adding, changing, renaming or removing an agent's metadata fields. Long-term memory is a different setting.
---

# Agent Metadata Fields

An agent's metadata fields label each of its runs. Each field has a title and a type, and every run then carries one value per field, shown on its task and usable as a task filter. In the app they are the fields on the agent's Memory tab. They are not long-term memory, which is the separate `memory` setting.

Change them with `relevance_update_agent_metadata_fields`. `relevance_update_agent` doesn't accept `custom_metadata`. Changes go to the agent's draft: publish to make them live.

## Choose how each field is filled

| The value is…                                                       | Use                                                                                              |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| Read out of the run: a count in a summary, a category, a short note | `fill: { mode: "agent" }`, with a `description` that says what to record and where it comes from |
| Whether a tool succeeded or a trigger fired, or how many times      | `fill: { mode: "rule", on: "tool" \| "trigger", ids: [...] }` on a boolean or number field       |

- **Agent-filled fields:** the agent records the value during the run. Its `description` is the agent's only instruction, so be specific, e.g. "The Errors count in the summary the Handbook tool returns".
- **Rule-filled fields:** each time a listed tool finishes without an error, or a listed trigger fires:
  - a boolean field is set to true;
  - a number field counts up by 1.

  Tool ids are the agent's attached tools' `chain_id`s; trigger ids are its triggers' ids. A rule never reads a value out of a run, so it fills no other type.

Types: `boolean`, `number`, `string`, `single_option` and `multiple_option`. The two option types need `options`, the exact values the agent picks from.

## Titles are keys

- **A title is the key every run's value is stored under.** `upsert` edits the field with that exact title, or adds one; it never renames. Use `rename` only when the user asks: runs already labelled keep the old title, and saved task filters on it stop matching.
- **New titles are limited.** They take letters, digits, spaces, `_`, `-` and `.`, and start with a letter or digit. Two titles that differ only in case, or only in characters outside letters, digits, `_` and `-`, count as the same field.
- **Untouched fields stay as they are.** Fields a call doesn't name are left exactly as stored, including ones made in the app.

## Check your work

- The tool returns every field after saving, plus warnings to pass on to the user:
  - a rename;
  - a changed type;
  - dropped options;
  - the agent's conversation tags turning off or back on.
- An agent's conversation-tagging tool is off while it has any metadata field.
- After publishing, run the agent once and read the run's task to confirm the values are recorded.
