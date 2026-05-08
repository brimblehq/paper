# Webhooks

Brimble emits webhooks for deployment, project, domain, environment, DNS, autoscaling, subscription, and payment events. Wire them into your own endpoint, a Discord channel, or a Slack channel.

## Prerequisites

- A subscription on a plan that includes webhooks (Hacker and above).
- An HTTPS endpoint to receive POSTs, or a Discord/Slack incoming-webhook URL.

## How webhooks are configured

Webhooks are configured **per subscription**, not per project. Three URLs can be configured, independently:

| URL field | Format expected | Receives |
|---|---|---|
| `webhookUrl` | An HTTPS endpoint you control | Signed JSON POSTs (one event per request). |
| `discordUrl` | A Discord channel webhook URL (`https://discord.com/api/webhooks/.../...`) | Discord-formatted messages. No signature; Discord authenticates by URL secrecy. |
| `slackUrl` | A Slack incoming webhook URL (`https://hooks.slack.com/services/...`) | Slack-formatted messages. No signature; Slack authenticates by URL secrecy. |

A single events list applies to all three destinations — whichever ones are set.

## Set up a webhook

1. Open the dashboard.
2. Go to **Settings → Webhooks** (or your team's webhook settings, if you're configuring a team subscription).
3. Paste an endpoint URL into one or more of **Webhook URL**, **Discord URL**, or **Slack URL**.
4. Pick the events you want to receive. Use **Select all** to subscribe to every event.
5. Save.

![TODO: screenshot of the Webhooks settings page showing the three URL input fields (Webhook, Discord, Slack), the events checklist grouped by category (Deployment, Project, Domain, Environment, DNS, Autoscaling, Subscription, Payment), and a Save button](./images/PLACEHOLDER.png)

*The Webhooks settings page with the three destination URLs and the event subscription checklist.*

The next matching event delivers to every configured destination.

From the API:

```bash
curl -X PATCH https://api.brimble.io/v1/webhooks \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "webhookUrl": "https://your-endpoint.example.com/brimble",
    "discordUrl": "",
    "slackUrl": "",
    "events": ["deployment.success", "deployment.failed", "payment.failed"]
  }'
```

Pass `["*"]` to subscribe to every event without listing them. Pass `[]` to disable delivery without removing the URL.

## What an HTTP webhook delivery looks like

```http
POST /brimble HTTP/1.1
Host: your-endpoint.example.com
Content-Type: application/json
X-Brimble-Webhook: 9b8a7c…   ← HMAC-SHA256 of the raw body, hex-encoded
X-Brimble-Test: true         ← only on test events

{
  "event": "deployment.success",
  "subscription_id": "65a1f4c2b8d9e7f3a2b1c0d4",
  "data": {
    "_id": "65a2b3c4d5e6f7a8b9c0d1e2",
    "status": "ACTIVE",
    "sha": "a1b2c3d4e5f6...",
    "branch": "main",
    "message": "Add /healthz endpoint",
    "committer": {
      "name": "Ada Lovelace",
      "email": "ada@example.com",
      "username": "adalovelace"
    },
    "startTime": "2026-05-08T14:22:24.000Z",
    "endTime": "2026-05-08T14:23:11.482Z"
  }
}
```

Full payload schema for every event is in [Webhook events reference](../reference/webhook-events.md).

## Verifying signatures

Every HTTP webhook delivery includes `X-Brimble-Webhook`, an HMAC-SHA256 of the raw request body, hex-encoded. Verify it before trusting any payload — anyone can post arbitrary JSON to your endpoint, but only Brimble has the signing secret.

The signing secret is the **subscription's deployment secret key**, available in the dashboard under **Settings → Webhooks → Show secret**.

```javascript
// Express, TypeScript
import { createHmac, timingSafeEqual } from "crypto";
import express from "express";

const app = express();

// Capture the raw body — JSON.parse changes the bytes you signed.
app.use(express.json({
  verify: (req, res, buf) => { (req as any).rawBody = buf; },
}));

const secret = process.env.BRIMBLE_WEBHOOK_SECRET!;

app.post("/brimble", (req, res) => {
  const sig = req.header("X-Brimble-Webhook") ?? "";
  const expected = createHmac("sha256", secret)
    .update((req as any).rawBody)
    .digest("hex");

  const sigBuf = Buffer.from(sig, "hex");
  const expectedBuf = Buffer.from(expected, "hex");

  if (sigBuf.length !== expectedBuf.length || !timingSafeEqual(sigBuf, expectedBuf)) {
    return res.status(401).send("invalid signature");
  }

  const isTest = req.header("X-Brimble-Test") === "true";
  handle(req.body, { isTest });

  res.status(200).end();
});
```

