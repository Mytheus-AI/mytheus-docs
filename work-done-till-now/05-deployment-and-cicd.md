# Deployment and CI/CD

Status: DONE WITH GAPS. Last verified 2026-08-07.

Not a plan band. This records the deployment substrate that every band depends on: how the
VPS stack is built, published, deployed, and reached from the internet. It exists because
that substrate was assumed to work and did not.

Covers `mytheus-platform` (deploy descriptor, deploy workflow) and `mytheus-backend`
(image publishing, deploy trigger). Related: [16-deployment-and-infrastructure.md](../16-deployment-and-infrastructure.md),
[ADR-0016](../adr/0016-self-hosted-vps-mvp-substrate.md).

## 1. What landed, per repo

### mytheus-backend

- `.github/workflows/ci.yml`, `docker` job: builds and pushes
  `ghcr.io/mytheus-ai/mytheus-backend` on pushes to `main`, tagged with both `main` and the
  commit SHA. Pull requests build but never push. Uses the built-in `GITHUB_TOKEN` with
  `packages: write`, so no personal access token exists and there is nothing to rotate.
- `.github/workflows/ci.yml`, `deploy` job: deploys to the VPS after the image publishes and
  after `lint`, `test`, `migrations`, `authz-check` and `security` all pass.
- `security` job: replaced `gitleaks-action` with the gitleaks binary. The action requires a
  paid licence for organisation-owned repositories and had been hard-failing since the repo
  moved into the `Mytheus-AI` org.
- `uv.lock`: `aiohttp` 3.14.1 to 3.14.3, `cryptography` 49.0.0 to 50.0.0.
- `tests/test_org_model_reconciliation.py`: one E501 fix.

### mytheus-platform

- `deploy/compose.prod.yml`: pulls the published image instead of building from a sibling
  checkout; `caddy` service removed; secret paths behind `MYTHEUS_ENV_DIR` and
  `MYTHEUS_PG_INIT` with absolute defaults under `/etc/dokploy/mytheus-secrets`.
- `docker-compose.yml` (new, repo root): entrypoint that includes `deploy/compose.prod.yml`,
  so a deploy tool's project directory and its `.env` are the same directory.
- `.github/workflows/deploy.yml` (new): deploys on changes to the deploy descriptor.
- `deploy/deploy.sh`: pulls images rather than building; no longer expects a sibling repo.
- `deploy/README.md`, `deploy/postgres-init/README.md`: corrected.
- `env/prod.db.temporal.env.example` (new).

### On the VPS, not in any repo

- `/usr/local/sbin/mytheus-deploy.sh`: the only command the CI key may run.
- `/usr/local/sbin/mytheus-port-lock.sh` and its systemd unit: drops external ingress to
  Dokploy's 3000 and Traefik's 8080/8443.
- `/etc/dokploy/mytheus-secrets/`: the credential files.

## 2. Architecture of the deploy path

Push to `main` in `mytheus-backend` runs the full gate suite. If every gate passes, `docker`
publishes the image and `deploy` opens one SSH connection to the VPS. The key installed there
carries a forced command in `authorized_keys`, so it can only run
`/usr/local/sbin/mytheus-deploy.sh`: no shell, no other script, no port forwarding. That is
why CI holds no other secret. Every credential the stack uses lives on the host and is never
handed to GitHub.

The server script fetches the pinned revision, asserts that every credential file exists AND
that the rendered compose config actually carries a `DATABASE_URL` for all four backend roles,
and only then touches the running stack. It finishes by waiting on the container healthcheck;
CI then checks `https://api.mytheus.cloud/healthz` so a green deploy cannot mean "container up,
nobody can reach it".

Dokploy remains connected and can deploy the same stack from its UI, under the same project
name `mytheus` and from the same compose file. Its auto-deploy is deliberately OFF: it triggers
on the same push webhook CI does, and both running `docker compose up` against one project
would race.

## 3. Defects found and closed

Each of these was live, and none was visible from the documentation.

| Defect | Consequence | Fix |
|---|---|---|
| `compose.prod.yml` built the backend from `context: ../../mytheus-backend` | The stack could only be deployed by hand on a box where someone had arranged two sibling clones. No deploy tool could adopt it, the running image matched no commit, and rollback had nothing to roll back to. | Pull a published image |
| `caddy` service published 80 and 443 | Those ports are held by the `atithivishram-prod` Caddy. The service had never started, but any tool bringing the full stack up would fail on a port collision. | Removed |
| `gitleaks-action` requires a paid org licence | The secret-scanning gate had not run since the repo moved into the org, and its failure aborted the job before `pip-audit`, hiding 4 dependency advisories. | Use the gitleaks binary |
| `aiohttp` and `cryptography` advisories | 4 known vulnerabilities shipping in the image. | Lockfile bump |
| Dokploy runs compose INSIDE its container | `env_file` paths under `/srv` resolve against the container filesystem, where `/srv` is not mounted. The entries are `required: false`, so nothing errored: services came up with no `DATABASE_URL`. | Secrets moved to `/etc/dokploy/mytheus-secrets`, which is bind mounted at the same path in both namespaces |
| `postgres-init/README.md` instructed `chmod 600` | `psql` reads the hook as the non-root `postgres` user inside the container, so the hook failed with "Permission denied", `public` stayed owned by `pg_database_owner`, and migrations died later on "permission denied for schema public". Production had been repaired by hand, which hid the fact that every rebuild and any disaster recovery would fail identically. | `0644` file inside a `0700` directory |
| `${POSTGRES_PASSWORD}` interpolation | Interpolation cannot read an `env_file`, so the superuser password had to be handed to the deploy tool. Dokploy stores env in its database and returns it in plaintext to any API-key holder, which would have made that API key equivalent to the database superuser credential. | `postgres` reads it from `prod.env`; `temporal` gets its own one-line file |
| Dokploy ports 3000/8080/8443 world-open | The Dokploy admin panel answered 200 on the raw IP, bypassing Cloudflare. `cf-origin-lock` covers only 80/443. Introduced by this work. | `mytheus-port-lock` systemd unit |
| No TLS for `vps.mytheus.cloud` / `api.mytheus.cloud` | Cloudflare error 525. | Both hosts added to the existing Caddy with `tls internal` |

