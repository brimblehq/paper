# Regions

A region is the datacenter where your project runs. Pick one when you create a project.

## Region tiers

| Tier | Available on | Description |
|---|---|---|
| **Free** | Free plan and up | Limited selection. Suitable for side projects and testing. |
| **Paid** | Hacker, Pro, Team | Wider selection across continents. Better latency for production workloads. |

The dashboard shows available regions when you create a project. Free-plan projects can only deploy to free regions; upgrading the plan unlocks paid regions.

## Regions

Brimble has regions across:

- **Africa** — including major hubs in West and East Africa.
- **Europe** — including Frankfurt, Helsinki, and Nuremberg.
- **North America** — including Ashburn (US-East) and Hillsboro (US-West).
- **South America** — including São Paulo.
- **Asia** — including Singapore and Tokyo.

Region codes follow the pattern `<city>-<sequence>` (e.g. `fra1`, `nyc1`, `lhr1`). The dashboard shows the city, country, and region code side by side.

## Picking a region

Three things matter:

1. **User proximity.** Pick the region closest to your users. Cross-continent traffic adds 100–300ms of latency before your app responds.
2. **Co-location.** Put a project's database in the same region as the project that uses it.
3. **Compliance.** If you have data-residency requirements, pick a region in the relevant jurisdiction.

The dashboard pre-fills a default region based on your account location. Override if your users are elsewhere.

## Region status

Each region is in one of three states:

- **Active** — accepting new deployments.
- **Degraded** — accepting deployments, but performance may be reduced. See [status.brimble.io](https://status.brimble.io) for active incidents.
- **Maintenance** — temporarily not accepting new deployments. Existing deployments continue running.

If a region is in maintenance and you create a project there, the first deployment stays `pending` until the region is back. You can move to a different region instead.

## Switching a project's region

Region is set at project creation. To move a project, create a new project in the target region from the same repo, copy environment variables, attach domains, and delete the old project. Database projects need a separate data migration.

## Storage pricing per region

Storage cost is multiplied by a per-region factor. Most regions are at the base rate; a few high-cost regions carry a small multiplier. The exact rate per region appears during project creation and on the project's billing page.

## Outbound IP ranges

Outbound traffic from your project comes from a region-specific IP range. If you're whitelisting Brimble at a third-party API, request the current ranges from support — they aren't fixed forever.
