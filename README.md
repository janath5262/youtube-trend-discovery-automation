# YouTube Trend Discovery Automation (n8n)

A portfolio workflow for finding recently published YouTube videos, measuring engagement, filtering candidates, checking title language with Google Gemini, and logging selected videos to Airtable.

> **Portfolio status:** Imported from a locally built n8n workflow and sanitized for public sharing. This repository documents the configuration; a clean import and end-to-end execution have **not** been independently verified. The workflow is **not** an autonomous AI agent or MCP server. It is an n8n automation with an AI-assisted language-filtering step.

## Overview

**Input:** A search query entered into the n8n chat trigger.  
**Output:** Records created in an Airtable table for videos that pass the filters.

### Workflow

```text
Chat Trigger (topic)
  -> YouTube: Get many videos (last seven days, US, relevance)
  -> HTTP Request: YouTube videos endpoint (statistics)
  -> JavaScript: derive engagement metrics
  -> If: threshold/title filters
  -> Remove Duplicates
  -> LLM Chain: Gemini classifies English-language titles
      + Structured Output Parser
  -> If: Answer == Yes
  -> Airtable: Create record
```

The nodes named `Filter Bad words` and `Filter Languages` are n8n **If** filters. The language step evaluates **title language only**, not audio/transcript language. There is no published site or scheduled job included in this export.

## Technology

- **n8n**: Chat trigger, routing, JavaScript processing, and external integrations
- **YouTube Data API**: video search and statistics
- **Google Gemini**: English-title classification
- **Airtable**: saved video shortlist

## Filter logic in the uploaded workflow

The provided JavaScript computes:

```javascript
likeRatio = views > 0 ? (likes / views) * 100 : 0;
commentRatio = views > 0 ? (comments / views) * 100 : 0;
```

All four conditions must pass:

| Test | Condition |
| --- | --- |
| Like ratio | Greater than 5% |
| Comment ratio | Greater than 0.3% |
| Views | Greater than 1,000 |
| Video title | Does not contain `crypto` (case-sensitive in the export) |

The source search is scoped to videos published in the **previous seven days**, with `regionCode = US`, `safeSearch = moderate`, and `order = relevance`. Some legitimate videos will not pass these deliberately narrow rules.

## Prerequisites

1. A running [n8n](https://n8n.io/) instance with the bundled YouTube, HTTP Request, Gemini, Airtable, and AI chain nodes supported by your installation.
2. A Google account authorized to use the YouTube n8n node and a Google Cloud project with **YouTube Data API v3** enabled.
3. A **newly issued** YouTube Data API key for the statistics request, stored in an **n8n credential**, not in `workflow.json`.
4. Google Gemini credentials configured in n8n.
5. An Airtable account, API token, base, and table.

### Airtable schema

Create a table with the following exact field names (types in parentheses):

| Field | Type | Source in workflow |
| --- | --- | --- |
| Title | Single line text | Video title |
| URL | URL | YouTube watch link |
| View Score | Number | **Like ratio percentage**, not number of views |
| Comment Score | Number | Comment ratio percentage |

Consider renaming `View Score` to `Like Ratio (%)` in your own version; if you rename the Airtable field, update its n8n mapping as well.

## Import and configure

1. Download/clone this repository. In n8n, create a workflow and use **Import from File** to select `workflow.json`.
2. Reconnect **YouTube** OAuth credentials in `Get many videos`.
3. In the **Get Video Data** HTTP Request node, create/select a secure **Query Auth** credential whose parameter name is `key` and value is your **rotated** YouTube API key. Configure the node's **Authentication** to use that credential. The public JSON intentionally contains **no API key**; without the credential, the node will not authenticate.
4. Reconnect Google Gemini credentials in `Google Gemini Chat Model`.
5. Create the Airtable schema above, set the Airtable base/table in `Create a record` (the JSON contains placeholders), and reconnect Airtable credentials.
6. Confirm `Filter Language` can use the Gemini model and its Structured Output Parser. Expected parsed form: `{ "Answer": "Yes" }` or `{ "Answer": "No" }`.
7. Save the workflow. Leave it inactive until all account-specific parameters are set and tests pass.

**Security note:** The originally exported workflow included a **literal YouTube Data API key**. Treat that key as exposed: revoke it and issue a fresh one in Google Cloud. Removing it from the public file does not make the exposed key safe again. Review previous versions and git history if the original was ever committed.

## Test checklist

- [ ] Enter a search phrase in n8n chat; verify the YouTube search returns video items.
- [ ] Confirm `Get Video Data` returns `items[0].statistics` for a sample `videoId`.
- [ ] Confirm the JavaScript node produces `title`, `youtubeUrl`, `views`, `likes`, `comments`, `likeRatio`, `commentRatio`.
- [ ] Check both the true and false paths of the numeric/title filter.
- [ ] Confirm the Gemini step emits structured `Answer` (`Yes` or `No`).
- [ ] Confirm the final filter accepts only `Yes`.
- [ ] Verify a qualifying sample appears once in Airtable, with correct mapping.
- [ ] Verify empty search results, API quota limits, and missing statistics are handled acceptably.

Avoid using real personal information or credentials in demo screenshots. See [SECURITY.md](SECURITY.md).

## Known limitations / improvements

- **No verified end-to-end run is provided** with this export; screenshots and a demo recording are not yet attached.
- The first YouTube node uses `returnAll: true`. Broad queries can consume quota quickly and increase run time; consider limiting returned records.
- There is no explicit error-handling path for YouTube API failures, quotas, Gemini failures, or Airtable failures.
- The JS node returns an error item when YouTube statistics are missing; subsequent filters may quietly discard that item rather than surface a useful failure.
- Title-language classification may be imperfect, and title language is not a guarantee of video language.
- `Remove Duplicates` is configured with default options in the export; verify whether it removes duplicates across executions versus within one run for your use case.
- `Filter Bad words` currently excludes only the literal string `crypto` as configured; it is not a general content-safety classifier.
- The workflow does not explicitly return a formatted shortlist to the chat user; its primary output is Airtable record creation.
- The workflow uses AI to classify titles, but it does **not** expose autonomous tool selection like the separately built AI Business Assistant Agent.
