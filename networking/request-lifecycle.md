# Request lifecycle

What happens between a user typing your URL and your service returning a response. This page covers the request path end-to-end so you can debug latency, errors, and unexpected behavior with the right mental model.

## High-level flow

```
User
 │  1. DNS resolves your-domain.com → Brimble edge IPs
 ▼
Edge (TLS termination + ingress)
 │  2. TLS handshake — Let's Encrypt cert
 │  3. Match the hostname to a Brimble domain
 │  4. Apply ingress rules (redirect? maintenance? auth?)
 │
 ▼
Gateway (the proxy)
 │  5. Strip dangerous headers
 │  6. Inject Brimble headers
 │  7. Resolve which container instance to send to
 │  8. Forward
 │
 ▼
Your service
 │  9. Code runs, returns response
 │
 ▼
Gateway → Edge → User
```

The next sections cover what each step actually does.

## 1. DNS

Your domain's DNS resolves to Brimble's edge. For default URLs (`<project>.brimble.app`), this is a wildcard A record under Brimble's nameservers. For a custom domain, your CNAME points at `gateway.brimble.app` (or an A record at the apex).

Until DNS propagates, the edge never sees the request. If `dig your-domain.com +short` doesn't return Brimble's edge, no other layer matters.

## 2. TLS

The edge terminates TLS using a Let's Encrypt certificate. For Brimble-managed domains the cert is issued automatically; for custom domains the cert is issued the first time the domain points at the edge and renewed before expiry.

After TLS, the request is plain HTTP/2 internally. Your service speaks HTTP/1.1 or HTTP/2 (h2c for gRPC) — never raw TLS. You don't configure certificates on your container.

## 3. Hostname matching

The edge looks up your hostname against the domain table. Three outcomes:

- **Match found, project attached** — proceed to step 4.
- **Match found, no project** — return 404 "Deployment Not Found".
- **No match** — return a generic "not connected to Brimble" page.

The hostname is an exact match. `app.example.com` is a different lookup from `example.com`.

## 4. Ingress rules

Before the request reaches your container, the gateway runs through several gates in order. Each can short-circuit the request.

### Project disabled

If the project is paused or disabled (typically due to billing), the gateway returns **403 Deployment Disabled**. Your container is not reached.

### Maintenance mode

If you've toggled maintenance on for the project, the gateway returns **200** with a maintenance page. Your container is not reached.

### Password protection

If the project has password protection on, the gateway checks for a session cookie (`x-brimble-session`). If the cookie is missing or invalid, the gateway returns **401** with a password prompt page. Successful login sets the cookie; subsequent requests pass through without prompting until the cookie expires.

### MCP authentication

For MCP server projects with authentication enabled, the gateway checks for the `x-brimble-key` header. Missing or invalid keys get **401 Unauthorized**.

### Redirect

If the domain has a redirect URL configured, the gateway returns the configured status code (301, 302, 307, or 308) with the destination in the `Location` header. Your container is not reached.

## 5. Header strip

Before forwarding, the gateway removes a small set of dangerous request headers — used by frameworks for internal-only signaling that should never be set by an external client:

- `x-react-router-spa-mode`
- `x-react-router-prerender-data`
- `x-middleware-subrequest`

If a client sends one of these, your code never sees it.

## 6. Header inject

The gateway adds a few headers and rewrites a couple more before forwarding:

| Header | Value |
|---|---|
| `x-forwarded-proto` | `https` (always, even if the original request was http) |
| `x-forwarded-ssl` | `on` |
| `host` | rewritten to your domain |
| `x-brimble-host` | the gateway server that handled the request |
| `x-brimble-id` | a fresh UUID for this request — log this for correlation |
| `x-brimble-project-version` | an ISO timestamp identifying which deployment is serving (`project.updatedAt`) |

For document requests (`Accept: text/html`, `Sec-Fetch-Dest: document`), conditional cache headers (`if-none-match`, `if-modified-since`, `if-match`, `if-unmodified-since`, `if-range`) are stripped — the edge always serves fresh HTML, never a 304. Static assets (JS, CSS, images, fonts) keep their conditional headers and respect normal cache behavior.

## 7. Backend resolution

The gateway picks which container instance to send the request to.

- **Single-container projects** — the request goes to the project's container at its known IP and port.
- **Multi-container projects** (with an autoscaling group) — the gateway picks the container with the fewest active connections (least-connection algorithm) and increments its connection count. If that container errors at connection time, the gateway falls back to another container.

The selection happens per request — sticky sessions aren't the default. If your service depends on session affinity, store session state in a database or a shared cache (Redis), not in the container's memory.

## 8. Forward to your container

The gateway opens a connection to your container's IP and port and forwards the request — including:

- The original method, path, and query string.
- The original body (with content-length preserved).
- All headers, after the strip and inject above.

For uploads (`Content-Type: multipart/form-data` or large bodies), the gateway streams the body through without buffering the full payload.

For WebSockets (paths like `/ws/*`, `/socket.io/*`, or any request with `Upgrade: websocket`), the gateway upgrades the connection and proxies bidirectionally.

For gRPC (`Content-Type: application/grpc*`), the gateway forwards using HTTP/2 cleartext (h2c) so streaming methods work.

## 9. Your service handles the request

Your code runs. It has access to:

- All your environment variables (resolved — `{{shared.X}}` and `{{@slug.X}}` references are already substituted).
- The injected headers from step 6.
- The standard `process.env.PORT` to bind to.

The response goes back through the gateway, which:

- Sets `Cache-Control: no-store` for HTML document responses.
- Strips `etag` and `last-modified` headers on document responses.
- Forwards everything else as-is.

Static assets keep their cache headers untouched.

## 10. Back to the user

The response travels back through the edge, gets re-encrypted with TLS, and lands in the user's browser.

Total round trip: typically tens to hundreds of milliseconds, dominated by:

- **User-to-edge network latency** — depends on how close the user is to your project's region.
- **Your service's processing time** — the only thing you control.
- **Edge-to-container traffic** — generally microseconds for in-region traffic.

## Debugging request flow

When a request behaves unexpectedly, work through the layers in order:

1. **DNS first.** `dig your-domain.com +short` — does it point at Brimble?
2. **TLS.** `curl -I https://your-domain.com` — does it return any HTTP response, or fail at the connection level?
3. **Edge hostname match.** Does the response say "not connected" or render a Brimble error page? Check the project's domain attachment.
4. **Ingress rules.** Look at the response — 401 (auth), 403 (disabled), 200 maintenance page (maintenance), 3xx redirect (redirect rule).
5. **Backend.** Open the project in the dashboard. Is it deployed? Active? Are runtime logs flowing?
6. **Application.** If the request reaches your service, the answer is in your application logs.

The `x-brimble-id` header on every response is a unique ID for the request — log it on your side and Brimble's side correlates the same request across both.

## Next steps

- [Networking and the edge](networking.md) — limits and edge behavior.
- [502 errors](../troubleshooting/502-errors.md) — what to check when the gateway can't reach your service.
- [DNS troubleshooting](../troubleshooting/dns.md) — when the request doesn't even reach Brimble.
