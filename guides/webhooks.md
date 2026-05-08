# Webhooks

Brimble emits webhooks for deploy, domain, environment, and DNS events. Wire them into Slack, Discord, or your own endpoint to react to platform changes.

## Prerequisites

- A project on a plan that includes webhooks (Hacker and above).
- An endpoint to receive POSTs, or a Slack/Discord channel.

## Add a webhook

1. Open the project.
2. Go to **Settings** → **Webhooks**.
3. Click **Add webhook**.
4. Pick a delivery target:
   - **HTTP endpoint** — your own URL.
   - **Slack** — paste an incoming webhook URL.
   - **Discord** — paste a webhook URL from a channel.
5. Pick which events to subscribe to.
6. Save.

The next matching event delivers to the target.

## Available events

| Event | Fires when |
|---|---|
| `deployment.started` | A new deployment begins building. |
| `deployment.success` | A deployment becomes active. |
| `deployment.failed` | A deployment errors during build or deploy. |
| `project.created` | A new project is created. |
| `project.deleted` | A project is deleted. |
| `project.updated` | A project's settings change. |
| `domain.created` | A custom domain is added. |
| `domain.purchased` | A domain is purchased through Brimble. |
| `domain.renewed` | A purchased domain auto-renews. |
| `domain.renewal_failed` | A domain auto-renewal fails. |
| `environment.variables.added` | An env var is added. |
| `environment.variables.updated` | An env var value changes. |
| `environment.variables.deleted` | An env var is deleted. |
| `dns.record.created` | A DNS record is added. |
| `dns.record.updated` | A DNS record changes. |
| `dns.record.deleted` | A DNS record is removed. |
| `autoscaling.group.created` | An autoscaling group is configured. |
| `autoscaling.group.updated` | An autoscaling group's settings change. |
| `autoscaling.group.deleted` | An autoscaling group is removed. |

See [Webhook events reference](../reference/webhook-events.md) for the payload schema of each event.

## Payload format

Every webhook delivery is a `POST` with `Content-Type: application/json`. The body looks like:

```json
{
  "event": "deployment.success",
  "timestamp": "2026-05-08T14:23:11.482Z",
  "project": {
    "id": "prj_8x21",
    "name": "acme-api"
  },
  "data": {
    "deploymentId": "dep_1k99",
    "commit": {
      "sha": "a1b2c3d",
      "message": "Add /healthz endpoint",
      "branch": "main",
      "committer": "ada@example.com"
    },
    "duration": 47,
    "url": "https://acme-api.brimble.app"
  }
}
```

The `data` shape varies per event. The `event`, `timestamp`, and `project` fields are always present.

## Authentication

For HTTP endpoints, you can configure:

- **Basic auth.** Username and password sent in `Authorization: Basic …`.
- **API key.** Sent in a header you specify.
- **Signed payload.** Brimble HMACs the body with a shared secret you provide and sends the signature in `X-Brimble-Signature`. Verify on your end before trusting the payload.

Pick one when creating the webhook. Slack and Discord URLs are pre-authenticated by URL — no extra auth needed.

## Verifying a signed payload

If you chose signed payloads, verify each request before processing:

```javascript
import { createHmac, timingSafeEqual } from "crypto";

function verify(req, secret) {
  const sig = req.header("X-Brimble-Signature");
  const expected = createHmac("sha256", secret).update(req.rawBody).digest("hex");
  return timingSafeEqual(Buffer.from(sig), Buffer.from(expected));
}
```

Reject requests where the signature doesn't match — they didn't come from Brimble.

## Retries

If your endpoint returns a non-2xx status or fails to respond within 10 seconds, Brimble retries with exponential backoff:

- 1st retry: 1 minute later
- 2nd retry: 5 minutes later
- 3rd retry: 30 minutes later
- 4th retry: 2 hours later
- 5th retry: 12 hours later

After 5 failed retries, the delivery is dropped. The webhook itself is **not** disabled — future events keep being delivered.

## Test a webhook

Click **Send test event** on the webhook in the dashboard, or:

```bash
curl -X POST https://api.brimble.io/v1/webhooks/test \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"webhookId": "<webhook-id>", "event": "deployment.success"}'
```

Brimble sends a synthetic payload to your endpoint with `"_test": true` set in the body, so your code can ignore test events if it should.

## Verification

After adding a webhook, push a commit to trigger a deployment. You should see:

- `deployment.started` deliver as soon as the build begins.
- `deployment.success` (or `deployment.failed`) deliver when the deployment completes.

The webhook page in the dashboard logs every delivery with timestamp, status, and response.

## Troubleshooting

**Webhook never fires.** Check the events list — you may not be subscribed to the event you expect. Also verify the webhook is enabled (the toggle on the row).

**Webhook fires but my endpoint never gets it.** Check delivery logs in the dashboard. If they show a non-2xx response, your endpoint rejected the request. If they show a timeout, Brimble couldn't reach the URL or your endpoint took too long to respond.

**Signature verification fails on every request.** The most common cause is parsing the body before verifying — `JSON.parse` followed by `JSON.stringify` produces different bytes than what was signed. Verify against the raw request body, not the parsed object.

**Slack/Discord webhook stopped working.** Slack and Discord rotate or revoke incoming webhook URLs. Generate a new one and update the webhook in Brimble.

## Next steps

- [Webhook events reference](../reference/webhook-events.md) — every event and its full payload schema.
- [Deployments](../concepts/deployments.md) — what triggers each `deployment.*` event.
