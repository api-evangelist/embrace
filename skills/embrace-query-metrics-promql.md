---
name: Query the Embrace Metrics API with PromQL
description: Pull Embrace session, crash and custom metrics over the Prometheus HTTP API, with the right host, granularity and freshness window.
api: embrace-metrics-api
surface: rest
base_url: https://api.embrace.io/metrics
generated: '2026-08-12'
method: generated
source: https://embrace.io/docs/metrics-forwarding/metrics-api/
---

# Query the Embrace Metrics API with PromQL

Embrace publishes no OpenAPI for this API; it is the Prometheus HTTP API (`/api/v1`) behind a
bearer token, so any PromQL client works. Embrace's own samples use `prometheus-query` (Node) and
`prometheus-api-client` (Python). Every metric name and query below is copied from Embrace's
documentation — do not guess metric names, enumerate them instead (see step 2).

## Connect

- Base URL: `https://api.embrace.io/metrics`, with `/api/v1` as the client's base path.
- Regional data residency changes the HOST, not a header or path: US accounts use
  `https://api-us1.embrace.io/metrics`, EU accounts `https://api-eu1.embrace.io/metrics`. A client
  pointed at the default host for a residency-enabled org will not silently fall back.
- Auth: `Authorization: Bearer <Metrics API token>`, taken from the Embrace dashboard at
  Settings → Organization → API → "Metrics API". This is a different token from the Custom Metrics
  API token and from the Symbol Upload token.

## Steps

1. Choose the granularity by choosing the metric name prefix. Embrace encodes granularity in the
   name: `five_minute_*`, `hourly_*`, `daily_*`. There is no `step`-driven rollup — asking for a
   five-minute step on an `hourly_` metric does not produce five-minute data.
2. Enumerate what is queryable before composing anything: `labelValues('__name__', …)` in Node, or
   `prom.all_metrics()` in Python. This is the honest substitute for a spec.
3. Scope the query by app:
   - one app — `sum(hourly_sessions_total{app_id="a1b2C3"})`
   - several apps — `sum(hourly_sessions_total{app_id=~"a1b2C3|Z9Y8x7"})` (pipe-delimited regex)
   - every app in the org — omit `app_id` entirely: `sum(hourly_custom_metric_sessions_total{})`
4. Group with `by(...)` on the dimensions the metric supports, e.g.
   `sum(daily_sessions_total{app_id="…",app_version="1.2.3"}) by (city, state)`.
5. Compute rates yourself. Embrace deliberately removed rate metrics because a rate is not
   composable — the documented replacement for `daily_crash_free_session_rate` is
   `1 - daily_crashes_total / daily_sessions_total`. Crash-free percentage by device is
   `1 - sum(hourly_crashes_total{app_id="$app_id"}) by (device_model) / sum(hourly_sessions_total{app_id="$app_id"}) by (device_model) * 100`.

## Rules

- Respect the freshness lag before reporting a window as complete: five-minute data lands ~4
  minutes after the point is calculated, hourly ~15 minutes, daily ~14 hours. A daily point for
  `2026-08-12 00:00` is not readable until `2026-08-12 14:00`.
- Steps smaller than one hour are rounded up to an hour. A 5-minute step on a 24-hour range does
  not return 288 points.
- Anything with a `_deprecated` suffix is gone — deprecated October 2023, removed April 2026. If a
  saved query still references one it returns nothing; rewrite it from the Replacements table at
  https://embrace.io/docs/metrics-forwarding/metrics-api/metrics-deprecated/.
- No rate-limit headers are published on this API. Do not build a backoff loop that reads
  `Retry-After` or `X-RateLimit-Remaining`; neither is returned. Pace on the freshness windows above.

## Creating a custom metric

Custom metrics are created through a separate API, `https://api.embrace.io/custom-metrics`, with
its own token issued by an Embrace onboarding specialist — it is not self-service. Once created,
the metric becomes queryable here. Status codes and the error envelope for that API are in
`errors/embrace-problem-types.yml`; the filter tree shape and the `duration_bucket` first-match
gotcha are in `conventions/embrace-conventions.yml`.
