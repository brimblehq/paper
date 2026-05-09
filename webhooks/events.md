# Webhook events reference

Every event Brimble emits, with the exact payload type your endpoint receives. To configure delivery, see [Webhooks](overview.md).

## Envelope

Every webhook delivery is a `POST` with `Content-Type: application/json`. Every event uses the same envelope:

```typescript
interface WebhookPayload<T> {
  event: WebhookEvent;
  subscription_id?: string;   // ObjectId of the Brimble subscription that owns the resource
  data: T;                    // event-specific — see each event below
}
```

There is no top-level `timestamp`, `project`, or `team`. Resource information lives inside `data`.

## Headers Brimble sends

| Header | Value |
|---|---|
| `Content-Type` | `application/json` |
| `X-Brimble-Webhook` | HMAC-SHA256 of the raw request body, hex-encoded. |
| `X-Brimble-Test` | `true`, only present on test events. |

## Common types

These types are referenced from multiple events.

```typescript
// Mongo ObjectIds are always 24-char hex strings on the wire
type ObjectId = string;

// ISO 8601 strings, never Date objects on the wire
type ISODate = string;

enum ServiceType {
  WebService = "web-service",
  Static = "static",
  Worker = "worker",
  Database = "database",
  Mcp = "mcp"
}

enum ProjectStatus {
  Pending = "PENDING",
  InProgress = "INPROGRESS",
  Active = "ACTIVE",
  Failed = "FAILED",
  Cancelled = "CANCELLED",
  Inactive = "INACTIVE",
  Degraded = "DEGRADED",
  Payment = "PAYMENT"
}

enum Environment {
  Production = "PRODUCTION",
  Preview = "PREVIEW"
}

enum GitType {
  GitHub = "github",
  GitLab = "gitlab",
  Bitbucket = "bitbucket",
  Docker = "docker"
}

interface Repo {
  name: string;
  full_name: string;
  id: number;
  branch: string;
  installationId: number;
  git: GitType;
}

interface Specs {
  memory: number;     // MB
  cpu: number;        // vCPU (fractional)
  storage: number;    // MB
  region: ObjectId;   // populated to a Region object on the wire
}

interface Committer {
  name: string;
  email: string;
  username: string;
}
```

## Project payload type

`IProject` is shared by `project.created`, `project.updated`, `project.deleted`, and `deployment.started`.

```typescript
interface ProjectPayload {
  _id: ObjectId;
  name: string;
  uuid: number;
  pid: number;
  serviceType?: ServiceType;
  framework: string;
  status: ProjectStatus;
  description: string;

  // Build configuration
  installCommand: string;
  buildCommand: string;
  startCommand: string;
  preStartCommand?: string;
  outputDirectory: string;
  rootDir?: string;
  watchPaths: string[];
  buildCacheEnabled?: boolean;
  healthCheckPath?: string;

  // Runtime
  port: number;
  ip: string;
  dir: string;
  tlsEnabled: boolean;
  passwordEnabled: boolean;
  authEnabled: boolean;
  password: string | null;
  isPrivate: boolean;
  maintenance: boolean;
  disabled: boolean;
  isPaused: boolean;
  isPaid: boolean;
  billable: boolean;
  hasUpdates?: boolean;

  // Persistence
  diskSize?: number;
  volumeMount: string;
  backupEnabled: boolean;
  last_backup_url?: string;
  last_backup_at?: ISODate;

  // Source
  repo: Repo | null;
  user_id: ObjectId;
  team: ObjectId;

  // Compute
  specs: Specs;
  server: ObjectId;
  nomadJobId: string;
  deployedInCluster: boolean;
  replica_ready: boolean;
  replicas: number;
  autoscaling_group?: ObjectId;

  // Database-specific (when serviceType === Database)
  dbImage?: ObjectId;
  whiteListedIps?: string[];

  // Free-tier
  free_tier_expires_at?: ISODate | null;
  free_tier_expiry_notified_at?: ISODate | null;

  // Misc
  domains: ObjectId[];
  monitor_id: string;
  tracking_token: string;
  from: string;
  inherit_environment_vars: boolean;
  screenshot: { image: string; public_id: string };
  last_requested: ISODate;
  lastProcessed: number;
  container_stats_schedule_id: string | null;

  createdAt: ISODate;
  updatedAt: ISODate;
}
```

