# Roll back a deployment

If a new deployment breaks production, roll back to a previous one. Rollbacks reuse the previous artifact, no rebuild, and finish in seconds.

## Prerequisites

- A project with at least two completed deployments.
- The previous deployment must have been **Active** at some point, Brimble can only redeploy artifacts that finished building successfully.

## Roll back

1. Open the project.
2. Click **Deployment history**.
3. Find the deployment you want to restore. Each row shows the commit SHA, the message, the timestamp, and the status.
4. Click **⋯** → **Redeploy this version**.
5. Confirm. Brimble re-runs that deployment's image and flips traffic to it within ~30 seconds.

{% hint style="info" %}
**Image needed:** screenshot of the Deployment history page showing several deployment rows with status chips, commit messages, branches, deployer avatars, timestamps, and the row-level "⋯" menu open with the "Redeploy this version" option highlighted
{% endhint %}

The rolled-back deployment becomes the new active one. Your deployment history adds a new row pointing at the same commit SHA.

## What carries over and what doesn't

A rollback restores the **code artifact only**. It does not roll back:

- **Environment variables.** If you changed a variable since the old deployment, the rollback uses the current value, not the historical one.
- **Database schema.** If a migration ran on the new deployment, the rolled-back code may be incompatible. Either roll back the migration manually, or write migrations that are forward- and backward-compatible.
- **External state.** Side effects (emails sent, payments processed, files written) stay in place.

For migrations specifically, prefer additive changes: add columns before reading them, never drop a column in the same release that stops writing to it. This keeps rollbacks safe.

## Verification

After the rollback completes, hit your project's URL or health endpoint:

```bash
curl -I https://<project-name>.brimble.app/healthz
```

The response includes `X-Brimble-Project-Version`, an ISO timestamp identifying the deployment serving the request. Compare it to the deployment timestamp in the dashboard to confirm the rollback took effect.

## Troubleshooting

**"Redeploy this version" is greyed out.** The deployment never finished building, or the artifact has been pruned. Pick a more recent successful deployment.

**The rollback failed to start.** Open the new deployment row. The logs show the same phases as a normal deployment; usually the failure is in the deploy phase, not the build, since there's no rebuild. Check the runtime logs for startup errors.

**Traffic still goes to the broken version.** Edge routing flips on health-check pass. If the rolled-back deployment is failing its health check, the edge keeps the previous one. Check **Logs** for startup errors and compare to a known-good deployment.

**The old version doesn't run with the current database schema.** A migration ran on the broken deployment and the old code can't read the new schema. Roll the migration back manually, or write a forward-fix in a new deployment.

## Cancel a stuck deployment first

If a deployment is hanging in `pending` or `in progress` and you want to roll back without waiting for it to fail, click **Cancel** on the deployment in the dashboard. Once cancelled, you can roll back to the previous active deployment.

## Next steps

- [Deployments](deployments.md), full lifecycle and what each state means.
- [Build failures](../troubleshooting/build-failures.md), diagnose why the new deployment broke.
