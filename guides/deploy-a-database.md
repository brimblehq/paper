# Deploy a database

Provision a managed database on Brimble. You pick an engine, version, region, and size; Brimble runs the database, gives you a connection string, and manages the volume.

## Prerequisites

- A Brimble account with a paid plan, or a free-tier project (some engines are restricted on free).
- A project that will use the database. The database can live on its own; you wire it into your service via an environment variable.

## Supported engines

| Engine | Common use |
|---|---|
| **PostgreSQL** | Default relational database. Pick this if you don't have a strong reason. |
| **MySQL** | Relational. Use if your stack expects MySQL specifically. |
| **MariaDB** | MySQL-compatible drop-in. |
| **MongoDB** | Document store. |
| **Redis** | In-memory key/value. Caching, sessions, queues. |
| **Valkey** | Drop-in Redis fork. |
| **RabbitMQ** | Message broker. |
| **Neo4j** | Graph database. |
| **ClickHouse** | Columnar analytics database. |

Versions available per engine are shown in the dashboard during provisioning.

## Provision

1. Open the dashboard.
2. Click **New project** → **Database**.
3. Pick the engine and version.
4. Choose a region. Co-locate the database with the service that will use it — cross-region database calls add latency to every query.
5. Pick a sizing tier (CPU, memory, storage). Start small; you can resize later.
6. Set a name. Use lowercase with dashes (e.g. `acme-pg-prod`).
7. Click **Provision**.

Provisioning typically takes 60–120 seconds. The dashboard shows the status: `provisioning` → `active`.

![TODO: screenshot of the database provisioning dialog with engine, version, region, and size selected](./images/PLACEHOLDER.png)

*The database provisioning flow.*

## Get the connection string

Once active, the database's overview page shows:

- **Connection string** — a fully-formed URL you can plug into your app.
- **Host**, **Port** — the public endpoint, plus a private endpoint for in-region traffic.
- **User** and **Password** — credentials.
- **Database name** — the default database (for engines that have the concept).

Click the eye icon to reveal the password. Copy the connection string into your service's environment as `DATABASE_URL` (or whatever your code expects).

## Connect from a Brimble service

In the service that needs the database:

1. Go to **Environment**.
2. Add a variable: `DATABASE_URL` set to the connection string.
3. Redeploy.

If the service and the database share a region, use the **private** endpoint — it's faster and doesn't count against bandwidth. Otherwise, use the public endpoint.

## Restrict who can connect

Public endpoints are reachable from anywhere by default. To restrict access to specific source IPs:

1. Open the database project.
2. Go to **Networking** (or **Whitelisted IPs**).
3. Add the IPs or CIDR ranges that should be allowed.

When the whitelist is empty, connections are open from any source. With at least one entry, only those sources can connect. Service-to-service traffic from your other Brimble projects in the same region uses the private endpoint and isn't subject to the public whitelist.

## Resize a database

CPU, memory, and storage can be scaled up at any time. Storage can only grow, not shrink.

1. Open the database project.
2. Go to **Settings** → **Sizing**.
3. Pick the new size and click **Apply**.

Scaling causes a short restart (typically under 30 seconds). Connections drop and reconnect — make sure your client has reconnect logic.

## Backups

Backups can be enabled on any database from **Settings → Backups**. When enabled, Brimble takes scheduled snapshots and retains them for the period configured on the project. You can also take an on-demand backup with **Back up now**.

Restoring from a backup happens via support today — open a ticket with the database project and the snapshot timestamp.

## Rotate the password

To rotate the database password:

1. Open the database project.
2. Go to **Settings** → **Credentials**.
3. Click **Rotate password**.

Rotating requires 2FA. The new password takes effect immediately; existing connections are dropped. Update the connection string in any service using the database, then redeploy that service.

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

**Connection works locally but fails from the service.** Make sure the service and database are in the same region for private-endpoint use, or that you're using the public endpoint with TLS otherwise.

**Provisioning failed.** Open the database project. The status page shows the provisioning state and any error. Common causes: region out of capacity, name conflict, payment issue. Click **Retry** on the project to attempt provisioning again.

## Next steps

- [Manage environment variables](environment-variables.md) — wiring the connection string into a service.
- [Service types reference](../reference/service-types.md) — full database options.