`password`, `vaultPath`, and `vaultToken` are present in the database model but stripped from webhook payloads.

## Log payload type

Used by `deployment.success` and `deployment.failed`.

```typescript
interface LogPayload {
  _id: ObjectId;
  name: string;            // branch name
  key: string;             // unique deployment key
  status: ProjectStatus;   // ACTIVE on success, FAILED on failure

  commit: {
    sha: string;
    branch: string;
    message: string;
    committer: Committer;
  };

  project: ObjectId;
  user?: ObjectId;
  team?: ObjectId;
  preview?: ObjectId;       // set for PR-triggered preview deploys
  environment: Environment;

  jobs: Array<{
    id: number;
    name: string;
  }>;

  startTime: ISODate;
  endTime: ISODate;
  createdAt: ISODate;
  updatedAt: ISODate;
  deleted: boolean;
}
```

## Domain payload type

Used by `domain.created`, `domain.purchased`, `domain.transfer_initiated`, and (with extra merged fields) `domain.renewed` and `domain.renewal_failed`.

```typescript
interface DomainPayload {
  _id: ObjectId;
  name: string;
  project: ObjectId;
  user_id: ObjectId;
  team_id?: ObjectId;
  primary: boolean;
  preview?: ObjectId;

  provider: string;                    // domain registrar identifier
  subscription: ObjectId;
  cashier_subscription_id: string | null;

  purchased: boolean;
  auto_renewal: boolean;
  renewal_duration: number;            // years
  renewal_date: string;                // ISO date string
  renewal_price: number;
  is_discounted: boolean;
  privacy_enabled: boolean;

  job_identifier: string;
  trigger_created: boolean;
  trigger_created_at: string;

  nameservers: string[];
  dns: ObjectId[];                     // referenced DNS record IDs

  is_pending_verification: boolean;

  redirect?: {
    url: string;
    status?: 301 | 302 | 307 | 308;
  };

  project_environment?: ObjectId;
  createdAt: ISODate;
  updatedAt: ISODate;
}
```

For `domain.renewed`, the payload merges in the new expiry:

```typescript
interface DomainRenewedPayload extends DomainPayload {
  renewal_date: string;     // updated to the new expiry
  auto_renewal: boolean;    // current auto-renewal setting
}
```

For `domain.transfer_failed` and `domain.transfer_out_completed`, only the domain name is sent:

```typescript
interface DomainTransferLitePayload {
  domain_name: string;
}
```

## DNS payload type

Used by `dns.record.created`, `dns.record.updated`, and `dns.record.deleted`.

```typescript
interface DnsPayload {
  _id: ObjectId;
  name: string;             // host portion ("api", "@", "www")
  type: string;             // "A" | "AAAA" | "CNAME" | "MX" | "NS" | "TXT" | "SPF"
  value: string;            // the record's answer/value
  ttl: number;              // seconds
  domain: ObjectId;         // parent domain ID
  isProxied?: boolean;      // only meaningful for A and CNAME
  createdAt: ISODate;
  updatedAt: ISODate;
}
```

## Environment variable payload type

Used by `environment.variables.added`, `environment.variables.updated`, and `environment.variables.deleted`.

```typescript
interface EnvPayload {
  _id: ObjectId;
  name: string;             // variable name (e.g. "DATABASE_URL")
  value: string;            // see note below
  project: ObjectId;
  user: ObjectId;
  environment: Environment | string;
  is_system?: boolean;
  createdAt: ISODate;
  updatedAt: ISODate;
}
```

**About `value`.** The variable's value is included in the payload. Brimble stores secrets in an encrypted secret store; the `value` field on the webhook payload is the encrypted form for variables that have been written through the secret store (the typical path), and may be plaintext for variables created in legacy flows. Treat the field as sensitive in either case, log handlers should never write `value` to disk or to a log aggregator.

