---
title: Common Tool Patterns
description: Reusable tool design patterns — search + analyse, scrape + extract, multi-query loops, URL processing, enrichment, conditional branching, error handling. Load when designing multi-step tools or you need worked examples.
---

# Common Tool Patterns

Reusable patterns for building Relevance AI tools.

## Search + LLM Analysis

Search for information and analyze results.

```typescript
{
  studio_id: "search-analyze",
  title: "Search and Analyze",
  params_schema: {
    type: "object",
    properties: {
      query: { type: "string", title: "Query" }
    },
    required: ["query"]
  },
  transformations: {
    steps: [
      {
        name: "search",
        transformation: "serper_google_search",
        params: { search_query: "{{params.query}}" },
        output: { results: "{{organic}}" }
      },
      {
        name: "analyze",
        transformation: "prompt_completion",
        params: {
          prompt: "Analyze these search results about {{params.query}}:\n\n{{steps.search.output.results}}"
        },
        output: { analysis: "{{answer}}" }
      }
    ]
  }
}
```

## Knowledge Search (over a knowledge set)

Semantically search a knowledge base. The step id is `search` (the catalog's
friendly name is "knowledge search"). Its source param is `dataset_id`, and it
**only** runs on a knowledge set — the value must resolve to `knowledge:<knowledge_set_id>`
(or `knowledge:*` for all knowledge). A plain dataset name fails at run time with
_"Search can only be performed on a knowledge set. Please convert the dataset to
knowledge and try again."_

> **The #1 knowledge-search bug — same shape as the OAuth footgun.** Exposing the
> source as a plain `type: "string"` input (or defaulting it to a bare dataset name)
> renders a free-text box, not the knowledge-set picker. The step then receives a
> bare name, which resolves to a _dataset_, and fails — even though the input "looks"
> filled in. Search requires a **vectorized knowledge set**; convert a plain dataset
> to knowledge first (Knowledge tab in the app) if needed.

```typescript
// ❌ WRONG — plain string input defaulting to a bare dataset name.
//    Renders a text box; the step resolves it to a dataset and fails.
{
  params_schema: {
    properties: {
      knowledge_set: {
        type: "string",
        title: "Knowledge Table",
        default: "inbound_lead_source_messaging" // bare name — NOT a knowledge set
      }
    }
  },
  transformations: {
    steps: [
      {
        name: "search",
        transformation: "search",
        params: { dataset_id: "{{params.knowledge_set}}", query: "{{params.query}}", query_type: "vector" }
      }
    ]
  }
}

// ✅ RIGHT — knowledge-set input. Renders the picker, which yields a `knowledge:<id>` value.
{
  params_schema: {
    type: "object",
    properties: {
      knowledge_base: {
        type: "string",
        title: "Knowledge Base",
        description: "Select the knowledge set to search",
        metadata: { content_type: "knowledge_set" }
      },
      query: { type: "string", title: "Query" }
    },
    required: ["knowledge_base", "query"]
  },
  transformations: {
    steps: [
      {
        name: "search",
        transformation: "search",
        params: {
          dataset_id: "{{params.knowledge_base}}", // resolves to "knowledge:<id>"
          query: "{{params.query}}",
          query_type: "vector"
        },
        output: { results: "{{vector_search}}" }
      }
    ]
  }
}
```

To search a **fixed** knowledge base instead of taking it as an input, hardcode the
prefixed id: `dataset_id: "knowledge:<knowledge_set_id>"` (get ids from
`relevance_list_knowledge_sets`).

The related `summarize_knowledge` (source param `knowledge`) and `advanced_retrieval`
(source param `knowledge_set`) steps take a knowledge set the same way — wire their
source to a `content_type: "knowledge_set"` input or a `knowledge:`-prefixed value.

## Scrape + Extract

Scrape webpage and extract structured data.

```typescript
{
  studio_id: "scrape-extract",
  title: "Scrape and Extract",
  params_schema: {
    type: "object",
    properties: {
      url: { type: "string", title: "URL" }
    },
    required: ["url"]
  },
  transformations: {
    steps: [
      {
        name: "scrape",
        transformation: "browserless_scrape",
        params: {
          website_url: "{{params.url}}",
          method: "Text"
        },
        output: { content: "{{output.page}}" }
      },
      {
        name: "extract",
        transformation: "prompt_completion",
        params: {
          prompt: `Extract key information from this webpage:

{{steps.scrape.output.content}}

Return JSON with: title, main_points, contact_info`
        },
        output: { data: "{{answer}}" }
      }
    ]
  }
}
```

## Multi-Query Loop

Generate multiple queries and search each. This pattern shows where to place markdown notes for documentation — see [documentation.md](documentation.md) for the full rubric.

```typescript
{
  studio_id: "multi-search",
  title: "Multi-Query Search",
  params_schema: {
    type: "object",
    properties: {
      topic: { type: "string", title: "Topic" }
    },
    required: ["topic"]
  },
  transformations: {
    steps: [
      // Header note — orients anyone reading the tool
      {
        name: "overview",
        transformation: "markdown",
        params: {
          markdown: "### Multi-query web search and synthesis\n\nTakes a **{{params.topic}}** and produces a synthesised summary. The chain fans out (LLM generates queries → parallel search → synthesise) so the final summary is grounded in a wider slice of the web than a single query would return."
        },
        output: {}
      },
      // Generate queries as JSON
      {
        name: "generate_queries",
        transformation: "prompt_completion",
        params: {
          prompt: `Generate 5 search queries for: {{params.topic}}
Output ONLY a JSON array of strings:
["query 1", "query 2", ...]`
        },
        output: { queries_json: "{{answer}}" }
      },
      // Inline note — explains why the parse step exists (non-obvious data reshape)
      {
        name: "why_parse",
        transformation: "markdown",
        params: {
          markdown: "The LLM returns a JSON string in **{{steps.generate_queries.output.queries_json}}**, but `loop` requires an actual array — not a string. This step parses it and strips any markdown code-fence the LLM might wrap around the JSON."
        },
        output: {}
      },
      // Parse JSON to array
      {
        name: "parse_queries",
        transformation: "python_code_transformation",
        params: {
          code: `
import json
json_str = """{{steps.generate_queries.output.queries_json}}"""
json_str = json_str.strip()
if json_str.startswith("\`\`\`"):
    json_str = json_str.split("\\n", 1)[1]
if json_str.endswith("\`\`\`"):
    json_str = json_str[:-3]
return {"queries": json.loads(json_str.strip())}
`
        }
      },
      // Loop through queries
      {
        name: "search_loop",
        transformation: "loop",
        params: {
          items: "{{steps.parse_queries.output.transformed.queries}}",
          execution_mode: "parallel",
          steps: [
            {
              name: "search",
              transformation: "serper_google_search",
              params: {
                search_query: "{{item}}",
                num: 5
              },
              output: { results: "{{organic}}" }
            }
          ]
        },
        output: { all_results: "{{results}}" }
      },
      // Synthesize all results
      {
        name: "synthesize",
        transformation: "prompt_completion",
        params: {
          prompt: `Synthesize findings about {{params.topic}} from these search results:

{{steps.search_loop.output.all_results}}

Provide a comprehensive summary.`
        },
        output: { summary: "{{answer}}" }
      }
    ]
  }
}
```

## URL List Processing

Process a list of URLs in parallel.

```typescript
{
  studio_id: "process-urls",
  title: "Process URL List",
  params_schema: {
    type: "object",
    properties: {
      urls: {
        type: "array",
        title: "URLs",
        items: { type: "string" }
      }
    },
    required: ["urls"]
  },
  transformations: {
    steps: [
      {
        name: "scrape_all",
        transformation: "loop",
        params: {
          items: "{{params.urls}}",
          execution_mode: "parallel",
          error_handling: "continue",
          steps: [
            {
              name: "scrape",
              transformation: "browserless_scrape",
              params: {
                website_url: "{{item}}",
                method: "Text"
              },
              output: { content: "{{output.page}}" }
            }
          ]
        },
        output: { pages: "{{results}}" }
      }
    ]
  }
}
```

## Data Enrichment

Enrich data with external lookups.

