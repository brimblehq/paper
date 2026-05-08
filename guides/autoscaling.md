# Configure autoscaling

Run more than one container for a project, with bounds you control. Useful for traffic that varies by time of day or unpredictable spikes.

By default, a Brimble project runs as a single container. Autoscaling groups let you run more than one and bound how high they can scale.

## Prerequisites

- A project on a paid plan that includes autoscaling.
- The project must be a web service or worker. Static sites don't need scaling (they serve from the edge), and databases scale separately.

## How it works

You attach an **autoscaling group** to a project. The group sets:

- **Minimum containers** — the floor. Brimble runs at least this many at all times.
- **Maximum containers** — the ceiling. Brimble runs at most this many.
- **Max CPU per container** — the upper bound on CPU each container can use.
- **Max memory per container** — the upper bound on memory each container can use.
- **Active** — whether the group is running. Toggle off to freeze scaling without losing the configuration.

When a group is active, Brimble keeps the running container count between `min_containers` and `max_containers`. Containers are added when load on existing ones is high enough to justify them and removed when there's slack. The current container count is tracked as the group's `replicas` field.

## Create an autoscaling group

1. Open the project.
2. Go to **Scaling**.
3. Click **Add autoscaling group**.
4. Set:
   - **Name** — a label so you can identify the group later.
   - **Minimum containers**.
   - **Maximum containers**.
   - **Max CPU per container** (e.g. `1` for one vCPU).
   - **Max memory per container** in MB.
5. Save.

The group activates immediately. Container count adjusts on the next scaling cycle.

## Recommended starting bounds

For a typical web service:

- Minimum: 2 (so a single container failure doesn't take you offline)
- Maximum: 10
- Max CPU: matches your project's compute size
- Max memory: matches your project's compute size

Adjust as you see how your traffic actually behaves. Most teams start narrower than they think they need and widen the cap once they have data.

## Workers

For background workers, autoscaling adjusts the number of consumer containers. If you scale to four containers, four copies of your worker run at once — each consuming from the same queue. Make sure your queue protocol distributes work safely across consumers (most do).

If your worker maintains in-process state that shouldn't be duplicated (a singleton scheduler, for example), don't autoscale — set `min_containers = max_containers = 1`.

## Edit a group

In **Scaling**, click the group, edit values, save. Changes take effect on the next scaling cycle.

## Pause or remove a group

Toggle **Active** off on the group to pause without losing the configuration. The current container count freezes; future cycles don't scale.

To remove the group entirely, click **Delete**. Containers stay at whatever the count was when you deleted; future scaling stops, and you manage the project as a single-container deployment again.

## Verification

Open **Scaling** and watch the **Replicas** field on the group. Generate some load against the project (with `wrk`, `vegeta`, or just real traffic) and observe whether the count climbs toward your maximum.

Once load drops, the count returns toward your minimum after a stabilization period.

## Troubleshooting

**Containers never scale up.** Either load isn't actually crossing the scaling threshold, or the group is paused. Check **Active** is on, then look at observability metrics — if CPU/memory never approach the per-container ceilings, scaling won't trigger.

**Containers scale up and down repeatedly (flapping).** Your minimum and maximum are too close, or the per-container ceilings are too tight relative to traffic. Widen the bounds.

**Container count stays at the maximum even when traffic drops.** Some traffic patterns (long-lived WebSocket connections, server-sent events) prevent containers from being safely terminated. Brimble drains connections before stopping a container; long-lived connections delay this. This is intentional — it prevents dropping users.

**Workers scale up but only one is doing work.** Your consumer protocol isn't distributing across containers. Check that your queue client uses a consumer-group pattern (Redis BLPOP, RabbitMQ work queues, BullMQ workers — these all distribute correctly).

## Next steps

- [Deployments](../concepts/deployments.md) — how new containers come online and pass health checks.
- [Plans and pricing](../reference/plans.md) — billing impact of additional containers.