When a bulk import adds many variables in one operation, Brimble emits one `environment.variables.added` event per variable, not a single batched event.

## Autoscaling group payload type

Used by `autoscaling.group.created`, `autoscaling.group.updated`, and `autoscaling.group.deleted`.

```typescript
interface AutoScalingGroupPayload {
  _id: ObjectId;
  name: string;
  user_id?: ObjectId;
  team_id?: ObjectId;
  subscription_id: ObjectId;

  min_containers: number;
  max_containers: number;
  replicas: number;          // current count managed by Brimble
  max_cpu: number;
  max_memory: number;        // MB
  active: boolean;

  meta: Record<string, any>;
  createdAt: ISODate;
  updatedAt: ISODate;
}
```

## Subscription payload type

Used by every `subscription.*` event.

```typescript
interface SubscriptionEventData {
  subscription: {
    id: ObjectId;
    stripe_id: string;
    stripe_status: string;          // "active" | "past_due" | "unpaid" | ...
    plan_type: string;              // "free-plan" | "hacker-plan" | ...
    type: string;                   // subscription "name" — usually "default"
    ends_at?: ISODate;              // present on subscription.canceled
    previous_plan?: string;         // present on subscription.upgraded
    current_plan?: string;          // present on subscription.upgraded
    previous_price?: string;        // present on subscription.downgraded
  };
  user: {
    id: ObjectId;
    email: string;
  };
  team: {
    id: ObjectId;
  } | null;                         // null for personal subscriptions

  cancellation_reason?: string | null;   // subscription.canceled
  failure_reason?: string | null;        // subscription.renewal_failed
  invoice?: {                            // subscription.renewed, subscription.renewal_failed
    stripe_invoice_id: string;
    amount_paid?: number;
    amount_due?: number;
    currency?: string;
    attempt_count?: number;
  };
}
```

## Payment payload type

Used by `payment.successful` and `payment.failed`.

```typescript
interface PaymentEventData {
  invoice: {
    stripe_invoice_id: string;
    amount_paid?: number;            // payment.successful
    amount_due?: number;             // payment.failed
    currency: string;                // "usd", "ngn", etc.
    status: string;                  // "paid" | "open" | ...
    paid?: boolean;                  // payment.successful
    attempt_count?: number;          // payment.failed
  };
  user: {
    id: ObjectId;
    email: string;
  };
  subscription: {
    id: ObjectId;
    stripe_id: string;
    plan_type: string;
    stripe_status: string;
  };
  team: {
    id: ObjectId;
  } | null;

  failure_reason?: string | null;    // payment.failed
}
```

## Event index

### Deployment events

| Event | `data` type |
|---|---|
| `deployment.started` | `ProjectPayload` |
| `deployment.success` | `LogPayload` (status === ACTIVE) |
| `deployment.failed` | `LogPayload` (status === FAILED) |

### Project events

| Event | `data` type |
|---|---|
| `project.created` | `ProjectPayload` (also fires when a database project finishes provisioning) |
| `project.updated` | `ProjectPayload` |
| `project.deleted` | `ProjectPayload` |

### Domain events

| Event | `data` type |
|---|---|
| `domain.created` | `DomainPayload` |
| `domain.purchased` | `DomainPayload` |
| `domain.renewed` | `DomainRenewedPayload` |
| `domain.renewal_failed` | `DomainPayload` |
| `domain.transfer_initiated` | `DomainPayload` |
| `domain.transfer_completed` | `DomainPayload` |
| `domain.transfer_failed` | `DomainTransferLitePayload` |
| `domain.transfer_out_completed` | `DomainTransferLitePayload` |
| `project.domain.updated` | `DomainPayload` |

### Environment variable events

| Event | `data` type |
|---|---|
| `environment.variables.added` | `EnvPayload` |
| `environment.variables.updated` | `EnvPayload` |
| `environment.variables.deleted` | `EnvPayload` |

### DNS record events