```python
# FastAPI
from fastapi import FastAPI, Request, HTTPException
import hmac, hashlib, os

app = FastAPI()
SECRET = os.environ["BRIMBLE_WEBHOOK_SECRET"].encode()

@app.post("/brimble")
async def brimble(request: Request):
    raw = await request.body()
    expected = hmac.new(SECRET, raw, hashlib.sha256).hexdigest()
    sig = request.headers.get("x-brimble-webhook", "")

    if not hmac.compare_digest(expected, sig):
        raise HTTPException(401, "invalid signature")

    is_test = request.headers.get("x-brimble-test") == "true"
    handle(await request.json(), is_test=is_test)
    return {"ok": True}
```

```go
// Go (net/http)
import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
    "io"
    "net/http"
    "os"
)

var secret = []byte(os.Getenv("BRIMBLE_WEBHOOK_SECRET"))

func brimble(w http.ResponseWriter, r *http.Request) {
    body, _ := io.ReadAll(r.Body)
    mac := hmac.New(sha256.New, secret)
    mac.Write(body)
    expected := hex.EncodeToString(mac.Sum(nil))
    sig := r.Header.Get("X-Brimble-Webhook")
    if !hmac.Equal([]byte(expected), []byte(sig)) {
        http.Error(w, "invalid signature", http.StatusUnauthorized)
        return
    }
    // ... handle
    w.WriteHeader(http.StatusOK)
}
```

The single biggest mistake is signing against `JSON.parse(...)` output instead of the raw bytes. JSON parsing reorders keys, normalizes whitespace, and re-encodes — the signature won't match. Capture the raw body before any parser touches it.

## Handling test events

Test events have:

- The exact same envelope and signature as a real event.
- An additional `X-Brimble-Test: true` header.
- A payload **you supplied** when calling **Send test event** (or `POST /v1/webhooks/test`) — Brimble doesn't synthesize one for you.

If your handler has side effects (sending emails, writing to a database), short-circuit on the test header:

```javascript
if (req.header("X-Brimble-Test") === "true") {
  console.log("test event received:", req.body.event);
  return res.status(200).end();
}
```

## Discord and Slack delivery

Discord and Slack URLs are pre-authenticated by URL secrecy — Brimble doesn't sign requests to them. Each event is rendered as a channel message:

- **Discord** — embed-style message with project name, event, and short summary.
- **Slack** — block-formatted message with the same fields.

You can configure all three URLs at once. The same event delivers to each independently — if your HTTP endpoint times out, the Discord/Slack messages still post.

## Delivery semantics

- **At-least-once.** A delivery may be retried if your endpoint fails or times out. Make your handler idempotent — keying on `data._id` + `event` works for most resources.
- **Order is not guaranteed.** Two events from the same project can arrive out of order. Use `createdAt`/`updatedAt`/`startTime`/`endTime` from `data` to reconcile.
- **Best-effort delivery.** After repeated failures, a delivery is dropped — your webhook stays enabled and continues to receive future events.
- **No webhook is automatically disabled** by Brimble. If your endpoint has been broken for hours, deliveries are dropped, but new events keep being attempted.

## Test a webhook

Click **Send test event** on the webhook in the dashboard, or:

```bash
curl -X POST https://api.brimble.io/v1/webhooks/test \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-endpoint.example.com/brimble",
    "type": "webhook",
    "payload": {
      "event": "deployment.success",
      "subscription_id": "65a1f4c2b8d9e7f3a2b1c0d4",
      "data": { "_id": "test-deployment", "status": "ACTIVE" }
    }
  }'
```

`type` is one of `webhook` (signed, with `X-Brimble-Test: true`), `discord` (unsigned, formatted for a Discord channel), or `slack` (unsigned, formatted for a Slack channel). The test request times out after 10 seconds and is not retried.

## Verification

After saving a webhook, push a commit. Within seconds you should see:

- `deployment.started` deliver as the build begins.
- `deployment.success` (or `deployment.failed`) deliver when the deployment completes.

If you don't see them:

1. Check **Settings → Webhooks**. The toggle on each URL must be on.
2. Confirm the events list includes the events you expected.
3. Verify your endpoint accepts POSTs and returns within 10 seconds.

## Troubleshooting

**No deliveries at all.** The webhook URL may not be saved. Check **Settings → Webhooks** in the dashboard. Also confirm the events list isn't empty.

**Signature mismatch on every request.** You're signing against the parsed body, not the raw bytes. Capture the body before JSON parsing, sign that.

**Some events arrive, others don't.** The events you're missing aren't in your subscription list. Add them, or pass `["*"]` to subscribe to everything.

**Discord/Slack worked once, now silent.** Discord and Slack rotate or revoke incoming webhook URLs. Generate a new URL and update it in **Settings → Webhooks**.

**Endpoint times out under load.** Acknowledge fast (return 200 immediately), then process asynchronously. Don't run downstream calls inside the request handler.

**Rapid duplicate deliveries.** Most likely your endpoint returned a non-2xx status, triggering a retry. Confirm you return 200 even when the body is malformed — never error inside the handler before sending the response.

## Next steps

- [Webhook events reference](../reference/webhook-events.md) — every event with full payload schema.
- [Deployments](../concepts/deployments.md) — what triggers each `deployment.*` event.
