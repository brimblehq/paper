# Regions

A **region** is the physical datacenter where your project runs. You pick a region when you create a project. Pick the one closest to your users.

## Why region matters

Latency between your users and the region they hit is the single biggest controllable factor in app responsiveness. A user in São Paulo hitting a project in Frankfurt pays ~200ms in network round-trip before your code does anything. If your users are in one place, deploy there.

Region also determines:

- Where the project's database and disk live.
- Which IP range outbound traffic comes from (relevant if you whitelist Brimble at a third-party API).
- The price of storage. Storage cost is multiplied by a per-region factor.

## Region tiers

| Tier | Available on | Description |
|---|---|---|
| **Free** | Free plan and up | Limited selection. Suitable for side projects and testing. |
| **Paid** | Hacker, Pro, Team | Wider selection across continents. |

The dashboard marks paid regions clearly during project creation. You can't deploy to a paid region from the free plan.

## Available regions

Brimble has regions across:

- **Africa** — major hubs in West and East Africa.
- **Europe** — including Frankfurt, Helsinki, and Nuremberg.
- **North America** — including Ashburn (US-East) and Hillsboro (US-West).
- **South America** — including São Paulo.
- **Asia** — including Singapore and Tokyo.

Region codes follow the pattern `<city><sequence>` (e.g. `fra1`, `nyc1`, `lhr1`). The dashboard shows the city, country, and region code side by side when you create a project.

## Picking a region

Rules of thumb:

- **Single market.** Pick the region geographically closest to your users.
- **Multi-region traffic.** Pick the region closest to the largest user concentration. For two roughly-equal markets, weigh the latency cost on each side.
- **Database co-location.** Always put a project's database in the same region as the project that uses it. Cross-region database calls add round-trip latency to every query.
- **Compliance.** If you have data-residency requirements (GDPR, regional privacy laws), pick a region inside the relevant jurisdiction.

The dashboard pre-fills a default region based on your account location. Override if your users are elsewhere.

## Region status

Each region is in one of three states:

- **Active** — accepting new deployments.
- **Degraded** — accepting deployments, but performance may be reduced. See [status.brimble.io](https://status.brimble.io) for active incidents.
- **Maintenance** — temporarily not accepting new deployments. Existing deployments continue serving.

If a region is in maintenance and you create a project there, the first deployment stays `pending` until the region is back. You can move to a different region instead.

## Switching a project's region

Region is set at project creation and isn't editable in place. To move a project:

1. Create a new project in the target region from the same repo.
2. Copy environment variables across (the dashboard has a copy action under the source project's **Environment** tab).
3. Move custom domains to the new project once it's healthy.
4. Delete the old project.

Database projects need a separate data migration — open a support ticket if you need to move a managed database between regions.

## Storage pricing per region

Storage cost is multiplied by a per-region factor. Most regions are at the base rate; a few high-cost regions carry a small multiplier. The exact rate per region appears during project creation and on the project's billing page.

## Outbound IP ranges

Outbound traffic from your project comes from a region-specific IP range. If you're whitelisting Brimble at a third-party API, request the current ranges from support — they aren't fixed forever.
