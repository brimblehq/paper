# Deploy a database

Provision a managed database on Brimble. You pick an engine, version, region, and size; Brimble runs the database, gives you a connection string, and handles the volume.

## Prerequisites

- A Brimble account with a paid plan, or a free-tier project (some engines are restricted on free).
- A project to connect the database to. The database can live in its own project; you'll wire it into your service via an environment variable.

## Supported engines

| Engine | Common use |
|---|---|
| **PostgreSQL** | Default relational database. Pick this if you don't have a strong reason. |
| **MySQL** | Relational. Use if your stack expects MySQL specifically (Rails legacy, WordPress). |
| **MariaDB** | MySQL-compatible drop-in. |
| **MongoDB** | Document store. |
| **Redis** | In-memory key/value. Caching, sessions, queues. |
| **Valkey** | Drop-in Redis fork. Pick if you specifically need Valkey. |
| **RabbitMQ** | Message broker. |
| **Neo4j** | Graph database. |
| **ClickHouse** | Columnar analytics database. |

Versions available per engine are shown in the dashboard during provisioning.

## Provision

1. Open the dashboard.
2. Click **New project** → **Database**.
3. Pick the engine and version.
4. Choose a region. Co-locate the database with the service that will use it — cross-region database calls add latency to every query.
5. Pick a sizing tier (CPU, memory, storage). Start small; you can scale later.
6. Set a name. Use lowercase with dashes (e.g. `acme-pg-prod`).
7. Click **Provision**.

Provisioning typically takes 60–120 seconds. The dashboard shows the status: `provisioning` → `active`.

![TODO: screenshot of the database provisioning dialog with engine, version, region, and size selected](./images/PLACEHOLDER.png)

*The database provisioning flow.*

## Get the connection string

Once active, the database's overview page shows:

- **Connection string** — a fully-formed URL you can plug into your app.
- **Host** — the public endpoint (and the private endpoint, for in-region traffic).
- **Port** — the database port.
- **User** and **Password** — credentials.
- **Database name** — the default database (for engines that have the concept).

Click the eye icon to reveal the password. Copy the connection string into your service's environment as `DATABASE_URL` (or whatever your code expects).

## Connect from a Brimble service

In the service that needs the database:

1. Go to **Environment**.
2. Add a variable: `DATABASE_URL` set to the connection string from the database overview.
3. Redeploy.

If the service and the database are in the same region, use the **private** endpoint — it's faster and doesn't count against bandwidth. Otherwise, use the public endpoint over TLS.

## Connect from your laptop

Public endpoints are reachable from anywhere, but you can restrict access to specific IPs:

1. Open the database project.
2. Go to **Networking** or **Whitelisted IPs**.
3. Add your IP or CIDR range.

When the whitelist is empty, connections are open from any source. When it has at least one entry, only those sources can connect.

## Resize a database

You can scale CPU, memory, and storage up at any time. Storage can only grow, not shrink.

1. Open the database project.
2. Go to **Settings** → **Sizing**.
3. Pick the new size and click **Apply**.

Scaling causes a short restart (typically under 30 seconds). Connections drop and reconnect — make sure your client has reconnect logic.

## Replicas and high availability

You can run 1–10 replicas. Set the count under **Settings** → **Replicas**.

- **1 replica** — single instance. Cheapest. Restart causes downtime.
- **2+ replicas** — automatic failover. The connection string points at a load balancer that routes to a healthy replica.

Replication is engine-specific (Postgres streaming replication, MongoDB replica sets, etc.). Read patterns vary; check your engine's documentation for whether reads can hit replicas.

## Backups

Backups can be enabled on any database from **Settings** → **Backups**. When enabled:

- Brimble takes a snapshot on a schedule (daily by default).
- Snapshots are kept for the retention period configured on the project.
- You can take an on-demand backup with **Back up now**, or via:

```bash
curl -X POST https://api.brimble.io/v1/projects/database/backup \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"projectId": "<database-project-id>"}'
```

Restoring from a backup happens via support today. Self-serve restore is on the roadmap.

## Update credentials

To rotate the database password:

1. Open the database project.
2. Go to **Settings** → **Credentials**.
3. Click **Rotate password**.

Rotating the password requires 2FA. The new password takes effect immediately; existing connections are dropped. Update any service using the connection string and redeploy.

## Verification

From a Brimble service, log connection success on startup:

```javascript
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const { rows } = await pool.query("SELECT version()");
console.log("DB connected:", rows[0].version);
```

From your laptop, with the engine's CLI:

```bash
psql "$DATABASE_URL" -c "SELECT version();"
```

## Troubleshooting

**Connection times out.** Either the database isn't yet `active`, or your IP isn't on the whitelist. Check both.

**Connection refused.** The database may have crashed or be restarting after a resize. Check the database project's status in the dashboard.

**Connection works locally but fails from the service.** Make sure the service and database are in the same project's environment, and that you're using the right endpoint (private for in-region, public otherwise). Both endpoints use TLS.

**Provisioning failed.** Open the database project. The status page shows the provisioning logs. Common causes: region out of capacity, name conflict, payment issue. You can retry with `POST /v1/projects/database/retry`.

## Next steps

- [Manage environment variables](environment-variables.md) — wiring the connection string into a service.
- [Service types reference](../reference/service-types.md) — full database options.
