# Builds

A **build** turns your source code into something Brimble can run. This page covers what runs during a build, in what order, and what the runner does between "you pushed a commit" and "your service is live."

## What runs your build

Brimble has a dedicated **build runner** — a Node-based service that lives on its own pool of machines, separate from where your code eventually runs. Every deployment starts with the runner picking up the job, preparing a fresh workspace, and walking through the build pipeline.

The runner is a queue consumer. When you push a commit (or click **Redeploy**), Brimble's core API publishes a deployment job to a message queue. A free worker pulls the job, claims a build slot, and starts. If all your slots are busy, the job waits in the queue.

## The build pipeline

Each deployment moves through these stages, and each one shows up as a section in the streaming logs.

### 1. Clone

The runner fetches your commit from GitHub, GitLab, or Bitbucket using the credentials Brimble has on file for your account. Submodules are cloned recursively. The clone is shallow — only the commit being built and a small history.

If your repo is private, the runner authenticates using the Git provider integration you authorized when connecting the repo. No SSH keys or tokens of your own are needed.

### 2. Detect

The runner inspects the repo to decide which **builder** to use:

- **A `Dockerfile` at the project root** — the runner uses your Dockerfile directly.
- **A static-site framework** (Vite, Astro, Next.js export, Hugo, etc.) — the runner uses the static builder, which produces a directory of files served from the edge.
- **A backend stack** (Node, Python, Ruby, Go, Java, PHP, Rust, Elixir, etc.) — the runner uses **Railpack** to detect the framework, install dependencies, and build a container image without you writing a Dockerfile.
- **A Nix-based project** (a `flake.nix` or `default.nix`) — the runner uses Nix to materialize the dependency closure.

The detected stack, version, and chosen builder are logged. If the auto-detection picks the wrong thing, override the build command, start command, or framework under **Settings → Build**.

### 3. Inject secrets

Before the install or build runs, the runner pulls your environment variables from Brimble's secret store and exposes them to the build environment. They're available to every command that runs from this point forward — install scripts, build scripts, your code.

Secrets you've set in the active environment (Production, Preview, etc.) are present. System variables Brimble injects (`BRIMBLE_REGION`, `BRIMBLE_COMMIT_SHA`, `CI`, `BRIMBLE_BUILD=1`, etc.) are also present.

### 4. Install

The runner runs your install command — `npm ci`, `pip install -r requirements.txt`, `bundle install`, `go mod download`, whatever the framework needs. The default install command is auto-detected; override it under **Settings → Build**.

