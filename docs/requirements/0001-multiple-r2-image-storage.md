# 0001: Multiple R2 targets for image storage

Status: pending-verification

- Owner: ImageStorageService and ImageStorageTargetRepository
- Created: 2026-10-21
- Updated: 2026-10-21
- Related issues/ADRs: ADR 0003, ADR 0004

## Problem and user outcome

Image storage currently exposes one WebDAV configuration. Users need to
configure multiple Cloudflare R2 buckets for generated images, select R2 as the
image storage mode, and manage those targets from Settings. R2 backup remains
one independent target using the existing backup settings.

## Current behavior and evidence

- `ImageStorageService` owns image save, read, delete, cleanup, compression, and
  synchronization. The gallery index is `data/image_index.json`.
- Image storage settings currently expose local, WebDAV, and combined WebDAV
  modes through `contracts/settings.py` and `services/settings_management_service.py`.
- R2 backup settings and execution are independent in `BackupService` and the
  `backup` settings projection.
- `D:\FlowRealm-Go` provides a reference implementation for R2 target
  persistence, per-asset target identity, target testing, allocation, and
  fallback. Its capacity and API-operation accounting is outside this scope.

## Scope

- Add multiple R2 image targets with enable/disable, credential update,
  connection test, and deletion when no gallery asset references the target.
- Choose the enabled target with the least indexed R2 image bytes, using stable
  target ID as a tie-breaker; try other enabled targets when upload fails.
- Store target ID and exact object key on each R2-backed gallery entry so reads,
  deletes, cleanup, and synchronization address the same bucket and key.
- Preserve local fallback when every R2 upload fails and allow a later R2 sync
  to retry pending local copies.
- Import a complete legacy single-image-R2 configuration as the reserved
  `legacy-default` target when encountered.
- Keep R2 backup on its existing single-target settings and execution path.

## Non-goals

- R2 capacity quotas, Cloudflare API usage accounting, bucket analytics, or
  automatic bucket migration.
- Moving Image Assets or the gallery index into the Application Database.
- Changing the single R2 target used by backups.

## Domain language

Use **Image Asset** and **Image Storage Target** from
[`../../CONTEXT.md`](../../CONTEXT.md).

## Unique owners

| Concern | Authoritative owner | Interface consumed by others | Forbidden mirror or fallback |
| --- | --- | --- | --- |
| R2 target configuration and credentials | `ImageStorageTargetRepository` | Target CRUD and public target projection | Settings JSON or Vue as another credential store |
| Image assignment, object keys, reads, deletes, synchronization | `ImageStorageService` | Gallery index and stored-image result | API routes performing object operations |
| HTTP validation and authorization | `api/system.py` contracts/routes | Authenticated JSON endpoints | Vue-side credential validation as authority |
| Drafts, target actions, loading and feedback | Settings page runtime | `settingsApi` and storage panel events | Separate settings-page copy of target status |
| Backup R2 configuration and operations | `BackupService` and existing `backup` settings | Existing backup settings/API | Image target list or shared target selection |

## Backend Modules and Interfaces

`ImageStorageTargetRepository` stores target credentials and health in its own
Application Database table. `ImageStorageService` selects targets, persists
target identity with each asset, and owns upload fallback, read, deletion, and
sync behavior. Settings API routes expose validated target CRUD and test
commands; responses never include the secret access key.

The image index remains the authority for asset references. A target cannot be
deleted while indexed assets reference it, and its account or bucket identity
cannot change while those assets remain. Application Database and image index
updates keep their existing independent transaction boundaries.

## Frontend interaction and responsive behavior

The Settings storage panel lists configured targets, managed image count and
bytes, enabled state, and health result. It supports adding and editing a
target, connection tests, enable/disable, and deletion. The existing storage
mode selector gains R2. Target actions require saved image-storage settings
when they depend on mode and show loading, empty, and error states.

Backup remains in its existing Settings tab with one R2 credential set.

## Persistence impact

Add a domain-owned `image_storage_targets` table to the existing Application
Database. Image objects and their target ID/key references remain in R2 and the
JSON gallery index respectively. The legacy single-image-R2 settings are
imported once into `legacy-default`; backup settings are not migrated.

## Concurrency, security, and failure behavior

- Admin routes authenticate target management and connection tests.
- Secret access keys are persisted only in the repository and omitted from
  response projections; updates with an empty secret preserve the stored key.
- Upload uses enabled targets in least-managed-bytes order and records the
  successful target in the gallery index. Failed targets are marked unhealthy
  and the next target is tried.
- If all target uploads fail, the image is retained locally as pending R2 sync.
- Assets remain readable through their recorded target even after that target
  is disabled. Deletion is blocked while assets reference a target.
- Backup continues to use one independent R2 configuration and bucket prefix.

## Acceptance criteria

- [ ] Admin can create, list, edit, test, enable/disable, and delete unused R2 image targets.
- [ ] Target responses omit secret access keys, and editing without a new secret preserves the existing key.
- [ ] New R2 uploads distribute by managed image bytes, retry another enabled target after failure, and retain local fallback if all fail.
- [ ] R2 gallery reads, deletion, local cleanup, compression reconciliation, and full sync use the asset's recorded target and key.
- [ ] A referenced target cannot be deleted or have its account/bucket identity changed.
- [ ] Existing local/WebDAV paths and single-target R2 backups continue to work.
- [ ] Current storage maps and CHANGELOG describe only behavior verified in code and tests.

## Verification matrix

| Acceptance criterion | Verification level | Evidence required | Status |
| --- | --- | --- | --- |
| Target CRUD and secret projection | Repository/API contract tests | 44 targeted tests passed | verified |
| Assignment, retry, fallback, read, delete, and sync | ImageStorageService interface tests | 44 targeted tests passed | verified |
| Settings target management | Vue runtime test and build | Runtime tests and production build passed | verified |
| Existing storage and backup behavior | Python regression suite | 1041 passed; one pre-existing clock-patching failure remains | pending |
| Current docs and release notes | Documentation review | Maps, storage docs, README, and CHANGELOG reviewed | verified |

## Documentation and CHANGELOG impact

Update `CONTEXT.md`, `docs/storage-architecture.md`,
`docs/maps/backend-map.md`, `docs/maps/frontend-map.md`,
`docs/maps/critical-flows.md`, and `CHANGELOG.md` after their described
behavior is verified.

## ADR requirement

Not required. The accepted ADRs already assign Image Asset mutations to
`ImageStorageService`, allow separate domain repositories in the Application
Database, and keep asset objects outside that database.

## Rollback

Set image storage back to local or WebDAV mode. Keep the target rows and R2
objects intact until all gallery references are migrated or removed; do not
delete external objects as part of a code rollback.

## Unresolved questions

- None. The accepted scope is multiple R2 targets for image storage only;
  backups retain one R2 target.
