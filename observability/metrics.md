# Project metrics

Every Brimble project has an **Observability** tab that shows CPU, memory, network, and response-time metrics in real time. Use it to size your project, debug latency spikes, and watch traffic patterns.

This is separate from [web analytics](../analytics/web-analytics.md), which counts visitors and pageviews on the front end. Observability metrics are server-side — what your container is doing.

## Open the Observability tab

1. In the dashboard, open the project.
2. Click **Observability**.

The tab loads with metrics for the last hour by default.

![TODO: screenshot of the Observability tab on a web-service project showing two semi-gauges (CPU, Memory) at the top and a time-series chart below with tabs for Memory Usage, CPU Usage, Network Egress, and Response Times](./images/PLACEHOLDER.png)

*The Observability tab on a project.*

## Top: live gauges

Two semi-circular gauges show the **current** snapshot:

- **CPU usage** — percentage of allocated CPU in use, with the allocation in vCPU shown alongside (e.g. `42.3% — 1.0 vCPU`).
- **Memory usage** — percentage of allocated memory in use, with the allocation in GB (e.g. `61.8% / 2.0 GB`).

These are point-in-time readings. The chart below shows how they've moved over your selected window.

## Bottom: time-series chart

Pick a metric from the tabs above the chart:

| Metric | Unit | Description |
|---|---|---|
| **Memory Usage** | % | Memory consumption as a percentage of the project's allocated memory. |
| **CPU Usage** | % | CPU consumption as a percentage of the project's allocated CPU. |
| **Network Egress** | KB/s | Outbound network throughput from your container. |
| **Response Times** | ms | Latency percentiles for HTTP responses. Toggle between **P90**, **P95**, **P99**, and **Average**. Not available on database projects. |

The chart's X-axis is time; the Y-axis is the metric. Hover any point to see the exact value at that timestamp.

### Response Time percentiles

For web services and MCP servers, the Response Times tab is the most useful one for catching tail latency:

- **P90** — 90% of responses were faster than this. Useful for "typical worst case."
- **P95** — 95% of responses were faster. Use for SLOs.
- **P99** — 99% of responses were faster. Most actionable signal for tail latency — if P99 is bad, a meaningful slice of users are seeing it.
- **Average** — sum of all response times divided by request count. Easy to read but hides outliers; prefer percentiles for real analysis.

Toggle between them with the **P90 / P95 / P99 / Average** segmented control on the right side of the chart header.

## Time range

The dropdown in the top right of the tab picks the time window:

- Last 1 Hour (default)
- Last 6 Hours
- Last 24 Hours
- Last 7 Days
- Last 30 Days

Switching the range refreshes the chart and the gauges. Longer ranges aggregate more — minute-level resolution at 1 hour, hourly aggregates at 7+ days.

## What the metrics tell you

A few common patterns and what they usually mean:

- **CPU pinned at 100%** — your service is CPU-bound. Either optimize hot paths, scale up the project's compute size, or add containers via [scaling](../scaling/overview.md).
- **Memory climbing steadily over hours/days** — memory leak. Restart the deployment to confirm (the saw-tooth on restart proves the leak), then find what's holding allocations.
- **Memory near 100%** — likely getting OOM-killed periodically. Check runtime logs for restart loops; bump the project's memory size or fix the allocation pattern.
- **Network egress spiky** — traffic comes in bursts. Normal for batch jobs and scheduled work; check the timing matches your scheduler.
- **Network egress flat-high** — sustained high outbound traffic. Investigate whether you're proxying large responses, leaking data, or a CDN/cache is bypassed.
- **P99 response time much higher than P90** — a small slice of requests are slow. Usually a slow downstream (database query, third-party API) on specific paths. Cross-reference with request logs.
- **Response times rising during peak hours** — you're at capacity. Scale or optimize.

## Database projects

Database projects show **Memory Usage**, **CPU Usage**, and **Network Egress** — the same gauges and the same chart. They don't have **Response Times** because Brimble doesn't time individual queries from the platform side; for query-level performance, use the database engine's native tools (e.g. `pg_stat_statements` for PostgreSQL).

## Workspace bandwidth

Per-project network egress is on the Observability tab. The **workspace's total bandwidth** is on the home page in the **Bandwidth** tile, with a small chart showing the usage trend across the current cycle.

Bandwidth is what's billed against your plan's allowance — see [Plans and pricing](../billing/plans.md) for the limits and overage rate.

## Refresh and freshness

Metrics auto-refresh when you change the time range. They don't auto-tick within a fixed window — to pull the latest second-by-second data, switch ranges (e.g. flip from 1 Hour to 6 Hours and back).

There's no manual refresh button on the Observability tab. The data behind it has a few-seconds lag from the source.

## What's not shown here

A few things observability doesn't cover, intentionally:

- **Request count / RPS over time.** For request volume, the request logs tab on the project shows individual requests with timestamps; aggregate from there.
- **Per-endpoint metrics.** All metrics are project-aggregate. To see latency on a specific endpoint, instrument your code (Sentry, OpenTelemetry, custom metrics).
- **Error rate.** Inferable from request logs (filter on 5xx) but not surfaced as a chart on Observability.

For deeper application instrumentation, ship telemetry to a third-party service (Sentry, Datadog, Grafana Cloud, etc.) — Brimble's metrics are about platform health, not full APM.

## Troubleshooting

**Gauges show 0% with traffic.** The metrics pipeline lags by a few seconds for fresh deployments — wait a minute and refresh. If still zero after a few minutes, runtime metrics may be unavailable temporarily; check [status.brimble.io](https://status.brimble.io).

**Response Times tab is missing.** You're on a database, worker, or static-site project. Response times only apply to projects that handle HTTP requests (web service, MCP server).

**Numbers look wrong vs. my own monitoring.** Brimble's CPU and memory percentages are relative to the **allocation** you picked under **Settings → Resources**, not the host's total capacity. If your container's `top` shows 30% but Brimble shows 90%, your container is using 90% of what Brimble gave it.

## Next steps

- [Configure scaling](../scaling/overview.md) — when CPU or memory hit ceiling.
- [Web analytics](../analytics/web-analytics.md) — visitor-facing analytics, separate from these metrics.
- [Plans and pricing](../billing/plans.md) — bandwidth allowance and overage.
