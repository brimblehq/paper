# Add a custom domain

Point your own domain at a deployed Brimble project. Brimble issues a TLS certificate via Let's Encrypt automatically.

## Prerequisites

* A project already running at `<project-name>.brimble.app`. If you haven't deployed yet, follow the [quickstart](quickstart.md) first.
* A domain you control, with access to its DNS settings.

## Step 1: Add the domain in Brimble

1. Open your project in the [dashboard](https://app.brimble.io).
2. Go to the **Domains** tab.
3. Click **Add domain** and enter the hostname you want to use (e.g. `app.example.com`).

Brimble shows you the DNS record to set at your provider.

![TODO: screenshot of the Add Domain dialog showing the CNAME record Brimble expects](./images/PLACEHOLDER.png)

*The Add Domain dialog shows the exact DNS record to copy.*

## Step 2: Set the DNS record

You have two options at your DNS provider, depending on whether you're pointing a subdomain or an apex domain.

### Option A — CNAME (recommended for subdomains)

Use this for any subdomain like `app.example.com`, `api.example.com`, or `www.example.com`.

| Type  | Name              | Value                  |
|-------|-------------------|------------------------|
| CNAME | `app` (subdomain) | `gateway.brimble.app`  |

### Option B — A record (for apex domains)

Most DNS providers don't allow CNAMEs at the root (`example.com`). For an apex, use the **A record** value shown in the dashboard.

If your DNS provider supports CNAME flattening or ALIAS records (Cloudflare, Route 53, DNSimple, and others), point an `ALIAS` or flattened `CNAME` at `gateway.brimble.app` — that's preferable to a hardcoded A record because it tracks edge IP changes automatically.

## Step 3: Wait for verification

Once DNS propagates, Brimble:

1. Verifies the record resolves to its edge.
2. Requests a TLS certificate from Let's Encrypt.
3. Marks the domain as **active** in the dashboard.

Propagation usually takes a few minutes but can take up to 24 hours, depending on your DNS provider's TTL.

## Verification

Open `https://your-domain.com` in a browser. You should see your app, served over HTTPS with a valid certificate.

From the terminal:

```bash
curl -I https://your-domain.com
```

A successful response shows `HTTP/2 200` (or whatever status your app returns for `/`) and an HTTPS connection that didn't error.

## Troubleshooting

**Domain stuck on "verifying."** DNS hasn't propagated yet. Check from a third party:

```bash
dig your-domain.com +short
```

The output should match the value Brimble showed you. If it doesn't, double-check the record at your DNS provider — typos in the target hostname are the most common cause.

**TLS certificate fails to issue.** Let's Encrypt requires the domain to resolve to Brimble's edge before issuing a certificate. If DNS is correct, Brimble retries automatically; wait a few minutes. If it still fails after an hour, check that no `CAA` record on your domain blocks Let's Encrypt:

```bash
dig your-domain.com CAA +short
```

If a `CAA` record exists and doesn't list `letsencrypt.org`, add `letsencrypt.org` or remove the record.

**Mixed-content warnings in the browser.** Your app is loading some assets over HTTP on an HTTPS page. Audit your code for hardcoded `http://` URLs.

## Removing a domain

In **Domains**, click the trash icon next to the domain. Brimble revokes the certificate and stops accepting traffic for that hostname. Remove or update the DNS record at your provider afterward so it doesn't dangle.

## Manage DNS in Brimble

If you'd rather manage DNS records inside the Brimble dashboard than at an external provider, change your domain's nameservers to:

* `ns1.brimble.io`
* `ns2.brimble.io`

Once those nameservers are active, you can manage all DNS records (A, CNAME, MX, TXT, SPF, NS) for the domain from the dashboard.

## Next steps

* Push a new deployment — your custom domain stays attached across deployments.
* Add a second domain to the same project for staging vs. production hostnames.
