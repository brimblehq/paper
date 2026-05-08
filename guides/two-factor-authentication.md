# Set up two-factor authentication

Add a second factor to your Brimble account using a TOTP app (Google Authenticator, 1Password, Authy, Bitwarden, etc.) or a passkey.

Brimble requires 2FA for several sensitive operations: deleting a project, deleting a domain, rotating a database password, transferring a domain out, and transferring team ownership. Setting it up once unlocks all of them.

## Prerequisites

- A TOTP app installed on your phone or in your password manager — **or** a passkey-capable device (Touch ID / Face ID on Mac, Windows Hello, hardware security key, etc.).

## Set up TOTP

1. Open the dashboard.
2. Click your avatar → **Account settings** → **Security**.
3. Under **Two-factor authentication**, click **Enable TOTP**.
4. Scan the QR code with your TOTP app, or copy the secret manually.
5. Enter the 6-digit code your app shows. Click **Verify**.
6. Brimble shows you 8 recovery codes. **Save these somewhere safe** — they're the only way back into your account if you lose access to your TOTP app.
7. Click **Done**.

2FA is now active on your account. Future logins prompt for a TOTP code.

![TODO: screenshot of the 2FA setup screen with a QR code and recovery codes shown](./images/PLACEHOLDER.png)

*The TOTP setup screen.*

## Set up a passkey

A passkey lets you sign in or step up to a sensitive action with a fingerprint, face scan, or hardware key — no codes to type.

1. Go to **Account settings** → **Security**.
2. Under **Passkeys**, click **Add passkey**.
3. Follow your browser's prompt. On Mac, it'll suggest Touch ID. On Windows, Hello. With a hardware key, plug it in and tap.
4. Name the passkey (e.g. "MacBook", "YubiKey blue") so you can recognize it later.

You can add multiple passkeys. Adding more than one is recommended — if your only passkey is on a device you lose, you'll need a recovery code to get back in.

## Recovery codes

Brimble generates 8 recovery codes when you enable TOTP. Each code is single-use. They're shown **once**, at setup. Store them in a password manager or print them and lock them up.

To regenerate (invalidating the old set):

1. Go to **Security**.
2. Click **Regenerate recovery codes** under TOTP.
3. Re-authenticate with your current TOTP code.
4. Save the new codes.

## Sign in with 2FA

After 2FA is enabled, login flows like this:

1. Enter email and password (or click an OAuth provider).
2. Brimble prompts for a TOTP code or passkey challenge.
3. Enter the 6-digit code from your app, or complete the passkey prompt.
4. You're signed in.

You get **5 TOTP attempts** per challenge. After 5 wrong codes, the challenge locks for a cooling-off period.

## Step-up authentication

Some actions require 2FA even when you're already signed in:

- Deleting a project.
- Deleting a custom domain.
- Rotating a database password.
- Transferring a domain out of Brimble.
- Transferring team ownership.

When you trigger one of these, Brimble pops a 2FA prompt before the action runs. Enter the code or pass the passkey challenge. The step-up is good for a few minutes — repeated sensitive actions in quick succession only prompt once.

## I lost my 2FA device

If you lose access to your TOTP app:

1. Go to the login page.
2. Enter your email and password.
3. On the 2FA prompt, click **Use a recovery code**.
4. Enter one of your saved recovery codes.

Each code works once. After signing in with a recovery code, regenerate your TOTP setup and a new set of codes.

If you've also lost your recovery codes:

- If you have a passkey on a device you can still access, use that.
- Otherwise, contact support. Account recovery without 2FA is manual and may take days.

## Disable 2FA

1. Go to **Security**.
2. Click **Disable two-factor authentication**.
3. Enter your current TOTP code (or use a passkey or recovery code).
4. Confirm.

Disabling 2FA removes the requirement for sign-in *and* step-up actions. We recommend leaving 2FA on.

## Verification

To confirm setup is correct, sign out and sign back in. You should be prompted for the second factor. If you're not, 2FA isn't actually enabled — repeat the setup.

## Troubleshooting

**TOTP code is rejected even though it looks right.** Your phone's clock is drifting. TOTP relies on accurate time. Sync your phone's clock to a network time source.

**QR code won't scan.** Use the manual setup secret instead. Most apps let you paste a string directly.

**Passkey prompt doesn't appear.** Your browser or device doesn't support WebAuthn. Update your browser, or use TOTP.

**Locked out of recovery code prompt.** After too many wrong attempts, the prompt locks. Wait 15 minutes and try again, or contact support.

## Next steps

- [Manage teams](manage-teams.md) — invite team members and manage roles.
- [Account security best practices](../reference/security.md) — full list of recommendations.
