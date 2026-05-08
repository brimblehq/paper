# Regions

A **region** is the physical datacenter where your project runs. You pick a region when you create a project. Pick the one closest to your users.

## Why region matters

Latency between your users and the region they hit is the single biggest controllable factor in app responsiveness. A user in São Paulo hitting a project in Frankfurt pays ~200ms in network round-trip before your code does anything. If your users are in one place, deploy there.

Region also determines:

- Where the project's database and disk live.
- Which IP range the outbound traffic comes from (relevant if you whitelist Brimble at a third-party API).
- The price of storage. Storage cost is multiplied by a per-region factor; some regions are more expensive than others.

## Region availability

Brimble has regions across Africa, Europe, the Americas, and Asia. The full list is visible when you create a project.

Two tiers exist:

- **Free regions.** Available on the free plan. Limited selection.
- **Paid regions.** Available on Hacker, Pro, and Team plans. Wider selection, including regions optimized for specific traffic patterns.

The dashboard marks paid regions clearly during project creation. You can't deploy to a paid region from a free plan.

## Picking a region

Rules of thumb:

- **Single market.** Pick the region geographically closest to your users.
- **Multi-region traffic.** Pick the region closest to the largest user concentration. For two roughly-equal markets, weigh the latency cost on each side.
- **Database co-location.** Always put a project's database in the same region as the project. Cross-region database calls add round-trip latency to every query.
- **Compliance.** If you have data-residency requirements (GDPR, regional privacy laws), pick a region inside the relevant jurisdiction.

You can see the full list of regions and their codes in [Regions reference](../reference/regions.md).

## Switching regions

Region is set at project creation and isn't editable in place. To move a project to a different region:

1. Create a new project in the target region from the same repo.
2. Copy environment variables across (the dashboard has a copy action under the source project's **Environment** tab).
3. Move custom domains to the new project once it's healthy.
4. Delete the old project.

For projects with a managed database, follow [Migrate a database between regions](../guides/migrate-database-between-regions.md) — moving the data is a separate step.

## Region status

Each region in the dashboard shows one of:

- **Active** — accepting deployments normally.
- **Degraded** — accepting deployments, but performance may be reduced. Brimble posts an incident at [status.brimble.io](https://status.brimble.io).
- **Maintenance** — temporarily not accepting new deployments. Existing deployments continue serving.

If a region is in maintenance and you try to deploy, the deployment stays `pending` until the region is back, or you can move the project to a different region.

## Next steps

- [Regions reference](../reference/regions.md) — full list of region codes and tiers.
- [Plans and pricing](../reference/plans.md) — which regions each plan can use.
