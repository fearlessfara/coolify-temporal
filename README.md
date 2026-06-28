# Temporal Docker (Coolify-ready)

Self-hosted [Temporal](https://temporal.io/) workflow engine with PostgreSQL persistence, the Temporal Web UI, and admin tools — packaged as a Docker Compose stack designed for [Coolify](https://coolify.io/) deployment.

## What this stack is

This repository runs four services:

| Service | Image | Purpose |
|---------|-------|---------|
| `postgres` | `postgres:16-alpine` | Persistent storage for Temporal |
| `temporal` | `temporalio/auto-setup` | Temporal server (auto-provisions schema) |
| `temporal-ui` | `temporalio/ui` | Web UI for workflows and namespaces |
| `temporal-admin-tools` | `temporalio/admin-tools` | CLI tools (`tctl`, `temporal`) for administration |

**Port model:**

- **Internal only:** PostgreSQL (`5432`), Temporal gRPC frontend (`7233`), admin-tools (no ports)
- **Public via Coolify:** Temporal UI (`8080` inside the container, proxied by Coolify/Traefik to your domain)

The Temporal gRPC port (`7233`) is **not** published to the host. Workers and clients must connect over the Docker network or through a private tunnel.

## Coolify deployment steps

1. **Create a new resource** in Coolify → **Docker Compose** → point to this repository (or paste the compose file).

2. **Set environment variables** in Coolify using [`.env.example`](.env.example) as a reference. The compose file includes defaults for image tags and Postgres settings, so a minimal deploy works without any env vars — but you **should** set at least `POSTGRES_PASSWORD` (and `TEMPORAL_CORS_ORIGINS` once you assign a domain).

3. **Deploy the stack.** Coolify will build/pull images and start all four services on a shared Docker network.

4. **Assign a domain to `temporal-ui`:**
   - In Coolify, open the `temporal-ui` service.
   - Add a domain (e.g. `temporal.example.com`).
   - Map it to **container port `8080`** (not host port — Coolify attaches to the container network).

5. **Update CORS** — set `TEMPORAL_CORS_ORIGINS` to your Coolify domain, e.g.:
   ```
   TEMPORAL_CORS_ORIGINS=https://temporal.example.com
   ```

6. **Redeploy** after changing env vars.

7. **Register a namespace** (see below) before running workflows.

## Required environment variables

- `POSTGRES_USER` — PostgreSQL username (default: `temporal`)
- `POSTGRES_PASSWORD` — PostgreSQL password (**change this**)
- `POSTGRES_DB` — PostgreSQL database name (default: `temporal`)
- `TEMPORAL_VERSION` — Tag for `temporalio/auto-setup` and `temporalio/admin-tools` (default: `1.29.6`). Must exist on **both** images — their tag sets differ; see [Docker Hub](https://hub.docker.com/r/temporalio/auto-setup/tags).
- `TEMPORAL_UI_VERSION` — Tag for `temporalio/ui` (default: `2.51.1`)
- `TEMPORAL_CORS_ORIGINS` — Allowed browser origins for the UI (set to your Coolify domain in production)

> **Note:** `temporalio/auto-setup` also creates a `temporal_visibility` database automatically for advanced visibility queries.

## Connecting workers from another Coolify service

Workers running in a separate Coolify service can reach Temporal over the internal Docker network.

### Option A: Connect to the same predefined network

1. In Coolify, find the Docker network name used by this Temporal stack (often `<project-name>_temporal-network` or similar).
2. In your worker service settings, enable **Connect to Predefined Network** and select that network.
3. Point your worker/client at:
   ```
   TEMPORAL_ADDRESS=temporal:7233
   ```
   The hostname `temporal` resolves to the Temporal frontend container on the shared network.

### Option B: Same Coolify project / compose group

If Coolify places both services on the same user-defined network, use the same address:

```
temporal:7233
```

### Local development against a running stack

If you need to connect from your laptop (not on the Docker network), use an SSH tunnel or Tailscale — **do not** expose port `7233` publicly:

```bash
# Example: tunnel through your Coolify host
ssh -L 7233:temporal:7233 user@your-coolify-host
# Then connect to localhost:7233 locally
```

## Register a namespace

Temporal requires at least one namespace before you can run workflows. The `default` namespace is not created automatically.

### Using `tctl` (legacy CLI)

```bash
docker compose exec temporal-admin-tools tctl --ns default namespace register
```

Verify:

```bash
docker compose exec temporal-admin-tools tctl namespace list
```

### Using `temporal` CLI (modern)

```bash
docker compose exec temporal-admin-tools temporal operator namespace create \
  --namespace default \
  --address temporal:7233
```

List namespaces:

```bash
docker compose exec temporal-admin-tools temporal operator namespace list \
  --address temporal:7233
```

## Check the UI

1. Open the domain you assigned to `temporal-ui` in Coolify (e.g. `https://temporal.example.com`).
2. The UI connects to `temporal:7233` internally — no extra configuration needed.
3. Select your namespace (e.g. `default`) from the namespace dropdown.
4. You should see an empty workflow list until workers are connected and workflows are started.

For local testing without Coolify, you can temporarily add a port mapping to `temporal-ui` in a local override file — but remove it before deploying to production.

## Backup notes for Postgres volume

Temporal state lives in the `postgres-data` named volume. The actual volume name on disk is prefixed by your Compose/Coolify project name, e.g. `<project>_postgres-data`.

### Logical backup (recommended)

```bash
docker compose exec postgres pg_dump -U temporal -d temporal > temporal-backup.sql
docker compose exec postgres pg_dump -U temporal -d temporal_visibility > temporal-visibility-backup.sql
```

### Volume snapshot

```bash
# Replace <project>_postgres-data with your actual volume name (docker volume ls)
docker run --rm \
  -v <project>_postgres-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/postgres-data-$(date +%Y%m%d).tar.gz -C /data .
```

### Restore

Stop the stack, restore the volume or replay `pg_dump` output, then redeploy.

> **Tip:** Schedule regular backups. Losing the Postgres volume means losing all workflow history.

## Security notes

- **Never expose port `7233` publicly.** The Temporal frontend has no built-in authentication. Anyone who can reach it can start workflows, query history, and administer the cluster.
- **Only expose the UI** through Coolify/Traefik on port `8080`, and consider enabling Coolify's built-in authentication or putting the UI behind an SSO/reverse-proxy auth layer.
- **Use strong passwords** for `POSTGRES_PASSWORD`. Rotate them periodically.
- **Keep image tags pinned** via `TEMPORAL_VERSION` and `TEMPORAL_UI_VERSION` — avoid `latest` in production.
- **Restrict network access:** only services that need Temporal (workers, your app) should join the Temporal Docker network.
- **Set `TEMPORAL_CORS_ORIGINS`** to your actual domain — do not use `*` in production.

## Production notes

> **Warning:** The Temporal frontend gRPC port (`7233`) must remain **private and internal** in production. Exposing it to the internet without protection is a critical security risk.

Acceptable ways to reach `7233` from outside the Docker network:

- **VPN** (WireGuard, OpenVPN)
- **Tailscale** or similar mesh network
- **mTLS** with a sidecar/proxy that terminates TLS and authenticates clients
- **SSH tunnel** for admin/debug access only

Do **not** add `ports: "7233:7233"` to `docker-compose.yml` or publish it through Coolify unless you have one of the above protections in place.

For high-availability production deployments, consider Temporal Cloud or the [official Temporal Helm charts](https://github.com/temporalio/helm-charts) with proper mTLS, monitoring, and multi-node clustering. This compose stack is suitable for self-hosted single-node setups behind Coolify.

## Node.js SDK connection example

Install the Temporal TypeScript SDK:

```bash
npm install @temporalio/client @temporalio/worker @temporalio/workflow @temporalio/activity
```

### Worker (runs on the same Docker network as Temporal)

```typescript
import { NativeConnection, Worker } from '@temporalio/worker';
import * as activities from './activities';

async function run() {
  const connection = await NativeConnection.connect({
    address: process.env.TEMPORAL_ADDRESS ?? 'temporal:7233',
  });

  const worker = await Worker.create({
    connection,
    namespace: 'default',
    taskQueue: 'my-task-queue',
    workflowsPath: require.resolve('./workflows'),
    activities,
  });

  await worker.run();
}

run().catch((err) => {
  console.error(err);
  process.exit(1);
});
```

Set in your worker service environment:

```
TEMPORAL_ADDRESS=temporal:7233
```

### Client (start workflows)

```typescript
import { Client, Connection } from '@temporalio/client';

async function startWorkflow() {
  const connection = await Connection.connect({
    address: process.env.TEMPORAL_ADDRESS ?? 'temporal:7233',
  });

  const client = new Client({
    connection,
    namespace: 'default',
  });

  const handle = await client.workflow.start('myWorkflow', {
    taskQueue: 'my-task-queue',
    workflowId: 'workflow-' + Date.now(),
    args: ['hello'],
  });

  console.log('Started workflow:', handle.workflowId);
}

startWorkflow().catch(console.error);
```

### Local dev (via SSH tunnel)

If tunneling `7233` to your machine:

```typescript
const connection = await NativeConnection.connect({
  address: 'localhost:7233',
});
```

## Sharing this repo publicly

If you publish this repository (GitHub, GitLab, etc.):

- **Never commit `.env`.** It contains database passwords and other secrets.
- **Only share `.env.example`.** It documents required variables with placeholder values.
- **Rotate any secrets** that were accidentally committed — treat them as compromised.
- The [`.gitignore`](.gitignore) in this repo excludes `.env` by default; verify it is in place before your first push.

## Quick start (local)

```bash
cp .env.example .env
# Edit .env and set a strong POSTGRES_PASSWORD

docker compose up -d

# Wait for services to become healthy, then register the default namespace
docker compose exec temporal-admin-tools tctl --ns default namespace register

# Optionally expose UI locally via override (not for production)
# echo 'services:\n  temporal-ui:\n    ports:\n      - "8080:8080"' > docker-compose.override.yml
# docker compose up -d
```

## License

Use and modify freely. Temporal is licensed separately — see [temporalio/temporal](https://github.com/temporalio/temporal).
