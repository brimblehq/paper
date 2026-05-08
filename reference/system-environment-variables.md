# System environment variables

Brimble injects a small set of environment variables into every deployment, on top of the ones you configure. They're available at build time and at runtime.

## Variables Brimble sets

| Variable | Set when | Value |
|---|---|---|
| `PORT` | Runtime, web service and MCP server | The port your service must listen on. Brimble assigns it at deploy time; never hardcode. |
| `NODE_ENV` | Build (Node only) | `production` by default for production deploys. Override per-environment if needed. |
| `CI` | Build | `true` — many tools use this to suppress interactive prompts. |

## Reading them in code

```javascript
// Node
const port = Number(process.env.PORT);
```

```python
# Python
import os
port = int(os.environ["PORT"])
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

`PORT` is controlled by Brimble — you can't set it, and any value you put in your env config is ignored at runtime.

## Detecting environment

If you need to switch behavior between local development and a deployed environment, use a variable you set yourself (for example, an `APP_ENV` you set per-environment in the dashboard). Brimble doesn't inject a generic "running on Brimble" flag.

For framework-specific conventions (`NODE_ENV`, `RAILS_ENV`, etc.), set those in the project's environment under **Environment**.
