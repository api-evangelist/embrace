---
name: Investigate an Embrace crash to root cause
description: Take a crash from "it is the top crash" to a stack trace and a scoped blast radius, using the Embrace MCP crash and exception tools.
api: mcp/embrace-mcp.yml
surface: mcp
operations: [list_crashes, get_crash_details, get_crash_distribution, get_crash_stack_samples, list_exceptions, get_exception_details]
generated: '2026-08-12'
method: generated
source: https://embrace.io/docs/mcp/
---

# Investigate an Embrace crash to root cause

All six tools are published by Embrace at https://embrace.io/docs/mcp/. Connect and authenticate
exactly as in `embrace-daily-app-health-check.md`; `mcp:read` plus `mcp:tools:call` is enough.

## Steps

1. `list_crashes` — rank crashes by total occurrences and unique users affected. When the question
   is version-specific, filter with the `app_versions` parameter rather than ranking globally and
   eyeballing.
2. `get_crash_details` — scope of the crash group and which app versions it appears in. This tells
   you whether you are looking at a regression (new in a recent version) or a long tail.
3. `get_crash_distribution` — spread the crash group across one dimension (OS version, device
   model, country) *compared against session baselines*. The baseline comparison is the point: a
   crash that is 60% Android 14 is unremarkable if 60% of sessions are Android 14.
4. `get_crash_stack_samples` — fetch real stack traces by crash group ID. This is the only tool
   that returns code-level evidence; everything before it is triage.
5. If the failure surfaces as a handled exception rather than a crash, switch to `list_exceptions`
   and `get_exception_details`, which additionally return affected-user counts and history.

## Rules

- Stack traces only symbolicate if the build's symbols were uploaded. If frames come back as
  addresses or minified tokens, the answer is a build-pipeline gap, not an unknowable crash — see
  `cli/embrace-cli.yml` for `embtool`, `embrace-web-cli` and the symbol-upload CI actions.
- For web apps specifically, Embrace symbolicates function names only where the function was
  declared with the `function` keyword; functions assigned to a const or variable stay
  unsymbolicated. Do not read an unsymbolicated frame as a missing sourcemap without checking that.
- Do not diagnose from a distribution alone. Steps 3 and 4 answer different questions — who is
  affected, and why — and a root-cause claim needs both.
- Never restate a stack frame as a fix. Report the frame, the versions it appears in, and the
  scope; propose a fix only when the trace plus the distribution actually support one.

## Adjacent surface

Where a crash correlates with network failure, `list_network_domains`, `list_network_endpoints`
and `get_network_endpoint_errors` break errors down by HTTP status and connection error type.
Embrace's MCP server annotates connection errors with urgency ratings and plain-English
explanations that separate benign device-offline events from genuine failures — carry that
distinction through instead of counting every connection error as an incident.
