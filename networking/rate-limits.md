# Rate limits

The hard limits Brimble's edge enforces on incoming traffic. These are infrastructure-level guardrails, they protect your service from abusive single-source traffic and don't depend on your project's compute capacity.

## Public traffic

| Limit | Value |
|---|---|
| **Requests** | 500 per 10-minute window, per IP, per domain |
| **Body size** | 5 MB per request |
| **Header size** | 8 KB |
| **URL length** | 8 KB |

Requests that exceed the request rate get `429 Too Many Requests` until the window rolls. Over-size bodies get `413 Payload Too Large`.

## Internal traffic

Requests from internal Brimble IPs (the dashboard, the API, your other Brimble projects) are exempt from public rate limits. This means service-to-service calls between your Brimble projects don't count against each other's limit.

## Build minutes

A separate limit governs build runners:

| Plan | Build minutes / month | Concurrent builds |
|---|---|---|
| Free | 100 | 0 (queued) |
| Hacker | 500 | 1 |
| Pro | 2,000 | 2 |
| Team | 5,000 + 1,000 / extra build | Variable |

Once monthly build minutes are exhausted on a paid plan, overage bills at $0.002/minute and rolls into the next invoice. On the free plan, builds queue indefinitely until the cycle resets.

## Webhook delivery

Brimble retries failed webhook deliveries with exponential backoff:

| Retry | Wait |
|---|---|
| 1 | 1 minute |
| 2 | 5 minutes |
| 3 | 30 minutes |
| 4 | 2 hours |
| 5 | 12 hours |

After 5 failed retries, the delivery is dropped. The webhook is not disabled; future events still try to deliver.

Each delivery has a **10-second timeout**. Your endpoint must return a response within 10 seconds.

## Sensitive operations

A few operations require step-up 2FA before running:

- Delete a project.
- Delete a custom domain.
- Rotate a database password.
- Transfer a domain out of Brimble.
- Transfer team ownership.

The 2FA prompt allows 5 wrong attempts per challenge, then locks for a cooling-off period.

## Higher limits

For workloads that need more than the default public traffic limit (high-traffic public APIs, large file ingest, etc.), contact support. Custom rate limit policies, custom body size limits, and per-project carve-outs are available on Pro and Team plans.

## What you can't change in the dashboard

The current dashboard surfaces the rate-limit thresholds and overage rates but doesn't let you raise them in place. Raise them through support or by upgrading your plan, depending on which limit you're hitting.
