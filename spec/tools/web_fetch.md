# web_fetch

Fetch a URL and return its content as markdown.

## Description (shown to the model)

Fetch any URL and return its main content as markdown. Use this for reading articles, documentation, GitHub pages, blog posts, etc. For web search, fetch a search engine URL — DuckDuckGo's HTML version (`https://duckduckgo.com/html/?q=...`) works reliably without JavaScript.

For JavaScript-heavy sites (single-page apps, Google search results), the default `fetch` backend may return mostly empty content. If that happens, the user can configure a JS-rendering backend (`LOGOS_WEB_FETCH_BACKEND=jina` or `playwright`) — see the recipe.

Heavy / repeated fetching is best done via `delegate_task` with this tool, so the raw page contents don't accumulate in the main context.

## Input

```ts
{ url: string }
```

`url` must include a scheme (`http://` or `https://`). File URLs and other schemes are rejected.

## Output

```ts
{ ok: true, url: string, title: string, content_markdown: string, fetched_at: string }
| { ok: false, error: string }
```

- `url` is the final URL after any redirects.
- `title` is the page title (from `<title>` or first H1; empty string if none).
- `content_markdown` is the cleaned main-content as markdown, truncated to ~50 KB. The agent's per-message token guard handles anything that survives the truncation.
- `fetched_at` is an ISO-8601 timestamp.
- Errors include 4xx/5xx HTTP status, timeout, blocked-by-bot-detection, network error, etc.

## Backends

The recipe ships three backends, selectable via `LOGOS_WEB_FETCH_BACKEND`:

| Backend | Default? | JS-rendered | Local | Privacy | Setup |
|---------|----------|-------------|-------|---------|-------|
| **`fetch`** (default) | ✅ | ❌ | ✅ | Best — direct connection to target site | npm: `@mozilla/readability`, `jsdom` |
| **`jina`** | opt-in | ✅ | ❌ | URL + content sent to Jina (third party) | env: `JINA_API_KEY` (optional, higher rate limit) |
| **`playwright`** | opt-in | ✅ | ✅ | Best — local headless browser | npm: `playwright`, then `npx playwright install chromium` (~150 MB) |

### `fetch` (default)

- Use Node's built-in `fetch` to retrieve the HTML.
- Run the response through Mozilla Readability (with jsdom) to extract main content.
- Convert the extracted HTML to markdown using a small HTML-to-markdown library (e.g. `turndown`).
- Pros: zero third-party dependencies at runtime, direct connection (best privacy), fast, ~50 KB of npm deps.
- Cons: returns mostly empty content on heavily JavaScript-rendered sites (SPAs, Google search results, some news sites). Works fine for Wikipedia, GitHub HTML pages, most blogs and documentation, Hacker News, DuckDuckGo HTML version, etc.

### `jina` (opt-in)

- Construct `https://r.jina.ai/{url}` and fetch it.
- Jina's hosted service runs a headless browser, extracts main content, returns clean markdown.
- If `JINA_API_KEY` is set in `config/.env`, send it as a bearer token for higher rate limits.
- Pros: handles JS rendering, no local install, free tier.
- Cons: every URL you fetch is sent to Jina along with their server's view of the rendered content. See **Privacy** below.

### `playwright` (opt-in)

- Spawn a headless Chromium via Playwright, navigate to the URL, wait for the page to settle, extract the rendered HTML.
- Convert via Readability + turndown.
- Pros: full JS rendering, runs locally (best privacy), no third-party dependency.
- Cons: ~150 MB browser binary download, slower per-request (~1–3s), higher memory footprint while fetching.

## Privacy

The default `fetch` backend connects directly from your machine to the target URL — no third party involved. Privacy is whatever the target site's privacy is.

The `jina` backend routes through Jina AI's servers. They see:

- Every URL you fetch (logged with your IP, or with your account if you use an API key)
- The rendered content of every page
- Your fetch patterns over time (potentially building a profile of interests)

This is fine for general research but worth knowing if your assistant looks up health, financial, or other sensitive information. They don't see anything behind your auth (cookies aren't sent), so they can't access private content.

The `playwright` backend is fully local — same privacy as `fetch`, with JS support. Pay the install cost if privacy matters and you need JS rendering.

## Environment variables

- `LOGOS_WEB_FETCH` — `true` (default) or `false`. When false, the tool refuses every call with `web_fetch is disabled` and never appears as available.
- `LOGOS_WEB_FETCH_BACKEND` — `fetch` (default) | `jina` | `playwright`.
- `JINA_API_KEY` — optional, only used when backend is `jina`.
- `LOGOS_WEB_FETCH_TIMEOUT_MS` — optional, default 15000.

## Behavior

- Reject non-`http(s)` URLs.
- Apply the configured backend.
- Truncate `content_markdown` at 50 KB; append `\n…[truncated]` if cut.
- Catch any backend exception and return `{ ok: false, error: <message> }`. Don't throw.
- Include the final URL after redirects in the response so the agent knows what it actually got.

## Dependencies

Default (`fetch`) backend:
- `@mozilla/readability`
- `jsdom`
- `turndown` (HTML → markdown)

Plus the backend-specific deps when those backends are selected.

## Implementation notes

- Search via the agent: just fetch a search engine URL. DuckDuckGo HTML (`https://duckduckgo.com/html/?q=...`) works with the default `fetch` backend. Google search results need `jina` or `playwright`.
- Tool description should mention the search-via-fetch pattern with an example, so the agent doesn't waste a step asking how to search.
- Best paired with `delegate_task` for research tasks: spawn a sub-agent with `tools: ["web_fetch"]` so the raw fetched content stays in the sub-agent's context, not the main agent's.