```typescript
{
  studio_id: "enrich-company",
  title: "Enrich Company Data",
  params_schema: {
    type: "object",
    properties: {
      company_name: { type: "string", title: "Company Name" },
      domain: { type: "string", title: "Domain (optional)" }
    },
    required: ["company_name"]
  },
  transformations: {
    steps: [
      // Header note — the website scrape is conditional on `domain`, worth flagging
      {
        name: "overview",
        transformation: "markdown",
        params: {
          markdown: "### Enrich a company from name (+ optional domain)\n\nTakes **{{params.company_name}}** and an optional **{{params.domain}}**. Always searches the web; additionally scrapes the company's homepage when a domain is supplied. The final LLM step fuses both sources into a structured JSON profile."
        },
        output: {}
      },
      // Search for company info
      {
        name: "search_company",
        transformation: "serper_google_search",
        params: {
          search_query: "{{params.company_name}} company overview",
          num: 5
        },
        output: { results: "{{organic}}" }
      },
      // Scrape website if domain provided
      {
        name: "scrape_site",
        transformation: "browserless_scrape",
        if: "{{params.domain}}",
        params: {
          website_url: "https://{{params.domain}}",
          method: "Text"
        },
        output: { website_content: "{{output.page}}" }
      },
      // Extract structured data
      {
        name: "extract_info",
        transformation: "prompt_completion",
        params: {
          prompt: `Extract company information:

Company: {{params.company_name}}
Search results: {{steps.search_company.output.results}}
Website content: {{steps.scrape_site.output.website_content}}

Return JSON:
{
  "name": "",
  "description": "",
  "industry": "",
  "size": "",
  "headquarters": "",
  "website": "",
  "social_links": []
}`
        },
        output: { company_data: "{{answer}}" }
      }
    ]
  }
}
```

## Error Handling Pattern

Robust error handling with Python.

```typescript
{
  name: "safe_parse",
  transformation: "python_code_transformation",
  params: {
    code: `
import json

try:
    json_str = """{{steps.previous_step.output}}"""
    json_str = json_str.strip()

    # Handle markdown code blocks
    if json_str.startswith("\`\`\`"):
        json_str = json_str.split("\\n", 1)[1]
    if json_str.endswith("\`\`\`"):
        json_str = json_str[:-3]

    data = json.loads(json_str.strip())
    return {"data": data, "success": True, "error": None}

except Exception as e:
    return {"data": None, "success": False, "error": str(e)}
`
  }
}
```

## Conditional Branching

Different actions based on input.

```typescript
{
  studio_id: "conditional-process",
  transformations: {
    steps: [
      {
        name: "overview",
        transformation: "markdown",
        params: {
          markdown: "### Route input to a type-specific handler\n\nClassifies **{{params.input}}** as URL, email, or other, then runs only the matching handler via `if:` branches. The classifier output (`{{steps.determine_type.output.type}}`) is the routing key."
        },
        output: {}
      },
      {
        name: "determine_type",
        transformation: "prompt_completion",
        params: {
          prompt: "Is '{{params.input}}' a URL, email, or other? Reply with just: url, email, or other",
          model: "openai-gpt-4o-mini"
        },
        output: { type: "{{answer}}" }
      },
      {
        name: "process_url",
        transformation: "browserless_scrape",
        if: "{{steps.determine_type.output.type == 'url'}}",
        params: {
          website_url: "{{params.input}}"
        }
      },
      {
        name: "process_email",
        transformation: "prompt_completion",
        if: "{{steps.determine_type.output.type == 'email'}}",
        params: {
          prompt: "Extract info from email: {{params.input}}"
        }
      }
    ]
  }
}
```

## Pagination Pattern

Handle paginated data sources.

```typescript
{
  name: "fetch_all_pages",
  transformation: "python_code_transformation",
  params: {
    code: `
import requests

all_items = []
page = 1
has_more = True

while has_more and page <= 10:  # Max 10 pages safety
    response = requests.get(
        "{{params.api_url}}",
        params={"page": page, "per_page": 100}
    )
    data = response.json()
    all_items.extend(data.get("items", []))
    has_more = data.get("has_more", False)
    page += 1

return {"items": all_items, "total_pages": page - 1}
`
  }
}
```
