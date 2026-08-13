# n8n

n8n workflow automation, queue mode (main + worker + external task runner). Standalone-usable
with plain Docker Compose (bundles its own Postgres + Valkey), or deployed as a tenant app on
Coolify using this fleet's shared Postgres and shared Valkey instead.

## Standalone

```bash
cp .env.example .env
# set N8N_ENCRYPTION_KEY and N8N_RUNNERS_AUTH_TOKEN in .env (openssl rand -hex 32 each)
docker compose up -d
```
Editor at http://localhost:5678.

## On Coolify

Deployed with `docker_compose_location` set to `/docker-compose.coolify.yaml`, which joins
the `coolify` external network to reach the shared `postgresql` and `valkey` instances by
service name, and gets a public domain on the `n8n` service only.

## Database and queue

Needs a dedicated Postgres database (`n8n`) on the shared `postgresql` instance and a
dedicated Valkey ACL user (`n8n_user`, scoped to the `n8n:*` key pattern) on the shared
`valkey` instance — see the Hub's README "Adding a tenant app" runbook (its "If it needs
Postgres" and "If it needs Valkey" steps) for the exact provisioning commands.
