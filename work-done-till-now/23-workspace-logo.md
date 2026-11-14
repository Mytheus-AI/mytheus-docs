# 23. The workspace logo

Not a plan band ([ADR-0048](../adr/0048-workspace-logo.md)). Status: **DONE WITH GAPS** (built and verified locally on 2026-11-14; committed on branch `feat/org-logo` in four repos, not pushed, not deployed).

Workspace admins upload and edit the organisation's logo. It is stored privately in Cloudflare R2 and drawn wherever the workspace appears, replacing the dark initial circle ("K" for Kestrel Labs).

## What landed, per repo

### mytheus-backend

- Migration `e9f0a1b2c3d4` (revises `d8e3f4a5b6c7`, the single head on 2026-11-14): `tenant_settings.logo_key`, `logo_content_type`, `logo_version` (default 0), `logo_bytes`, `logo_width`, `logo_height`, `logo_updated_at`, with CHECKs (key under the row's own `{org_id}/branding/`, type in PNG/JPEG/WebP, size up to 2 MB, sides 16 to 4096, key and type set together). RLS is the table's existing `tenant_isolation` policy; no new grant.
- `services/identity_tenant_service/branding/`: `images.py` (magic bytes and header parsing for PNG, JPEG and WebP, SVG refused, dimension check), `store.py` (the `mytheus-branding` store over the shared S3 client, an in-memory `FakeLogoStore` for tests), `routes.py`:
  - `GET /api/v1/org/logo` (member), `PUT /api/v1/org/logo` (admin, raw image body), `DELETE /api/v1/org/logo` (admin, idempotent), `GET /api/v1/org/logo/image?v=` (member, immutable for the current version, ETag and 304).
  - Audit actions `tenant_settings.logo.updated` and `tenant_settings.logo.removed`; per-workspace hourly limit `ORG_LOGO_UPLOADS_PER_HOUR` (default 20) at `nexus:{org_id}:ratelimit:org-logo:{hour}`.
- `GET /orgs` gains `logo_url` and `logo_updated_at` (additive).
- `libs/shared/object_storage.py`: the boto3 client factory that `billing_metering_service/storage.py` and `integration_service/storage.py` each copied; both now call it, and the logo store reuses it.
- `design/openapi/identity-tenant-service.yaml` regenerated; `tests/test_org_logo.py` (23 tests); `tests/test_orgs_rbac_routes.py` answers the new logo read in its fake session.

### mytheus-ui

- `components/tenant/OrgLogoCard.tsx`: the editor in the Workspace console's Overview, under the workspace name. Drag and drop or file picker; a square crop and position step on a canvas (drag, zoom slider and buttons, arrow keys, plus and minus, 0 to centre) with live previews at the sidebar and chooser sizes; staged progress; Replace; Remove (asks first via `RemoveButton`); wrong-type, too-big, unreadable and server-refused errors; a saved check; read-only for members (Settings > Profile).
- `components/tenant/OrgMark.tsx`: the logo (rounded square, `object-fit: cover`, hairline ring) or the initial, with fallback on load error. Used in the sidebar switcher (open and rail), the Workspace console header and the workspace chooser at `/onboarding/organization`.
- `app/actions/org-logo.ts` (`uploadOrgLogo`, `removeOrgLogo`): Server Actions to the API, then the Clerk organization image is mirrored best effort, then `router.refresh()` in the card. `next.config.mjs` raises the Server Action body limit to 3 MB.
- `app/api/org-logo/route.ts`: the same-origin relay for `<img>`, `private, max-age=31536000, immutable`, ETag passed through. `lib/api/org-logo.ts` (gateway calls), `lib/org-logo.ts` (validation, initial, URL mapping, crop geometry).
- `resolveOrgContext()` reads the logo in parallel with the Clerk lookups; `OrgContextValue.org.logoUrl` carries it.
- The Workspace overview skeleton gains the logo card. Tests: `tests/org-logo.test.ts`; `tests/e2e/org-logo-a11y.spec.tsx` (21 CT tests, every state in both themes, rail centring, 360 px). The CT actions stub also gains the five ops-panel exports the full suite was missing on main (it did not build).

### mytheus-platform

- `env/prod.env.example`: the backend `S3_*` variables, `S3_BUCKET_BRANDING=mytheus-branding`, `ORG_LOGO_UPLOADS_PER_HOUR=20`.
- `scripts/push-secrets.sh storage`: copies the R2 key and endpoint from `mytheus-intelligence/.env` into `prod.env` with the branding bucket (`--dry-run` prints names only).

### mytheus-docs

- `_canon/02-data-model.md`: the `tenant_settings.logo_*` columns and the `mytheus-branding` bucket; `_canon/01-service-catalog.md`: the routes and ownership; `05-data-architecture.md`: the bucket; ADR-0048; README index row.

## Decisions and deviations, and why

- **Columns on `tenant_settings`, not `organizations`.** `organizations` is not RLS-policied by design (ADR-0002 D3); the task asked for RLS on new columns.
- **A new bucket instead of a `branding/` prefix in an existing one.** The brief preferred reuse. `mytheus-recordings` has a per-tenant `{org_id}/` expiration rule (`bucket_lifecycle.py`) that would delete logos at transcript retention; `mytheus-documents` is ingestion's and is swept into `storage_gb`; neither is written by `identity-tenant-service`.
- **SVG refused by the API, rasterised by the web editor.** No safe sanitiser among runtime dependencies.
- **No 512 px and 128 px server variants.** Pillow is not a dependency; the original is stored. The editor exports a 512 px square, so in practice every web upload is normalised; API clients store what they send (validated).
- **Raw body, not multipart.** `python-multipart` is not a runtime dependency.
- **Versioned proxy instead of a presigned GET.** A presigned URL changes per read and expires; see ADR-0048 D5.
- **Clerk mirror from the UI Server Action.** It already holds `clerkClient` (the rename does the same); the chooser lists workspaces from Clerk, where the backend cannot serve other workspaces' logos.
- **Progress is staged, not byte-accurate.** A Server Action reports no upload progress; the export is about 20 to 200 KB, so the bar shows preparing, uploading and saving.

## Open gaps and owner steps

1. Create the private R2 bucket `mytheus-branding` (no lifecycle rule) and allow the backend's R2 token on it.
2. Run `scripts/push-secrets.sh storage` (puts `S3_*` and `S3_BUCKET_BRANDING` in `prod.env`); until then uploads answer 503 "not connected" and every screen shows the initial.
3. Deploy in order: backend (migration `e9f0a1b2c3d4`), then UI. One repo at a time.
4. RTBF and tenant deletion do not yet purge `{org_id}/branding/` in `mytheus-branding`.
5. Mobile does not draw the logo yet (`GET /orgs` and `/api/v1/org/logo` carry it).
6. Pre-existing, not caused here: `tests/test_only_mytheus.py::test_weekly_digest_is_built_filtered_and_sent_once` fails on a Saturday (a due date two days out falls outside the digest week); CT `ledger-a11y` (date fixture) and `room-sidebar-a11y` "suggested question" (qdock contrast 4.49) fail on this branch and do not touch these files.

## Verification

Backend (throwaway Postgres `supabase/postgres:15.8.1.060` on 5471 and Redis on 6471; `DATABASE_URL`, `BOOTSTRAP_DATABASE_URL`, `REDIS_URL` exported):

- `uv run ruff check .`, `uv run black --check .`, `uv run mypy .` (293 files): clean. `uv run python scripts/check_authz.py`: passed.
- `uv run pytest tests/test_org_logo.py`: 23 passed. `uv run pytest`: 1696 passed, 1 skipped, 2 failed (the test_orgs fake-session test, since fixed, and the Saturday digest test above).
- `uv run python -m scripts.emit_openapi`; `docker run --rm -v <dir>:/w tufin/oasdiff breaking /w/base/identity-tenant-service.yaml /w/head/identity-tenant-service.yaml --fail-on ERR`: exit 0. Spectral with `design/.spectral.yaml`: no errors.

UI:

- `npx tsc --noEmit -p .`, `npx next lint`: clean. `npm test`: 561 passed.
- `CT_PORT=3611 npx playwright test -c tests/e2e/playwright-ct.config.ts org-logo --workers=4`: 21 passed. Full suite `--workers=4`: 597 passed, 473 skipped, 4 failed (the pre-existing ones above, one flaky and passing on rerun).
- `npx -y node@20 ./node_modules/next/dist/bin/next build` with `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` exported from the main checkout's `.env`: compiled, `/api/org-logo` listed.

## Follow-ups, 2026-11-14 (mytheus-ui)

- Settings > Profile shows the logo card to admins too, editable (`9cb86da`); before, it showed only to members, read-only, and an admin found no way to set the logo outside the Workspace console.
- Start a workspace takes an optional logo (`9919ef0`): picked with the editor's checks, previewed beside the name, and uploaded right after the workspace is set up as the same centred 512 px square (the upload is an admin call scoped to the new workspace, so it cannot happen earlier). A failed upload never undoes the workspace; the page says why and offers Continue. The image helpers moved to `src/lib/org-logo-browser.ts` unchanged (`596bbaa`).
- Production: saving a logo answers the generic "could not be saved just now". The `S3_*` variables are set (otherwise the message names storage as not connected), so the write to R2 fails: owner step 1 above (create the private bucket `mytheus-branding` and allow the backend's R2 token on it) is the likely missing piece. Not confirmed from the VPS logs, which were not read; `docker logs` on identity-tenant-service shows `org logo: upload failed` with the cause.
- Verified: CT `location-prompt-a11y` covers the Start a workspace picker (pick, remove, a non-image refused) in both themes, axe clean.
