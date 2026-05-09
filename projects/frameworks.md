# Frameworks supported

Brimble's build pipeline auto-detects most popular frameworks and pre-fills the install, build, and start commands. This page lists what's detected and what defaults you get.

If your stack isn't listed, set the commands manually under **Settings → Build**. Anything that runs in a Linux container will work, detection is a convenience, not a requirement.

## How detection works

Brimble inspects:

- The repo root for known files (`package.json`, `requirements.txt`, `Gemfile`, `go.mod`, `Cargo.toml`, `composer.json`, `pom.xml`, `build.gradle`, `mix.exs`, `Dockerfile`).
- The contents of those files for framework-specific signals (e.g. `next` in `dependencies`).
- A few file-pattern checks (`Procfile`, `wrangler.toml`, etc.).

A `Dockerfile` at the repo root takes precedence, Brimble uses it directly and skips framework detection.

## Node.js

Detected via `package.json`.

| Framework | Service type | Default commands |
|---|---|---|
| **Next.js (server)** | Web service | `npm install`, `npm run build`, `npm start` |
| **Next.js (export)** | Static site | `npm install`, `npm run build`, output `out/` |
| **Nuxt 3** | Web service | `npm install`, `npm run build`, `node .output/server/index.mjs` |
| **SvelteKit** | Web service | `npm install`, `npm run build`, `node build` |
| **Remix** | Web service | `npm install`, `npm run build`, `npm start` |
| **Vite** | Static site | `npm install`, `npm run build`, output `dist/` |
| **Astro** | Static site or Web service | `npm install`, `npm run build`, output `dist/` |
| **Express / Fastify / Koa** | Web service | `npm install`, no build, `npm start` |
| **NestJS** | Web service | `npm install`, `npm run build`, `npm run start:prod` |
| **Hono** | Web service | `npm install`, `npm run build`, `npm start` |
| **AdonisJS** | Web service | `npm install`, `npm run build`, `node build/server.js` |

Package manager auto-detected from lockfile: `package-lock.json` → npm, `yarn.lock` → yarn, `pnpm-lock.yaml` → pnpm, `bun.lockb` → bun.

Node version auto-detected from `engines.node` in `package.json` or a `.nvmrc` file. Latest LTS by default.

## Python

Detected via `requirements.txt`, `pyproject.toml`, or `Pipfile`.

| Framework | Service type | Default commands |
|---|---|---|
| **Django** | Web service | `pip install -r requirements.txt`, `gunicorn <project>.wsgi` |
| **Flask** | Web service | `pip install -r requirements.txt`, `gunicorn app:app` |
| **FastAPI** | Web service | `pip install -r requirements.txt`, `uvicorn main:app --host 0.0.0.0 --port $PORT` |

Python version detected from `.python-version`, `runtime.txt`, or `pyproject.toml`. Defaults to a recent stable version.

Package manager: pip (requirements.txt), Poetry (pyproject.toml with `[tool.poetry]`), or pipenv (Pipfile).

## Ruby

Detected via `Gemfile`.

| Framework | Service type | Default commands |
|---|---|---|
| **Rails** | Web service | `bundle install`, `bundle exec rails assets:precompile`, `bundle exec rails server -p $PORT -b 0.0.0.0` |
| **Sinatra** | Web service | `bundle install`, no build, `bundle exec ruby app.rb -o 0.0.0.0 -p $PORT` |

Ruby version from `.ruby-version` or `Gemfile`.

## Go

Detected via `go.mod`.

- Service type: Web service.
- Build: `go build -o app .`
- Start: `./app`

Go version from `go.mod`.

## PHP

Detected via `composer.json`.

| Framework | Service type | Default commands |
|---|---|---|
| **Laravel** | Web service | `composer install --no-dev`, `php artisan migrate --force`, `php artisan serve --host=0.0.0.0 --port=$PORT` |
| **Symfony** | Web service | `composer install --no-dev`, `bin/console cache:clear`, `php -S 0.0.0.0:$PORT public/index.php` |

## Java / Kotlin

Detected via `pom.xml` (Maven) or `build.gradle` / `build.gradle.kts` (Gradle).

| Framework | Service type | Default commands |
|---|---|---|
| **Spring Boot** | Web service | `./mvnw package -DskipTests` (or Gradle equivalent), `java -jar target/*.jar` |

JVM version from `pom.xml`'s `<java.version>` or Gradle's `targetCompatibility`.

## Rust

Detected via `Cargo.toml`.

- Service type: Web service.
- Build: `cargo build --release`
- Start: `./target/release/<bin-name>`

Rust toolchain from `rust-toolchain.toml` or stable.

## Elixir

Detected via `mix.exs`.

- Service type: Web service.
- Build: `mix deps.get`, `mix compile`, asset compilation if Phoenix.
- Start: `mix phx.server` or `MIX_ENV=prod elixir --erl '-detached' -S mix run --no-halt`.

## Static-only frameworks

| Framework | Default commands |
|---|---|
| **Hugo** | `hugo`, output `public/` |
| **Jekyll** | `bundle install`, `bundle exec jekyll build`, output `_site/` |
| **Eleventy (11ty)** | `npm install`, `npx eleventy`, output `_site/` |
| **Gatsby** | `npm install`, `npm run build`, output `public/` |
| **Docusaurus** | `npm install`, `npm run build`, output `build/` |
| **MkDocs** | `pip install mkdocs`, `mkdocs build`, output `site/` |

For all of the above, Brimble creates a static site project (no runtime container).

## Dockerfile

If a `Dockerfile` exists at the project root, Brimble uses it directly:

- Build: `docker build -t <image> .`
- Start: `docker run -e PORT=$PORT -p $PORT <image>`

The Dockerfile must:

- Use `EXPOSE` for the listener port (or the container must listen on `$PORT` regardless).
- Run as a non-root user where possible.
- End with a `CMD` that launches your service in the foreground.

## Custom commands

For any framework, detected or not, you can override under **Settings → Build**:

- **Install command**, typically the package manager install step.
- **Build command**, anything that compiles or bundles. Empty for runtime-only languages.
- **Pre-start command**, runs once per build, after the build, before the artifact is pushed. Useful for migrations.
- **Start command**, launches the runtime container. Must keep the process in the foreground.

The commands run in `bash`. Standard shell features (pipes, redirects, `&&`) work.

## Language version selection

Most frameworks default to a recent stable version. Pin a specific version via the language's standard mechanism:

- **Node:** `engines.node` in `package.json`, or `.nvmrc`.
- **Python:** `.python-version`, `runtime.txt`, or `pyproject.toml`.
- **Ruby:** `.ruby-version` or `Gemfile`.
- **Go:** `go.mod`'s `go` directive.
- **Java:** `pom.xml` `<java.version>` or Gradle `targetCompatibility`.

If detection picks the wrong version, the deployment logs will show the chosen version under the **Detect** phase. Pin explicitly to fix.
