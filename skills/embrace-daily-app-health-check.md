---
name: Embrace daily app health check
description: Read an Embrace app's overall health for a period — session volume, crash-free rate, top versions and any new crashes — using the Embrace MCP server.
api: mcp/embrace-mcp.yml
surface: mcp
operations: [list_apps, get_app_details, get_top_versions, list_crashes]
generated: '2026-08-12'
method: generated
source: https://embrace.io/docs/mcp/
---

# Embrace daily app health check

Every tool name below is published by Embrace at https://embrace.io/docs/mcp/. Embrace ships no
OpenAPI document, so these are MCP tools rather than REST operationIds. Do not invent tools or
parameters: the live `inputSchema` for each tool is auth-gated (anonymous `tools/list` returns
401), so pass only arguments you have confirmed from a successful authenticated `tools/list`.

## Connect

- Endpoint: `https://mcp.embrace.io/mcp` (Streamable HTTP).
- Auth, option A — OAuth 2.0 authorization code with PKCE (S256) against
  `https://dash-api.embrace.io`. Best for an interactive session; the token inherits your own
  Embrace app permissions.
- Auth, option B — a service-account bearer token, `Authorization: Bearer emb_sa_…`. It must carry
  `mcp:tools:call`, plus `mcp:read` for everything in this skill. Use this in CI or any headless
  agent. Read it from `EMBRACE_API_TOKEN`; never inline it.

## Steps

1. `list_apps` — resolve the app you care about and its Embrace App ID (a 5-character identifier).
   Do this first even when you think you know the ID; a service account may be scoped to an
   explicit app list and silently see fewer apps than a human would.
2. `get_app_details` — pull health metrics, crash-free rate and session counts. Pass `time_window`
   to set the period. To describe a trend rather than a snapshot, call it twice with different
   `time_window` values and compare, rather than asserting a direction from one reading.
3. `get_top_versions` — order app versions by session count. Report health against the versions
   that actually carry traffic; a catastrophic crash rate on a version with 12 sessions is noise.
4. `list_crashes` — top crashes ranked by occurrences and unique users affected. Flag anything
   that was not present in the previous run.

## Rules

- Never present a rate without its denominator. `get_app_details` returns session counts; carry
  them into the summary so a 99.2% crash-free rate reads as "99.2% of 41,000 sessions".
- Watch data freshness before you claim "today". Embrace publishes its own lag: five-minute
  metrics land ~4 minutes late, hourly ~15 minutes, and daily ~14 hours. A daily figure for
  00:00 is not readable until 14:00. If the window you were asked about is inside that lag, say so.
- Embrace retains events for 3 days on Free, 14 on Pro and 30 on Enterprise. A question about
  last month may be unanswerable on the account's plan — that is an answer, not a failure.
- The MCP surface is read-oriented. If a request implies a change to Embrace configuration, stop:
  all 21 published tools are `list_`/`get_`.

## Errors

Embrace publishes no error catalog for the MCP surface. Treat an HTTP 401 as an auth problem, not
an empty result — an expired or unscoped token looks identical to "no data" if you do not check.
See `errors/embrace-problem-types.yml`.
