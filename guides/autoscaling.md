# Configure autoscaling

Scale a project's replica count automatically based on load. Useful for traffic that varies by time of day or unpredictable spikes.

## Prerequisites

- A project on a paid plan.
- The project must be a web service or worker (autoscaling is not available for static sites or databases).

## How it works

You create an **autoscaling group** for a project. The group has:

- A **minimum** and **maximum** replica count.
- A **scaling strategy** (linear, exponential, or target).
- A **metric threshold** (CPU%, memory%, or request count).

Brimble polls the metric, compares it to the threshold, and adjusts replica count.

## Strategies

| Strategy | Behavior |
|---|---|
| **Linear** | Add or remove one replica at a time when the threshold is crossed. Predictable, slow to react to large spikes. |
| **Exponential** | Double the replicas when scaling up; halve when scaling down. Reacts fast, may overshoot. |
| **Target** | Add or remove replicas to keep the metric at a target value. Best for steady-state services with known capacity per replica. |

Pick **Target** if you've measured how much traffic one replica can handle. Pick **Linear** otherwise.

## Create an autoscaling group

1. Open the project.
2. Go to **Scaling**.
3. Click **Add autoscaling group**.
4. Set:
   - **Minimum replicas** — the floor. Don't go below this even when idle.
   - **Maximum replicas** — the ceiling. Don't go above this even under heavy load.
   - **Strategy** — linear, exponential, or target.
   - **Metric** — CPU%, memory%, or requests per second.
   - **Scale-up threshold** — when the metric exceeds this, add replicas.
   - **Scale-down threshold** — when the metric is below this, remove replicas.
   - **Cooldown** — minimum time between scaling actions, in seconds. Prevents thrashing.
5. Save.

The group activates immediately. The next metric poll triggers a scaling action if thresholds are met.

## Recommended settings

For a typical web service:

- Min: 2 (so a single replica failure doesn't take you offline)
- Max: 10
- Strategy: Linear
- Metric: CPU%
- Scale-up: 70%
- Scale-down: 30%
- Cooldown: 120 seconds

Adjust as you learn what your service's CPU profile looks like under real traffic.

## Workers

For workers, the relevant metric is usually queue depth or the work-rate of the queue you're consuming from. Brimble's built-in metrics are infrastructure-level (CPU, memory) — to scale on queue depth, expose a custom metric or use the platform's `requests` proxy with an internal endpoint that reports work backlog.

## Edit a group

Open **Scaling**, click the group, edit values, save. Changes take effect on the next metric poll.

## Pause or remove a group

To temporarily pause autoscaling without deleting the configuration, toggle **Active** off on the group. The current replica count freezes; future metric polls don't trigger actions.

To remove the group entirely, click **Delete**. The replica count stays at whatever it was — autoscaling stops, you manage replicas manually.

## Verification

After enabling autoscaling, generate load (with `wrk`, `vegeta`, or similar) above your scale-up threshold and watch:

```bash
wrk -t4 -c200 -d2m https://<project-name>.brimble.app/
```

Open **Scaling** in the dashboard. You should see the replica count rise within a poll cycle (typically 30–60 seconds). After load stops, replicas drop back to the minimum after the cooldown.

The **Activity** tab on the autoscaling group logs every scaling action with timestamp, trigger metric value, and replica count before/after.

## Troubleshooting

**Replicas never scale up.** Check the metric is actually crossing your threshold. Open **Observability** for the project — if CPU never crosses 70%, scaling never triggers. Either lower the threshold or pick a different metric.

**Replicas scale up and down repeatedly (flapping).** Your scale-up and scale-down thresholds are too close. Widen the gap. With CPU, try scale-up at 70% and scale-down at 30% rather than 60/40.

**Replicas hit the maximum and stay there.** Either your traffic genuinely exceeds capacity at the cap, or the metric stays elevated even after replicas are added (a downstream bottleneck). Investigate before raising the cap.

**Scale-down happens but old replicas keep serving.** Brimble drains connections from a replica before terminating it. Long-lived connections (WebSockets, server-sent events) keep the replica running until they close. This is intentional.

## Next steps

- [Deployments](../concepts/deployments.md) — how new replicas come online and pass health checks.
- [Plans and pricing](../reference/plans.md) — billing impact of additional replicas.
