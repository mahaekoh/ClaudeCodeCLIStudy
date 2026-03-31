# Claude Code — WebFetch Tool

How the WebFetch tool retrieves, processes, and returns web content.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Request Flow](#request-flow)
  - [1. Input Validation](#1-input-validation)
  - [2. Permission Check](#2-permission-check)
  - [3. Cache Lookup](#3-cache-lookup)
  - [4. Domain Blocklist Preflight](#4-domain-blocklist-preflight)
  - [5. HTTP Fetch](#5-http-fetch)
  - [6. Redirect Handling](#6-redirect-handling)
  - [7. Content Conversion](#7-content-conversion)
  - [8. Secondary Model Processing](#8-secondary-model-processing)
  - [9. Result Assembly](#9-result-assembly)
- [Preapproved Domains](#preapproved-domains)
- [Caching](#caching)
- [Security](#security)
- [Limitations](#limitations)
- [Source Files](#source-files)

---

## Overview

WebFetch retrieves content from a URL, converts it to markdown, and processes it through a secondary AI model (Haiku) to extract the information requested by the user's prompt. The main Claude model never sees the raw page content — only the Haiku-processed summary.

**Tool name:** `WebFetch`
**Read-only:** Yes
**Permissions:** Required (unless the host is preapproved)
**Max result size:** 100,000 characters

**Input:**
| Parameter | Type | Required | Description |
|---|---|---|---|
| `url` | `string` | Yes | URL to fetch |
| `prompt` | `string` | Yes | What to extract from the content |

**Output:**
| Field | Type | Description |
|---|---|---|
| `bytes` | `number` | Size of fetched content |
| `code` | `number` | HTTP status code |
| `codeText` | `string` | HTTP status text |
| `result` | `string` | Processed content from the secondary model |
| `durationMs` | `number` | Total time for fetch + processing |
| `url` | `string` | The URL that was fetched |

---

## Architecture

WebFetch uses a **two-model architecture**:

```
User prompt + URL
       │
       ▼
┌─────────────────┐
│  Permission      │──▶ Preapproved host? User rule? Ask user?
│  Check           │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  LRU Cache       │──▶ Cache hit? Return immediately.
│  (15 min TTL)    │
└────────┬────────┘
         │ miss
         ▼
┌─────────────────┐
│  Domain Blocklist│──▶ Anthropic API: api.anthropic.com/api/web/domain_info
│  Preflight       │    Blocked? → Error. Allowed? → Continue.
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  HTTP GET        │──▶ axios, 60s timeout, 10 MB max, HTTPS upgrade
│  (via axios)     │    Redirects handled manually.
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  HTML → Markdown │──▶ Turndown library (pure JS DOM, not a browser)
│  (Turndown)      │    Non-HTML passed through as-is.
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Haiku Model     │──▶ Summarizes/extracts per the user's prompt.
│  (secondary LLM) │    Different guidelines for preapproved vs other domains.
└────────┬────────┘
         │
         ▼
    Tool result returned to main Claude model
```

**This is NOT a browser.** There is no Chromium, Puppeteer, or headless browser. It is a plain HTTP client (`axios`) that fetches raw bytes, converts HTML to markdown with Turndown (a pure-JS DOM parser via `@mixmark-io/domino`), and passes the text to Haiku for processing.

---

## Request Flow

### 1. Input Validation

**Source:** `utils.ts:139` — `validateURL()`

Before anything else, the URL is validated:

- Must be under **2,000 characters**
- Must parse as a valid URL
- Must not contain embedded **username/password** credentials
- Hostname must have at least **2 parts** (no bare `localhost`)

```typescript
// URL length cap (was 250, relaxed for JWT-signed URLs like cloud service signed URLs)
const MAX_URL_LENGTH = 2000
```

### 2. Permission Check

**Source:** `WebFetchTool.ts:104` — `checkPermissions()`

Three-tier permission resolution:

1. **Preapproved host** → Auto-allow (no user prompt). See [Preapproved Domains](#preapproved-domains).
2. **User-configured rules** → Check `allow`/`deny`/`ask` rules matched by `domain:<hostname>`.
3. **No matching rule** → Prompt the user for approval. Suggests adding a local allow rule for the domain.

Permission rules match on `domain:<hostname>`, not the full URL. Granting permission to `domain:example.com` allows all paths on that host.

### 3. Cache Lookup

**Source:** `utils.ts:356`

An **LRU cache** is checked before any network call:

```typescript
const CACHE_TTL_MS = 15 * 60 * 1000   // 15 minutes
const MAX_CACHE_SIZE_BYTES = 50 * 1024 * 1024  // 50 MB

const URL_CACHE = new LRUCache<string, CacheEntry>({
  maxSize: MAX_CACHE_SIZE_BYTES,
  ttl: CACHE_TTL_MS,
})
```

Cache hits return the full `FetchedContent` immediately — no network request, no domain check, no secondary model call.

Content is cached under the **original URL** (not the upgraded/redirected URL).

### 4. Domain Blocklist Preflight

**Source:** `utils.ts:176` — `checkDomainBlocklist()`

Unless `skipWebFetchPreflight` is set in settings (for enterprise customers whose security policies block outbound connections to `claude.ai`), the tool makes a **preflight GET** to Anthropic's domain check API:

```
GET https://api.anthropic.com/api/web/domain_info?domain=<hostname>
```

- **`can_fetch: true`** → domain is allowed, result cached for 5 minutes in a separate `DOMAIN_CHECK_CACHE`
- **`can_fetch: false`** → throws `DomainBlockedError` ("Claude Code is unable to fetch from \<domain\>")
- **Non-200 / network error** → throws `DomainCheckFailedError`

The domain check cache is **hostname-keyed** (not URL-keyed) to avoid duplicate preflight calls when fetching multiple paths on the same host.

### 5. HTTP Fetch

**Source:** `utils.ts:262` — `getWithPermittedRedirects()`

The actual HTTP request uses **axios**:

```typescript
axios.get(url, {
  signal,                                    // AbortController signal
  timeout: FETCH_TIMEOUT_MS,                 // 60 seconds
  maxRedirects: 0,                           // Redirects handled manually
  responseType: 'arraybuffer',               // Raw bytes
  maxContentLength: MAX_HTTP_CONTENT_LENGTH,  // 10 MB
  headers: {
    Accept: 'text/markdown, text/html, */*',
    'User-Agent': getWebFetchUserAgent(),     // Custom user agent string
  },
})
```

Key constraints:
| Setting | Value |
|---|---|
| Timeout | 60 seconds |
| Max content length | 10 MB |
| Max redirects | 10 hops (manual handling) |
| Protocol | HTTPS only (HTTP auto-upgraded) |
| Response type | `arraybuffer` (raw bytes) |

**No JavaScript execution.** No cookies. No sessions. No rendering engine. It's a plain HTTP GET.

### 6. Redirect Handling

**Source:** `utils.ts:207` — `isPermittedRedirect()`

Redirects are **not followed automatically** (per Product Security Review guidelines). Instead:

**Same-host redirects** (including `www.` addition/removal) are followed automatically, up to 10 hops:

```typescript
// Permitted: example.com → www.example.com (www added)
// Permitted: www.example.com → example.com (www removed)
// Permitted: example.com/old → example.com/new (path change, same host)
// NOT permitted: example.com → other-site.com (different host)
```

**Cross-host redirects** are NOT followed. Instead, the tool returns a `RedirectInfo` object to the model:

```
REDIRECT DETECTED: The URL redirects to a different host.

Original URL: https://example.com/page
Redirect URL: https://cdn.example.net/page
Status: 301 Moved Permanently

To complete your request, I need to fetch content from the redirected URL.
Please use WebFetch again with these parameters:
- url: "https://cdn.example.net/page"
- prompt: "<original prompt>"
```

The model must make a new, separate WebFetch call for the redirect target. This prevents open-redirect exploitation where a trusted domain redirects to a malicious one.

**Egress proxy detection:** If the response is a 403 with `X-Proxy-Error: blocked-by-allowlist` header, it's recognized as an enterprise egress proxy block and surfaced as an `EgressBlockedError`.

### 7. Content Conversion

**Source:** `utils.ts:454`

The raw response bytes are processed based on content type:

| Content-Type | Processing |
|---|---|
| `text/html` | Converted to markdown via **Turndown** |
| Everything else | Used as-is (raw text) |
| Binary (`application/pdf`, images, etc.) | Saved to disk with proper extension; UTF-8 decoded text still sent to Haiku |

**Turndown** is a pure-JavaScript HTML-to-markdown converter. It uses `@mixmark-io/domino` (a pure-JS DOM implementation) to parse HTML — not a browser engine. It is lazy-loaded as a singleton on first use to defer the ~1.4 MB import.

After conversion, the axios response buffer is explicitly freed (`response.data = null`) to allow GC to reclaim up to 10 MB before Turndown builds its DOM tree.

### 8. Secondary Model Processing

**Source:** `utils.ts:484` — `applyPromptToMarkdown()`

The markdown content is sent to **Haiku** (a fast, cheap Claude model) with the user's prompt:

```typescript
const assistantMessage = await queryHaiku({
  systemPrompt: asSystemPrompt([]),      // Empty system prompt
  userPrompt: modelPrompt,               // Content + prompt + guidelines
  signal,
  options: { querySource: 'web_fetch_apply', ... },
})
```

The model prompt is constructed by `makeSecondaryModelPrompt()`:

```
Web page content:
---
<markdown content, truncated to 100K chars>
---

<user's prompt>

<guidelines>
```

**Guidelines differ by domain type:**

For **preapproved domains** (code documentation, official docs):
```
Provide a concise response based on the content above. Include relevant details,
code examples, and documentation excerpts as needed.
```

For **non-preapproved domains** (all others):
```
Provide a concise response based only on the content above. In your response:
- Enforce a strict 125-character maximum for quotes from any source document.
  Open Source Software is ok as long as we respect the license.
- Use quotation marks for exact language from articles; any language outside of
  the quotation should never be word-for-word the same.
- You are not a lawyer and never comment on the legality of your own prompts and
  responses.
- Never produce or reproduce exact song lyrics.
```

The non-preapproved guidelines are **copyright/legal safeguards** that prevent verbatim reproduction of copyrighted content.

**Exception:** If the URL is preapproved, the content type is `text/markdown`, and the content is under 100K characters, the Haiku step is **skipped entirely** — the raw markdown is returned directly.

### 9. Result Assembly

**Source:** `WebFetchTool.ts:286`

The Haiku response becomes the `result` field. If binary content was additionally saved to disk:

```
[Binary content (application/pdf, 2.4 MB) also saved to /path/to/file.pdf]
```

The final output:

```typescript
{
  bytes: 45000,                          // Raw fetch size
  code: 200,                             // HTTP status
  codeText: 'OK',                        // HTTP status text
  result: 'Haiku-processed summary...',  // Processed content
  durationMs: 3200,                      // Total time
  url: 'https://example.com/page',       // Requested URL
}
```

---

## Preapproved Domains

**Source:** `preapproved.ts`

Approximately **80 code-related domains** are preapproved for fetching without user permission. These are GET-only exceptions — the sandbox system deliberately does NOT inherit this list for network restrictions, as POST/upload access to these domains could enable data exfiltration.

Categories include:

| Category | Examples |
|---|---|
| **Anthropic** | `code.claude.com`, `modelcontextprotocol.io` |
| **Language docs** | `docs.python.org`, `doc.rust-lang.org`, `go.dev`, `developer.mozilla.org` |
| **Web frameworks** | `react.dev`, `vuejs.org`, `nextjs.org`, `angular.io` |
| **Python ecosystem** | `docs.djangoproject.com`, `fastapi.tiangolo.com`, `pytorch.org`, `numpy.org` |
| **Cloud providers** | `docs.aws.amazon.com`, `cloud.google.com`, `learn.microsoft.com` |
| **Databases** | `www.postgresql.org`, `redis.io`, `www.mongodb.com` |
| **DevOps** | `kubernetes.io`, `www.docker.com`, `www.terraform.io` |
| **Mobile** | `developer.apple.com`, `developer.android.com`, `reactnative.dev` |
| **Tools** | `git-scm.com`, `bun.sh`, `nodejs.org`, `jestjs.io` |

Path-scoped entries (e.g., `github.com/anthropics`) enforce segment boundaries — `/anthropics` does NOT match `/anthropics-evil/malware`.

---

## Caching

Two separate caches:

### URL Content Cache

```typescript
const URL_CACHE = new LRUCache<string, CacheEntry>({
  maxSize: 50 * 1024 * 1024,  // 50 MB
  ttl: 15 * 60 * 1000,        // 15 minutes
})
```

Stores the full fetched-and-converted content (markdown, HTTP status, content type, persisted path). Keyed by the **original URL** (before HTTPS upgrade or redirect following). Size-weighted by content byte length.

### Domain Check Cache

```typescript
const DOMAIN_CHECK_CACHE = new LRUCache<string, true>({
  max: 128,                    // max entries
  ttl: 5 * 60 * 1000,         // 5 minutes
})
```

Caches only `allowed` results from the Anthropic domain blocklist API. Hostname-keyed to avoid duplicate preflight calls for different paths on the same host.

Both caches can be cleared via `clearWebFetchCache()`.

---

## Security

| Measure | Implementation |
|---|---|
| **Domain blocklist** | Preflight check against `api.anthropic.com/api/web/domain_info` |
| **No credentials in URLs** | Rejected during validation |
| **HTTPS enforcement** | HTTP auto-upgraded to HTTPS |
| **Cross-host redirect blocking** | Returns redirect info to model; model must re-fetch explicitly |
| **Same-host redirect cap** | Maximum 10 hops to prevent redirect loops |
| **Content size limit** | 10 MB max HTTP response |
| **URL length limit** | 2,000 characters |
| **Copyright protection** | 125-char quote limit for non-preapproved domains |
| **Egress proxy detection** | Recognizes `X-Proxy-Error: blocked-by-allowlist` headers |
| **Permission boundary** | User must approve each domain (unless preapproved) |
| **Sandbox separation** | Preapproved list is NOT shared with sandbox network allowlist |

---

## Limitations

- **No JavaScript execution** — Single-page apps (React, Angular, etc.) that require JS to render will return empty or skeleton HTML
- **No bot detection evasion** — Sites with Cloudflare, CAPTCHAs, or similar protections will block the request
- **No cookies or sessions** — Authenticated pages (Google Docs, Confluence, Jira, private GitHub repos) will fail
- **No rendering** — CSS, layout, and visual elements are ignored
- **10 MB size cap** — Large pages or files over 10 MB will fail
- **100K char truncation** — Content over 100,000 characters is truncated before Haiku processing
- **US-centric** — Domain blocklist API may have regional availability differences
- **Haiku quality ceiling** — The secondary model is fast but less capable than the main model; complex extraction may lose nuance

The tool's prompt explicitly advises the model to prefer MCP-provided fetch tools when available, as they may offer authenticated access or browser-based rendering.

---

## Source Files

| File | Purpose |
|---|---|
| `tools/WebFetchTool/WebFetchTool.ts` | Tool definition, permission check, call orchestration |
| `tools/WebFetchTool/utils.ts` | HTTP fetch, caching, redirect handling, Haiku processing |
| `tools/WebFetchTool/preapproved.ts` | Preapproved domain list and matching logic |
| `tools/WebFetchTool/prompt.ts` | Tool description and secondary model prompt template |
| `tools/WebFetchTool/UI.tsx` | Terminal UI rendering (progress, result display) |
