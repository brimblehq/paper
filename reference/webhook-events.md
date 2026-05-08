# Webhook events reference

Every event Brimble can emit, with payload schema. To subscribe, see [Webhooks](../guides/webhooks.md).

## Common envelope

Every webhook delivery has the same outer shape:

```json
{
  "event": "<event-name>",
  "timestamp": "<ISO 8601>",
  "project": {
    "id": "<project-id>",
    "name": "<project-name>"
  },
  "team": {
    "id": "<team-id-or-null>",
    "name": "<team-name-or-null>"
  },
  "data": { /* event-specific */ }
}
```

The `data` field varies per event.

## Deployment events

### `deployment.started`

A new deployment has begun building.

```json
{
  "event": "deployment.started",
  "data": {
    "deploymentId": "dep_1k99",
    "environment": "Production",
    "commit": {
      "sha": "a1b2c3d",
      "message": "Add /healthz endpoint",
      "branch": "main",
      "committer": {
        "name": "Ada Lovelace",
        "email": "ada@example.com",
        "username": "adalovelace"
      }
    }
  }
}
```

### `deployment.success`

A deployment is now active and serving traffic.

```json
{
  "event": "deployment.success",
  "data": {
    "deploymentId": "dep_1k99",
    "environment": "Production",
    "commit": { /* same shape as above */ },
    "duration": 47,
    "url": "https://acme-api.brimble.app"
  }
}
```

`duration` is in seconds, from clone start to traffic flip.

### `deployment.failed`

A deployment errored.

```json
{
  "event": "deployment.failed",
  "data": {
    "deploymentId": "dep_1k99",
    "environment": "Production",
    "commit": { /* ... */ },
    "phase": "build",
    "error": "exit code 1: npm ERR! ..."
  }
}
```

`phase` is one of `clone`, `detect`, `install`, `build`, `package`, `push`, `deploy`, `health`. `error` is a short summary; full logs are in the dashboard.

## Project events

### `project.created`

```json
{
  "event": "project.created",
  "data": {
    "serviceType": "web-service",
    "region": "fra1"
  }
}
```

### `project.updated`

Settings on the project changed (build commands, health check path, sizing, etc.).

```json
{
  "event": "project.updated",
  "data": {
    "changedFields": ["buildCommand", "startCommand"]
  }
}
```

### `project.deleted`

```json
{
  "event": "project.deleted",
  "data": {
    "deletedAt": "2026-05-08T14:23:11.482Z"
  }
}
```

## Domain events

### `domain.created`

A custom domain was attached to a project.

```json
{
  "event": "domain.created",
  "data": {
    "domain": "app.example.com"
  }
}
```

### `domain.purchased`

A domain was purchased through Brimble.

```json
{
  "event": "domain.purchased",
  "data": {
    "domain": "example.com",
    "price": 12.99,
    "currency": "USD",
    "renewalDate": "2027-05-08T00:00:00.000Z"
  }
}
```

### `domain.renewed`

A purchased domain auto-renewed successfully.

```json
{
  "event": "domain.renewed",
  "data": {
    "domain": "example.com",
    "price": 12.99,
    "currency": "USD",
    "newRenewalDate": "2028-05-08T00:00:00.000Z"
  }
}
```

### `domain.renewal_failed`

A purchased domain's auto-renewal failed (typically a payment issue).

```json
{
  "event": "domain.renewal_failed",
  "data": {
    "domain": "example.com",
    "expiresAt": "2026-05-15T00:00:00.000Z",
    "reason": "card_declined"
  }
}
```

## Environment variable events

### `environment.variables.added`

```json
{
  "event": "environment.variables.added",
  "data": {
    "environment": "Production",
    "variables": ["DATABASE_URL", "STRIPE_KEY"]
  }
}
```

Values are not included in webhook payloads. Names only.

### `environment.variables.updated`

```json
{
  "event": "environment.variables.updated",
  "data": {
    "environment": "Production",
    "variables": ["STRIPE_KEY"]
  }
}
```

### `environment.variables.deleted`

```json
{
  "event": "environment.variables.deleted",
  "data": {
    "environment": "Production",
    "variables": ["OLD_TOKEN"]
  }
}
```

## DNS events

### `dns.record.created`

```json
{
  "event": "dns.record.created",
  "data": {
    "domain": "example.com",
    "record": {
      "type": "A",
      "host": "api",
      "answer": "1.2.3.4",
      "ttl": 3600
    }
  }
}
```

### `dns.record.updated`

```json
{
  "event": "dns.record.updated",
  "data": {
    "domain": "example.com",
    "record": { /* ... */ },
    "previousRecord": { /* ... */ }
  }
}
```

### `dns.record.deleted`

```json
{
  "event": "dns.record.deleted",
  "data": {
    "domain": "example.com",
    "record": { /* ... */ }
  }
}
```

## Autoscaling events

### `autoscaling.group.created`

```json
{
  "event": "autoscaling.group.created",
  "data": {
    "min": 2,
    "max": 10,
    "strategy": "linear",
    "metric": "cpu",
    "scaleUpThreshold": 70,
    "scaleDownThreshold": 30
  }
}
```

### `autoscaling.group.updated`

Same shape as `created`, plus `previousConfig` showing the prior values.

### `autoscaling.group.deleted`

```json
{
  "event": "autoscaling.group.deleted",
  "data": {
    "deletedAt": "2026-05-08T14:23:11.482Z"
  }
}
```

## Test events

When you click **Send test event**, the payload includes `"_test": true` at the top level. Use this to ignore tests in handlers that have side effects.

```json
{
  "_test": true,
  "event": "deployment.success",
  "timestamp": "...",
  "project": { /* ... */ },
  "data": { /* ... */ }
}
```

## Delivery guarantees

- **At-least-once.** Brimble retries failed deliveries up to 5 times. Your handler should be idempotent.
- **Order is not guaranteed.** Two events from the same project can arrive out of order. Use the `timestamp` field to reconcile.
- **10-second timeout.** Your endpoint must respond within 10 seconds, even if processing is async. Acknowledge fast, handle later.