If a build cache exists for this project (and the lockfile hasn't changed since the last successful build), the runner restores the cached install output instead. A warm cache typically takes the install phase from 60–120 seconds down to under 10. See [Build cache](#build-cache) below.

### 5. Build

The runner runs your build command — `npm run build`, `mix compile`, `cargo build --release`, etc. For Dockerfile-based projects, this is where `docker build` (using BuildKit / `buildx` under the hood) constructs the image layer by layer.

For Railpack-based builds, Railpack assembles a base image, applies the language toolchain, copies your source, runs your build command, and produces a runnable container.

For static sites, this is the only phase that produces output; no container is built afterwards.

### 6. Pre-start

If you've set a **pre-start command** under **Settings → Build**, it runs once after the build, before the artifact is pushed. Common use: running database migrations against the production database before flipping traffic. The pre-start command runs in the build runner with the same environment variables as the build phase.

If pre-start exits non-zero, the deployment fails — traffic does not switch.

### 7. Push

The runner pushes the finished artifact (a container image, or a directory of static assets) to Brimble's internal storage. For container images, the runner pushes only the layers that don't already exist — unchanged base layers don't transfer.

While the push happens, build cache layers are uploaded asynchronously to object storage so the next build can start warm.

### 8. Orchestrate

The runner hands the artifact off to the orchestration layer, which schedules the container in your project's region. Brimble uses Nomad for scheduling and Consul for service discovery and health checking.

The orchestration layer:

- Allocates a slot on a host with capacity for your project's CPU, memory, and storage.
- Pulls the image (or unpacks the static bundle).
- Starts the container with your start command and the runtime environment variables.
- Registers the service in Consul with health checks defined by your `healthCheckPath`.

### 9. Health check

For web services, Brimble sends a GET to your `healthCheckPath` (default `/`). For workers, the check is process liveness. For static sites, no probe — the artifact is served directly from the edge.

Once the check passes, the edge flips traffic to the new deployment. The previous deployment continues running for a short drain window so in-flight requests finish, then it's stopped.

If the health check never passes, the deployment is marked **degraded** and traffic stays on the previous deployment. After repeated failures the deployment moves to **failed**.

## Build cache

Brimble caches your install output between builds when:

- The project's **Build cache** toggle is on (it is, by default).
- Your lockfile (`package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `requirements.txt`, `Gemfile.lock`, `go.sum`, `Cargo.lock`, `composer.lock`) hasn't changed since the last successful build.
- The package manager and language version are unchanged.

For Docker-based builds, BuildKit's layer cache is also used — unchanged layers are reused from the previous build, dramatically cutting rebuild time.

Cache layers are uploaded after each successful build to durable object storage. Future builds (on any runner host) restore from there, so caching works even when builds land on different machines.

### Clearing the cache

If a stale cache is producing weird results, clear it:

1. Open **Settings → Build** for the project.
2. Click **Clear build cache**.
3. The next build runs cold; subsequent ones cache again.

## The build environment

Inside the build runner:

- Your repo at the project's root directory (or the `rootDir` you configured for monorepos).
- Standard package managers for the detected language — npm, yarn, pnpm, bun, pip, poetry, bundler, go, cargo, composer, mvn, gradle, mix.
- All environment variables for the active environment.
- Network access for fetching dependencies from public registries.
- Plenty of memory and CPU for typical app builds. Large frontend bundles (multi-GB Next.js builds, big Webpack bundles) work without special config; for outliers you can bump build memory under **Settings → Build**.

You don't have:

- Access to your production runtime container or its filesystem.
- Persistent state between builds beyond the cache.
- The ability to spawn long-running processes that outlive the build.
- Access to other tenants' builds or runtimes — every build runs in an isolated runner.

`BRIMBLE_BUILD=1` is set during the build phase only. Use it to gate code that should only run during a build (prerendering, static config generation):

```javascript
if (process.env.BRIMBLE_BUILD === "1") {
  // build-time only
}
```

At runtime, `BRIMBLE_BUILD` is unset.

## Watch paths

For monorepos, configure **Watch paths** under **Settings → Build** to scope which file changes trigger a build. For example, `apps/web/**` means pushes that only modify other apps in the monorepo don't redeploy this project.

If watch paths aren't set, every push to the tracked branch builds.

## Build minutes and concurrency

Each plan grants:

- A monthly **build-minute** allowance.
- A number of **concurrent build slots**.

Build minutes are clock time from the start of clone to the end of push. If you exhaust your monthly allowance:

- **Free plan:** new builds queue indefinitely until the cycle resets.
- **Paid plans:** overage bills at the per-minute rate from [Plans](../reference/plans.md) and rolls into the next invoice.

Concurrent slots determine how many builds can run at the same time. Above your slot count, builds queue (status: `pending`) until a slot frees up. Cancel a queued or in-progress build to free a slot from **Deployments**.

## Custom Dockerfiles

If you put a `Dockerfile` at the project root, Brimble uses it directly:

- The image is built with `docker build`, which uses BuildKit.
- Multi-stage Dockerfiles work; only the final stage's image is deployed.
- The container must listen on `process.env.PORT` and bind to `0.0.0.0`.
- The image runs as the user defined in the Dockerfile (use a non-root user where possible).

Your Dockerfile has full control — anything Docker can do, you can do. Brimble doesn't inspect or modify the Dockerfile.

## docker-compose

For projects that need multiple coordinated containers (a service plus a sidecar, for example), Brimble understands `docker-compose.yml` files. The compose file is parsed and each service in it is built and deployed as part of the same project.

Compose-style `deploy.replicas` and `deploy.mode` are honored when set — the parser respects `replicated`, `replicated-job`, and `global` modes.

## Status updates back to the dashboard

The runner streams status back to Brimble's API in real time as it works. Each phase emits start/finish events. The dashboard's logs drawer subscribes to that stream so you see logs land within hundreds of milliseconds of the runner producing them.

When a deployment finishes (success or failure), the runner publishes a final event. That event triggers:

- The status badge update in the dashboard.
- The webhook delivery for `deployment.success` or `deployment.failed` (if you've configured webhooks).
- The flip of edge traffic to the new deployment.
- The old deployment's drain and stop.

## Why builds fail

The build pipeline has well-defined failure modes — see [Build failures](../troubleshooting/build-failures.md) for the catalog and how to debug each one.

## Next steps

- [Deployments](deployments.md) — how a build's output becomes a running service.
- [Build failures](../troubleshooting/build-failures.md) — common errors during a build.
- [Frameworks supported](../reference/frameworks.md) — the full list of detected stacks.
