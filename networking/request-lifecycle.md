# Request lifecycle

What happens between a user typing your URL and your service returning a response. This page covers the request path end-to-end so you can debug latency, errors, and unexpected behavior with the right mental model.

Brimble runs every public request through **Cloudflare** before it reaches Brimble's own infrastructure. Cloudflare provides DDoS protection, edge caching, and the secure tunnel into Brimble's network. From your service's perspective, the request comes from Brimble's gateway — but the first hop, every time, is Cloudflare.

## High-level flow

```
User
 │  1. DNS resolves your-domain.com → Cloudflare edge
 ▼
Cloudflare (global edge)
 │  2. TLS handshake
 │  3. WAF + DDoS filtering, optional caching
 │  4. Tunnel into Brimble
 │
 ▼
Brimble Gateway (the proxy)
 │  5. Strip dangerous headers
 │  6. Match the hostname to a Brimble domain
 │  7. Apply ingress rules (redirect? maintenance? auth?)
 │  8. Inject Brimble headers
 │  9. Resolve which container instance to send to
 │ 10. Forward
 │
 ▼
Your service
 │ 11. Code runs, returns response
 │
 ▼
Gateway → Cloudflare → User
```

The next sections cover what each step actually does.

## 1. DNS

Your domain's DNS resolves to **Cloudflare's** edge. For default URLs (`<project>.brimble.app`), Brimble already has the wildcard pointed at Cloudflare. For a custom domain, your CNAME pointing at `gateway.brimble.app` (or an A record at the apex) ultimately resolves to Cloudflare's anycast IPs.

Until DNS propagates, no other layer matters. If `dig your-domain.com +short` doesn't resolve to Cloudflare's edge, the request never enters Brimble's flow.

## 2. Cloudflare TLS

Cloudflare terminates TLS at the user-facing edge using a Let's Encrypt certificate Brimble provisions for the hostname. For Brimble-managed domains the cert is issued automatically; for custom domains the cert is issued the first time the domain points at the edge and renewed before expiry.

After TLS, the request is plaintext HTTP between Cloudflare and Brimble's gateway, but it's tunneled — see step 4.

## 3. WAF, DDoS, and caching

Before forwarding, Cloudflare:

- **Filters DDoS and abusive traffic** at the network and application layers. A request that triggers Cloudflare's filters never reaches Brimble.
- **Optionally serves from cache** for cacheable responses. By default, Brimble configures the cache to leave HTML uncached and respect cache headers for static assets. When a deployment goes active, Brimble purges the cache for the project's hostnames so users see the new version immediately.
- **Rate-limits** abusive single-source traffic. The limits are independent of your project's compute size — they're an infrastructure-level guardrail.

A small set of paths (websocket upgrades, gRPC, anything with auth headers) is always passed through untouched.

## 4. Cloudflare tunnel into Brimble

Cloudflare reaches Brimble's gateway through a **Zero Trust tunnel**. Each Brimble server runs a tunnel agent that registers with Cloudflare; Cloudflare sends the request through the tunnel to the right server. There's no public IP for Brimble's gateway — it's only reachable through Cloudflare.

What this means for you:

- Your service's origin IP is never directly exposed.
- DDoS attacks hit Cloudflare's edge, not Brimble.
- Brimble can move servers around without DNS changes.

## 5. Brimble Gateway: header strip

Once inside, the gateway is the first thing in Brimble's own network to see the request. It removes a small set of dangerous request headers — used by frameworks for internal-only signaling that should never be set by an external client:

- `x-react-router-spa-mode`
- `x-react-router-prerender-data`
- `x-middleware-subrequest`

If a client tries to set one, the gateway drops it before your code runs.

## 6. Hostname matching

The gateway looks up the request's hostname against the domain table. Three outcomes:

- **Match found, project attached** — proceed.
- **Match found, no project** — return 404 "Deployment Not Found".
- **No match** — return a generic "not connected to Brimble" page.

The hostname is an exact match. `app.example.com` is a different lookup from `example.com`.

## 7. Ingress rules

Before the request reaches your container, the gateway runs through several gates in order. Each can short-circuit the request.

### Project disabled

If the project is paused or disabled (typically due to billing), the gateway returns **403 Deployment Disabled**.

### Maintenance mode

If you've toggled maintenance on for the project, the gateway returns **200** with a maintenance page.

### Password protection

If the project has password protection on, the gateway checks for a session cookie (`x-brimble-session`). Missing or invalid → **401** with a password prompt page. Successful login sets the cookie; subsequent requests pass through until the cookie expires.

### MCP authentication

For MCP server projects with authentication enabled, the gateway checks the `x-brimble-key` header. Missing or invalid → **401 Unauthorized**.

### Redirect

