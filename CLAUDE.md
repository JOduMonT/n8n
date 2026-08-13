# CLAUDE.md — n8n

Deploy config for [n8n](https://github.com/n8n-io/n8n), queue mode with an external task
runner. Read `README.md` first for the standalone-vs-Coolify basics — this file is the
"don't repeat past mistakes" layer.

## This repo must stay tenant-agnostic

`docker-compose.coolify.yaml` must never contain a hardcoded domain, database name beyond
`n8n` (fixed by convention — every tenant gets a database named after the app), or secret.
Every tenant-specific value is a required env var (`${VAR:?VAR must be set}`) supplied via
Coolify environment variables, never committed here.

## A `ports:` block in `docker-compose.coolify.yaml` would be a real security incident

Same class of bug this fleet has hit before (`metamcp`, `qdrant`): publishing a port on the
Coolify variant serves the app in plaintext on the server's public IP, bypassing Traefik/TLS
entirely. Traefik reaches every container over the `coolify` network and auto-selects each
image's own `EXPOSE`d port — a host port mapping is never needed there. The standalone
`docker-compose.yaml` (never deployed, local `docker compose up` only) is fine to publish
ports.

## Version pinning

All three images (`n8nio/n8n` for `n8n`/`n8n-worker`, `n8nio/runners` for `task-runners`)
pin the same concrete tag — check `gh api repos/n8n-io/n8n/releases/latest` for the current
one before bumping, or let Renovate open the PR. `n8nio/runners` publishes a matching tag per
n8n release; if a `-distroless` variant exists for the resolved tag, `task-runners` uses it
(see the hardening section below) — confirm with `docker manifest inspect
n8nio/runners:<tag>-distroless` before assuming it exists for every release.

## Three services, one queue

`n8n` (main, public domain) and `n8n-worker` (internal-only) run the identical image —
`command: worker` is the only difference — and both mount the same `n8n-data` volume at
`/home/node/.n8n`, since both need the same instance data. `task-runners` (internal-only)
is the external runner broker that actually executes Code-node JavaScript, kept as a
separate container/process rather than inline in `n8n`/`n8n-worker` for isolation, per n8n's
own recommended default.

## Secrets

- `N8N_ENCRYPTION_KEY` must be byte-identical across `n8n` and `n8n-worker` or the worker
  can't decrypt stored credentials. This is the single most consequential secret in this
  stack — **losing it makes every stored credential permanently unrecoverable**. Back it up
  out-of-band (e.g. a password manager entry) before the first production deploy; it is not,
  and must never be, stored in git.
- `N8N_RUNNERS_AUTH_TOKEN` is shared between `n8n`/`n8n-worker` (broker clients) and
  `task-runners` (broker server) — internal-only, never exposed publicly.
- `QUEUE_BULL_REDIS_USERNAME`/`QUEUE_BULL_REDIS_PASSWORD` authenticate as a dedicated Valkey
  ACL user scoped to the `n8n:*` key pattern (`QUEUE_BULL_PREFIX: n8n`), not the shared
  Valkey instance's own admin password — see the `valkey` repo's README for how this user is
  provisioned. Keeps this consumer's queue keys isolated from any future Valkey consumer's.

## Security hardening

All four services (`n8n`, `n8n-worker`, `task-runners`, standalone-only `postgres`/`valkey`)
carry an OWASP hardening block, validated against real containers (including a full
webhook → queue → worker → Postgres execution, and for `task-runners` specifically a real
Code-node execution) before landing — see the implementation plan's Tasks 5-7 for exactly
what was tested and why each value is what it is.

**`n8n`/`n8n-worker` deliberately have no `read_only: true`** — same reasoning as `metamcp`:
write surface beyond the declared `/home/node/.n8n` volume isn't fully enumerable ahead of
time. Revisit if/when n8n's actual runtime write paths are fully documented upstream.

**`task-runners` is the one service that IS `read_only: true`**, plus a distroless image
(where available), the unprivileged `nobody` user (`65532:65532`), and `cap_drop: ALL` — per
[n8n's task runner hardening guide](https://docs.n8n.io/deploy/host-n8n/configure-n8n/security/harden-task-runners).
It's the highest-risk component (executes user-authored JavaScript from Code nodes), so it
gets the tightest posture; the other three services can't be `read_only` safely yet, so
`task-runners` carries more of the isolation burden.

**`task-runners` has no Docker healthcheck at all — confirmed impossible, not merely
omitted.** The `n8nio/runners:*-distroless` image ships no shell (no `/bin/sh`, so any
`CMD-SHELL` test fails immediately with "exec: /bin/sh: no such file or directory") and no
HTTP client (no `wget`/`curl`, so no exec-form check can run either) — verified live via
`docker exec` against the running container in Task 7. This is the distroless image's
intended design (it deliberately excludes package managers, shells, and other utilities not
needed at runtime), and adding a client back in would defeat the point of choosing it.
Liveness for this one service is only observable via container run-state (`docker ps` /
`docker inspect .State.Status`), never Docker's own HEALTHCHECK status — any health
verification tooling in this fleet must account for that rather than expect a "healthy"
reading here.

**Known gap: no AppArmor profile on `task-runners`.** The upstream hardening guide
recommends one (denying access to `/proc/*/{environ,mounts}` specifically, to stop
environment-variable/secret exposure from within a compromised runner). This fleet has no
mechanism yet to provision and load a custom AppArmor profile on the Coolify host — every
other hardening control here is expressible entirely in `docker-compose.coolify.yaml`, this
one isn't. Deliberately deferred, not forgotten; tracked in `specs/n8n.md`.
