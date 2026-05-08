# Builds

A **build** turns your source code into an artifact Brimble can run. Two builders exist: **Railpack** for backend services, and Brimble's **frontend builder** for static sites and SPAs. The right one is picked automatically based on your repo.

## What gets built

Every deployment runs a build, except for rollbacks (which reuse an existing artifact) and database projects (which run a managed image you don't build).

The build runs in an isolated, ephemeral environment called the **runner**. Each build gets its own runner, scrubbed at the end. Files written during a build don't persist to the next one — except through the **build cache**, described below.

## Builder selection

Brimble inspects your repo and picks one of two builders:

- **Railpack** — used for backend services and most application types. Detects your language, picks the right base image, installs dependencies, runs your build, and produces a container image.
- **Frontend builder** — used for projects with a static-site framework or a build that emits a directory of static assets. Detects the framework, runs the build, and serves the output directory from the edge directly (no container).

You can override the selection in **Settings → Build** if the auto-detection guesses wrong.

## Framework detection

For Node, Brimble reads `package.json` to identify Next.js, Nuxt, SvelteKit, Remix, Vite, Astro, Hugo, Express, Fastify, NestJS, and others. For Python, it reads `requirements.txt`, `pyproject.toml`, or `Pipfile`. For Ruby, `Gemfile`. For Go, `go.mod`. For PHP, `composer.json`. For Rust, `Cargo.toml`. For Java, `pom.xml` or `build.gradle`.

If a `Dockerfile` is present at the repo root, Brimble uses it directly and skips framework detection.

When detection succeeds, Brimble pre-fills:

- The install command.
- The build command (or none, if the framework doesn't need one).
- The start command.
- The output directory (for static sites).

You can edit any of these in **Settings → Build**.

## The build pipeline

A build runs through these stages, each visible as a section in the deployment logs:

1. **Clone** — fetch the commit from Git.
2. **Detect** — pick the builder and framework.
3. **Install** — run the install command. Cached layers are reused if the lockfile hasn't changed.
4. **Build** — run the build command.
5. **Package** — assemble the runnable artifact (container image or static bundle).
6. **Push** — upload the artifact to Brimble's internal registry.

Each stage has its own log section in the dashboard.

## Build cache

Brimble caches install output between builds when:

- The lockfile (`package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `requirements.txt`, `Gemfile.lock`, `go.sum`, `Cargo.lock`, `composer.lock`) hasn't changed.
- The package manager and language version are unchanged.
- The cache is enabled for the project (it is, by default).

A warm cache typically cuts install time from 60–120 seconds to under 10. To disable the cache for a project (for example, if it's masking a real install bug), turn off **Build cache** under **Settings → Build**.

To clear the cache without disabling it, click **Clear build cache** in the same panel. The next build will be cold; subsequent ones cache again.

## Build environment

Inside the build runner you have:

- Your repo at the project's root directory.
- Standard package managers for the detected language (npm, yarn, pnpm, pip, bundler, go, cargo, composer, etc.).
- All environment variables for the active environment, applied during install and build.
- Network access for fetching dependencies.

You don't have:

- Access to the production runtime container or its filesystem.
- Persistent storage between builds, beyond the cache.
- The ability to spawn long-running processes that outlive the build.

Build-time resource limits are generous enough for typical app builds (multi-GB Next.js builds, large Python ML deps, etc.). If you hit a limit, the failure shows up in the deployment logs as an out-of-memory or timeout error.

## Build minutes

Every plan includes a monthly allowance of build minutes. Build time is measured from clone start to push completion. Once the allowance is used up:

- **Free plan.** New builds queue indefinitely until the cycle resets.
- **Paid plans.** Overage is billed at $0.002 per minute and rolled into the next invoice.

The current usage is visible under **Billing → Usage** in the dashboard.

## Watch paths

For monorepos, you can tell Brimble to only trigger a build when files in a specific directory change. Set the **Watch paths** in **Settings → Build** to `apps/web/**` and pushes that only modify other apps won't redeploy this project.

If watch paths aren't set, every push to the tracked branch triggers a build.

## Custom build commands

Set the install, build, and start commands explicitly under **Settings → Build** if the defaults don't fit. They run in this order, in the project's root directory:

1. Install command (e.g. `pnpm install --frozen-lockfile`).
2. Pre-start command, if set (runs once per build, after build but before push).
3. Build command (e.g. `pnpm build`).
4. Start command — runs in the deployed container, not the build runner.

Each command runs in `bash`. Standard shell features (pipes, redirects, `&&`) work.

## Build secrets

Use environment variables for build-time secrets (private package tokens, fetch keys, etc.). Variables in the project's environment are injected into the build runner. See [Environment variables](environments.md).

## Verification

Look at the deployment logs after a successful build. Each phase reports its duration and exit code. To compare two builds:

```bash
curl https://api.brimble.io/v1/logs/<deployment-id> -H "Authorization: Bearer <token>"
```

The response includes per-phase timestamps so you can see exactly where time was spent.

## Next steps

- [Build failures](../troubleshooting/build-failures.md) — common build errors and how to fix them.
- [Frameworks supported](../reference/frameworks.md) — the full list of detected stacks.
- [Environment variables](environments.md) — making secrets and config available at build time.
