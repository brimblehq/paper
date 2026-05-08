# Environments

An **environment** is a named set of configuration that a project deploys with. Brimble gives every project a `Production` environment by default. You can create more — typically `Preview`, `Staging`, or one per developer — and each holds its own environment variables.

## Why environments exist

Most apps need different configuration in different contexts. A staging build talks to a staging database; production talks to production. An engineer testing locally points at a sandbox API key. Without environments, you'd swap env vars by hand on every deploy.

In Brimble, an environment carries:

- A **name** (`Production`, `Preview`, `staging`, anything you choose).
- A set of **environment variables**.
- Optionally, a **parent environment** to inherit variables from.

Deployments are always tied to one environment. The variables for that environment are injected into the build and the runtime container.

## Production and Preview

Two environments are always available:

- **Production.** The default. Variables here apply to every build of the tracked branch.
- **Preview.** Optional. Applied to deployments built from pull requests, so you can vet changes against staging credentials before merging.

A pull request opened against the tracked branch builds with the Preview environment's variables. Once merged, the next build uses Production.

You can create additional environments — for example, a `staging` environment tied to a specific branch — under **Environments** in the dashboard.

## Environment variables

Each environment holds its own set of key/value pairs. Inside a deployment, they're injected into the runtime container the same way any Linux process sees env vars: through `process.env`, `os.environ`, `ENV[...]`, or whatever your language uses.

A variable set in one environment is **not** visible in another unless you opt in via inheritance.

| Property | Meaning |
|---|---|
| **Name** | The variable name. Convention: uppercase with underscores. |
| **Value** | The variable value. Stored encrypted. |
| **Environment** | Which environment this value belongs to. |
| **Inheritable** | Whether child environments can inherit this value. |

System variables Brimble injects automatically (like `PORT`) live in a separate namespace and override nothing. See [System environment variables](system-variables.md).

## Inheritance

A child environment can inherit variables from a parent. A common pattern:

```
Production  (DB_URL=prod, STRIPE_KEY=live, LOG_LEVEL=info)
└── Preview (inherits from Production, overrides DB_URL=staging)
```

When you create or edit an environment, you can pick a parent. Variables from the parent flow through unless you set the same key on the child — child values always win. Only variables marked `inheritable` are passed down; secrets you want to keep production-only stay there.

To set up inheritance:

1. Open **Environments** in the project sidebar.
2. Edit the child environment.
3. Under **Inherit from**, pick the parent.
4. Save. The next deployment uses the merged set.

## Build-time vs runtime

Environment variables are visible **both** at build time and at runtime. If your build script needs a token to fetch a private package, set it in the environment — it'll be available during `npm install` and during your start command.

If you specifically don't want a value present at build time (for example, secrets you only want available to the running container), there's no separate runtime-only namespace. Instead, fetch the secret at runtime from a vault or secret store using a token you set as a build-time variable.

## Editing variables

You can add, edit, or delete variables in the dashboard under **Environment** on the project page. Editing a variable does **not** redeploy automatically — the new value applies on the next deployment. To pick it up immediately, click **Redeploy** on the latest deployment.

## Verification

To confirm a variable is set in the running container, log it from your app on startup:

```javascript
console.log("DATABASE_URL configured:", !!process.env.DATABASE_URL);
```

Don't log the value itself — Brimble's runtime logs are not encrypted at rest the same way the variable store is, and any log output is visible to anyone with access to the project.

## Next steps

- [Environment variables guide](environment-variables.md) — task-oriented walkthrough.
- [System environment variables](system-variables.md) — what Brimble injects automatically.
- [Deployments](deployments.md) — how environment variables flow into a deployment.
