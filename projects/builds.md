# Builds

A **build** turns your source code into something Brimble can run. This page covers what runs during a build, in what order, and what the runner does between "you pushed a commit" and "your service is live."

## What runs your build

Brimble runs every build on its **remote builder fleet**, a pool of ephemeral build machines kept warm with a persistent layer cache. The fleet is separate from where your code eventually runs, and the machine that runs your build is fresh every time.

Two things matter to you here:

* **Builders are ephemeral.** A build machine doesn't survive the build. State written to the local filesystem during a build is gone the moment the build ends.
* **The cache is persistent.** Even though the machine is fresh, the layer cache attached to your project survives. The next build pulls cached layers from object storage and skips the work it already did.

That combination means cold builds get a clean machine every time (no surprise state from previous runs) while warm builds reuse layers and finish in a fraction of the time.

## The build pipeline

Each deployment moves through these stages, and each one shows up as a section in the streaming logs.

### 1. Clone

The runner fetches your commit from GitHub, GitLab, or Bitbucket using the credentials Brimble has on file for your account. Submodules are cloned recursively. The clone is shallow, fetching only the commit being built and a small history.

If your repo is private, the runner authenticates using the Git provider integration you authorized when connecting the repo. No SSH keys or tokens of your own are needed.

### 2. Detect

The runner inspects the repo to decide which **builder** to use:

* **A `Dockerfile` at the project root.** The runner uses your Dockerfile directly.
* **A static-site framework** (Vite, Astro, Next.js export, Hugo, and similar). The runner uses the static builder, which produces a directory of files served from the edge.
* **A backend stack** (Node, Python, Ruby, Go, Java, PHP, Rust, Elixir, and similar). The runner uses **Railpack** to detect the framework, install dependencies, and build a container image without you writing a Dockerfile.
* **A Nix-based project** (a `flake.nix` or `default.nix`). The runner uses Nix to materialize the dependency closure.

The detected stack, version, and chosen builder are logged. If the auto-detection picks the wrong thing, override the build command, start command, or framework under **Configuration**.

### 3. Inject secrets

Before the install or build runs, the runner pulls your environment variables from Brimble's secret store and exposes them to the build environment. They're available to every command that runs from this point forward, including install scripts, build scripts, and your code.

Secrets you've set in the active environment (Production, Preview, etc.) are present.

### 4. Install

The runner runs your install command (`npm ci`, `pip install -r requirements.txt`, `bundle install`, `go mod download`, whatever the framework needs). The default install command is auto-detected; override it under **Configuration**.

If a layer cache exists for this project and the lockfile hasn't changed since the last successful build, the runner restores the cached install output instead of running the install fresh. A warm cache typically takes the install phase from a minute or two down to a few seconds. See [Build cache](#build-cache) below.

### 5. Build

The runner runs your build command (`npm run build`, `mix compile`, `cargo build --release`, and so on). For Dockerfile-based projects, this is where the image is built layer by layer using BuildKit.

For Railpack-based builds, Railpack assembles a base image, applies the language toolchain, copies your source, runs your build command, and produces a runnable container.

For static sites, this is the only phase that produces output; no container is built afterwards.

### 6. Pre-start

If you've set a **pre-start command** under **Configuration**, it runs once after the build, before the artifact is pushed. A common use is running database migrations against the production database before flipping traffic. The pre-start command runs in the build runner with the same environment variables as the build phase.

If pre-start exits non-zero, the deployment fails. Traffic does not switch.

### 7. Push

The runner pushes the finished artifact (a container image, or a directory of static assets) to Brimble's internal storage. For container images, the runner pushes only the layers that don't already exist; unchanged base layers don't transfer.

While the push happens, the new layers are added to the project's persistent cache so the next build can reuse them.

### 8. Orchestrate

The runner hands the artifact off to the orchestration layer, which schedules the container in your project's region. Brimble uses Nomad for scheduling and Consul for service discovery and health checking.

The orchestration layer:

* Allocates a slot on a host with capacity for your project's CPU, memory, and storage.
* Pulls the image (or unpacks the static bundle).
* Starts the container with your start command and the runtime environment variables.
* Registers the service in Consul with health checks defined by your `healthCheckPath`.

### 9. Health check

For web services, Brimble sends a GET to your `healthCheckPath` (default `/`). For workers, the check is process liveness. For static sites, no probe; the artifact is served directly from the edge.

Once the check passes, the edge flips traffic to the new deployment. The previous deployment continues running for a short drain window so in-flight requests finish, then it's stopped.

If the health check never passes, the deployment is marked **degraded** and traffic stays on the previous deployment. After repeated failures the deployment moves to **failed**.

## Build cache

Brimble's build cache is a content-addressed layer cache attached to your project. It's the part that makes warm builds fast.

