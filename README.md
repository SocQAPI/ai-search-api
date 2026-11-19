<!-- Generated from the SocQ API catalog and docs. Do not edit by hand. -->
# AI Search API

[![SocQ AI Search API](https://img.shields.io/badge/API-SocQ%20AI%20Search%20API-F64C31)](https://socq.ai/apis/ai-search?utm_source=github&utm_medium=readme&utm_campaign=socq-ai-search-api)
[![API documentation](https://img.shields.io/badge/Docs-docs.socq.ai-111827)](https://docs.socq.ai/api-manual/ai-search/overview)
[![License: MIT](https://img.shields.io/badge/License-MIT-2563EB)](LICENSE)
[![Check examples](https://github.com/SocQAPI/ai-search-api/actions/workflows/check.yml/badge.svg)](https://github.com/SocQAPI/ai-search-api/actions/workflows/check.yml)

4 endpoints for public AI Search data through one SocQ API key, with runnable cURL, Node.js and Python examples and a valid request body for every endpoint.

[Browse AI Search APIs](https://socq.ai/apis/ai-search?utm_source=github&utm_medium=readme&utm_campaign=socq-ai-search-api) · [Get an API key](https://socq.ai/dashboard/api-key?utm_source=github&utm_medium=readme&utm_campaign=socq-ai-search-api) · [Documentation](https://docs.socq.ai/api-manual/ai-search/overview) · [All SocQ examples](https://github.com/SocQAPI/socq-examples)

## Endpoints

Prices are in SocQ credits; 1 credit = $0.005.

| Endpoint | publicId | What it returns | Price | Launched |
| --- | --- | --- | --- | --- |
| [LLM Mentions API](#llm-mentions-api) | `ai-search/llm-mentions` | Find the AI Overviews and ChatGPT answers that cite a domain or mention a keyword, with the full answer, cited sources and AI search volume. | 40 credits ($0.20) per call + 0.5 credits ($0.0025) per result | 2026-11-17 |
| [AI Search Visibility API](#ai-search-visibility-api) | `ai-search/llm-brand-metrics` | Measure how often AI answers mention a domain or keyword, with AI search volume, the most cited source domains and the brands named alongside it. | 40 credits ($0.20) per call + 0.5 credits ($0.0025) per result | 2026-11-17 |
| [LLM Top Sources API](#llm-top-sources-api) | `ai-search/llm-top-mentioned` | Rank the ten domains or pages that AI answers cite most for a keyword or domain, by mentions and AI search volume. | 40 credits ($0.20) per call + 0.5 credits ($0.0025) per result | 2026-11-17 |
| [Google AI Mode API](#google-ai-mode-api) | `ai-search/google-ai-mode` | Get the full Google AI Mode answer for a search in Markdown, with every cited source, for any country and language. | 2 credits ($0.01) per call | 2026-11-17 |

## Quick start

1. Create an API key at [socq.ai/dashboard/api-key](https://socq.ai/dashboard/api-key?utm_source=github&utm_medium=readme&utm_campaign=socq-ai-search-api).
2. Export it. The examples read only `SOCQ_API_KEY`.

```bash
git clone https://github.com/SocQAPI/ai-search-api.git
cd ai-search-api
export SOCQ_API_KEY="your-api-key"
```

3. Run any endpoint by its publicId. Without a second argument the script sends `payloads/<slug>.json`; pass a JSON file or an inline JSON object to change the input.

```bash
# Node.js 20+
node examples/node/socq.mjs ai-search/llm-mentions

# Python 3.8+ (standard library only)
python3 examples/python/socq.py ai-search/llm-mentions payloads/llm-mentions.json

# cURL + jq
bash examples/curl/socq.sh ai-search/llm-mentions '{"target":"notion.so","platform":"google","results_limit":100}'
```

Each script submits the task, polls it until it succeeds or fails, follows every result page, and writes the records to `output/<slug>-<task_id>.json`. Run `npm run check` to validate the payloads and scripts offline.

Never expose `SOCQ_API_KEY` in browser code, mobile apps, public repositories, screenshots, fixtures or logs.

## How the API works

Base URL `https://api.socq.ai`. Every request sends the API key as a bearer token:

```http
Authorization: Bearer <SOCQ_API_KEY>
Content-Type: application/json
```

All endpoints are asynchronous:

```text
POST /v1/<publicId>  ->  data.task_id  ->  GET /v1/tasks/{task_id}  ->  queued | running | succeeded | failed
```

1. **Submit** the JSON body to `POST https://api.socq.ai/v1/<publicId>`. The response returns `data.task_id` immediately; it means the task was created, not that data is ready.
2. **Poll** `GET https://api.socq.ai/v1/tasks/{task_id}` until `data.status` is `succeeded` or `failed` (a failed task carries `data.error_message`).
3. **Read results.** A succeeded task returns normalized records in `data.results.items`. Use `limit` (default 50, max 500) and pass `data.results.next_cursor` as `cursor` while `data.results.has_more` is true. `fields` selects up to 50 comma-separated fields or dot paths.
4. **Raw files (optional).** `GET https://api.socq.ai/v1/tasks/{task_id}/files` lists raw JSONL exports.

```bash
curl "https://api.socq.ai/v1/tasks/$TASK_ID?limit=100" \
  -H "Authorization: Bearer $SOCQ_API_KEY"

curl "https://api.socq.ai/v1/tasks/$TASK_ID?limit=100&cursor=$NEXT_CURSOR" \
  -H "Authorization: Bearer $SOCQ_API_KEY"
```

`callback_url` (where an endpoint accepts it) makes SocQ POST to your URL when the task reaches a terminal state; keep polling as a fallback.

**Billing.** A submit may reserve credits from `results_limit` or the number of submitted inputs; when the task finishes SocQ settles the charge from the actual result count and returns unused reserved credits. `402` means insufficient credits.

**Errors.** `400` invalid body or unsupported field, `401` missing or invalid key, `402` insufficient credits, `403` IP allowlist or task owned by another account, `404` endpoint or task not found, `422` input validation failed, `429` rate or credit-window limit (respect `Retry-After`), `500` submission failed. A client-side timeout does not cancel a task: resume polling the same `task_id` instead of submitting again.

Reference: [Authentication](https://docs.socq.ai/api-manual/agent-api/authentication) · [Task status and results](https://docs.socq.ai/api-manual/agent-api/status) · [Task files](https://docs.socq.ai/api-manual/agent-api/files) · [Callbacks](https://docs.socq.ai/api-manual/agent-api/callbacks) · [Errors](https://docs.socq.ai/api-manual/agent-api/errors)

## Endpoint reference

### LLM Mentions API

Find the AI Overviews and ChatGPT answers that cite a domain or mention a keyword, with the full answer, cited sources and AI search volume.

- **Request:** `POST https://api.socq.ai/v1/ai-search/llm-mentions`
- **publicId:** `ai-search/llm-mentions`
- **Price:** 40 credits ($0.20) per call + 0.5 credits ($0.0025) per result
- **Launched:** 2026-11-17
- **Example body:** [`payloads/llm-mentions.json`](payloads/llm-mentions.json)

More and more buyers ask an AI before they search. This endpoint shows the questions where Google AI Overviews or ChatGPT cite your domain, or where your brand or keyword comes up in the answer. Each row is one question with the full answer in Markdown, the pages it cited, how many people ask it each month, and, for ChatGPT, the brands it named and the follow-up searches it ran. Use it to audit generative engine optimization, find the pages AI trusts in your niche, or catch answers that describe your product wrongly.

Provide a domain in target or a keyword in query, not both. platform is google (AI Overviews, the default) or chat_gpt (tracked for the United States in English). results_limit defaults to 100 and accepts integers from 1 to 1000. Each request costs 40 credits plus 0.5 credits per answer returned; the 40 credits also apply when no answer is found.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `callback_url` | string (uri) | No | - |
| `language` | string | No | - |
| `language_code` | string | No | (length 2 to 10) |
| `location_code` | integer | No | Search market location code. (value at least 1) |
| `location_name` | string | No | Search market location name. |
| `platform` | string | No | AI answers to read: google (AI Overviews) or chat_gpt (ChatGPT, tracked for the United States in English). (one of `chat_gpt`, `google`; default `"google"`) |
| `query` | string | One of* | Keyword or brand name to track in AI answers, such as crm software. Give either target or query. |
| `results_limit` | integer | No | (value 1 to 1000; default `100`) |
| `target` | string | One of* | Domain to track in AI answers, such as hubspot.com. Give either target or query. |

*Required rule: send at least one of `query`, `target`; do not send `query` and `target` together.

```bash
curl -X POST "https://api.socq.ai/v1/ai-search/llm-mentions" \
  -H "Authorization: Bearer $SOCQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "target": "notion.so",
  "platform": "google",
  "results_limit": 100
}'
```

Full docs: [https://docs.socq.ai/api-manual/ai-search/llm-mentions](https://docs.socq.ai/api-manual/ai-search/llm-mentions) · Landing page: [socq.ai/apis/ai-search/llm-mentions](https://socq.ai/apis/ai-search/llm-mentions?utm_source=github&utm_medium=readme&utm_campaign=socq-ai-search-api)

### AI Search Visibility API

Measure how often AI answers mention a domain or keyword, with AI search volume, the most cited source domains and the brands named alongside it.

- **Request:** `POST https://api.socq.ai/v1/ai-search/llm-brand-metrics`
- **publicId:** `ai-search/llm-brand-metrics`
- **Price:** 40 credits ($0.20) per call + 0.5 credits ($0.0025) per result
- **Launched:** 2026-11-17
- **Example body:** [`payloads/llm-brand-metrics.json`](payloads/llm-brand-metrics.json)

A share-of-voice report for AI search. One request returns the total number of AI answers that mention your domain or keyword and the monthly search volume behind them, then breaks those answers down: the domains they cite most, the domains in the web results the AI looked at, and on ChatGPT the brands and brand categories named in the same answers. Run it for your brand and two competitors and you have a quick comparison of who AI recommends.

Provide a domain in target or a keyword in query, not both. platform is google (the default) or chat_gpt (tracked for the United States in English); brand rows appear on chat_gpt. Rows are the total followed by up to 10 entries for each breakdown. Each request costs 40 credits plus 0.5 credits per row; the 40 credits also apply when nothing is found.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `callback_url` | string (uri) | No | - |
| `language` | string | No | - |
| `language_code` | string | No | (length 2 to 10) |
| `location_code` | integer | No | Search market location code. (value at least 1) |
| `location_name` | string | No | Search market location name. |
| `platform` | string | No | AI answers to read: google (AI Overviews) or chat_gpt (ChatGPT, tracked for the United States in English). (one of `chat_gpt`, `google`; default `"google"`) |
| `query` | string | One of* | Keyword or brand name to track in AI answers, such as crm software. Give either target or query. |
| `target` | string | One of* | Domain to track in AI answers, such as hubspot.com. Give either target or query. |

*Required rule: send at least one of `query`, `target`; do not send `query` and `target` together.

```bash
curl -X POST "https://api.socq.ai/v1/ai-search/llm-brand-metrics" \
  -H "Authorization: Bearer $SOCQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "target": "hubspot.com",
  "platform": "chat_gpt"
}'
```

Full docs: [https://docs.socq.ai/api-manual/ai-search/llm-brand-metrics](https://docs.socq.ai/api-manual/ai-search/llm-brand-metrics) · Landing page: [socq.ai/apis/ai-search/llm-brand-metrics](https://socq.ai/apis/ai-search/llm-brand-metrics?utm_source=github&utm_medium=readme&utm_campaign=socq-ai-search-api)

### LLM Top Sources API

Rank the ten domains or pages that AI answers cite most for a keyword or domain, by mentions and AI search volume.

- **Request:** `POST https://api.socq.ai/v1/ai-search/llm-top-mentioned`
- **publicId:** `ai-search/llm-top-mentioned`
- **Price:** 40 credits ($0.20) per call + 0.5 credits ($0.0025) per result
- **Launched:** 2026-11-17
- **Example body:** [`payloads/llm-top-mentioned.json`](payloads/llm-top-mentioned.json)

Which sites does AI quote when people ask about your category? Give a keyword such as `crm software`, or a domain, and get the top ten domains or individual pages cited across the matching AI answers, ranked by how many answers cite them. The page view is a ready shortlist for digital PR and content outreach: these are the articles worth getting your product into.

Provide a keyword in query or a domain in target, not both. content_type is domains (the default) or pages. platform is google (the default) or chat_gpt (tracked for the United States in English). results_limit defaults to 10, which is also the maximum. Each request costs 40 credits plus 0.5 credits per ranked domain or page; the 40 credits also apply when nothing is found.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `callback_url` | string (uri) | No | - |
| `content_type` | string | No | Rank whole domains (domains) or single pages (pages). (one of `domains`, `pages`; default `"domains"`) |
| `language` | string | No | - |
| `language_code` | string | No | (length 2 to 10) |
| `location_code` | integer | No | Search market location code. (value at least 1) |
| `location_name` | string | No | Search market location name. |
| `platform` | string | No | AI answers to read: google (AI Overviews) or chat_gpt (ChatGPT, tracked for the United States in English). (one of `chat_gpt`, `google`; default `"google"`) |
| `query` | string | One of* | Keyword or brand name to track in AI answers, such as crm software. Give either target or query. |
| `results_limit` | integer | No | (value 1 to 10; default `10`) |
| `target` | string | One of* | Domain to track in AI answers, such as hubspot.com. Give either target or query. |

*Required rule: send at least one of `query`, `target`; do not send `query` and `target` together.

```bash
curl -X POST "https://api.socq.ai/v1/ai-search/llm-top-mentioned" \
  -H "Authorization: Bearer $SOCQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "query": "crm software",
  "platform": "google",
  "content_type": "domains"
}'
```

Full docs: [https://docs.socq.ai/api-manual/ai-search/llm-top-mentioned](https://docs.socq.ai/api-manual/ai-search/llm-top-mentioned) · Landing page: [socq.ai/apis/ai-search/llm-top-mentioned](https://socq.ai/apis/ai-search/llm-top-mentioned?utm_source=github&utm_medium=readme&utm_campaign=socq-ai-search-api)

### Google AI Mode API

Get the full Google AI Mode answer for a search in Markdown, with every cited source, for any country and language.

- **Request:** `POST https://api.socq.ai/v1/ai-search/google-ai-mode`
- **publicId:** `ai-search/google-ai-mode`
- **Price:** 2 credits ($0.01) per call
- **Launched:** 2026-11-17
- **Example body:** [`payloads/google-ai-mode.json`](payloads/google-ai-mode.json)

AI Mode is the conversational tab in Google Search that writes a complete answer instead of listing links. Send a query and get that answer as Markdown, plus the deduplicated list of sources it cites with titles, domains and excerpts. Track what Google tells people about your category, check whether your pages are among the sources, and compare answers across countries and languages.

Provide a search query in query. The defaults are United States (2840) and English (en). Each request costs 2 credits and returns one row with the answer and its sources.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `callback_url` | string (uri) | No | - |
| `language` | string | No | - |
| `language_code` | string | No | (length 2 to 10) |
| `location_code` | integer | No | Search market location code. (value at least 1) |
| `location_name` | string | No | Search market location name. |
| `query` | string | Yes | Search query, as typed into the search box. |

```bash
curl -X POST "https://api.socq.ai/v1/ai-search/google-ai-mode" \
  -H "Authorization: Bearer $SOCQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "query": "best crm for startups",
  "location_code": 2840,
  "language_code": "en"
}'
```

Full docs: [https://docs.socq.ai/api-manual/ai-search/google-ai-mode](https://docs.socq.ai/api-manual/ai-search/google-ai-mode) · Landing page: [socq.ai/apis/ai-search/google-ai-mode](https://socq.ai/apis/ai-search/google-ai-mode?utm_source=github&utm_medium=readme&utm_campaign=socq-ai-search-api)

## Repository contents

| Path | Purpose |
| --- | --- |
| [`examples/curl/socq.sh`](examples/curl/socq.sh) | Submit, poll and paginate with cURL and jq |
| [`examples/node/socq.mjs`](examples/node/socq.mjs) | Same workflow in Node.js 20+ (no dependencies) |
| [`examples/python/socq.py`](examples/python/socq.py) | Same workflow in Python 3.8+ (standard library) |
| [`payloads/`](payloads) | One documented request body per endpoint |
| [`endpoints.json`](endpoints.json) | Endpoint list, prices and request schemas used by the scripts and checks |
| [`fixtures/`](fixtures) | Synthetic submit and task response shapes |
| [`docs/production-notes.md`](docs/production-notes.md) | Retries, timeouts, pagination and data handling |
| [`scripts/check.mjs`](scripts/check.mjs) | Offline check: payloads, syntax and secrets |

## Responsible use and platform scope

- Use only publicly accessible AI Search data and the fields supported by each endpoint.
- Do not use the examples to access restricted content, login-only surfaces, authentication controls or non-public personal information.
- SocQ is not an official API of the represented platform and is not affiliated with or endorsed by it. Platform names and trademarks belong to their respective owners.
- Before production use, assess the laws, platform terms, privacy obligations and retention requirements that apply to your organization and use case.
- Collect only the fields needed for a defined purpose, restrict access, set retention periods, and support correction or deletion workflows where required.

This section describes the public-data boundary; it is not legal advice.

## License

[MIT](LICENSE)
