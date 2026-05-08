# What is Brimble

Brimble is a platform-as-a-service for deploying applications. You connect a Git repository, Brimble builds and runs it, and you get a public URL with HTTPS already set up.

If you've used Heroku, Render, or Railway, the model will feel familiar.

![TODO: screenshot of the Brimble dashboard home showing the welcome panel, the projects-this-week stats, recent deployed projects as cards, connected domains count, and the sidebar with Home / Projects / Domains / Scaling / Discover](./images/PLACEHOLDER.png)

*The Brimble dashboard home, after deploying a few projects.*

## What you can deploy

Each thing you deploy on Brimble is a **project**. A project has a type that determines how it builds and runs:

* **Web service** — long-running HTTP servers (Next.js, Express, Django, Rails, FastAPI, etc.)
* **Static site** — pre-built HTML, CSS, and JS bundles
* **Worker** — background jobs, queue consumers, scheduled tasks
* **Database** — managed PostgreSQL, MySQL, MongoDB, Redis, or SQLite
* **MCP server** — Model Context Protocol servers for AI tooling

You can run as many projects as your plan allows, in any combination.

## How a deployment works

1. You connect a Git repository (GitHub, GitLab, or Bitbucket) and pick a branch.
2. Brimble detects your framework and builds an artifact.
3. The artifact runs in an isolated sandbox in the region you chose.
4. The edge layer terminates TLS and routes traffic to your project at `<project-name>.brimble.app` — or your custom domain.

Every push to the connected branch triggers a new build and deployment. Old deployments stay around so you can roll back.

## Who Brimble is for

* Developers shipping side projects, MVPs, or production apps who don't want to operate infrastructure.
* Teams who want bare-metal performance without running Kubernetes.
* Anyone who's tired of YAML.

## Next steps

* [Quickstart →](quickstart.md) — deploy your first project
* [Add a custom domain →](custom-domains.md)
