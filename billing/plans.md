# Plans and pricing

Brimble pricing has three pieces:

1. A **base plan price** you pay every month for the plan's features and included quotas.
2. **Metered overage** for any compute, storage, or bandwidth above what your plan includes.
3. Optional **build minute top-ups** when you burn through your monthly allowance.

There's no hidden line. Every charge above the base plan price is a unit price times an amount, and the dashboard shows what's included, what's metered, and what you've used.

## Plans

There are four plans. Free, Hacker, and Pro are personal plans, one per user. Team is a separate workspace plan with its own billing, sized by member count and concurrent builds.

### Personal plans

| | Free | Hacker | Pro |
| - | ---- | ------ | --- |
| Base price | $0/mo | $5/mo | $15/mo |
| Projects | 3 | 10 | Unlimited |
| Build minutes/mo | 100 | 400 | 1,000 |
| Concurrent builds | 1 | 1 | 3 |
| Bandwidth/mo | 10 GB | 100 GB | 400 GB |
| Storage included | 1 GB | 5 GB | 20 GB |
| Log retention | 1 day | 7 days | 14 days |
| Custom domain | No | Yes | Yes |
| Webhooks | No | No | Yes |
| Autoscaling | No | No | Yes |
| Web analytics | No | No | Yes |
| PR previews | No | Yes | Yes |
| Org Git deploy | No | Yes | Yes |
| Database CPU max | 0.1 vCPU | Unlimited | Unlimited |
| Database memory max | 0.25 GB | Unlimited | Unlimited |
| Database storage max | 10 GB | Unlimited | Unlimited |
| Support | Community | Email | Priority |

### Team plan

Team is dynamic. The base price comes from how the workspace is configured:

```
monthly = (members × $5) + (concurrent_builds × $8)
```

A 5-member workspace with 2 concurrent builds is `5 × $5 + 2 × $8 = $41/mo`. Adjust either lever and the cost recalculates. Both are editable from **Workspace settings → Billing**.

Team specs:

| | Team |
| - | ---- |
| Projects | Unlimited |
| Build minutes/mo | 2,000 |
| Concurrent builds | 5 default, configurable |
| Bandwidth/mo | 1,000 GB |
| Storage included | 50 GB |
| Log retention | 30 days |
| Custom domain | Yes |
| Webhooks | Yes |
| Autoscaling | Yes |
| Web analytics | Yes |
| PR previews | Yes |
| Support | Dedicated |

A user can be on a personal plan and be a member of one or more teams at the same time. Personal subscriptions and team subscriptions are billed independently.

### Plan name mapping

The dashboard says **Pro**; the API and webhooks say `DEVELOPER_PLAN`. They're the same thing.

| Dashboard | API value |
| --------- | --------- |
| Free | `FREE_PLAN` |
| Hacker | `HACKER_PLAN` |
| Pro | `DEVELOPER_PLAN` |
| Team | `TEAM_PLAN` |

## Compute metering

Plans include a baseline of CPU and memory at no extra charge. Anything you provision above that baseline is metered.

| Resource | Plan default included | Overage rate |
| -------- | --------------------- | ------------ |
| CPU | Free 0.25, Hacker 0.5, Pro 1, Team 1 (vCPU equivalents) | $4 / GB-month above default |
| Memory | Free 0.25, Hacker 0.5, Pro 1, Team 1 (GB) | $4 / GB-month above default |
| Persistent storage | None included for persistent disks | $0.25 / GB-month |
| Bandwidth | Per the table above | $0.25 / GB above plan |
| Build minutes | Per the table above | $0.002 / min above plan |

### How metering actually works

Compute is billed by the **GB-hour**. Brimble tracks a project's resources as a sequence of segments, each with a start time, an end time, and the resources allocated during that segment. Two examples:

**A web service running at 0.5 GB CPU and 1 GB memory on the Hacker plan, all month.** Hacker default is 0.5 GB CPU and 0.5 GB memory.

```
CPU excess     = max(0.5 - 0.5, 0) = 0 GB → $0
Memory excess  = max(1.0 - 0.5, 0) = 0.5 GB
Memory cost    = 0.5 GB × $4 / GB-month = $2.00
Total compute  = $2.00 for the month
```

**The same service scaled up to 2 GB memory at the 15th of the month.** Brimble closes the first segment (`0.5 / 1.0` for ~360 hours) and opens a new one (`0.5 / 2.0` for ~360 hours).

