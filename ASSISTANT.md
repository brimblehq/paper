# Assistant guide

This file is for AI assistants (Claude, Cursor, Copilot, etc.) working on the Brimble docs. It captures the conventions, the hard rules, and the pitfalls that have already been worked through. Read it before making changes.

If you are a human reading this, the operational stuff for editors lives in `README.md`. This file is the *why* and the *how to not break things*.

---

## 1. What this repo is

The source for **Brimble's public documentation site**, built with [Mintlify](https://mintlify.com).

* Pages are `.mdx` files organized by feature area (`projects/`, `domains/`, `environments/`, etc.).
* Navigation, theming, fonts, and brand colors live in `docs.json` at the root.
* Run locally with `mintlify dev`.

The site is published from `main` after Mintlify validates `docs.json` and the MDX files.

---

## 2. The single hardest rule

**The codebase is the only source of truth.** Never document something based on what feels right, what other PaaS docs say, or what plausibly should be true. Verify against the relevant repo first.

Brimble services and where to look:

| Concern | Repo |
|---------|------|
| Project lifecycle, deployments, domains, env vars, autoscaling, webhook emission | `~/Documents/Archive/PERSONAL/work/core` |
| Build pipeline, runner, build cache | `~/Documents/Archive/PERSONAL/work/runner` |
| Database provisioning, backups | `~/Documents/Archive/PERSONAL/work/heracle` |
| Edge proxy, request routing, password protection, MCP auth | `~/Documents/Archive/PERSONAL/work/proxy` |
| DNS service, Caddy config | `~/Documents/Archive/PERSONAL/work/dns` |
| Subscriptions, Stripe, build minutes, refunds | `~/Documents/Archive/PERSONAL/work/brimble-payment` |
| Login, OAuth, 2FA, passkeys | `~/Documents/Archive/PERSONAL/work/auth-service` |
| Webhook delivery (factory + dispatch), email templates | `~/Documents/Archive/PERSONAL/work/brimble-email` |
| Dashboard UI, all settings panels, plan defaults | `~/Documents/brimble/apps/dashboard/src` |
| Marketing site styling, tokens | `~/Documents/brimble/apps/web/src` |

If you are about to claim a behavior, search the relevant repo for a function, route, schema field, or config value that backs it up. If you can't find one, **say so explicitly** and either ask the user or write more conservatively. Do not split the difference with a confident-sounding guess.

When the user calls out a fabrication, fix it and look back at *adjacent* claims in the same doc. They were almost certainly produced by the same flawed inference.

---

## 3. Document only what users can do

The mongoose documents, RabbitMQ events, internal queues, and HashiCorp stack details are **not the public contract**. Public docs cover:

* What the user sees and does in the dashboard.
* Behaviors that the user observes from outside (URLs, headers, response codes, webhook payload shapes after the factory transforms them).
* The CLI and webhook configuration — anything users wire into their own systems.

Internal plumbing, infra components, and database schemas don't belong in public docs even when they're real. The smell test: would an engineer at a competitor learn something operational from this doc? If yes, cut it.

The webhook payloads are a good example: `core` emits the full `IProject` mongoose document, but `brimble-email/src/factory/webhook.factory.ts` reshapes it to a clean schema before delivery. The clean schema is what we document. The raw `IProject` with `vaultPath`, `vaultToken`, `tracking_token`, `nomadJobId`, etc. is not.

---

## 4. Files and structure

```
docs.json                  Mintlify config: nav, theme, fonts, colors, navbar links
index.mdx                  Docs home page (cards grid)
README.md                  Human-facing repo readme (mintlify dev, conventions)
ASSISTANT.md               This file
fonts/                     ABC Marfa woff2 files (referenced from docs.json)
images/<area>/             Screenshots, organized by doc area

getting-started/           First-deploy flow
projects/                  Project lifecycle, builds, deployments, service types, scaling
domains/                   Custom domains, buying, transfer, DNS
environments/              Environments, env vars, references
networking/                Edge, request lifecycle, internal services, rate limits
scaling/                   Scaling groups
observability/             Metrics, logs
analytics/                 Web analytics
workspaces-and-teams/      Workspaces, team management
security/                  2FA, passkeys
billing/                   Plans, build minutes
notifications/             Notification preferences
webhooks/                  Webhook setup + events reference
troubleshooting/           Symptom-indexed debugging
glossary.mdx               Term reference
```

