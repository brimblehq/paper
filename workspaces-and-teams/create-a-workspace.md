# Create a workspace

A **workspace** holds projects shared with a team. Workspaces are billed separately from your personal account and have their own seats, concurrent builds, and projects.

This guide covers creating a paid team workspace from scratch. To invite people into an existing workspace, see [Manage teams](manage-teams.md).

## Prerequisites

- A Brimble account with 2FA recommended (the workspace owner has full control of billing).
- A payment method on file. The workspace is charged immediately on creation.

## Step 1: Name and slug

1. Click your avatar → **New workspace**.
2. Fill in:
   - **Workspace image** — optional logo (PNG, JPG, or WebP).
   - **Workspace name** — the human name (e.g. "Acme Engineering").
   - **Workspace URL** — the slug used in URLs. Auto-generated from the name; edit if you need something specific. The slug is the part after `brimble.app/` in workspace links.
3. Click **Continue**.

![TODO: screenshot of step 1 of workspace creation showing the image upload, name field, and URL slug input prefixed with brimble.app/](./images/PLACEHOLDER.png)

*Workspace name and slug.*

## Step 2: Size and concurrent builds

This step sets the workspace's monthly cost.

| Setting | Description |
|---|---|
| **Team size** | Number of seats. Pick from 3, 5, 10, 15, 25, or 50. Each seat costs **$5/month**. |
| **Concurrent builds** | How many builds can run in parallel. Each concurrent build container costs **$7.50/month**. |
| **Startup promo code** | Optional. If you have one, enter it and click **Verify**. Valid codes apply a discount; invalid ones show an error. |

The page shows a running total: `seats × $5 + builds × $7.50 = $X/month`.

A workspace with 5 seats and 2 concurrent builds is `5 × $5 + 2 × $7.50 = $40/month`.

You can change both values later under **Workspace settings → Billing**, but you can't go below the seats currently in use without removing members first.

Click **Continue**.

## Step 3: Invite members

Invite the people who'll be in the workspace. For each row:

- **Email** — the invitee's address. They'll get an email with a link.
- **Role** — pick from `Member`, `Administrator`, or `Viewer`. The Creator role is reserved for the workspace owner.

Click **Add another** to invite more. You can leave this step empty and invite people later.

The page shows a billing summary: seats cost + builds cost + total. If you've added more invites than your team size, a warning appears: "You've invited N members but only have M seats — additional seats will be charged."

You can't invite yourself — Brimble blocks the email matching your account.

## Step 4: Create

Click **Create workspace**. The card on file is charged immediately for the first month.

After creation:

- The workspace appears in the workspace switcher in the top-left.
- Pending invites send out as emails.
- You become the **Creator** of the workspace.

## After creation

Things you'll typically do next:

- **Create a project** under the workspace. Switch into the workspace via the switcher and click **New project**.
- **Set up webhooks** for the workspace under **Settings → Webhooks**.
- **Configure 2FA** on your account if you haven't — required for ownership transfer.
- **Adjust limits.** Bumping seats or builds is one click in **Settings → Billing**.

## Switching between workspaces

The workspace switcher in the top-left of the dashboard groups your workspaces:

- **Personal Accounts** — your individual workspace.
- **Teams** — every team workspace you're a member of.

Switching context filters the project list, the dashboard home, and most settings to the active workspace.

## Roles and what each can do

| Role | Permissions |
|---|---|
| **Creator** | Full control. Manages billing, members, projects. Can transfer ownership. One per workspace. |
| **Administrator** | Same as Creator except can't transfer ownership. |
| **Member** | Read and write to projects. Deploy, manage env vars, add domains. Cannot manage billing or members. |
| **Viewer** | Read-only across projects, deployments, logs. |

## Verification

After creating the workspace:

1. Switch into it from the workspace switcher.
2. Confirm you see an empty project list.
3. Open **Settings → Members** and confirm pending invites are listed.
4. Open **Settings → Billing** and confirm the next charge date and current seats/builds.

## Troubleshooting

**"Card declined" on creation.** The card on file failed when Brimble tried to charge for the first month. Add a different card under **Billing → Payment methods** and try again. The workspace doesn't get created until payment goes through.

**Promo code says "invalid."** Codes are case-sensitive and time-limited. Re-check exactly as provided. If it's still rejected, contact whoever gave you the code — promo codes can be revoked or expire without warning.

**Invitee never got the email.** Check spam. Revoke the pending invite from **Settings → Members** and re-invite to the right address. Typos are the most common cause.

**Want to start small.** Pick 3 seats and 0 concurrent builds. You can upgrade later as the team grows. Note: 0 concurrent builds means workspace projects queue builds indefinitely, like the personal free plan — generally pick at least 1.

## Next steps

- [Manage teams](manage-teams.md) — invite people, change roles, transfer ownership.
- [Plans and pricing](../billing/plans.md) — full pricing breakdown including overages.
- [Two-factor authentication](two-factor-authentication.md) — required for ownership transfer.
