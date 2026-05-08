# Manage environment variables

Set, edit, and delete environment variables for a project. Variables are injected into both the build runner and the running container.

## Prerequisites

- A project deployed on Brimble. If you haven't deployed yet, follow the [quickstart](../getting-started/quickstart.md).

## Add a variable

In the dashboard:

1. Open the project.
2. Go to **Environment** in the project sidebar.
3. Pick the environment (Production, Preview, or any custom one) from the dropdown.
4. Click **Add variable**.
5. Enter the name and value. Click **Save**.

Or, from the API:

```bash
curl -X POST https://api.brimble.io/v1/envs/<project-id>/Production \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"name": "DATABASE_URL", "value": "postgres://user:pass@host:5432/db"}'
```

The variable is stored encrypted. The next deployment picks it up.

## Add many variables at once

For pasting in a `.env` file:

1. Open **Environment**.
2. Click **Bulk import**.
3. Paste the contents of your `.env` file. Lines starting with `#` are treated as comments.
4. Click **Import**.

Each line of the form `KEY=value` becomes a variable. Quoted values are unquoted. Values with `=` or whitespace inside should be quoted with single or double quotes.

## Apply changes

Editing variables doesn't redeploy your project automatically. To pick up changes:

- **For new deployments.** Push a commit, or click **Redeploy**.
- **For the current deployment.** There's no in-place env update for a running container. You must redeploy.

This is intentional — environment changes are tied to a specific build, so you always know which version of your code is running with which config.

## Edit a variable

In the dashboard, click the variable name in the **Environment** list. Update the value and save. Or from the API:

```bash
curl -X PATCH https://api.brimble.io/v1/envs/<project-id>/<env-var-id> \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"value": "new-value"}'
```

## Delete a variable

```bash
curl -X DELETE https://api.brimble.io/v1/envs/<project-id>/<env-var-id> \
  -H "Authorization: Bearer <token>"
```

Or use the trash icon next to the variable in the dashboard.

## Inherit variables from another environment

A child environment can inherit variables from a parent. This is useful when you want Preview to share most of Production's config but override a few values.

1. Open **Environments** (separate tab from variables — `https://app.brimble.io/projects/<id>/environments`).
2. Edit the environment you want to inherit into (e.g. `Preview`).
3. Set **Inherit from** to the parent (e.g. `Production`).
4. Save.

Only variables marked `inheritable` flow through. Toggle the inheritable flag on individual variables in the parent environment.

If a variable is set on both the child and the parent, the child's value wins.

## Use variables in your code

Variables show up as standard process environment variables. Examples:

```javascript
// Node
const dbUrl = process.env.DATABASE_URL;
```

```python
# Python
import os
db_url = os.environ["DATABASE_URL"]
```

```go
// Go
import "os"
dbUrl := os.Getenv("DATABASE_URL")
```

```ruby
# Ruby
db_url = ENV.fetch("DATABASE_URL")
```

## Use variables in build commands

Variables are also injected into the build runner. You can reference them in install scripts, build scripts, or `package.json`:

```json
{
  "scripts": {
    "build": "NODE_ENV=$NODE_ENV next build"
  }
}
```

For private package registries, set the auth token as a variable and reference it in `.npmrc` or `pip.conf`:

```
//registry.npmjs.org/:_authToken=${NPM_TOKEN}
```

## Verification

After redeploying, log a non-secret variable on startup:

```javascript
console.log("Boot config:", {
  hasDb: !!process.env.DATABASE_URL,
  region: process.env.BRIMBLE_REGION
});
```

Don't log the value of a secret. Logs are visible to anyone with project access.

## Troubleshooting

**Variable shows in dashboard but not in `process.env`.** You haven't redeployed since adding it. Click **Redeploy** on the latest deployment.

**Variable is set in Production but not in Preview.** Variables are scoped per environment. Either set it in Preview directly, or enable inheritance from Production.

**Special characters in value get mangled.** When pasting via bulk import, wrap values containing `=`, `#`, or whitespace in quotes: `KEY="value with spaces"`.

**Build fails reading a variable that's set.** The variable might be missing the `inheritable` flag if you're using inheritance. Check the parent environment.

## Next steps

- [Environments concept](../concepts/environments.md) — how environments and inheritance work.
- [System environment variables](../reference/system-environment-variables.md) — variables Brimble injects automatically.
