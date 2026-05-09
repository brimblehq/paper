# Deploy a worker

Run a long-lived background process, queue consumer, scheduler, message handler, without exposing an HTTP port.

## Prerequisites

- A Git repository with the worker code.
- A Brimble account.

A worker is just a process that runs forever. It doesn't listen on a port, and Brimble won't probe it for an HTTP health check. If your script exits, Brimble restarts it.

## Step 1: Create the worker project

1. Open the dashboard.
2. Click **New project**.
3. Connect the repository and pick a branch.
4. Under **Service type**, choose **Worker**.

{% hint style="info" %}
**Image needed:** screenshot of the new-project flow with service type set to "Worker" and the Git source selected
{% endhint %}

## Step 2: Configure the build

A worker uses the same build pipeline as a web service. Brimble auto-detects the framework. Override the build and start commands if needed.

| Setting | Example |
|---|---|
| **Build command** | `npm run build` (omit if no build needed) |
| **Start command** | `node dist/worker.js` |
| **Region** | Same region as the queue or database the worker talks to. |
| **Compute size** | Match to your worker's memory profile. |

Workers don't need `PORT` and don't expose a public URL.

## Step 3: Set environment variables

Workers need credentials to talk to whatever they consume. Common ones:

- `DATABASE_URL`, primary database connection string.
- `REDIS_URL`, queue or cache connection string.
- `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`, for SQS, S3, etc.

Add these under **Environment** before deploying. See [Manage environment variables](environment-variables.md).

## Step 4: Deploy

Click **Deploy**. The build phase runs the same as a web service. Once the artifact is built, Brimble starts the container with your start command.

## Verification

A healthy worker is one whose process is still running. Check the dashboard:

- **Status: Active** means the process is alive.
- **Status: Failed** means the process exited and couldn't be restarted (after retries).

To verify the worker is doing work, log a heartbeat or processed-count to your logs:

```javascript
setInterval(() => {
  console.log(`processed ${count} jobs in the last 60s`);
  count = 0;
}, 60_000);
```

Then watch the runtime logs in the dashboard.

## Restart behavior

If the worker process exits, clean exit or crash, Brimble restarts it. After repeated rapid restarts (5 in 60 seconds), the deployment is marked **Failed** and stops being restarted automatically. This is to prevent infinite crash loops from churning resources.

To force a restart, click **Restart** on the deployment.

## Scheduled jobs

For cron-style jobs (run every hour, run at 3 AM nightly), you have two options:

1. **In-process scheduler.** Use a library like `node-cron`, `APScheduler`, or `whenever` inside your worker. The worker runs continuously; the library triggers your job on schedule.
2. **Separate cron service**, a forthcoming option will let you define cron jobs declaratively. For now, use the in-process pattern.

## Multiple workers

By default a worker runs as a single container. To run more than one in parallel, configure an autoscaling group on the project, see [Autoscaling](autoscaling.md). Each container runs the same image with the same env vars; your code is responsible for distributing work safely across them, typically by consuming from a shared queue with a consumer-group pattern.

If your worker keeps singleton state in memory (a scheduler, an in-process lock), don't scale past one container. Set the autoscaling group's minimum and maximum to 1, or skip autoscaling entirely.

## Logs

Workers stream logs the same as web services. Open **Logs** on the project to see live output. Anything written to stdout or stderr lands in the logs.

## Troubleshooting

**Worker starts and immediately exits.** Your start command runs to completion. Workers must run forever, usually that means consuming from a queue or sleeping inside an event loop. If your code does its work and returns, wrap it in a loop or scheduler.

**Worker crashes silently.** Make sure your runtime catches and logs unhandled exceptions:

```javascript
process.on("unhandledRejection", (err) => console.error("unhandled:", err));
process.on("uncaughtException", (err) => console.error("uncaught:", err));
```

**Restarts every few seconds.** The process is exiting fast enough to trigger the rapid-restart limit. Look at the last few log lines before each restart, there's almost always an error there.

**Worker can't connect to the database.** Make sure the worker's region matches the database's region, and use the private connection string for in-region traffic.

## Next steps

- [Deployments](deployments.md), how restarts and health checks work.
- [Deploy a database](deploy-a-database.md), provision the queue or DB the worker reads from.
