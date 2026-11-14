# ADR-0048: The workspace logo: stored in R2, served behind the tenant check, drawn wherever the workspace appears

## Related documents

- [_canon/02-data-model.md](../_canon/02-data-model.md): the `tenant_settings.logo_*` columns (section 2) and the `mytheus-branding` bucket (section 6), added in the same change.
- [_canon/01-service-catalog.md](../_canon/01-service-catalog.md): the `/org/logo` routes of `identity-tenant-service`.
- [ADR-0023](0023-object-storage-cloudflare-r2.md): object storage is Cloudflare R2, private buckets, `{org_id}/` prefixes.
- [ADR-0002](0002-multi-tenancy-model.md) D3: `organizations` is not RLS-policied; tenant-confidential data belongs on `tenant_settings`.
- [ADR-0040](0040-polish-program.md): additive API evolution (oasdiff), recording playback by presigned GET.

## Status

**Accepted** (2026-11-14).

## Context

A workspace is drawn in the sidebar switcher, the workspace chooser and the Workspace console as a dark circle with its initial ("K" for Kestrel Labs). Workspace admins asked to upload their organisation's logo. Questions to settle: where the image lives and who writes it, how the browser loads a private image, what file types are safe, and whether Clerk's own organization image is kept in step.

## Decision

### D1. Owner and columns

`identity-tenant-service` owns the logo, as it owns `tenant_settings`. The pointer lives on `tenant_settings` (not `organizations`, which is not RLS-policied and holds directory metadata only): `logo_key`, `logo_content_type`, `logo_version`, `logo_bytes`, `logo_width`, `logo_height`, `logo_updated_at`. The table's `tenant_isolation` policy covers them. A CHECK pins `logo_key` under the row's own `{org_id}/branding/` prefix.

### D2. A bucket of its own: `mytheus-branding`

Keys are `{org_id}/branding/logo-{version}.{png|jpg|webp}`. `logo_version` is bumped on every upload and never reset, so each upload is a new key and a new URL. The previous object is deleted after the new row commits; removal deletes it too.

Why not a `branding/` prefix in an existing bucket: `mytheus-recordings` expires everything under `{org_id}/` at the tenant's transcript retention (one lifecycle rule per tenant prefix), which would delete a logo after a year; `mytheus-documents` belongs to ingestion and is swept into `storage_gb`; neither has `identity-tenant-service` as its writer, and every store has one writer. `mytheus-branding` is private and has no lifecycle rule. It uses the same `S3_*` credentials as every backend bucket, named by `S3_BUCKET_BRANDING`. The boto3 client construction that billing and integration each carried is now one shared factory, `libs/shared/object_storage.py`, which the logo store reuses.

### D3. Accepted files

The API reads the bytes, not the declared type: PNG, JPEG or WebP by magic bytes and a parsed header, at most 2 MB and 16 to 4096 px per side (a small file can still declare a huge canvas). **SVG is refused** by the API: no SVG sanitiser is a runtime dependency (`defusedxml` is only in the lock file through dev tools) and a hand-written XML allowlist is not a safe substitute. The web editor **rasterises** an SVG on a canvas before upload, so people can still choose one. Pillow is not a dependency, so the server stores the bytes as uploaded; the editor's crop step is the normalisation: it exports a 512 px square, WebP where the browser encodes it, PNG otherwise. There is no separate 128 px variant.

### D4. Routes

- `PUT /api/v1/org/logo` (owner/admin): the image as the raw body with its `Content-Type` (no multipart: `python-multipart` is not a runtime dependency, and a raw body streams through the gateway). Refusals are `VALIDATION_FAILED` with a `reason` (`too_large` 413, `unsupported_type` and `svg_not_accepted` 415, `bad_dimensions` and `corrupt_image` 422). Limited to `ORG_LOGO_UPLOADS_PER_HOUR` (default 20) per workspace in Redis, failing closed. Audited as `tenant_settings.logo.updated`.
- `DELETE /api/v1/org/logo` (owner/admin): idempotent 204; audited as `tenant_settings.logo.removed` when there was one.
- `GET /api/v1/org/logo` (member): `logo_url`, `logo_updated_at`, `logo_version`, type, size and dimensions; `logo_url` null when there is none. `GET /orgs` gains `logo_url` and `logo_updated_at` additively.
- `GET /api/v1/org/logo/image?v={version}` (member): the bytes, with `ETag`, `X-Content-Type-Options: nosniff` and a sandboxing CSP.

The workspace is always the token's: no route names one, so another workspace's logo cannot be asked for, and "no logo" is a 404.

### D5. A versioned proxy, not a presigned URL

`logo_url` is `/api/v1/org/logo/image?v={version}`. With the current `v` the response is `Cache-Control: private, max-age=31536000, immutable`. Recordings and briefing audio use short presigned GETs; a logo does not, because it is drawn on every screen: a presigned URL changes on every read (so the browser never caches it) and expires after minutes (so a tab left open would show a broken image). The web app relays the route same-origin at `/api/org-logo?v=`, forwarding the session, because an `<img>` cannot carry the bearer token. A stale `v` is a 404 and the mark falls back to the initial.

### D6. Clerk mirror, R2 the source of truth

After a successful save, the web app's Server Action also sets the Clerk organization's image (`updateOrganizationLogo`), and removal deletes it, best effort. This keeps Clerk's own screens and the workspace chooser (which lists every workspace a person belongs to from Clerk, including ones whose tokens the backend would refuse) in step. Mytheus never reads the logo from Clerk inside a workspace. Clerk serves its copy from its own CDN; an organisation logo is not confidential, and the R2 copy stays private.

### D7. Where it is drawn

One component, `OrgMark`: the logo in a rounded square (`object-fit: cover`, a hairline ring so a white logo still has an edge on cream and dark), or the initial when there is none or it fails to load. It replaces the initial in the sidebar switcher (open and rail, still centred on the rail line), the workspace chooser and the Workspace console header. Admins edit it in the console's Overview; members see it read-only in Settings > Profile.

## Consequences

- One more private bucket to create in R2 (`mytheus-branding`, no lifecycle rule) and the backend `S3_*` credentials in `prod.env` (`scripts/push-secrets.sh storage`). Until then uploads answer 503 "not connected" and every screen shows the initial.
- RTBF and tenant deletion must also purge `{org_id}/branding/` in `mytheus-branding`; the saga does not yet list the bucket.
- The storage sweep (`storage_gb`) does not count logos: they are capped at 2 MB per workspace.
- A logo image costs one proxied read per version per browser; nothing else changes on the hot path except one extra gateway read in the app shell, run in parallel with the existing ones.
