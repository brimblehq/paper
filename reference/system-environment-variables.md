# System environment variables

Brimble injects a small set of environment variables into every deployment, on top of the ones you configure. They're available at build time and at runtime.

## Variables Brimble sets

| Variable | Set when | Value |
|---|---|---|
| `PORT` | Runtime, web service and MCP server | The port your service must listen on. Brimble assigns it; never hardcode. |
| `BRIMBLE` | Always | `1` — present so your code can detect Brimble. |
| `BRIMBLE_REGION` | Always | The region code (e.g. `fra1`). |
| `BRIMBLE_PROJECT` | Always | The project name. |
| `BRIMBLE_PROJECT_ID` | Always | The numeric project ID. |
| `BRIMBLE_ENVIRONMENT` | Always | The active environment name (e.g. `Production`). |
| `BRIMBLE_DEPLOYMENT_ID` | Always | The deployment ID this container was built for. |
| `BRIMBLE_COMMIT_SHA` | Always | The Git commit SHA being deployed. |
| `BRIMBLE_BRANCH` | Always | The Git branch the deployment came from. |
| `NODE_ENV` | Build (Node only) | `production` by default. Override per-environment if needed. |
| `CI` | Build | `true` — many tools use this to suppress interactive prompts. |
| `BRIMBLE_BUILD` | Build only | `1` — distinguishes build runner from runtime container. |

## Reading them in code

```javascript
// Node
const port = process.env.PORT;
const region = process.env.BRIMBLE_REGION;
```

```python
# Python
import os
port = int(os.environ["PORT"])
region = os.environ.get("BRIMBLE_REGION")
```

```go
// Go
import "os"
port := os.Getenv("PORT")
```

## `PORT` rules

For web services and MCP servers, `PORT` is mandatory:

- Read it. Don't hardcode.
- Listen on `0.0.0.0`, not `localhost` or `127.0.0.1` — Brimble's edge can't reach localhost-only listeners.

A common boot pattern:

```javascript
const port = Number(process.env.PORT) || 3000;
app.listen(port, "0.0.0.0", () => console.log(`listening on ${port}`));
```

The `|| 3000` fallback only matters when running locally; in production `PORT` is always set.

## What you can't override

System variables can't be overridden by your environment configuration. If you set `BRIMBLE_REGION` as a custom env var, the system value still wins.

`PORT` is the exception — you cannot set it; Brimble controls it. Setting `PORT` in your env config will be ignored at runtime.

## Distinguishing build from runtime

`BRIMBLE_BUILD=1` is set during the build runner only. Use it to gate code that should only run during a build:

```javascript
if (process.env.BRIMBLE_BUILD === "1") {
  // generate static config, prerender pages, etc.
}
```

At runtime, `BRIMBLE_BUILD` is unset.

## Detecting Brimble

If your code runs both locally and on Brimble:

```javascript
const onBrimble = !!process.env.BRIMBLE;
```

Use this to switch between dev-only behavior (verbose logging, hot reload) and production behavior.

## Per-deployment identifiers

`BRIMBLE_DEPLOYMENT_ID` and `BRIMBLE_COMMIT_SHA` change with every deployment. Useful for:

- Sentry / error reporting `release` field.
- Log correlation.
- A `/version` endpoint that exposes which build is running.

```javascript
app.get("/version", (req, res) => {
  res.json({
    deployment: process.env.BRIMBLE_DEPLOYMENT_ID,
    commit: process.env.BRIMBLE_COMMIT_SHA,
    region: process.env.BRIMBLE_REGION
  });
});
```