| Event | `data` type |
|---|---|
| `dns.record.created` | `DnsPayload` |
| `dns.record.updated` | `DnsPayload` |
| `dns.record.deleted` | `DnsPayload` |

### Autoscaling events

| Event | `data` type |
|---|---|
| `autoscaling.group.created` | `AutoScalingGroupPayload` |
| `autoscaling.group.updated` | `AutoScalingGroupPayload` |
| `autoscaling.group.deleted` | `AutoScalingGroupPayload` |

### Subscription events

| Event | `data` type |
|---|---|
| `subscription.created` | `SubscriptionEventData` |
| `subscription.upgraded` | `SubscriptionEventData` |
| `subscription.downgraded` | `SubscriptionEventData` |
| `subscription.renewed` | `SubscriptionEventData` |
| `subscription.renewal_failed` | `SubscriptionEventData` |
| `subscription.past_due` | `SubscriptionEventData` |
| `subscription.unpaid` | `SubscriptionEventData` |
| `subscription.paused` | `SubscriptionEventData` |
| `subscription.reactivated` | `SubscriptionEventData` |
| `subscription.canceled` | `SubscriptionEventData` |
| `subscription.incomplete_expired` | `SubscriptionEventData` |

### Payment events

| Event | `data` type |
|---|---|
| `payment.successful` | `PaymentEventData` |
| `payment.failed` | `PaymentEventData` |

## Discriminated union for handlers

If you handle multiple events in one TypeScript handler, you can switch on `event` cleanly:

```typescript
type WebhookDelivery =
  | { event: "deployment.started"; subscription_id?: string; data: ProjectPayload }
  | { event: "deployment.success" | "deployment.failed"; subscription_id?: string; data: LogPayload }
  | { event: "project.created" | "project.updated" | "project.deleted"; subscription_id?: string; data: ProjectPayload }
  | { event: "domain.created" | "domain.purchased" | "domain.transfer_initiated" | "domain.transfer_completed" | "project.domain.updated" | "domain.renewal_failed"; subscription_id?: string; data: DomainPayload }
  | { event: "domain.renewed"; subscription_id?: string; data: DomainRenewedPayload }
  | { event: "domain.transfer_failed" | "domain.transfer_out_completed"; subscription_id?: string; data: DomainTransferLitePayload }
  | { event: "environment.variables.added" | "environment.variables.updated" | "environment.variables.deleted"; subscription_id?: string; data: EnvPayload }
  | { event: "dns.record.created" | "dns.record.updated" | "dns.record.deleted"; subscription_id?: string; data: DnsPayload }
  | { event: "autoscaling.group.created" | "autoscaling.group.updated" | "autoscaling.group.deleted"; subscription_id?: string; data: AutoScalingGroupPayload }
  | { event: `subscription.${string}`; subscription_id?: string; data: SubscriptionEventData }
  | { event: "payment.successful" | "payment.failed"; subscription_id?: string; data: PaymentEventData };

function handle(payload: WebhookDelivery) {
  switch (payload.event) {
    case "deployment.success":
    case "deployment.failed":
      // payload.data: LogPayload — narrowed
      console.log(payload.data.commit.sha, payload.data.status);
      break;

    case "environment.variables.added":
      // payload.data: EnvPayload — narrowed
      console.log("env added:", payload.data.name);
      break;

    case "payment.failed":
      // payload.data: PaymentEventData — narrowed
      alertOps(payload.data.failure_reason);
      break;
  }
}
```

## Subscribing

Pass the events you want when configuring a webhook, or `["*"]` to receive everything. Pass `[]` to disable delivery without removing the configuration.

## Delivery model

- **At-least-once.** A delivery may be retried after a failed response. Make handlers idempotent, keying on `data._id` plus `event` works for most resources.
- **Order is not guaranteed.** Two events from the same project can arrive out of order. Use `createdAt` / `updatedAt` / `startTime` / `endTime` from `data` to reconcile.
- **Best-effort.** After repeated delivery failures, a single delivery is dropped. Your webhook itself stays enabled and continues to receive subsequent events.