How it works in practice:

* **Layer-level reuse.** For Docker and Railpack builds, the cache works at the layer level. If the inputs to a layer haven't changed (your lockfile, base image, and the commands run before it), the layer is fetched from cache instead of rebuilt.
* **Persistent across builds.** Layers live in durable object storage tied to your project. Any builder in the fleet can pull them, so caching works even when builds land on different machines.
* **Parallel cache uploads.** As a build runs, completed layers are uploaded in the background. By the time the build finishes, the next build's cache is already warm.
* **Fresh machine, warm cache.** Every build runs on an ephemeral builder, but the cache survives. You don't get cross-build pollution from filesystem state, but you don't pay for cold dependencies on every push.

### What's cached

* **Install output** for backend builds (`node_modules`, `~/.cache/pip`, `vendor/bundle`, `~/.cargo`, and similar), keyed on your lockfile.
* **Image layers** for Dockerfile and Railpack builds, keyed on each layer's input hash.
* **Static-builder intermediate output**, keyed on the source manifest.

### What's not cached

* Anything you write to a path outside the standard install or build directories.
* Output that depends on environment variables not declared as build inputs (changing one of these silently bypasses the cache).
* The build runner's working directory itself; it's wiped between builds.

### Cache lifetime

A project's build cache **clears automatically after 15 days** of inactivity. If you don't deploy a project for two weeks, the next deploy is a cold build. Active projects keep their cache indefinitely.

### Clearing the cache manually

If a stale cache is producing weird results, clear it:

1. Open **Configuration** for the project.
2. Toggle **Enable Build Cache on Redeploy** off, save, and redeploy.
3. The next build runs cold. Toggle the cache back on once it succeeds; subsequent builds will re-warm.

## The build environment

Inside the build runner:

* Your repo at the project's root directory, or the `rootDir` you configured for monorepos.
* Standard package managers for the detected language: npm, yarn, pnpm, bun, pip, poetry, bundler, go, cargo, composer, mvn, gradle, mix.
* All environment variables for the active environment.
* Network access for fetching dependencies from public registries.
* Plenty of memory and CPU for typical app builds. Large frontend bundles (multi-GB Next.js builds, big Webpack bundles) work without special config; for outliers you can bump build memory under **Configuration**.

You don't have:

* Access to your production runtime container or its filesystem.
* Persistent state between builds beyond the cache.
* The ability to spawn long-running processes that outlive the build.
* Access to other tenants' builds or runtimes; every build runs on an isolated runner.

## Watch paths

For monorepos, configure **Watch paths** under **Configuration** to scope which file changes trigger a build. For example, `apps/web/**` means pushes that only modify other apps in the monorepo don't redeploy this project.

If watch paths aren't set, every push to the tracked branch builds.

## Build minutes and concurrency

Each plan grants:

* A monthly **build-minute** allowance.
* A number of **concurrent build slots**.

Build minutes are clock time from the start of clone to the end of push. If you exhaust your monthly allowance:

* **Free plan:** new builds queue indefinitely until the cycle resets.
* **Paid plans:** overage bills at the per-minute rate from [Plans and pricing](../billing/plans.md) and rolls into the next invoice. You can also top up; see [Build minutes](../billing/build-minutes.md).

Concurrent slots determine how many builds can run at the same time. Above your slot count, builds queue (status `pending`) until a slot frees up. Cancel a queued or in-progress build to free a slot from **Deployments**.

## Custom Dockerfiles

If you put a `Dockerfile` at the project root, Brimble uses it directly:

* The image is built with BuildKit.
* Multi-stage Dockerfiles work; only the final stage's image is deployed.
* The container must listen on `process.env.PORT` and bind to `0.0.0.0`.
* The image runs as the user defined in the Dockerfile. Use a non-root user where possible.

Your Dockerfile has full control. Anything Docker can do, you can do. Brimble doesn't inspect or modify the Dockerfile.

## Status updates back to the dashboard

The runner streams status back to Brimble's API in real time as it works. Each phase emits start and finish events. The dashboard's logs drawer subscribes to that stream so you see logs land within hundreds of milliseconds of the runner producing them.

When a deployment finishes (success or failure), the runner publishes a final event. That event triggers:

* The status badge update in the dashboard.
* The webhook delivery for `deployment.success` or `deployment.failed`, if you've configured webhooks.
* The flip of edge traffic to the new deployment.
* The old deployment's drain and stop.

## Why builds fail

The build pipeline has well-defined failure modes. See [Build failures](../troubleshooting/build-failures.md) for the catalog and how to debug each one.

## Next steps

* [Deployments](deployments.md), how a build's output becomes a running service.
* [Build failures](../troubleshooting/build-failures.md), common errors during a build.
* [Frameworks supported](frameworks.md), the full list of detected stacks.
