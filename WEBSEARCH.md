# Claude Code — WebSearch Tool

How the WebSearch tool searches the web and returns results.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Request Flow](#request-flow)
  - [1. Input Validation](#1-input-validation)
  - [2. Permission Check](#2-permission-check)
  - [3. API Call with Server-Side Tool](#3-api-call-with-server-side-tool)
  - [4. Streaming and Progress](#4-streaming-and-progress)
  - [5. Response Parsing](#5-response-parsing)
  - [6. Result Formatting](#6-result-formatting)
- [How It Differs from WebFetch](#how-it-differs-from-webfetch)
- [Provider Support](#provider-support)
- [Prompt Requirements](#prompt-requirements)
- [Limitations](#limitations)
- [Source Files](#source-files)

---

## Overview

WebSearch performs web searches using Anthropic's **server-side `web_search_20250305` beta tool**. Unlike WebFetch, WebSearch does NOT make HTTP requests from the local machine. Instead, it delegates the entire search to the Anthropic API, which performs the search, fetches pages, and returns structured results — all server-side.

**Tool name:** `WebSearch`
**Read-only:** Yes
**Permissions:** Required (passthrough — always prompts user)
**Max result size:** 100,000 characters

**Input:**
| Parameter | Type | Required | Description |
|---|---|---|---|
| `query` | `string` | Yes | Search query (minimum 2 characters) |
| `allowed_domains` | `string[]` | No | Only include results from these domains |
| `blocked_domains` | `string[]` | No | Exclude results from these domains |

**Output:**
| Field | Type | Description |
|---|---|---|
| `query` | `string` | The executed search query |
| `results` | `array` | Mix of `SearchResult` objects and `string` text commentary |
| `durationSeconds` | `number` | Total time for the search operation |

Each `SearchResult` contains:
```typescript
{
  tool_use_id: string,
  content: Array<{ title: string, url: string }>
}
```

---

## Architecture

WebSearch works fundamentally differently from WebFetch. It uses Anthropic's **server-side tool** mechanism — the search happens entirely within the Anthropic API infrastructure:

```
Claude Code                          Anthropic API
─────────────                        ─────────────
                                    
User query: "latest React 19 features"
       │
       ▼
┌─────────────────┐
│  Permission      │──▶ Always asks user for approval
│  Check           │
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌──────────────────────────────────┐
│  API Request     │────▶│  Anthropic Messages API           │
│  with web_search │     │  ┌──────────────────────────┐    │
│  tool schema     │     │  │ Server-side web_search    │    │
│                  │     │  │ tool executes searches    │    │
│                  │◀────│  │ (up to 8 queries)         │    │
│  Stream events   │     │  └──────────────────────────┘    │
└────────┬────────┘     └──────────────────────────────────┘
         │
         ▼
┌─────────────────┐
│  Parse response  │──▶ Extract search results + text blocks
│  blocks          │
└────────┬────────┘
         │
         ▼
    Formatted results returned to main Claude model
```

**Key distinction:** No HTTP requests leave the local machine for the search. The Anthropic API performs the search, reads pages, and synthesizes results server-side. Claude Code only sends one API call and streams the response.

---

## Request Flow

### 1. Input Validation

**Source:** `WebSearchTool.ts:235` — `validateInput()`

Simple checks:
- `query` must be non-empty (minimum 2 characters from the Zod schema)
- Cannot specify both `allowed_domains` AND `blocked_domains` in the same request

```typescript
if (allowed_domains?.length && blocked_domains?.length) {
  return {
    result: false,
    message: 'Error: Cannot specify both allowed_domains and blocked_domains',
    errorCode: 2,
  }
}
```

### 2. Permission Check

**Source:** `WebSearchTool.ts:209` — `checkPermissions()`

WebSearch always returns `behavior: 'passthrough'` — it relies on the standard permission system to prompt the user. It suggests adding a local allow rule for the `WebSearch` tool name.

Unlike WebFetch, there are no preapproved domains or per-domain rules. The permission is tool-level, not domain-level.

### 3. API Call with Server-Side Tool

**Source:** `WebSearchTool.ts:254` — `call()`

This is the core of the tool. It makes a **streaming API call** to the Anthropic Messages API with the `web_search_20250305` beta tool:

```typescript
const toolSchema: BetaWebSearchTool20250305 = {
  type: 'web_search_20250305',
  name: 'web_search',
  allowed_domains: input.allowed_domains,
  blocked_domains: input.blocked_domains,
  max_uses: 8,   // Maximum 8 search queries per invocation
}
```

The API call:

```typescript
const queryStream = queryModelWithStreaming({
  messages: [userMessage],  // "Perform a web search for the query: <query>"
  systemPrompt: asSystemPrompt([
    'You are an assistant for performing a web search tool use',
  ]),
  tools: [],                 // No local tools
  options: {
    model: useHaiku ? getSmallFastModel() : mainLoopModel,
    extraToolSchemas: [toolSchema],   // Server-side web_search tool
    querySource: 'web_search_tool',
    ...
  },
})
```

The model decides **what to search** — it can make up to 8 separate search queries within a single API call to comprehensively answer the request. The search execution, page fetching, and content extraction all happen server-side within the Anthropic API.

A **GrowthBook feature flag** (`tengu_plum_vx3`) controls whether Haiku or the main model is used for the search. When Haiku is used, thinking is disabled and `toolChoice` forces the `web_search` tool.

### 4. Streaming and Progress

**Source:** `WebSearchTool.ts:299`

The response is streamed, and the tool emits progress events to the UI:

**Query update** — when the model starts a new search query:
```typescript
// Extracted from input_json_delta events via regex:
// /"query"\s*:\s*"((?:[^"\\]|\\.)*)"/
onProgress({
  data: { type: 'query_update', query: 'react 19 new features 2025' }
})
```

**Search results received** — when results come back for a query:
```typescript
onProgress({
  data: {
    type: 'search_results_received',
    resultCount: 10,
    query: 'react 19 new features 2025'
  }
})
```

The UI displays these as live updates:
```
Searching: react 19 new features 2025
Found 10 results for "react 19 new features 2025"
```

And the final result:
```
Did 3 searches in 4s
```

### 5. Response Parsing

**Source:** `WebSearchTool.ts:86` — `makeOutputFromSearchResponse()`

The API response is a sequence of content blocks:

```
[text] → [server_tool_use] → [web_search_tool_result] → [text + citations] → ...
```

These are parsed into a flat array of results:

- **`server_tool_use`** blocks mark the start of a search (skipped, used only as delimiters)
- **`web_search_tool_result`** blocks contain the search hits — extracted as `{ title, url }` pairs
- **`text`** blocks contain the model's commentary/synthesis — kept as strings
- Error results (e.g., `error_code` in the result) are logged and converted to error strings

The result is an interleaved array of search results and text:

```typescript
[
  "Here's what I found about React 19:",          // string (model text)
  { tool_use_id: "...", content: [                 // SearchResult
    { title: "React 19 Release Notes", url: "https://react.dev/blog/..." },
    { title: "What's New in React 19", url: "https://..." },
  ]},
  "The key features include...",                    // string (model text)
  { tool_use_id: "...", content: [...] },           // SearchResult (2nd search)
  "In summary..."                                   // string (model text)
]
```

### 6. Result Formatting

**Source:** `WebSearchTool.ts:401` — `mapToolResultToToolResultBlockParam()`

The results are formatted into a text block for the main Claude model:

```
Web search results for query: "latest React 19 features"

Here's what I found about React 19:

Links: [{"title":"React 19 Release Notes","url":"https://react.dev/blog/..."},
{"title":"What's New in React 19","url":"https://..."}]

The key features include...

Links: [{"title":"...","url":"..."}]

In summary...

REMINDER: You MUST include the sources above in your response to the user
using markdown hyperlinks.
```

The final reminder enforces the mandatory sourcing requirement from the tool's prompt.

---

## How It Differs from WebFetch

| Aspect | WebFetch | WebSearch |
|---|---|---|
| **Where search/fetch happens** | Local machine (axios HTTP client) | Anthropic API server-side |
| **Network requests from client** | Yes — direct HTTP to target site | No — only API call to Anthropic |
| **What it does** | Fetches a single known URL | Searches the web for a query |
| **Content processing** | Haiku processes raw HTML→markdown | API model synthesizes search results |
| **Bot detection** | Vulnerable — plain HTTP client | Handled server-side by Anthropic |
| **JavaScript rendering** | None — raw HTML only | Server-side (Anthropic's infrastructure) |
| **Caching** | 15-minute LRU cache locally | No local caching |
| **Domain control** | Preapproved list + per-domain rules | `allowed_domains`/`blocked_domains` params |
| **Permission model** | Per-domain (hostname-based rules) | Per-tool (single allow/deny) |
| **Max searches per call** | 1 URL | Up to 8 queries |
| **Use case** | "Read this specific page" | "Find information about X" |

---

## Provider Support

**Source:** `WebSearchTool.ts:168` — `isEnabled()`

WebSearch is not universally available. It depends on the API provider:

| Provider | Supported | Notes |
|---|---|---|
| **First-party** (Anthropic API) | Yes | Always enabled |
| **Vertex AI** | Conditional | Only Claude 4.0+ models (`claude-opus-4`, `claude-sonnet-4`, `claude-haiku-4`) |
| **Foundry** | Yes | All shipped models support it |
| **Other providers** | No | Tool is disabled entirely |

---

## Prompt Requirements

**Source:** `tools/WebSearchTool/prompt.ts`

The tool prompt imposes strict requirements on the model's behavior:

### Mandatory Source Attribution

```
CRITICAL REQUIREMENT - You MUST follow this:
  - After answering the user's question, you MUST include a "Sources:" section
  - In the Sources section, list all relevant URLs as markdown hyperlinks
  - This is MANDATORY - never skip including sources

  Example format:
    [Your answer here]

    Sources:
    - [Source Title 1](https://example.com/1)
    - [Source Title 2](https://example.com/2)
```

### Correct Year in Queries

```
IMPORTANT - Use the correct year in search queries:
  - The current month is <month year>. You MUST use this year when searching
    for recent information, documentation, or current events.
```

The current month/year is injected dynamically via `getLocalMonthYear()`.

### Domain Filtering

```
- Domain filtering is supported to include or block specific websites
- Web search is only available in the US
```

### Result Format

The tool result always ends with a reminder:
```
REMINDER: You MUST include the sources above in your response to the user
using markdown hyperlinks.
```

---

## Limitations

- **US availability only** — the web search beta is region-restricted
- **8 searches per call** — the `max_uses` is hardcoded to 8
- **Cannot use both domain filters** — `allowed_domains` and `blocked_domains` are mutually exclusive
- **No local caching** — every search hits the API (unlike WebFetch's 15-minute cache)
- **Provider-dependent** — disabled on non-supported API providers
- **No direct page access** — returns search result summaries, not full page content. Use WebFetch to read a specific URL from the results.
- **API cost** — each search is a full API call with streaming, using either Haiku or the main model
- **Beta API** — uses `web_search_20250305` which may change in future versions

---

## Source Files

| File | Purpose |
|---|---|
| `tools/WebSearchTool/WebSearchTool.ts` | Tool definition, API call, streaming, response parsing |
| `tools/WebSearchTool/prompt.ts` | Tool description with sourcing requirements |
| `tools/WebSearchTool/UI.tsx` | Terminal UI (search progress, result count display) |
