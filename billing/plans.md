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
| **CPU included per project** | 0.25 vCPU | 0.5 vCPU | 1 vCPU |
| **Memory included per project** | 0.25 GB | 0.5 GB | 1 GB |
| Build minutes/mo | 100 | 400 | 1,000 |
| Concurrent builds | 1 | 1 | 3 |
| Bandwidth/mo | 10 GB | 100 GB | 400 GB |
| Persistent disk max | 1 GB | 5 GB | 20 GB |
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

The **CPU included** and **Memory included** rows are the per-project compute baseline; you only pay metered overage above these amounts. See [Compute metering](#compute-metering) below.

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
| **CPU included per project** | 1 vCPU |
| **Memory included per project** | 1 GB |
| Build minutes/mo | 2,000 |
| Concurrent builds | 5 default, configurable |
| Bandwidth/mo | 1,000 GB |
| Persistent disk max | 50 GB |
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

Each plan includes a per-project baseline of CPU and memory at no extra charge (see the **CPU included** and **Memory included** rows in the plan tables). Anything you provision above that baseline is metered.

| Resource | Overage rate |
| -------- | ------------ |
| CPU above plan default | $4 / GB-month |
| Memory above plan default | $4 / GB-month |
| Persistent storage | $0.25 / GB-month (no plan default) |
| Bandwidth above plan | $0.25 / GB |
| Build minutes above plan | $0.002 / min |

### Choosing a compute size

When you create a project, you pick CPU and memory from sliders. Available sizes:

| Resource | Available sizes |
| -------- | --------------- |
| CPU | 0.5, 1, 2, 4, 8 vCPU |
| Memory | 0.5, 1, 1.5, 2, 4, 8, 12, 16 GB |

The smallest size you can pick (0.5 of each) is **higher** than what's included on Free and matches what's included on Hacker. So a Free-plan project at 0.5 vCPU / 0.5 GB pays metered overage on the 0.25 above its plan default for both. To stay fully inside Free's included compute, you'd need to provision below the slider's smallest stop, which the dashboard doesn't currently expose.

Database projects on Free are capped at 0.1 vCPU / 0.25 GB memory / 10 GB storage. The sliders are locked to those values; you can't pick a higher tier on Free for databases. Other plans get the full slider range for databases.

### How metering actually works

Compute is billed by the **GB-hour**. Brimble tracks each project's resources as a series of time segments. When you scale up CPU, change memory, or attach a disk, Brimble closes the current segment and opens a new one with the new values.

You're billed for the time at each configuration, not whichever was in effect at month-end. Scaling up halfway through the month means the second half is billed at the higher rate; the first half stays at the lower one. There's no rounding to whole hours.

The dashboard's **Billing → Usage** view shows the running cost for the current cycle so you can see what you'll owe before the invoice lands.

### Persistent storage

Persistent disks are billed by the GB-month at $0.25/GB at the base rate. Some regions carry a small multiplier; the actual monthly cost for each disk size is shown in the dropdown when you provision a disk.

The **Persistent disk max** in the plan tables is the largest disk you can provision on that plan. It's not "GB included free." Persistent disks meter from the first GB.

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

## Refunds

Refunds fall into two buckets.

**Automatic refunds** happen when Brimble couldn't deliver something you paid for:

* A domain purchase fails at the registrar, the charge is refunded automatically.
* A domain transfer-in fails or is rejected, the charge is refunded automatically.
* A renewal race produces a duplicate charge, Brimble issues the refund without you needing to ask.

These refunds go back to the original card via Stripe and you'll get a `Payment Reversed` email. The dashboard surfaces the reversal in **Billing → Invoices**.

**Manual refunds** for one-off charges (build-minute top-ups, accidental upgrades, anything else) go through support. Open a ticket with the transaction reference from **Billing → Invoices** and the reason. Eligibility:

* The charge must have settled successfully (pending or already-refunded charges can't be refunded again).
* You must be the user the charge was billed to.

Subscription fees behave differently from one-off charges; see [the FAQ](#faq) below for what happens when you downgrade or cancel mid-cycle.

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

## FAQ

**If I downgrade mid-cycle, do I get money back?** No. Downgrades are queued for the end of the current cycle. You keep the higher plan's features until the cycle ends, then drop to the lower one.

**If I upgrade mid-cycle, am I charged immediately?** No. Upgrades take effect right away (new features unlock, new metering rates apply going forward), but the new plan price is charged on the next billing date.

**Why was I charged more than the plan price this month?** Compute above plan defaults, persistent disks, bandwidth overage, or build-minute overage. Open **Billing → Invoices** and click the invoice; line items break the charge down by category.

**Can I get an invoice for accounting?** Every charge produces a Stripe invoice. They're all in **Billing → Invoices** with downloadable PDFs.

**Does Brimble add a markup on domain purchases?** Yes. The displayed price for a domain is the registrar's price plus a percentage charge from Brimble. The price you see in the buy flow is the final price you pay. Bringing an existing domain you already own (instead of buying through Brimble) doesn't trigger the markup, since you're not buying anything.

**Are there startup credits or promo codes?** The workspace-creation flow has a **Startup Promo Code** field, enter and verify the code before creating the workspace.

**What happens if I exhaust bandwidth mid-cycle?** On paid plans, you're billed $0.25/GB for everything above the included amount. On the free plan, projects start returning 503 once you've used your included amount, until the cycle resets.

**Are free-plan projects ever idled?** Yes. Free-plan, non-team projects that haven't received a request in 30 minutes are auto-paused. The next incoming request unpauses the project automatically. Team and paid-plan projects are not idled this way.

## Next steps

* [Build minutes](build-minutes.md), top-ups and the included allowance.
* [Persistent disk](../projects/persistent-disk.md), how disk pricing scales.
* [Project metrics](../observability/metrics.md), the live usage view per project.