Add new pages in the right area, then register them under the right `group` in `docs.json`'s `navigation.tabs[0].groups`.

---

## 5. Page conventions

**Frontmatter.** Every page has YAML frontmatter:

```yaml
---
title: "Page title"
description: "One-line summary, used as meta-description and previewed in search."
---
```

Mintlify renders the `title` as the H1, so **don't** write a duplicate `# Title` in the body.

**Internal links.** Use Mintlify's path-style links without the `.mdx` extension:

```mdx
See [Builds](/projects/builds) or [Plans and pricing](../billing/plans).
```

Both absolute (`/path`) and relative (`../area/page`) work. Drop `.mdx` either way.

**Headings.** Use `##` for top-level sections and `###` for subsections. The frontmatter `title` is the page H1; never start with `# ` in the body.

**Tone.** Direct, second person, active voice. Short sentences. No "simply," "just," "easily," "powerful," "seamless." Don't apologize for the product.

**Em-dashes are banned.** They make prose feel AI-generated. Use commas, periods, colons, or rewrite the sentence. Sweep them out before committing. Em-dashes inside fenced code blocks are fine.

---

## 6. Mintlify components

Use these instead of HTML or markdown image syntax.

**Callouts.** Available without imports: `<Info>`, `<Note>`, `<Tip>`, `<Warning>`, `<Check>`. They render as colored callout boxes:

```mdx
<Info>
  Database backups run every hour on the hour.
</Info>
```

**Image frames.** Use `<Frame>` with a single child `<img>`, plus a one-sentence `caption`:

```mdx
<Frame caption="The Add Domain dialog with the CNAME record to copy.">
  <img src="/images/domains/add-cname-record.jpeg" alt="Add Domain dialog showing the CNAME record" />
</Frame>
```

Image paths are absolute from the repo root. PNG and JPEG both work. Don't hotlink external CDNs unless you're explicitly intentional (Simple Icons in `projects/frameworks.mdx` is the one exception).

**Image placeholders.** When a page references UI but no screenshot exists yet, use:

```mdx
<Info>
  **Image needed:** description of exactly what to capture, in enough detail that someone else can take the screenshot without asking.
</Info>
```

When the screenshot lands in `/images/`, swap the `<Info>` for a `<Frame>` block. The placeholder convention makes outstanding screenshots greppable: `grep -rn "Image needed:" .`.

**Cards.** For landing-style pages (the home `index.mdx`), use `<CardGroup>` with `<Card>` children:

```mdx
<CardGroup cols={3}>
  <Card title="Quickstart" href="/getting-started/quickstart">
    Deploy your first project in under 10 minutes.
  </Card>
</CardGroup>
```

