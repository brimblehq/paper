# Two-factor authentication

Add a second factor to your Brimble account. Brimble supports **TOTP codes** (Google Authenticator, Authy, 1Password, Bitwarden, etc.) and **passkeys** (Touch ID, Windows Hello, hardware security keys).

You can use either, both, or neither. They're configured independently in your account's **Security** settings.

Brimble also requires step-up authentication for several sensitive operations: deleting a project, deleting a custom domain, rotating a database password, transferring a domain out, and transferring team ownership.

## Enable TOTP

1. In the dashboard, click your avatar → **Account settings → Security**.
2. Find **Two-factor authentication**. The current state shows "2FA is not enabled".
3. Click **Enable**.

Brimble walks you through three steps:

### Step 1, Scan

A QR code appears. Scan it with Google Authenticator, Authy, 1Password, or any TOTP app.

If your authenticator app doesn't support QR codes, copy the **manual setup key** shown below the code and paste it into the app.

Click **Continue**.

### Step 2, Verify

Enter the 6-digit code your authenticator app shows.

If the code is rejected, you have a clock-drift problem on the device generating it. Sync the device's clock and try the next code.

Click **Verify**.

### Step 3, Save your recovery codes

Brimble generates a set of recovery codes, single-use 8-character codes you can use instead of a TOTP code if you lose access to your authenticator app. **They're shown once.**

You have two options on this screen:

- **Copy all**, copy the codes to your clipboard.
- **Download**, save them as `brimble-2fa-recovery-codes.txt`.

Tick **I have saved my recovery codes** and click **Done** to close the modal. Brimble blocks the close action until the box is ticked, on purpose, losing both your authenticator and your recovery codes means a manual support recovery.

{% hint style="info" %}
**Image needed:** screenshot of the "Save your recovery codes" step showing the recovery codes grid, Copy all and Download buttons, and the "I have saved my recovery codes" checkbox
{% endhint %}

After this, the Security panel shows **2FA is enabled** with the count of recovery codes remaining.

## Use TOTP at sign-in

When 2FA is enabled, signing in works like this:

1. Enter email and password (or click an OAuth provider, or sign in with passkey).
2. Brimble shows the 2FA challenge with a countdown, "This challenge expires in MM:SS".
3. Enter the 6-digit code from your authenticator app.
4. Click **Verify & sign in**.

If you've lost access to your authenticator, click **Use a recovery code instead** at the bottom of the challenge. Enter one of your saved 8-character recovery codes. Each code works once.

## Step-up 2FA

Some actions require 2FA even when you're already signed in. When you trigger one, Brimble pops a 2FA prompt before the action runs:

- Delete a project.
- Delete a custom domain.
- Rotate a database password.
- Transfer a domain out of Brimble.
- Transfer team ownership.

Enter the code or pass the passkey challenge. The step-up is short-lived, repeated sensitive actions in quick succession only prompt once, but stepping up doesn't keep you "elevated" indefinitely.

## Recovery codes

After enabling 2FA, the Security panel shows **Recovery codes remaining: N**. Codes are single-use; the count drops by one every time you use a recovery code (at sign-in, or when disabling/regenerating 2FA).

Brimble warns you when codes are running low. Regenerate a fresh set:

1. Open **Security**.
2. Under **Two-factor authentication**, click **Regenerate recovery codes**.
3. Enter a current TOTP code.
4. Save the newly-generated codes (Copy all or Download).
5. Tick the acknowledgement and click **Done**.

Regenerating invalidates the entire previous set, old codes stop working immediately.

## Disable 2FA

1. Open **Security**.
2. Click **Disable**.
3. Enter a current TOTP code.
4. Confirm.

Disabling 2FA removes the requirement for sign-in **and** for step-up actions. We recommend leaving it on.

## Passkeys

Passkeys are a separate panel under **Security**. They let you sign in (or step up) using Touch ID, Windows Hello, or a hardware security key, no codes to type.

Passkeys can be used **with or without** TOTP. You don't need to enable TOTP first.

### Add a passkey

1. Under **Passkeys**, click **Add passkey**.
2. Type a **device name** so you can recognize it later (e.g. "MacBook", "YubiKey blue").
3. Follow the browser's prompt. On Mac you'll get Touch ID. On Windows, Hello. With a hardware key, plug it in and tap.
4. The new passkey appears in the list with the device name and creation date.

You can add multiple passkeys. **Adding more than one is recommended**, if your only passkey is on a device you lose, you'll need a recovery code to get back in.

### Sign in with a passkey

On the login page:

1. Enter your email.
2. Click **Sign in with passkey**.
3. Pick which passkey to use if you have more than one registered for this domain.
4. Pass the device's authentication prompt.

You're signed in.

### Rename or delete a passkey

In the **Passkeys** list, each passkey has **Rename** and **Delete** buttons.

Brimble blocks **Delete** on your last passkey if 2FA isn't also enabled, it's a guard against locking yourself out. Either enable TOTP first, or add a second passkey before removing the old one.

## Lost your authenticator and your recovery codes

If you have a passkey on a device you can still access, sign in with the passkey. From there, regenerate or disable 2FA.

If you've lost everything:

- The passkey recovery flow at `/passkey-recovery` lets you regain account access using one of your TOTP recovery codes, but that requires having saved them.
- Without the recovery codes either, contact support. Account recovery is manual and may take days.

This is why saving recovery codes during setup is mandatory in the UI, it's your last-line escape.

## Troubleshooting

**TOTP code is rejected even though it looks right.** Your phone's clock is drifting. TOTP relies on accurate time. Sync your phone's clock to a network time source.

**QR code won't scan.** Use the manual setup key shown below the QR. Most apps let you paste the secret directly.

**Passkey prompt doesn't appear.** Your browser or device doesn't support WebAuthn. Update the browser, or fall back to TOTP.

**"Save your recovery codes" Done button is disabled.** Tick the **I have saved my recovery codes** checkbox.

**"Delete" button on a passkey is greyed out.** It's the last passkey on your account and you don't have TOTP enabled. Add another passkey or enable TOTP first.

**Step-up prompt keeps appearing for the same action.** Step-up has a short trust window. Repeated sensitive actions in quick succession should only prompt once; if they're prompting every time, your tab/session may be expiring quickly. Sign in afresh and try again.

## Next steps

- [Manage teams](manage-teams.md), invite team members and manage roles.
- [Create a workspace](create-a-workspace.md), workspaces inherit your account's security settings.
