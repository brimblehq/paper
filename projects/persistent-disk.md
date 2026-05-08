# Persistent disk

Attach durable storage to a project so files survive restarts and redeployments. Without a persistent disk, anything your service writes to the filesystem is lost on every redeploy — the new container starts from a fresh image.

Use a persistent disk for:

- SQLite databases (small apps, single-instance).
- File uploads stored to disk before being moved to object storage.
- Cache/state that's expensive to rebuild on cold start.
- Self-hosted tools that need a data directory (Plausible, Umami, n8n, etc.).

## Prerequisites

- A project on a paid plan that includes persistent disks.
- The project must be a single-container service. Persistent disks don't share between containers; if you scale beyond one container, only one of them holds the data.

## Enable a persistent disk

1. Open the project.
2. Go to **Settings → Build** (or **Configuration**).
3. Scroll to **Persistent disk** and toggle it on.
4. Set:
   - **Mount path** — the path inside the container to mount the disk at (e.g. `/data`).
   - **Size** — pick one: 1 GB, 5 GB, 10 GB, 25 GB, 50 GB, or 100 GB.
5. Save.

![TODO: screenshot of the Persistent disk panel under Settings, showing the mount-path input and the size selector with options 1, 5, 10, 25, 50, 100 GB](./images/PLACEHOLDER.png)

*Persistent disk configuration on a project.*

The next deployment provisions the volume and mounts it at the path you set.

## Use the disk

Anything your service writes to the mount path persists across deploys, restarts, and resizes.

```javascript
// Node — write to the mounted disk
import fs from "fs/promises";

const DATA_DIR = "/data";
await fs.writeFile(`${DATA_DIR}/state.json`, JSON.stringify(state));
```

```python
# Python
import os, json

DATA_DIR = "/data"
with open(os.path.join(DATA_DIR, "state.json"), "w") as f:
    json.dump(state, f)
```

The mount is empty on first attach. Initialize whatever directory structure your service needs on startup.

## Resize a disk

Disks can grow but not shrink.

1. Open **Settings → Build → Persistent disk**.
2. Pick a larger size.
3. Save.

The resize happens on the next deployment. The container restarts to pick up the new size; existing data is preserved.

## Limits and constraints

- **Size** caps at 100 GB per project on standard plans. For larger volumes, contact support.
- **One disk per project.** You can't mount multiple persistent disks at different paths.
- **One container per disk.** A persistent disk is a local volume — it isn't shared across replicas. Don't enable autoscaling on a project that depends on a persistent disk for state, or all but one container will operate on stale local data.
- **Region-bound.** A persistent disk lives in the project's region. Moving the project to a different region requires a fresh disk.
- **Backups are your responsibility.** Brimble persists the disk across deployments and host moves but doesn't snapshot it. For data that must survive disaster, copy critical files to object storage on a schedule.

## When NOT to use a persistent disk

A persistent disk is the wrong choice when:

- **You need scale-out.** Multiple containers reading/writing the same dataset want a managed database or object storage, not a local volume.
- **Your data must be backed up automatically.** Use a managed database (PostgreSQL, MongoDB, etc.) — Brimble snapshots those.
- **Your data is huge.** Beyond ~100 GB, object storage with a small metadata DB scales better.

For most production apps, the right answer is "managed database for state, object storage for files, no persistent disk." Persistent disks are best for self-hosted tools, prototypes, and edge cases where local files are genuinely the right model.

## Verification

After enabling the disk and deploying, check the mount from your code:

```javascript
import fs from "fs";
console.log("disk mounted:", fs.existsSync("/data"));
console.log("disk writable:", fs.constants.W_OK & fs.statSync("/data").mode);
```

Or hit a debug endpoint:

```bash
curl https://<project>.brimble.app/debug/disk
```

Where your service exposes a quick stat of the mount.

## Troubleshooting

**Mount path doesn't exist after deploy.** The toggle might be off. Re-check **Settings → Build → Persistent disk** is enabled and the path is what you set.

**Files disappear on deploy anyway.** Files written to a path *outside* the mount don't persist. Confirm your code is writing under the configured mount path (e.g. `/data`, not `/var/data`).

**"Permission denied" writing to the mount.** Some images run as a non-root user that doesn't own the mount. In your Dockerfile, ensure the user has access — for example, `RUN mkdir -p /data && chown myuser:myuser /data`.

**Resize didn't take effect.** Resizes apply on the next deployment, not in place. Click **Redeploy**.

## Next steps

- [Deploy a database](deploy-a-database.md) — for state that needs scale-out, backups, and queryability.
- [Builds](../concepts/builds.md) — how the runtime container is built and started.