```
First half:    0.5 GB excess × ($4 / 720h) × 360h ≈ $1.00
Second half:   1.5 GB excess × ($4 / 720h) × 360h ≈ $3.00
Total memory:  $4.00 for the month
```

This is exactly how Brimble computes it under the hood. There's no rounding to whole hours, and no surprise floor.

### Persistent storage

Persistent disks are billed by the GB-month, every GB, no plan default. A 50 GB disk costs `50 × $0.25 = $12.50/mo` at the base rate.

The base rate is multiplied by a **regional storage factor** for some regions. Most regions are at the base; a few high-cost regions are higher. The actual rate per region is shown in the disk size dropdown when you provision a disk, for example `50 GB ($12.50/month)`.

The `storage` quota in the plan table above is the **maximum disk size** you can provision on that plan. It's not "GB included free." Storage you provision always meters from byte one.

### Bandwidth

Bandwidth is your project's total outbound traffic from Brimble's edge for the cycle. Up to your plan's included bandwidth is free. Above that, you're charged $0.25/GB.

You can see usage on the home page's **Bandwidth** tile, and the per-project breakdown on each project's **Observability** tab under Network Egress.

### Build minutes

Each plan includes a monthly build-minute allowance. A build minute is wall-clock time inside the build runner, from clone start to push end. See [Builds](../projects/builds.md) for what counts and what doesn't.

When you exhaust the allowance:

* **Free plan:** new builds queue indefinitely until the cycle resets.
* **Paid plans:** overage bills at $0.002/min and rolls into the next invoice. You can also top up with credits that never expire, see [Build minutes](build-minutes.md).

## Cycles, prorating, and changes

* **Billing cycle.** Monthly. The cycle starts on the date you first subscribed.
* **Upgrades.** Take effect immediately. The new plan and new features apply right away. You're not charged extra immediately; the new price kicks in on the next billing date. Compute meters keep accumulating against your usage, billed at cycle end.
* **Downgrades.** Don't take effect immediately. The current plan and features stay until the end of the current cycle. After the cycle, the lower limits apply. If you have more projects than the new plan allows, they aren't deleted; you keep them but can't create new ones over the limit.
* **Compute scaling within a cycle.** Resource changes (scaling up CPU, adding a disk, switching regions) are tracked as separate segments. You're billed for the time at each configuration, not whichever was in effect at the end of the month.

## Payment retries

Brimble retries failed charges automatically. The escalation:

1. **Days 1 to 7 after a failed charge:** Brimble retries the card. Builds and runtime keep working.
2. **After 7 days of failures:** new builds are disabled. Existing deployments keep serving.
3. **After 14 days (or 14 attempts):** the subscription deactivates. Projects on it are paused.

To recover, add a working card under **Billing → Payment methods** and click **Retry**. Once payment goes through, builds and projects come back online.

You can hold up to three cards on file; one is the default. Cards are added through redirects to the payment provider's hosted form (Stripe in most regions, Paystack for African markets); Brimble never sees card numbers.

## Currency

Pricing is set in USD. Some regions display equivalent local-currency amounts on the checkout page (set by the payment provider), but the underlying amounts are USD.

## Where to see usage

| What you want | Where it lives |
| ------------- | -------------- |
| Current cycle's bandwidth, build minutes, compute estimate | **Billing** in workspace or account settings |
| Per-project compute usage in real time | The project's **Observability** tab |
| Workspace-wide bandwidth chart | Home page → **Bandwidth** tile |
| Past invoices and Stripe receipts | **Billing → Invoices** |
| Pending downgrade or plan change | **Billing → Plan** |

## Changing plans

1. Open the dashboard.
2. Go to **Billing → Plan**.
3. Pick the new plan. The page shows the price difference and what changes.
4. Confirm.

Upgrades go through immediately. Downgrades are queued for the end of the current cycle. You can cancel a queued downgrade before it takes effect.

## Workspace billing vs. personal billing

Personal plans cover projects under your personal workspace. Team plans cover projects under their team workspace. The two are independent invoices. Compute, bandwidth, and build-minute usage on a team's projects bill against the team subscription, never your personal one.

## Pricing changes

When pricing changes, current customers are notified via email and dashboard banner with at least 30 days' notice. Existing subscriptions stay at their current rate until the change takes effect.

## Next steps

* [Build minutes](build-minutes.md), top-ups and the included allowance.
* [Persistent disk](../projects/persistent-disk.md), how disk pricing scales.
* [Project metrics](../observability/metrics.md), the live usage view per project.