If the domain has a redirect URL configured, the gateway returns the configured status code (301, 302, 307, or 308) with the destination in the `Location` header.

### Deployment not yet live

If the project hasn't finished its first deployment, or the latest deployment is `failed` and there's no previous active version, the gateway returns **202** (still deploying) or **409** (failed).

## 8. Header inject

The gateway adds and rewrites a few headers before forwarding to your container:

| Header | Value |
|---|---|
| `x-forwarded-proto` | `https` (always — even if the connection between Cloudflare and Brimble was plaintext) |
| `x-forwarded-ssl` | `on` |
| `host` | rewritten to your domain |
| `x-brimble-host` | the gateway server that handled the request |
| `x-brimble-id` | a fresh UUID for this request — log this for correlation |
| `x-brimble-project-version` | an ISO timestamp identifying which deployment is serving |

The original `x-forwarded-for` and `x-real-ip` headers from Cloudflare carry the user's real IP — your code can read them to know who's connecting.

For document requests (`Accept: text/html`, `Sec-Fetch-Dest: document`), conditional cache headers (`if-none-match`, `if-modified-since`, `if-match`, `if-unmodified-since`, `if-range`) are stripped — the edge always serves fresh HTML, never a 304. Static assets keep their conditional headers.

## 9. Backend resolution

The gateway picks which container instance to send the request to.

- **Single-container projects** — the request goes to the project's container at its internal IP and port.
- **Multi-container projects** (with an autoscaling group) — the gateway uses Brimble's internal service discovery to find healthy instances and picks the container with the fewest active connections (least-connection algorithm). If that container errors at connection time, the gateway falls back to another instance.

The selection happens per request — sticky sessions aren't the default. If your service depends on session affinity, store session state in a database or shared cache, not in the container's memory.

## 10. Forward to your container

The gateway opens a connection to your container's address and forwards the request, including:

- The original method, path, and query string.
- The original body (with content-length preserved).
- All headers, after the strip and inject above.

For uploads (`Content-Type: multipart/form-data` or large bodies), the gateway streams the body through without buffering.

For WebSockets (paths like `/ws/*`, `/socket.io/*`, or any request with `Upgrade: websocket`), the gateway upgrades the connection and proxies bidirectionally.

For gRPC (`Content-Type: application/grpc*`), the gateway forwards using HTTP/2 cleartext (h2c).

## 11. Your service handles the request

Your code runs. It has access to:

- All your environment variables (resolved — `{{shared.X}}` and `{{@slug.X}}` references are already substituted).
- The injected headers from step 8.
- The standard `process.env.PORT` to bind to.

The response goes back through the gateway, which:

- Sets `Cache-Control: no-store` for HTML document responses.
- Strips `etag` and `last-modified` on document responses.
- Forwards everything else as-is.

Static assets keep their cache headers untouched, which lets Cloudflare cache them on the way out.

## 12. Back to the user

The response travels back: gateway → Cloudflare tunnel → Cloudflare's edge → user's browser. Cloudflare may cache cacheable responses (static assets) at the edge so the next user gets it without round-tripping to Brimble.

Total round trip: typically tens to hundreds of milliseconds, dominated by:

- **User-to-Cloudflare latency** — short, since Cloudflare has anycast presence near most users.
- **Cloudflare-to-Brimble** — depends on the Brimble region.
- **Your service's processing time** — the only thing you fully control.
- **Edge-to-container traffic** — generally microseconds for in-region traffic.

## Debugging request flow

When a request behaves unexpectedly, work through the layers in order:

1. **DNS first.** `dig your-domain.com +short` — does it resolve to Cloudflare's edge?
2. **TLS.** `curl -I https://your-domain.com` — does it return any HTTP response, or fail at the connection level?
3. **Cloudflare interception.** A response with `Server: cloudflare` and a Cloudflare error page (1xxx codes) means Cloudflare blocked or couldn't reach Brimble.
4. **Brimble hostname match.** Does the response say "not connected" or render a Brimble error page? Check the project's domain attachment.
5. **Ingress rules.** Look at the response — 401 (auth), 403 (disabled), 200 maintenance page (maintenance), 3xx redirect (redirect rule).
6. **Backend.** Open the project in the dashboard. Is it deployed? Active? Are runtime logs flowing?
7. **Application.** If the request reaches your service, the answer is in your application logs.

The `x-brimble-id` header on every response is a unique ID for the request — log it on your side and Brimble's side correlates the same request across both.

## Next steps

- [Networking and the edge](overview.md) — limits and edge behavior.
- [Internal services](internal-services.md) — how your services reach each other privately.
- [502 errors](../troubleshooting/502-errors.md) — what to check when the gateway can't reach your service.
- [DNS troubleshooting](../troubleshooting/dns.md) — when the request doesn't even reach Brimble.
