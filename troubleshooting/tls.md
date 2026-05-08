# TLS certificate issues

Brimble issues TLS certificates from Let's Encrypt automatically. When it fails, the cause is almost always DNS or a CAA record. This page walks the common cases.

## Symptom: connection error in browser

The browser shows `ERR_SSL_PROTOCOL_ERROR`, `SSL_ERROR_NO_CYPHER_OVERLAP`, or "this connection is not private."

**Causes (most likely first):**

1. The certificate hasn't been issued yet.
2. A CAA record on your domain blocks Let's Encrypt.
3. DNS resolves to somewhere other than Brimble's edge.
4. The custom domain isn't actually attached to a project.

## Step 1: Check the dashboard

Open the project's **Domains** tab. Each domain shows a TLS status:

- **Active** — certificate issued, expiring date shown.
- **Pending** — Brimble is requesting the certificate. Wait a minute.
- **Failed** — Let's Encrypt rejected the request. Read the failure reason.
- **Verifying** — DNS hasn't propagated yet; certificate request hasn't started.

If status is **Verifying**, see [DNS issues](dns.md). TLS won't issue until DNS is right.

If status is **Failed**, the dashboard shows a reason — usually a CAA block, a domain mismatch, or a Let's Encrypt rate limit.

## Step 2: Check DNS

```bash
dig your-domain.com +short
```

The answer must point at Brimble's edge — either `gateway.brimble.app` (CNAME) or the edge IP shown when you added the domain (A record).

If the answer is empty or wrong, fix DNS first. TLS issuance fails on the first request and Brimble retries with backoff; aggressive retries can hit Let's Encrypt's rate limits, so don't bash refresh.

## Step 3: Check for CAA records

A CAA (Certification Authority Authorization) record on your domain restricts which CAs can issue certificates for it. If you have a CAA record that doesn't include Let's Encrypt, issuance fails with "issuance forbidden."

```bash
dig your-domain.com CAA +short
```

**No output:** No CAA record. Let's Encrypt is allowed by default.

**Output that includes `letsencrypt.org`:** Let's Encrypt is allowed.

**Output that doesn't include `letsencrypt.org`:** Add Let's Encrypt or remove the restriction:

```
@   CAA   0 issue "letsencrypt.org"
```

If you need other CAs to also issue, add them with additional CAA records.

## Step 4: Check Let's Encrypt rate limits

Let's Encrypt rate-limits per registered domain (the registrable part, e.g. `example.com`):

- **50 certificates per registered domain per week.**
- **5 duplicate certificate requests per week** (same exact set of names).
- **5 failed validations per account per hostname per hour.**

If you're testing repeatedly with a real domain, you can hit the limits. The dashboard surfaces "rate limit exceeded" with a wait time. There's no fix except waiting.

For active development, point a test subdomain (`dev.example.com`) at Brimble — that's a different rate-limit bucket than `example.com` itself.

## Step 5: Confirm domain is attached

If you removed the domain from the project (or it never finished attaching), Brimble won't request a certificate.

In the dashboard:

1. Open **Domains** in the project.
2. The domain should be listed with status **Active** (not just typed in but never saved).

If the domain isn't listed, add it.

## Symptom: certificate is for a different domain

The browser warns the certificate's name doesn't match the URL.

**Cause:** Your DNS resolves to Brimble's edge, but the edge sees a hostname it doesn't recognize as attached. It serves a default certificate (for `*.brimble.app`).

**Fix:** Add the domain to a project in Brimble. After DNS verification and certificate issuance (typically under 60 seconds), the warning goes away.

## Symptom: mixed-content warnings

The page loads over HTTPS, but the browser blocks some resources because they're being requested over HTTP.

**Cause:** Your code has hardcoded `http://` URLs, or it's reading the request's protocol as HTTP because it doesn't trust the proxy.

**Fix 1 — application code:** Replace hardcoded `http://` with `https://`, or use protocol-relative URLs (`//cdn.example.com/...`).

**Fix 2 — framework configuration:** Brimble sets `X-Forwarded-Proto: https` on every forwarded request. Configure your framework to trust it:

- **Express:** `app.set("trust proxy", true)`.
- **Django:** `SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")`.
- **Rails:** `config.force_ssl = true` (also handles redirects).
- **Phoenix:** `config :myapp, MyAppWeb.Endpoint, url: [scheme: "https", port: 443]`.

## Symptom: certificate expired

Brimble renews certificates automatically, well before expiry. If you see an expired certificate:

**Run:**

```bash
echo | openssl s_client -servername your-domain.com -connect your-domain.com:443 2>/dev/null | openssl x509 -noout -dates
```

If the `notAfter` is in the past, the cert wasn't renewed. The most common cause is that DNS broke between issuance and renewal — the renewal couldn't validate the domain.

**Fix:** Run through DNS troubleshooting above. Once DNS is right, Brimble retries the renewal within minutes.

## Verification

After fixing the underlying cause, force a fresh attempt:

```bash
curl -I https://your-domain.com
```

A successful response with HTTPS and a 2xx/3xx/4xx/5xx status (anything HTTP-level, not a connection error) means TLS is working. The browser will agree.

You can also verify the certificate's issuer and expiry:

```bash
echo | openssl s_client -servername your-domain.com -connect your-domain.com:443 2>/dev/null \
  | openssl x509 -noout -issuer -dates -subject
```

The issuer should be Let's Encrypt; `notAfter` should be ~90 days out.

## Still stuck

Contact support with:

- The full domain name.
- The output of `dig your-domain.com +short`.
- The output of `dig your-domain.com CAA +short`.
- The TLS status shown in the dashboard.

## Next steps

- [Custom domains](../domains/custom-domains.md) — initial setup.
- [DNS issues](dns.md) — when DNS itself is wrong.