## 4. Deviations from expectation, and why

- **Secrets live under `/etc/dokploy/`, which reads oddly for a Mytheus path.** It is the only
  directory bind mounted into the Dokploy container at an identical path, and both namespaces
  must resolve it: `env_file` is read by the compose CLI (container side) while the postgres
  bind mount is handed to the daemon as a host path (host side). The alternative, adding a
  mount to Dokploy's own swarm service, would have meant modifying the service that was doing
  the deploying.
- **Absolute defaults, not relative.** They tie the file to this host, which is already true
  of it (it names the box). The `required: false` entries mean a wrong path is silently empty
  rather than an error, so the default has to be right.
- **`mytheus-backend` deploys directly instead of dispatching to `mytheus-platform`.**
  `GITHUB_TOKEN` cannot dispatch across repositories, so that route needed a long-lived
  personal token whose only job is to say "deploy now". The forced-command SSH key grants
  strictly less.
- **Dokploy `appName` was corrected in Dokploy's own database.** Dokploy generates it with a
  random suffix and its API silently drops `appName` from updates. `appName` becomes
  `docker compose -p`, which overrides the compose file's `name: mytheus`, so leaving it would
  have created `mytheus-shrku5_pgdata` and left the production database orphaned behind an
  empty stack.

## 5. Verification

Commands run and output observed on 2026-08-07.

- Full pipeline green in `mytheus-backend`: `lint`, `test`, `migrations`, `authz-check`,
  `security`, `docker`, `deploy` all `success`. First fully green run.
- CI published for the deployed commit: the GHCR package carries both `main` and the commit
  SHA, created by the workflow rather than by hand.
- Forced command proven: `ssh -i <ci_key> root@<vps> 'cat /etc/shadow'` ran the deploy script
  instead of printing the file.
- Fresh install from zero, against a throwaway compose project: `migrate` exit 0,
  `public owner=postgres`, 31 tables built from an empty volume, all 7 services healthy. This
  is the first time disaster recovery has been demonstrated rather than assumed.
- Production deployed by both paths: GitHub Actions, and Dokploy (`composeStatus=done`,
  `migrate` exit 0).
- Data preserved across every deploy: `tables=31`, `alembic=56f72ac73fa9`, and
  `mytheus_pgdata` still created `2026-08-06T13:21:50Z`, identical to the fingerprint taken
  before any of this work.
- Public endpoints: `https://api.mytheus.cloud/healthz` returns
  `{"status":"ok","service":"core-api"}`; `https://vps.mytheus.cloud/` returns 200.
- Co-tenant `atithivishram-prod` unaffected throughout: 14/14 containers, all 5 domains
  responding.
- Ports 3000, 8080 and 8443 refuse external connections; 21 `cf-lock` and 6 `mytheus-lock`
  rules present.

Reported by earlier sessions and NOT re-verified here: the Plan 5-6 feature work itself.

## 6. Open gaps

| Gap | Blocks | Picked up by |
|---|---|---|
| Clerk JWT template must map `public_metadata.internal_org_id` into the claim | AI-tier endpoints return 401 | Dashboard step; documented in `mytheus-platform` `deploy/README.md` |
| `CLERK_WEBHOOK_SECRET`, Stripe and LiveKit values are placeholders | Webhooks, billing, meetings | Needs real credentials from the owner |
| Cloudflare SSL mode must stay Full, not Full (strict) | Both hostnames break on strict | Needs an origin certificate for `*.mytheus.cloud` |
| No staging environment | Every deploy is to production | Future |
| Grafana panels never built | Observability | Plan 14 |

Credential hygiene outstanding for the owner: three Hostinger API tokens, the Dokploy admin
password, and the Dokploy API key were all exposed in plaintext during these sessions and
should be rotated. Note that Dokploy's API returns stored registry passwords in plaintext to
any API-key holder, so that key is high value.

## 7. Application defects closed alongside this work

Found while chasing the deploy path, all in `mytheus-backend`, all verified by CI against a
real database (the suite went from 253 to 263 tests).

- **`organizations.clerk_org_id` was never written by any production path.**
  `libs/shared/org_resolver.py` resolves every authenticated request through that column, so a
  provisioned tenant was unreachable by any request: the API could not serve a real
  Clerk-authenticated caller at all. Only test fixtures populated it. The live database holds
  zero organizations, which is why nothing had surfaced it. `provision_organization` now takes
  and stores it, filling a NULL on a later run but never repointing an established link.
- **Nothing published `internal_org_id` to Clerk.** The AI tier authorizes from that claim and
  failed closed with 401 on every request. `libs/shared/clerk_admin.py` publishes it after
  commit, never inside the provisioning transaction.
- **`Idempotency-Key` reused across endpoints returned the wrong response.** One key space per
  service meant the same key on a second route replayed the first route's body with a 200: a
  success response for an operation that never ran. Now 409 `IDEMPOTENCY_KEY_REUSED`, a code
  that had been in the frozen enum since Plan 0 with nothing emitting it.
- **Every response was cached, including errors.** A transient 500 was pinned for the full 24h
  TTL, so retries replayed the error and the handler never ran again - the opposite of what a
  client retrying with an idempotency key wants. Only 2xx is stored now.
