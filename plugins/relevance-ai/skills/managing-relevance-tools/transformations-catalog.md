---
title: Popular Transformations Catalog
description: Categorised quick-lookup table of popular transformations (LLM, search, API, web scraping, code execution, email, integrations). Load when you need the right transformation name without reading the full reference.
---

# Popular Transformations Catalog

Quick reference for the most commonly used tool-step transformations in Relevance AI.

> **For implementation details, output patterns, and gotchas, see [transformations.md](transformations.md)**

---

## LLM & AI

| Name                       | Description                                                             |
| -------------------------- | ----------------------------------------------------------------------- |
| `prompt_completion`        | Send a prompt to an LLM and get a text response                         |
| `prompt_completion_vision` | Send images + text to vision-capable LLMs for image analysis            |
| `generate_image`           | Generate images from text descriptions (DALL-E, Stable Diffusion, etc.) |

## Data & Search

| Name            | Description                                                                                                                                                                                                                                                           |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `search`        | Semantic ("knowledge search") over a knowledge base. Source param `dataset_id` must be a knowledge set: `knowledge:<id>` or a `content_type: "knowledge_set"` input — a bare dataset name fails. See [patterns.md](patterns.md#knowledge-search-over-a-knowledge-set) |
| `retrieve_data` | Fetch records from Relevance AI datasets                                                                                                                                                                                                                              |
| `bulk_update`   | Upsert dataset records by `_id`. ⚠️ Replaces each matched document **wholesale** — fields you omit from a document are dropped, not merged. There is no partial-merge option; read the record first and re-send all fields you want to keep.                          |

## API Calls

| Name                 | Description                                                                                                                                                             |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api_call`           | Make HTTP requests to **external** APIs (no auth injection)                                                                                                             |
| `relevance_api_call` | Make HTTP requests to **Relevance platform** APIs (auto-injects caller's auth). Use for platform proxy endpoints like `/replicate/*`, `/knowledge/*`, `/agents/*`, etc. |

### Per-provider API-call steps — the escape hatch for "there's no step for that"

Most integrated providers ship a **generic authenticated API-call step** alongside their
action-specific steps. It handles auth for you, so it reaches **any endpoint the connected
account's scopes permit** — including actions that have no dedicated step.

> **When no action-specific step exists for what you need, reach for the provider's `*_api_call`
> step** — see [the step-type ladder](transformations.md#picking-a-step-type-the-ladder).

There are 86 native ones. Naming is `{provider}_api_call` or `{provider}_native_api_call`, sometimes
with a `_v2` suffix (prefer the highest version). A representative sample:

| Provider                              | Transformation                                                                                                                  |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| Google                                | `google_api_call_v2` (Gmail, Calendar, Tasks, …)                                                                                |
| Microsoft                             | `microsoft_api_call`, `microsoft_outlook_native_api_call`                                                                       |
| Slack                                 | `slack_api_call`                                                                                                                |
| HubSpot                               | `hubspot_api_call`                                                                                                              |
| Salesforce                            | `salesforce_api_call`, `salesforce_soql_api_call`                                                                               |
| Notion                                | `notion_native_api_call`                                                                                                        |
| Linear                                | `linear_native_api_call`                                                                                                        |
| Jira                                  | `jira_native_api_call`                                                                                                          |
| GitHub                                | `github_native_api_call`                                                                                                        |
| Airtable                              | `airtable_native_api_call`                                                                                                      |
| Zendesk                               | `zendesk_native_api_call`                                                                                                       |
| Google Sheets / Drive / Docs / Slides | `google_sheets_native_api_call`, `google_drive_native_api_call`, `google_docs_native_api_call`, `google_slides_native_api_call` |

**To find one for any provider**, browse the integration tag and read the `native` results:
`relevance_list_transformations({ integration: "<Provider>" })`.

> ⚠️ **Do not add `category: "api_call"` to narrow this.** ~93% of the whole catalogue carries that
> tag (every third-party action step does), so it filters nothing — and for third-party-backed
> providers it actively _excludes_ the generic step you're hunting, because those generic configs
> carry no integration tag. If `integration:` doesn't surface it, try a one-word `search` for the
> provider name instead: these steps are named `"<Provider> API call"`.

### ⚠️ `path` vs `url` — check the schema, don't assume

These steps do **not** share one parameter shape, and every one sets
`additionalProperties: false`, so passing the wrong parameter is **rejected outright, not ignored**:

| Shape                                           | Count | Examples                                                                                 |
| ----------------------------------------------- | ----- | ---------------------------------------------------------------------------------------- |
| relative **`path`**, joined to a fixed base URL | 72    | `slack_api_call`, `hubspot_api_call`, `notion_native_api_call`, `github_native_api_call` |
| full **`url`**                                  | 11    | `google_api_call_v2`, `google_api_call`, `elevenlabs_api_call`, `webflow_api_call`       |

Some also need an extra tenant/host param — `jira_native_api_call` requires `site_id`, and the
Freshdesk / Databricks / Supabase steps need their subdomain or workspace URL.

**So always call `relevance_get_transformation` before setting `params`.** Both shapes below are
correct for their own step and wrong for the other:

```typescript
// `url` shape — google_api_call_v2 takes a FULL url (required: url, method, oauth_account_id).
// Worked example: listing Google Calendar events, which has no dedicated native step
// (the Google Calendar natives are only check_google_calendar_v2 / create_google_calendar_event_v2).
{
  name: "get_calendar_events",
  transformation: "google_api_call_v2",
  params: {
    oauth_account_id: "{{params.google_account}}",
    method: "GET",
    url: "https://www.googleapis.com/calendar/v3/calendars/primary/events?timeMin={{params.start}}&timeMax={{params.end}}&singleEvents=true&orderBy=startTime"
  },
  output: { events: "{{response_body.items}}" }
}

// `path` shape — slack_api_call takes a RELATIVE path (base url https://slack.com/api is fixed).
// Passing `url` here fails schema validation.
{
  name: "list_channels",
  transformation: "slack_api_call",
  params: {
    oauth_account_id: "{{params.slack_account}}",
    method: "GET",
    path: "/conversations.list"
  },
  output: { channels: "{{response_body.channels}}" }
}
```

**Output (both shapes):** `{{response_body}}`, `{{status}}`, `{{response_headers}}`, `{{url}}`.

## Web & Scraping

| Name                      | Description                                     |
| ------------------------- | ----------------------------------------------- |
| `browserless_scrape`      | Scrape web pages using a headless browser       |
| `serper_google_search`    | Perform Google searches via Serper API          |
| `extract_website_content` | Extract clean, structured content from websites |

## Apify

| Name                | Description                                                                                                                                                          |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `run_apify_dynamic` | Run Apify actors dynamically for advanced web scraping and automation. Select from popular actors (Instagram, TikTok, Google Maps, etc.) and configure their inputs. |

## Replicate (AI Generations)

| Name                    | Description                                                                                                                                             |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `run_replicate_dynamic` | Generate AI content with Replicate models. Collections include: text-to-video, text-to-image, image-to-image, text-to-speech, speech-to-text, and more. |

## Unipile (Social Messaging)

| Name              | Description                                                                                                                                |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `linkedin_action` | LinkedIn actions via Unipile - Send messages, start chats, send invitations, create posts, send comments, get user profiles, get all chats |
| `whatsapp_action` | WhatsApp actions via Unipile - Send messages, start chats, get chat history                                                                |
| `telegram_action` | Telegram actions via Unipile - Send messages, manage chats                                                                                 |

## LinkedIn (Data Enrichment)

| Name                                | Description                                              |
| ----------------------------------- | -------------------------------------------------------- |
| `get_linkedin_profile`              | Fetch LinkedIn profile data for a person (via RapidAPI)  |
| `linkedin_people_search`            | Search for people on LinkedIn by criteria (via RapidAPI) |
| `linkedin_people_search_crustdata`  | Search for people on LinkedIn using Crustdata's API      |
| `linkedin_company_search_crustdata` | Search for companies on LinkedIn using Crustdata         |

## Code Execution

| Name                         | Description                                   |
| ---------------------------- | --------------------------------------------- |
| `python_code_transformation` | Execute custom Python code                    |
| `js_code_transformation`     | Execute custom JavaScript code (Deno runtime) |

## Flow Control

| Name     | Description                                        |
| -------- | -------------------------------------------------- |
| `branch` | Conditional if/else branching                      |
| `loop`   | Iterate over an array, running steps for each item |
| `delay`  | Pause execution for a specified duration           |

## Email

| Name                    | Description                     |
| ----------------------- | ------------------------------- |
| `send_sendgrid_email`   | Send emails via SendGrid        |
| `send_gmail_email_v2`   | Send emails via Gmail (OAuth)   |
| `send_outlook_email_v2` | Send emails via Outlook (OAuth) |

## CRM & Sales

| Name                   | Description                                     |
| ---------------------- | ----------------------------------------------- |
| `hubspot_api_call`     | Make HubSpot API calls                          |
| `salesforce_soql`      | Run Salesforce SOQL queries                     |
| `apollo_api_call`      | Make Apollo.io API calls for sales intelligence |
| `apollo_people_search` | Search for people in Apollo.io                  |
| `outreach_api_call`    | Make Outreach API calls                         |

## Browser Automation

| Name           | Description                                                                         |
| -------------- | ----------------------------------------------------------------------------------- |
| `airtop_*`     | Airtop browser automation - click, type, scroll, query page, take screenshots, etc. |
| `browseruse_*` | Browser Use automation - similar capabilities to Airtop                             |

## Document Processing

| Name            | Description                                                    |
| --------------- | -------------------------------------------------------------- |
| `pdf_to_text`   | Extract text from PDF files (with optional OCR)                |
| `file_to_text`  | Extract text from various file types (PDF, Word, Excel, audio) |
| `reducto_parse` | Parse documents with Reducto                                   |
| `mistral_ocr`   | OCR using Mistral's vision models                              |
| `firecrawl`     | Web scraping and crawling via Firecrawl                        |

## Communication

| Name                        | Description                                  |
| --------------------------- | -------------------------------------------- |
| `slack_message`             | Send Slack messages                          |
| `slack_message_advanced`    | Send Slack messages with advanced formatting |
| `whatsapp_message`          | Send WhatsApp messages (via official API)    |
| `whatsapp_template_message` | Send WhatsApp template messages              |

## Voice & Phone

| Name                | Description                                    |
| ------------------- | ---------------------------------------------- |
| `blandai_send_call` | Make AI phone calls via Bland.ai               |
| `vapi_*`            | Voice AI capabilities via Vapi                 |
| `text_to_speech`    | Convert text to speech (ElevenLabs)            |
| `audio_to_text_v2`  | Transcribe audio to text (Deepgram/AssemblyAI) |

## Project Management

| Name                   | Description           |
| ---------------------- | --------------------- |
| `create_linear_ticket` | Create Linear tickets |
| `jira_create_issue`    | Create Jira issues    |
| `jira_search_issues`   | Search Jira issues    |

## Other Integrations

| Name                     | Description                                       |
| ------------------------ | ------------------------------------------------- |
| `google_sheets_api_call` | Read/write Google Sheets                          |
| `google_docs_action`     | Create/edit Google Docs                           |
| `confluence_*`           | Confluence page/post operations                   |
| `zoom_api_call`          | Zoom API operations                               |
| `recallai_*`             | Meeting bot operations (join, record, transcribe) |

---

## Quick Selection Guide

| Need to...                              | Use                                                                 |
| --------------------------------------- | ------------------------------------------------------------------- |
| Generate/analyze text                   | `prompt_completion`                                                 |
| Analyze images                          | `prompt_completion_vision`                                          |
| Search your docs                        | `search` (needs a `knowledge:<id>` source)                          |
| Scrape a website                        | `browserless_scrape` or `firecrawl`                                 |
| Advanced scraping (social, e-commerce)  | `run_apify_dynamic`                                                 |
| Generate video/image/audio              | `run_replicate_dynamic`                                             |
| Send LinkedIn messages                  | `linkedin_action` (Unipile)                                         |
| Get LinkedIn data                       | `get_linkedin_profile`                                              |
| Call an integrated provider's API       | `{provider}_api_call` (auth injected — prefer this over `api_call`) |
| Call an external API                    | `api_call`                                                          |
| Call Relevance platform API (with auth) | `relevance_api_call`                                                |
| Run an existing tool as a step          | `run_chain`                                                         |
| Search Google                           | `serper_google_search`                                              |
| Run custom Python                       | `python_code_transformation`                                        |
| Run custom JS                           | `js_code_transformation`                                            |
| If/else logic                           | `branch`                                                            |
| Process a list                          | `loop`                                                              |
| Send email                              | `send_sendgrid_email`                                               |
| Wait/pause                              | `delay`                                                             |

---

## Notes

- Transformations chain together - outputs available via `{{step_name.output}}` syntax
- Most transformations support `skip_if` conditions
- Some transformations require OAuth connections (Unipile, Gmail, etc.) or API keys (Apify, Replicate, etc.)
- Use `relevance_list_transformations` to find more transformations (8000+ available)
- Use `relevance_get_transformation` to see full parameter schemas