**Steps, tabs, accordions.** Mintlify ships these too. See [Mintlify components](https://mintlify.com/docs/content/components) before reaching for HTML.

---

## 7. Code blocks

**Fences must be paired correctly:**

* The opener can carry a language tag: ` ```bash `, ` ```typescript `, ` ```text `.
* The closer must be **bare** ` ``` ` with no language. A closer with a tag (` ```bash ` at the bottom of a block) is *not* recognized as a closer; the parser keeps reading until it finds a bare fence, which collapses subsequent prose into one giant code block. This produces 404s in production and rendering chaos in preview. Always check that closers are bare.

**Tag every block.** Bare openers render without syntax highlighting. Use `text` for ASCII diagrams, DNS zone records, and other non-language content so they still get the code-block styling (border, copy button, monospace) without misleading colors.

Common tags by content:

| Content | Tag |
|---------|-----|
| Shell commands (`curl`, `dig`, `npm`, etc.) | `bash` |
| HTTP request/response | `http` |
| JSON | `json` |
| TypeScript or JS | `typescript` / `javascript` |
| YAML | `yaml` |
| `.npmrc`, `.env`, INI-style | `ini` (or `bash` for env-var assignments) |
| ASCII art, request flow, DNS records | `text` |

---

## 8. Two MDX gotchas you will hit

**HTML comments break the page.** MDX treats `<` as the start of JSX, so `<!-- ... -->` causes a parse error and the page returns 404. Use JSX comments instead:

```mdx
{/* this is a comment */}
```

**Curly braces in prose are JSX expressions.** `{{shared.NAME}}` outside a code span will be parsed by MDX as a JSX expression and fail. Always wrap brace-syntax env references in inline code:

```mdx
Reference a shared variable with `{{shared.NAME}}`.
```

The same applies to `{{@project-slug.NAME}}`. Inside fenced code blocks (` ``` ... ``` `) and inline code (`` `...` ``) braces are safe — MDX doesn't parse JSX inside code.

---

## 9. Hard editorial rules

* **No API endpoint references outside the webhook docs.** Don't write `curl https://api.brimble.io/v1/...` in any user guide. The webhook configuration page (and only that page) gets exceptions. For everything else, point at the dashboard.

* **No `<curl> | sh` install commands** unless the user explicitly asks.

* **No invented numbers.** If you don't have a value (rate limit, retry count, plan price), find it in the codebase or omit. The `core/src/services/v1/compute.service.ts` has the real metering math. `dashboard/src/utils/default-pricing.ts` has the real plan defaults. `brimble-payment/COMPUTE_PRICING_GUIDE.md` is internal but accurate.

* **No bullets that just describe what's not there.** "What's not shown" sections, "What you can't do" lists, defensive "the dashboard surfaces but you can't change in place" notes — fold the real point into the relevant section, drop the negation.

* **No "Verification" section just to add one.** A `curl -I https://your-app.brimble.app` and "you should see HTTP/2 200" doesn't help anyone. Keep Verification only when the verification step is non-obvious (the right `tools/list` JSON-RPC payload for an MCP server, the `wrk` invocation that actually exercises autoscaling, the `dig @1.1.1.1` for DNS troubleshooting).

* **No multi-language code triplets for trivial things.** Don't write the same `process.env.X` example in Node, Python, Ruby, and Go. Devs know how to read env vars in their language.

* **No worked-example math whiteboard for pricing.** The `(0.5 - 0.5) × $4 / 720h` style is alienating. State the rule, point at the dashboard view that shows the running cost, move on.

* **No fabricated "Discriminated union for handlers" / pattern-matching examples.** Webhook payload schemas are the doc; how the user dispatches on `event` is their business.

---

## 10. OPSEC

Public docs are reconnaissance fodder. The goal isn't to hide that Brimble uses Nomad / Consul / Vault / BuildKit / Cloudflare — those are fine to mention by name when the context calls for it. The goal is not to hand attackers a configuration map.

**Safe to mention.**

* Cloudflare sits in front of every request (the user can see this themselves from response headers).
* Brimble uses HashiCorp's stack, gVisor, BuildKit, Railpack at a high level.
* User-facing hostnames like `gateway.brimble.app`, `ns1.brimble.io`, `ns2.brimble.io`, the internal DNS suffix `*.service.brimble.internal` (which is what users wire into their own configs).
* Public response headers like `X-Brimble-Id`, `X-Brimble-Project-Version`.
* What the build does, what the runner does, what the cache does.

**Off-limits.**

* The names of third-party infrastructure providers Brimble runs on (referred to as "bare metal" or "Brimble's regions").
* Specific versions of any internal component.
* Internal config: ACL setup, sandbox/seccomp rules, network egress rules from runners, internal hostnames, default ports, ports on internal services, agent configs.
* Recovery, bootstrap, or admin procedures.
* Token, cookie, or session formats for internal auth between platform components.
* Internal API endpoints that aren't exposed to customers.
* How tenant isolation is enforced beyond "tenants are isolated."

If a section requires an internal hostname, port, or recovery sequence to be useful, it's an internal doc, not a public one.

---

## 11. Brand

Pulled from `apps/dashboard/src/styles.css` and `apps/web/src/styles.css` (identical tokens):

| Token | Light | Dark |
|-------|-------|------|
| Brand blue | `#006fff` | `#006fff` (unchanged) |
| Accent orange | `#f5a623` | `#f5a623` |
| Foreground | `#222528` | `#e8eaed` |
| Page background | `#fafafa` | `#1a1c1e` |

**Fonts** are **ABC Marfa** (body and headings) and **ABC Marfa Mono** (code), licensed from Dinamo. The variable `.woff2` files live in `/fonts/`. Trial cuts are wired today; production should use the licensed cuts when available. Mintlify only accepts `woff` / `woff2`, not TTF / OTF.

`docs.json` references the body font; the mono font isn't wired up yet.

---

## 12. Things that have already burned me

A running list. Add to it.

* **`_id` on the wire.** The webhook payload uses `id`, not `_id`. The dashboard backend strips Mongo's `_id` before sending. Don't expose `ObjectId` as a TypeScript type in docs; it's `string`.
* **Dashboard label vs runtime behavior.** The database overview card says "Backup frequency: Daily" but `heracle/internal/service/cron.go:42-43` cron is `0 0 * * * *` — every hour. The cron is the truth.
* **Hacker price.** $5/month, not $7. Pro is $15, not $19. Sourced from `dashboard/src/utils/default-pricing.ts` (the live config). Older internal docs have $7/$19; ignore them.
* **Plan free CPU/memory baseline.** Free is 0.25 vCPU / 0.25 GB. The compute slider's smallest stop is 0.5 — meaning Free-plan projects pick up metered overage as soon as they create a project. Document this honestly.
* **Persistent disk default.** 10 GB minimum, not 1. Sizes go 10–150 GB in 10 GB steps (`disk-size-options.ts`).
* **Password protection** uses a session cookie + form login (`x-brimble-session`), **not** HTTP Basic Auth. `curl -u user:pass` is wrong. There's no self-serve toggle in the dashboard yet.
* **MCP auth header** is `x-brimble-key`, not `Authorization: Bearer`.
* **`subscription_id` in webhook envelope** is in the internal RabbitMQ payload, but `brimble-email/src/factory/webhook.factory.ts` strips it before sending to the user. The user-visible envelope is `{event, data}`.
* **Webhook signing.** Real events (factory → Hookdeck → user) are unsigned. The `X-Brimble-Webhook` HMAC and `X-Brimble-Test` header only apply to the one-off test endpoint, not production deliveries. Don't claim users should verify HMAC.
* **Env var values are not in webhook payloads.** Only the variable name (`envData.name`). The `data.value` field doesn't exist in the factory output.
* **Region defaults are read at runtime** from the `PlanConfiguration` Mongo collection, not hardcoded. The internal `COMPUTE_PRICING_GUIDE.md` documents the values; the dashboard's `default-pricing.ts` mirrors them as fallbacks.
* **Hugging Face git integration was removed.** Don't reintroduce it without confirmation.

---

## 13. Workflow for adding or editing a page

1. Read the relevant codebase paths to ground the claims.
2. Write or edit the `.mdx` file.
3. If new, add it to `docs.json` under the right group.
4. Run `mintlify dev` and visually verify on `http://localhost:3000`. Check:
   * The page renders (no 404).
   * Code blocks have syntax highlighting where expected.
   * Internal links resolve.
   * No JSX parse errors in the terminal.
5. Sweep the file for em-dashes, fabrications, em-dash-y "Verification" sections, and HTML comments.
6. Commit. Reasonable message format:

```
Short summary (imperative, ≤72 chars)

Sourced from <repo>/<path>:<lines>.

- bullet of what changed
- bullet of what was removed
- bullet of any constraints (e.g. "kept ABC stays as <code> until UI ships")
```

---

## 14. When the user pushes back

Most edits in this repo come from a user pointing out something is wrong, missing, or over-explained. Default response:

1. **Look at the code first.** Before defending or extending what's there, search the codebase for the actual behavior. The user's instinct that something is wrong is usually right.
2. **Cut, don't decorate.** When in doubt, less is better. The user has repeatedly pulled out filler ("Verification" sections that just curl the URL, multi-language env-var examples, math-whiteboard pricing, "Why use X" sales sections, "Discriminated union" patterns). Those are high-likelihood removal candidates.
3. **Acknowledge the specific mistake.** Don't generalize. If the FAQ had six fabricated entries, name the six and what was wrong about each.
4. **Don't propose to keep "less sure about" items in the user's court while doing the rest unilaterally.** When the user says "go ahead," do all the cuts you flagged.

---

## 15. If you have to invent something to keep going

Don't. Stop and ask the user. The blast radius of a confident-sounding fabrication in docs is months of bad behavior in user code, support tickets, and trust erosion. Asking is cheaper than every alternative.
