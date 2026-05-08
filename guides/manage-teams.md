# Manage teams

A **team** holds projects shared across multiple people. Use a team for any project you don't want tied to a single account.

## Prerequisites

- A Brimble account.

## Create a team

1. In the dashboard, click your avatar → **New workspace**.
2. Enter the team name and an optional description. Upload a logo if you want.
3. Click **Create**.

The new team appears in the workspace switcher. Switch into it to see and create projects under that team.

## Roles

Members of a team have one of these roles:

| Role | Permissions |
|---|---|
| **Creator** | Full control. Created the team. Can transfer ownership. |
| **Administrator** | Same as Creator except can't transfer ownership. Manages billing, members, projects. |
| **Member** | Read and write to projects. Can deploy, manage env vars, add domains. Cannot manage billing or members. |
| **Viewer** | Read-only access to projects, deployments, and logs. |

A team always has exactly one Creator. There can be many Administrators, Members, and Viewers.

## Invite members

1. Open the team.
2. Go to **Settings** → **Members**.
3. Click **Invite member**.
4. Enter the email address and pick a role.
5. Click **Send invite**.

The invitee gets an email with a link. If they don't already have a Brimble account, the link prompts them to sign up; otherwise, they accept and are added to the team.

Pending invites stay listed under **Members** until accepted, expired, or revoked. To revoke, click **⋯** → **Revoke invite**.

## Change a member's role

1. Open **Settings** → **Members**.
2. Click the role on the member's row.
3. Pick the new role and save.

## Remove a member

1. Open **Settings** → **Members**.
2. Click **⋯** → **Remove from team** on the member's row.
3. Confirm.

Removed members lose access to all team projects immediately. Their personal projects are not affected.

## Transfer ownership

The Creator role can be transferred to another team member:

1. Open **Settings** → **Ownership**.
2. Pick the Administrator to transfer to.
3. Confirm. Ownership transfer requires 2FA.

After the transfer, the previous Creator becomes an Administrator.

## Billing

Each team has its own billing. Plans on a team apply to projects under that team.

- **Free.** Same limits as a personal free plan, scoped to the team.
- **Hacker / Pro.** Single-seat plans are typically used on personal accounts; for teams, prefer Team.
- **Team plan.** Per-member pricing plus per-concurrent-build pricing. See [Plans](../reference/plans.md).

A user can be in many teams, each with its own plan. Personal projects are billed separately.

## Switch between workspaces

The workspace switcher in the top-left of the dashboard lists your personal account and every team you're a member of. Switch context to see only the projects under that workspace.

## Verification

After inviting a member, ask them to:

1. Accept the invite.
2. Switch into the team in the workspace switcher.
3. Open a project.

If they can see the project list and open one, they're in.

## Troubleshooting

**Invitee never got the email.** Check spam. If still missing, revoke and re-invite. Confirm the email address is right — typos are common.

**Member sees a blank project list after accepting.** The invite went to the wrong email and they accepted with a different account. Remove and re-invite the right address.

**Can't transfer ownership.** Ownership transfers require 2FA on the Creator's account. Set up 2FA under **Settings** → **Security**, then retry.

## Next steps

- [Plans and pricing](../reference/plans.md) — what each plan includes.
- [Two-factor authentication](two-factor-authentication.md) — required for ownership transfer and a few other sensitive actions.
