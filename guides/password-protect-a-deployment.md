# Password-protect a deployment

Put a password in front of a Brimble project. Requests without correct basic auth credentials get a 401 at the edge.

Use this for staging deployments, internal tools, or any project that shouldn't be world-readable. The password check happens at the edge — no requests reach your service without it.

## Prerequisites

- A project deployed on Brimble.

## Enable password protection

1. Open the project.
2. Go to **Settings** → **Access**.
3. Toggle **Password protection** on.
4. Set a username and password. Save.

The next request to the project URL will see a `401 Unauthorized` with a `WWW-Authenticate: Basic` challenge. Browsers prompt for credentials; programmatic clients send them in the `Authorization` header.

## Test it

```bash
curl -I https://<project-name>.brimble.app
# HTTP/2 401

curl -I -u username:password https://<project-name>.brimble.app
# HTTP/2 200
```

In a browser, opening the URL pops a credentials dialog.

## Update the password

Same panel: **Settings** → **Access** → **Update password**. The new password applies immediately. Existing sessions in browsers (which cache basic auth) keep working until the browser is restarted; cached credentials match either the old or the new value depending on browser behavior.

For sensitive projects, plan a rotation by:

1. Setting a new password.
2. Telling everyone to clear their browser's cached credentials for the URL.
3. Confirming the old password no longer works.

## Disable password protection

Toggle **Password protection** off. The project becomes publicly accessible immediately.

## Apply only to some paths

The dashboard toggles password protection on the entire project. To allow specific paths through unauthenticated (e.g., `/healthz` for monitoring), implement that in your application — return 200 without checking auth, and let your other routes rely on the edge's basic auth.

This works because basic auth at the edge applies to **every** request. If your app exposes a healthcheck path that doesn't depend on auth, monitoring tools can probe it by including credentials, or you can leave password protection off and use [app-level auth](#app-level-authentication) for the protected routes.

## App-level authentication

For more control (per-user accounts, sessions, role-based access), implement auth in your code instead of edge basic auth. Common stacks:

- Express + Passport
- Django + django-allauth
- Rails + Devise
- A reverse proxy or auth provider (Auth0, Clerk, Cognito) sitting in front of your app

Disable password protection in Brimble when using app-level auth — otherwise users will be prompted twice.

## Verification

```bash
curl -I https://<project-name>.brimble.app
```

Without credentials, you should get `HTTP/2 401` and a `WWW-Authenticate: Basic` header. With credentials (`-u user:pass`), `HTTP/2 200`.

## Troubleshooting

**Browser doesn't prompt for credentials.** It cached a "no auth" response from before you enabled the protection. Clear site data for the URL, or open in a private window.

**Credentials are correct but I still see 401.** Make sure you're hitting the right hostname. The protection applies to the project's hostnames (default and custom). If you're hitting an old hostname that's no longer attached, you'll see a generic 404 or a "not connected" page, not a 401.

**Other 401s come from inside my app.** Edge basic auth and app-level auth both return 401. Check the response body — Brimble's edge returns a short auth-required page, while your app returns whatever you've coded.

## Next steps

- [Manage environment variables](environment-variables.md) — credentials your app uses internally.
- [Networking](../concepts/networking.md) — how the edge enforces auth.
