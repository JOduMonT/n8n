# Security Policy

Deployment config for n8n workflow automation in queue mode (main, worker, external task runner), standalone or as a Coolify tenant app using the fleet's shared Postgres and Valkey.

## Supported versions

Only the current `main` branch is supported.
Fixes land on `main`; there are no release branches.

## Reporting a vulnerability

Please report privately.
Do not open a public issue or pull request.

- **Preferred:** [report a vulnerability](https://github.com/JOduMonT/n8n/security/advisories/new) through GitHub private vulnerability reporting.
- **Email:** jodumont+security@gmail.com
- Include what you found, the affected file or service, steps to reproduce and the impact you see.
- Do not access, change or delete data that is not yours, and do not run denial-of-service or automated scanning against live systems.

You can expect an acknowledgement within 3 business days and a status update within 10.
Confirmed issues are fixed as quickly as severity allows, and you are credited in the fix unless you prefer not to be.

## Scope

In scope:

- `docker-compose.yaml` and `docker-compose.coolify.yaml`: exposed ports, how the encryption key and database credentials are supplied, default values.
- CI workflows that call `JOduMonT/fleet-toolkit`.

Out of scope:

- n8n itself (report to n8n), and workflows or credentials stored inside a running instance.
- Social engineering and physical attacks.

## How this repository is kept safe

- Dependabot alerts and security updates are on; a vulnerable dependency gets an automatic pull request.
  Routine version bumps are opened by Renovate, and `.github/dependabot.yml` keeps Dependabot's own version updates off to avoid duplicate pull requests.
- Dependabot pull requests are merged automatically by `.github/workflows/dependabot-auto-merge.yml` once every other check passes.
  Major version bumps are left open for review.
- GitHub secret scanning with push protection and CodeQL code scanning are enabled.
- The n8n encryption key and database passwords are injected by Coolify or `.env`, never committed.
- Image bumps come from Renovate and are smoke-tested in CI.
