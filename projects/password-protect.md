# Site password protection

Brimble can put a single shared password in front of a project. Visitors see a password prompt before any request reaches your service. Useful for staging deployments, internal tools, and anything that shouldn't be publicly readable.

The password check happens at the edge, no requests reach your container without it.

## How it works

- **One password per project.** No username. Anyone with the password can access the site. It's a single shared secret with no audit trail, so don't use it as the only access control for sensitive data.
- **Form-based login.** A visitor without a session sees a Brimble-rendered password page. They enter the password, submit, and are redirected back to the URL they came from.
- **Session cookie.** On a successful login, the proxy sets an `httpOnly` session cookie (`x-brimble-session`). Subsequent requests carry the cookie and pass through without re-prompting until the cookie expires.
- **Per-hostname.** Cookies are scoped to the hostname. Authenticating on `staging.example.com` doesn't grant access to `app.example.com`. Each protected hostname has its own session.
- **Not HTTP Basic Auth.** Brimble doesn't return a `WWW-Authenticate: Basic` challenge. `curl -u user:pass` won't authenticate, and browsers won't pop a credentials dialog, visitors interact with the password page.

## What visitors see

A request to a password-protected project returns a Brimble password page with a single password field. The visitor enters the password and submits. On success they're redirected to the URL they were trying to reach. On failure the page reloads with an "Invalid credentials" message.

The cookie is `httpOnly`, JavaScript can't read it, so the password page is the only way to obtain a session.

{% hint style="info" %}
**Image needed:** screenshot of the Brimble password page that's shown when an unauthenticated visitor hits a protected project, single password input, submit button, project hostname visible
{% endhint %}

## When it applies

Password protection covers **every request to the project**, top-level URL, asset paths, API endpoints, WebSocket upgrades. There's no per-path bypass.

If you need a public health check on an otherwise-protected project, run an unprotected sibling project at a different hostname for the public endpoint, or implement [app-level auth](#app-level-authentication) instead.

## Checking the current state

In the dashboard, open the project. The overview shows **Site password enabled: Yes** or **Site password enabled: No**. This is read-only on the overview page, it surfaces the current state but doesn't toggle it.

## Enabling, changing, or disabling

A self-serve UI to toggle site password protection or rotate the password isn't currently exposed in the dashboard. To enable, change, or remove it, contact support with the project name and the password you want set. This page will be updated when self-serve controls ship.

## App-level authentication

For more control (per-user accounts, sessions, role-based access), implement auth in your code instead. Common stacks:

- Express + Passport
- Django + django-allauth
- Rails + Devise
- A reverse proxy or auth provider (Auth0, Clerk, Cognito) sitting in front of your app

Use either site password protection **or** app-level auth, not both. Running both means visitors hit two prompts.

## Troubleshooting

**Form keeps rejecting the password.** The password is case-sensitive and has no leading/trailing whitespace. Confirm exactly what was set with whoever configured it.

**Logged in once, but every page reload re-prompts.** Your browser is dropping cookies. Check it accepts cookies from the project hostname, privacy modes and third-party-cookie blocks usually still allow first-party cookies, but extensions can interfere.

**Authenticated for `staging.example.com`, but `app.example.com` re-prompts.** Cookies are per-hostname. Each hostname has its own session.

**`curl -u user:pass` returns the password page.** That's expected, Brimble doesn't read Basic Auth headers. The browser's password page is the only login surface.

## Next steps

- [Networking and the edge](../networking/overview.md), how the edge enforces auth before requests reach your service.
- [Custom domains](../domains/custom-domains.md), password protection works on default and custom domains.
