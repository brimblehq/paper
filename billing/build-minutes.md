# Build minutes

Every Brimble plan includes a monthly allowance of build minutes, the clock time spent in the build runner across all of your projects. When the allowance runs out, you can top up with credits that never expire.

## How build minutes are counted

A build minute is one minute of wall-clock time inside the build runner, measured from the start of the clone phase to the end of the push phase. It includes:

- Cloning your repo.
- Detecting the framework.
- Installing dependencies.
- Running your build command.
- Pre-start commands (if configured).
- Pushing the artifact to internal storage.

It **does not** include:

- Time spent serving requests (your runtime is billed separately as compute).
- Time the build is queued waiting for a free slot.
- Cancelled builds (if cancelled before the build phase starts; cancelled mid-build still counts what ran).

## Where to see your usage

In the dashboard, build minutes are visible in two places:

1. **Account settings → Billing**, the **Build minutes** card shows used vs. included for the current cycle, with a progress bar and the next reset date.
2. **Home page**, the stats row at the top has a **Build minutes** widget with the same number; click it to jump to the billing page.

The card looks like:

```
Build minutes                            [ Top up ]
1,247 minutes left · resets Jun 5

Used this period          1,253 / 2,500 min
███████████░░░░░░░░░░░
+ 500 minutes in top-up credits
```

{% hint style="info" %}
**Image needed:** screenshot of the Build minutes card in Billing settings showing the used/included progress bar, "minutes left", reset date, top-up credits line, and the "Top up" button
{% endhint %}

## Cycles and resets

Each plan has a monthly cycle. Your **included minutes** reset to the plan's allowance on the cycle's renewal date. The reset date is shown on the card.

**Top-up credits do not reset**, they sit on top of the included minutes and stick around month over month until they're spent.

The order Brimble bills against:

1. Included minutes (this month's allowance).
2. Top-up credits (additive, never expire).

So a fresh cycle uses your monthly allowance first, then dips into credits if needed.

## Top up

When you're running low (or just want a buffer), top up:

1. Open **Billing → Build minutes**.
2. Click **Top up**.
3. Pick a preset amount or enter a custom value:
   - Presets: `$5`, `$10`, `$25`, `$50`.
   - Minimum custom amount: **$5**.
4. The modal previews how many minutes the amount buys at the current rate.
5. Confirm. The card on file is charged immediately, and the minutes appear as **top-up credits** on your account.

{% hint style="info" %}
**Image needed:** screenshot of the "Top up build minutes" modal showing the preset dropdown, custom amount field with $ prefix, "X minutes for $Y" preview line, and the "Pay with •••• 4242" summary
{% endhint %}

The conversion rate (minutes per dollar) is set globally and visible in the modal, "X minutes per $1 · credits never expire."

## What happens when you run out

When used minutes equal included minutes plus credits:

- **Free plan.** New builds queue indefinitely until the next cycle resets your included minutes. There's no overage billing on free.
- **Paid plans.** Builds **don't fail** mid-flight, running builds finish. New builds start to consume top-up credits if you have any. If credits are exhausted, builds queue until the next reset or the next top-up.

A "Builds disabled" state can also show up if billing failed for an extended period, that's separate from running out of minutes. See [Plans and pricing](plans.md).

## Estimating your usage

Typical builds:

- **Static site (Vite, Astro, simple Next.js export).** 30 seconds to 2 minutes.
- **Node web service with cache hit.** 30 seconds to 1 minute.
- **Node web service cold.** 1 to 4 minutes.
- **Large Next.js / monorepo build.** 3 to 10 minutes.
- **Python with native deps.** 1 to 5 minutes (cold), 30 seconds (cached).
- **Java / Gradle full build.** 3 to 8 minutes.

Multiply by your number of deploys per day across all projects. A team that pushes 20 times a day across 5 services typically lands somewhere between 200 and 1,500 minutes per month.

## Reduce your build time

A few cheap wins:

- **Don't disable the build cache.** It's on by default; turning it off triples cold-build times.
- **Use a lockfile.** `npm ci`, `pip --require-hashes`, `pnpm --frozen-lockfile` skip dependency resolution.
- **Set watch paths in monorepos.** Pushes that don't touch the project's directory don't trigger a build.
- **Multistage Dockerfiles.** Build in a heavy stage; run from a thin stage. The `docker build` cache reuses layers when only the source changes.
- **Skip unnecessary work in CI/build flows.** Don't run tests in the build phase, run them in your CI before merge instead.

## Troubleshooting

**"Add a payment method before topping up."** Add a card under **Billing → Payment methods**, then retry.

**Top-up confirmed but credits not visible.** Refresh the page. If still not visible after a minute, the payment is still being verified, Stripe sometimes takes a moment for SCA-protected cards. Check **Billing → Invoices** for the transaction status.

**My usage looks higher than expected.** Failed builds still count for the time they ran before failing. Likewise builds that pass health checks but stay in `in progress` until timeout. Check **Deployment history** for any unusually long deployments and investigate why they took that long.

## Next steps

- [Plans and pricing](plans.md), what each plan includes.
- [Builds](../projects/builds.md), what runs during a build, and how the cache works.
