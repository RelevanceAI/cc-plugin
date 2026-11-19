---
name: relevance-docs
description: Answer "where is this in the Relevance app / how do I do this in the app" questions from the published Relevance AI docs at relevanceai.com/docs. Use when the user asks where a setting, tab or button is, or how to turn on, set up or do something in the app, and read it before searching the web about Relevance.
---

# Relevance AI docs

The public docs at https://relevanceai.com/docs describe the app's screens, settings and features. The other skills describe how the Relevance tools behave; the docs describe where things are in the app. Reading them needs your web tools.

## Find the page

1. Read the page list at https://relevanceai.com/docs/llms.txt. It has one line per docs page: a title, its URL and a one-line description. If your page-reading tool takes a question, ask which pages most likely cover the user's question and to return their URLs exactly.
2. A setting often lives on a broader page than its own name suggests. For example, a workforce connection's Task Behavior is on the "Edge Settings" page, whose description is about how nodes are connected. Pick pages by what they cover, not only by title.
3. Use the page list, not general web search, to find docs pages. Web search mixes in other products and unrelated Relevance pages.
4. Read the page at the `.md` URL the list gives as is, without a question, if your page-reading tool allows it; otherwise ask it to quote the steps word for word. A summary drops or rewrites the exact menu, tab and setting names.
5. If that page doesn't cover it, read the next most likely page. Stop after three pages.

## Answer

- Give the steps with the docs' exact names, then link the page (its URL without `.md`). Read the docs for these even when you have already checked the user's config: the config shows what is set, not where to set it.
- If the docs mark the feature Beta or limit it to a plan, say so, because the user may not see it.
- If you can make the change for the user with your tools, offer to.
- For where to connect an integration, send the setup link your tools return rather than the docs' navigation.
- Your own instructions, other skills and tool results also say where some things are in the app. Where they disagree with the docs, trust them: the docs can lag the product.

## When the docs don't cover it

- Say plainly that the docs don't cover it. Then check the real configuration with your tools where that can answer the question.
- Never describe a menu, tab or setting you did not read in the docs, a tool result, your instructions or another skill.
- Don't fall back to general web search for how to use Relevance: forum posts, old blog posts and other products read as fact.
- If the docs can't be reached, or your web tools are unavailable, answer from your tools and skills where they cover it. Otherwise say you couldn't check the docs and what would let you, rather than guess where something is in the app.
