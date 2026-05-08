# DNS issues

DNS is the layer between users typing your domain name and reaching your project. When DNS is wrong, the symptoms vary widely — domain stuck on "verifying," intermittent 404s, the wrong site loading, no connection at all. This page indexes the symptoms.

## Quick triage

Three commands cover most DNS debugging:

```bash
dig your-domain.com +short              # what's the current answer
dig your-domain.com +trace              # where in the chain does it resolve
dig your-domain.com CAA +short          # any CAA records blocking TLS
```

Run them from a machine **not** on the same network as your DNS provider. Local network caches can show stale answers.

## Domain stuck on "verifying" in Brimble

The dashboard says "verifying" because Brimble polls DNS and the record isn't yet what it expects.

**Run:**

```bash
dig your-domain.com +short
```

**Expected:** Whatever value the dashboard shows under **Add domain** (typically `gateway.brimble.app` for a CNAME or a specific edge IP for an A record).

**If the answer is empty:** No DNS record exists yet. Add it at your DNS provider.

**If the answer is something else:** You set the record, but to a wrong value. Common typos: `gateway.brimble.io` instead of `gateway.brimble.app`; an old edge IP from documentation that's been updated.

**If the answer is right but Brimble still says "verifying":** DNS propagation is in flight. Wait. Most providers propagate within minutes; some can take up to 24 hours. While waiting, run:

```bash
dig @8.8.8.8 your-domain.com +short
dig @1.1.1.1 your-domain.com +short
```

If the answer is right from public resolvers (Google's `8.8.8.8`, Cloudflare's `1.1.1.1`), Brimble will see it shortly. If the answer is right from your provider but wrong from public resolvers, the record was just published and hasn't propagated yet.

## TLS certificate fails to issue

Brimble issues a Let's Encrypt certificate after DNS verification. If TLS is stuck:

**Step 1: Confirm DNS resolves correctly.** See above.

**Step 2: Check for a CAA record blocking Let's Encrypt:**

```bash
dig your-domain.com CAA +short
```

If you see a CAA record that doesn't include `letsencrypt.org`, Let's Encrypt won't issue. Add a CAA record allowing it, or remove the existing CAA record:

```
@   CAA   0 issue "letsencrypt.org"
```

**Step 3: Check for a wildcard mismatch.** Wildcard certs (`*.example.com`) require a `_acme-challenge` TXT record for DNS-01 validation. Brimble handles this automatically for managed domains; if you're using external DNS, the dashboard provides the exact TXT record to add.

**Step 4: Wait.** Let's Encrypt has rate limits. If multiple cert requests fail in succession, retries may be backed off for an hour. After that, Brimble retries automatically.

If TLS is still failing after an hour with correct DNS and no CAA blocks, contact support with the domain name.

## Wrong site loading

You point a domain at Brimble, but the site that loads belongs to someone else.

**Cause:** A previous tenant's hostname mapping is still cached at the edge, or DNS is pointing somewhere unexpected.

**Run:**

```bash
dig your-domain.com +short
curl -I https://your-domain.com
```

The `Server` and `X-Brimble-*` response headers tell you whether the edge served the response. If `X-Brimble-Host` shows a different domain than yours, the request was routed to the wrong project — typically because the domain was attached to a project elsewhere on Brimble.

**Fix:** Open the domain in the dashboard. Confirm it's attached to your project. If it's attached elsewhere, contact support — domain ownership may need verification.

## Apex (root) domain doesn't work

You added `example.com` (no subdomain) and it doesn't resolve. Most DNS providers don't allow CNAMEs at the apex.

**Fix:** Use one of these:

- **A record at apex** pointing to the IP shown in the dashboard's **Add domain** dialog.
- **ALIAS or flattened CNAME** at apex if your provider supports it (Cloudflare, Route 53, DNSimple). These let you point at `gateway.brimble.app` and the provider resolves it transparently.

For most users, `www.example.com` (CNAME) plus an apex A record (or ALIAS) plus an HTTP redirect from one to the other is the simplest setup.

## Subdomain works, www doesn't

You added `app.example.com` and it works, but `www.example.com` doesn't.

**Fix:** Add `www.example.com` as a separate domain in Brimble. Each hostname needs its own CNAME and its own attachment to the project. Wildcard custom domains aren't supported.

## CNAME at apex error

DNS providers reject CNAME at the apex with errors like "CNAME at zone apex." This is a hard rule of the DNS spec.

**Fix:** Use the A record value from the dashboard, or your provider's ALIAS / CNAME flattening feature. See [DNS records reference](../domains/dns-records.md).

## DNS records I added in Brimble's dashboard aren't taking effect

You're using Brimble's authoritative DNS (`ns1.brimble.io`, `ns2.brimble.io`) and a record you added isn't visible.

**Step 1:** Confirm your domain's nameservers are actually Brimble's:

```bash
dig your-domain.com NS +short
```

You should see `ns1.brimble.io` and `ns2.brimble.io`. If you see something else, your domain is still using your old DNS provider — Brimble's dashboard records don't apply.

**Step 2:** If nameservers are right, the record might be propagating. Brimble's authoritative DNS publishes within seconds, but resolvers cache the previous answer up to its TTL. Lower the TTL ahead of changes if you can.

**Step 3:** Check that the record host is correct. `@` for the apex; bare subdomain (e.g. `api`) for `api.example.com`, **not** the full `api.example.com`.

## Mixed DNS — some records at provider, some at Brimble

You can't mix authoritative DNS — your domain uses **either** Brimble's nameservers (and all records live in Brimble's dashboard) **or** an external provider (and you set CNAME/A records there). You can't have some records here and some there.

If you want some records managed by Brimble (for proxy routing) and some at an external provider (for mail), pick one as authoritative and add the others to it. There's no "merge" between authoritative DNS sources.

## Email broke after pointing domain at Brimble

You pointed your domain at Brimble for web traffic and now mail isn't reaching you.

**Cause:** The MX records that pointed at your mail provider were lost when you delegated to Brimble's nameservers without copying them.

**Fix:** Copy your mail provider's MX records into Brimble's DNS dashboard. Most mail providers (Google Workspace, Microsoft 365, Proton, Fastmail) publish their MX records on a setup page. Don't proxy MX records — they're for SMTP, not HTTP.

## Verification

After making any DNS change, verify from public resolvers:

```bash
dig @1.1.1.1 your-domain.com +short
dig @8.8.8.8 your-domain.com +short
```

If both return the expected answer, propagation is done.

For TLS:

```bash
curl -I https://your-domain.com
```

A 200 (or any response with a valid certificate) means TLS is working. A connection error or `SSL_ERROR_*` means the certificate isn't installed yet — wait, or check the dashboard.

## Next steps

- [Custom domains](../domains/custom-domains.md) — full setup guide.
- [DNS records](../domains/dns-records.md) — the supported record types and their fields.
- [TLS issues](tls.md) — when DNS is right but the certificate is the problem.
