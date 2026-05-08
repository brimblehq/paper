# DNS record types

The record types Brimble's authoritative DNS supports for managed domains. Manage them in the dashboard under **Domains → DNS** or via the API.

## Supported types

| Type | What it does | Example value |
|---|---|---|
| **A** | Maps a hostname to an IPv4 address. | `1.2.3.4` |
| **AAAA** | Maps a hostname to an IPv6 address. | `2001:db8::1` |
| **CNAME** | Aliases one hostname to another. | `gateway.brimble.app` |
| **MX** | Specifies a mail server for the domain. | `10 mail.example.com` |
| **NS** | Delegates a subdomain to other nameservers. | `ns1.other.com` |
| **TXT** | Arbitrary text. Used for SPF, DKIM, DMARC, domain verification. | `"v=spf1 include:_spf.google.com ~all"` |
| **SPF** | Alias for a specific TXT use case. Most providers use TXT instead. | `"v=spf1 -all"` |

## Fields per record

Every record has:

| Field | Meaning |
|---|---|
| **Type** | One of the above. |
| **Host** | The subdomain. Use `@` for the apex (root domain). Use `www` for `www.example.com`. |
| **Answer** | The value (IP, hostname, text, etc.). |
| **TTL** | Time-to-live in seconds. Default 3600 (1 hour). Lower for fast changes; higher for stable records. |
| **Proxied** | Only on `A` and `CNAME` records. When on, traffic to the host routes through Brimble's edge. |

## Apex (root) records

Standard DNS doesn't allow a `CNAME` at the apex (`example.com` itself). Brimble works around this:

- If you set a `CNAME` at `@`, Brimble stores it but synthesizes an `A` record pointing at the edge so resolvers see something they can use.
- In practice, set the apex record as `A` proxied to the edge, and set `www` as `CNAME` to `gateway.brimble.app`.

## Proxied vs not

A record marked **proxied** routes traffic through Brimble's edge. Use proxied records to:

- Get automatic TLS for the hostname.
- Hide the origin IP from public DNS.
- Apply edge features (rate limiting, caching, etc.).

A non-proxied record returns the raw value. Use non-proxied for:

- Mail records (`MX`, `TXT` with SPF/DKIM/DMARC) — never proxy these.
- Records pointing to non-Brimble services (SaaS verification, third-party hosts).

Only `A` and `CNAME` records can be proxied.

## TTL guidelines

| TTL | Use when |
|---|---|
| **300 (5 min)** | About to make a change. Lower the TTL ahead of time so the change propagates quickly. |
| **3600 (1 hour)** | Default. Reasonable for most records. |
| **86400 (1 day)** | Stable records you don't expect to change. Reduces DNS load. |

After making a change, raise the TTL back up to the long value once you're confident.

## Common patterns

### Pointing the apex and www at the same project

```
@        A      <edge IP shown in dashboard>   (proxied)
www      CNAME  gateway.brimble.app             (proxied)
```

Or, if your DNS provider supports CNAME flattening / ALIAS at apex:

```
@        ALIAS  gateway.brimble.app             (proxied)
www      CNAME  gateway.brimble.app             (proxied)
```

### Email through Google Workspace

```
@        MX     1 ASPMX.L.GOOGLE.COM
@        MX     5 ALT1.ASPMX.L.GOOGLE.COM
@        MX     5 ALT2.ASPMX.L.GOOGLE.COM
@        MX     10 ALT3.ASPMX.L.GOOGLE.COM
@        MX     10 ALT4.ASPMX.L.GOOGLE.COM
@        TXT    "v=spf1 include:_spf.google.com ~all"
```

### Subdomain on another nameserver

```
api      NS     ns1.somewhere.com
api      NS     ns2.somewhere.com
```

This delegates `api.example.com` to a different DNS provider. Useful when one team controls the apex and another controls a subdomain.

### Domain verification (Google, Stripe, GitHub, etc.)

```
@        TXT    "google-site-verification=abc123..."
```

Verification records always go on `@` (the apex) unless the provider says otherwise.

## Manage records

In the dashboard, open the domain (under **Domains** in your account or under **Domains** on a project) and use the **DNS** tab to add, edit, or delete records. Each row shows type, host, value, TTL, and whether the record is proxied through Brimble's edge.

## Verify a record

```bash
dig your-domain.com +short
dig api.your-domain.com +short
dig your-domain.com TXT +short
```

If the answer doesn't match what's in the dashboard, propagation is in flight — wait up to the TTL of the previous record. For new records, propagation is usually under 5 minutes.

## Limits

- A domain can hold many records, but a single host can only have one of each type (except `MX`, `NS`, and `TXT`, which support multiple).
- Record names must follow DNS label rules: lowercase letters, digits, dashes; max 63 characters per label, 253 for the full name.
- Proxied records must be `A` or `CNAME`. Other types can't be proxied.
