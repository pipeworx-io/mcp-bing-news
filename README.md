# @pipeworx/bing-news

Keyword news search via Bing News RSS — a third, independent news source
(different vendor and infrastructure from Google News/`country-news` and
GDELT/`gdelt`), returning recent matching articles ranked by Bing with the
real article link, publish time, source publication and a short description.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `bing_news_search(query, limit?)` — searches Bing News for `query` and
  returns up to `limit` (1-50, default 20) matching articles: `title`,
  `link` (the real destination article URL, not Bing's tracking redirect),
  `description`, `published` (RFC-822), `source` (the publication name).

## Auth

Keyless. No registration, no API key, no rate-limit documented by Bing for
this endpoint.

## Data sources

- `https://www.bing.com/news/search?q=<query>&format=rss` — Bing's own news
  search, RSS output. Not a documented/versioned API — it is the same feed a
  browser gets from the "News" tab's RSS link, so its shape can change
  without notice the way any scraped HTML/XML can.

Rediscoverable-the-hard-way notes:

- **The `link` element is a tracking-redirect wrapper, not the article URL.**
  Bing wraps every result as `https://www.bing.com/news/apiclick.aspx?ref=
  ...&url=` followed by the percent-encoded real article URL and more
  tracking params. The real destination is the value of that `url` query
  param, and it is present and decodable on every item observed (verified
  live 2026-10-09 against 5 search results — msn.com, finance.yahoo.com,
  pulse2.com). `realArticleUrl()` decodes it; this pack returns the decoded
  URL as `link`, never the apiclick wrapper. If Bing ever omits the `url`
  param, the code falls back to the raw link rather than dropping the item.
- **The source element is namespaced.** It is `News:Source`, not a plain
  `source` element — easy to miss if you copy a generic RSS-item parser from
  another pack (country-news's Google News feed has no equivalent element at
  all; it comes from a different field).
- **A 200 is not proof of a real result.** Like GDELT and Google News, a
  CAPTCHA/anti-bot challenge page can come back with HTTP 200 and
  `content-type: application/xml` or `text/html` instead of the RSS feed —
  guarded here by checking for a literal `<item` tag before parsing, so that
  case surfaces as a loud failure rather than a silent empty result.
- **Cloudflare Workers egress is NOT blocked for this endpoint** — edge-probed
  live 2026-10-09 from a throwaway `wrangler dev --remote` Worker on the prod
  account: `bing.com/news/search` answered the Worker's own egress HTTP 200,
  10,393 bytes, well-formed RSS. Unlike `country-news` (Google News blocks CF
  egress outright), this pack's direct fetch is not currently routed through
  the shared Supabase egress relay — the relay-fallback code exists (same
  `_proxyUrl`/`_proxyToken` convention as `country-news`/`gdelt`) but nothing
  in the gateway injects those args for this pack's slug, because nothing
  has needed to yet. Re-probe before assuming that's still true months later.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "bing-news": {
      "url": "https://gateway.pipeworx.io/bing-news/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/bing-news/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1856+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/bing_news_search \
  -H 'Content-Type: application/json' \
  -d '{"query":"\"Series A\" funding","limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/bing_news_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-bing-news` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/bing-news/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "bing-news": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-bing-news"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Bing News data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
