# Plans and pricing

Brimble has four plans. Pick one based on how many projects you run, how much traffic you serve, and how many concurrent builds you need.

## Plan summary

| Plan | Monthly | Projects | Bandwidth | Concurrent builds | Regions |
|---|---|---|---|---|---|
| **Free** | $0 | 5 | 10 GB | 0 (queued only) | Free regions |
| **Hacker** | $7 | 10 | 30 GB | 1 | All |
| **Pro** | $19 | 150 | 150 GB | 2 | All |
| **Team** | Variable | 500 | 500 GB | Variable | All |

The team plan is metered per member ($5/member/month) plus per concurrent build ($7.50/build/month). A 5-person team needing 3 concurrent builds is $5 × 5 + $7.50 × 3 = $47.50/month.

## What's included on every plan

- Automatic HTTPS with Let's Encrypt for default and custom domains.
- Git-triggered deploys.
- Real-time logs.
- Webhooks (Hacker and above).
- Managed databases (provisioned separately, billed by size).
- Custom domains.
- Brimble's authoritative DNS for managed domains.

## Compute and storage

CPU, memory, and storage are billed per project, on top of the plan price. Each plan includes a default amount; usage above the default is metered:

| Resource | Plan default included | Overage rate |
|---|---|---|
| **CPU** | Plan-specific | $4 / GB-month |
| **Memory** | Plan-specific | $4 / GB-month |
| **Storage** | None included | $0.25 / GB-month |
| **Bandwidth** | Plan limit | $0.25 / GB |
| **Build minutes** | Plan limit | $0.002 / minute |

Storage is always metered — no included amount on any plan. CPU and memory are included up to a plan-specific cap, then billed.

## Free plan limits

The free plan has hard limits that don't overflow:

- **5 projects max.** Trying to create a 6th prompts an upgrade.
- **10 GB bandwidth.** Once exhausted, projects return 503 until the cycle resets.
- **No concurrent builds.** Builds queue and run one at a time, indefinitely.
- **Free regions only.** A subset of the full region list.
- **No webhooks.**

## Trial

New accounts come with a 14-day free trial of paid features. After the trial, if you haven't added a payment method, the account drops to the free plan and projects exceeding free limits are paused (not deleted) until you upgrade.

## Payment

Payment methods supported:

- **Credit/debit card** via Stripe (US, EU, most countries).
- **Card** via Paystack (African markets).

You can hold up to 3 cards on file. One is the default; you can change it in **Billing → Payment methods**. Card management opens at the provider's hosted form — Brimble doesn't see card numbers.

## Billing cycle

Plans bill monthly. Compute, storage, bandwidth, and build-minute overages from the previous month are added to the current month's invoice.

If a charge fails, Brimble retries the card. After **7 days** of failed retries, builds are disabled (existing deployments keep serving). After **14 days**, the subscription deactivates and projects on it pause.

To prevent disruption, keep a card on file with sufficient credit and headroom on the limit.

## Pause vs delete

- A **paused** project is offline (returns 503) but retained. Re-activating it brings it back without rebuild.
- A **deleted** project is gone, including its data. Databases are deleted with the project. Custom domains detach.

Pausing happens automatically when payment fails for too long, or manually via **Settings → Pause project**. Deletion is always explicit and requires 2FA for projects on a paid plan.

## Switching plans

Upgrade in **Billing → Plan**. Upgrades take effect immediately and prorate the remainder of the current cycle.

Downgrades take effect at the end of the current cycle, so you don't get refunded for time already paid. If the new plan's limits are below your current usage (more projects than the new cap, more concurrent builds than allowed), you'll need to bring usage under the limit before the downgrade applies.

## Personal vs team billing

A user account can be on one personal plan, which covers projects under that user. Each team has its own plan. A user can be in many teams, each on a different plan, and the team's plan covers projects under that team.

Personal usage and team usage are billed separately on separate invoices.

## Viewing usage

In **Billing → Usage**, the dashboard shows the current cycle's:

- Bandwidth used / limit.
- Build minutes used / limit.
- CPU-hours, memory-hours, storage GB-hours per project.
- Estimated overage, if any.

Refreshes every few minutes. The final invoice number is computed at cycle end.

## Pricing changes

When pricing changes, current customers are notified via email and dashboard banner with at least 30 days' notice. Existing subscriptions stay at their current rate until the change takes effect.
