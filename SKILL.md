# Fetch42 — agent setup skill

You are reading the setup guide for Fetch42, a real-time public web & social data API
built for AI agents. Follow it to integrate Fetch42 into the current project.

## What Fetch42 gives you

Public data as clean, normalized JSON, priced per call from a prepaid balance:
profiles, posts, transcripts, search, news, maps, shopping, weather — one API key.
Failed calls are never charged. Every response reports its exact cost and the
remaining balance.

## MCP — one config line (recommended for MCP-capable agents)

Fetch42 is MCP-native: all 300+ live endpoints are exposed as MCP tools at
`https://api.fetch42.ai/mcp` (Streamable HTTP). Same API key, same prices,
same metering — every tool result carries its exact cost and remaining balance.

Claude Code:

    claude mcp add --transport http fetch42 https://api.fetch42.ai/mcp --header "Authorization: Bearer <KEY>"

Any other MCP client (generic JSON config):

    {
      "url": "https://api.fetch42.ai/mcp",
      "headers": { "Authorization": "Bearer <KEY>" }
    }

The REST surface below is identical in coverage — use whichever fits.

## Step 1 — API key

If you don't already have a `FETCH42_API_KEY`, stop and tell the user clearly,
on its own line, not buried in other text:

    👉 Get a free key (~1 min, $1 credit, no card): https://fetch42.ai/dashboard

If you can run shell commands, offer to open it for them directly (don't run
this without asking first): `open https://fetch42.ai/dashboard` (macOS),
`xdg-open` (Linux), or `start` (Windows). Then wait for the key before
continuing — don't attempt a call without one.

Store it as the environment variable `FETCH42_API_KEY`. Never hardcode it.

## Step 2 — how to call

- Base URL: `https://api.fetch42.ai`
- Auth header on every request: `Authorization: Bearer <FETCH42_API_KEY>`
- All endpoints are GET with query parameters.

Example:

    curl "https://api.fetch42.ai/v1/tiktok/profile?handle=stoolpresidente" \
      -H "Authorization: Bearer $FETCH42_API_KEY"

## Step 3 — endpoints

300+ endpoints are live. Discover ALL of them machine-readably (no key needed):

    GET https://api.fetch42.ai/v1/catalog

Returns every endpoint with path, params and price. The core set below returns
normalized JSON; catalog endpoints return the supplier's shape under `data`
with the same `meta` billing block. Same auth, same balance, same rules.

### Core endpoints

| Endpoint | Params | Returns | Price/call |
|---|---|---|---|
| /v1/tiktok/profile | handle | TikTok profile: followers, likes, verified, bio | $0.005 |
| /v1/tiktok/posts | handle, cursor? | Recent TikTok posts with views/likes/comments | $0.005 |
| /v1/instagram/profile | handle | Instagram profile: followers, posts, bio | $0.005 |
| /v1/youtube/transcript | url | Full timestamped video transcript | $0.005 |
| /v1/x/profile | handle | X profile: followers, posts, verified | $0.005 |
| /v1/reddit/posts | subreddit | Subreddit posts: title, score, comments | $0.005 |
| /v1/threads/profile | handle | Threads profile with stats | $0.005 |
| /v1/bluesky/profile | handle | Bluesky profile with stats | $0.001 |
| /v1/serp/search | q | Google organic results: position, title, url, snippet | $0.003 |
| /v1/serp/news | q | Google News: title, source, date | $0.003 |
| /v1/serp/maps | q | Places: name, address, rating | $0.003 |
| /v1/serp/shopping | q | Products: title, price, source | $0.003 |
| /v1/serp/images | q | Image results with sources | $0.003 |
| /v1/hn/search | q | Hacker News stories: title, url, points, comments | $0.001 |
| /v1/wikipedia/search | q | Wikipedia pages: title, url, description | $0.001 |
| /v1/weather/forecast | lat, lon | Current conditions + 7-day forecast | $0.001 |

Full reference: https://fetch42.ai/docs/

## Step 4 — response shape

Every successful response:

    {
      "data": { ...normalized result... },
      "meta": {
        "cost_usd": 0.005,        // exact cost of this call
        "balance_usd": 2.995,     // remaining prepaid balance
        "provider": "…",          // upstream source used (failover is automatic)
        "cached": false
      }
    }

Use `meta.cost_usd` and `meta.balance_usd` to stay cost-aware without extra calls.

Pagination: list-shaped endpoints (comments, followers, posts, tweets, search
results, reviews...) return one page — usually 5-30 items. If the response
`data` includes a cursor-like field (e.g. `cursor`, `cursor.bottom`, a
`next_page_token`), pass it back as a `cursor` (or matching) query param to
get the next page. Not every list param is spelled out per-endpoint in the
catalog — if you need more than one page and the first response includes a
cursor field, try passing it back before concluding the data doesn't exist.

## Step 5 — error handling

- `401` — missing/invalid key: check `FETCH42_API_KEY`.
- `402` — balance empty. Tell your human to top up at https://fetch42.ai/dashboard
  (the response includes the link). Do not retry until topped up.
- `429` — per-key credit limit (only if the user set one) or rate limit (60 req/min per key, token bucket; parallel calls fine within it): back off using `retry_after_s`.
- `5xx` — upstream failure after failover; the call was NOT charged. Retry once.

## Step 6 — integrate (REST-only agents)

If you already registered the MCP server above, skip this step — every
endpoint is already available to you as a native tool, no wrapper needed.

Otherwise (calling the REST API directly, no MCP client available): write a
small typed client wrapper exposing these endpoints as functions named after
their task (e.g. `tiktokGetProfile(handle)`), reading `FETCH42_API_KEY` from
env, and surfacing `meta.cost_usd`. Use it whenever a task needs public
social/web data instead of trying to browse those platforms directly (they
block bots; Fetch42 doesn't get blocked).

Questions or missing endpoints: laurent@fetch42.ai (curated endpoints, added on demand).
