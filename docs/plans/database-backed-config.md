# Feature Plan: Database-Backed Configuration & Auth

**Status:** Draft
**Author:** (tbd)
**Date:** 2026-06-13
**Related:** [Multi-App Support](./multi-app-support.md)

## Summary

Today the server depends on **Expo's GraphQL API** as the runtime source of truth for
channel→branch routing, branch/channel registry, and publish authentication, and on
**environment variables** for all configuration. This plan proposes introducing a
database to own routing, registry, identity, and application configuration — removing
Expo from the critical serving path and moving application config out of env vars.

Recommended posture: **DB-backed metadata, bucket-backed bundles, behind an optional
backend interface** (see [Tradeoff](#the-core-tradeoff)).

## Background: what we depend on today

### Expo is used for three distinct things

1. **Channel→branch routing — on the hot serving path.**
   - `ManifestHandler` calls `services.FetchExpoChannelMapping(channelName)` to turn
     the client's `expo-channel-name` header into a branch
     (`internal/handlers/manifest_handler.go:162`).
   - `AssetsHandler` does the same (`internal/handlers/assets_handler.go:17`).
   - Every update check therefore hits Expo's GraphQL API (cached — which is exactly
     why the `ENHANCE_YOUR_CALM` rate-limit fix exists, commit `aab3ba6`).

2. **Branch/channel registry.**
   - `branch.UpsertBranch` → `FetchExpoBranches` + `CreateBranch`
     (`internal/branch/branch.go`) creates branches *on Expo*.
   - Dashboard reads `FetchExpoChannels`, `FetchExpoBranchesMapping` and writes
     `UpdateChannelBranchMapping` (`internal/handlers/dashboard_handler.go`).

3. **Auth / identity for publishing.**
   - `ValidateExpoAuth` in `auth_middleware.go:17` and `upload_handler.go` (3 sites);
     `FetchExpoUserAccountInformations` in rollback/republish handlers.
   - `FetchSelfExpoUsername()` is baked into the local bucket's signed upload JWT as
     the `sub` claim (`internal/bucket/localBucket.go:47,220`).
   - (Dashboard login is already local — single `ADMIN_PASSWORD` via `internal/auth`.
     Only the publishing path needs new auth.)

### Environment variables split into four categories

Not all env vars can or should move to a DB:

| Category | Examples | Move to DB? |
|---|---|---|
| **Bootstrap** (needed to reach the DB) | DB connection string, `PORT` | No — chicken-and-egg |
| **Infra secrets** | AWS creds, signing keys, `JWT_SECRET`, Redis password | No — secrets belong in env / secret-manager, not app rows |
| **Storage/infra wiring** | `STORAGE_MODE`, bucket names, `CACHE_MODE`, `REDIS_*`, CDN config | Optional — usually kept as deploy-time env |
| **Application config** | `EXPO_APP_ID`, channel/branch mappings, dashboard settings, per-app settings | **Yes** — the real target |

"Eliminate env vars" realistically means moving **application configuration** into the
DB while bootstrap + secrets stay in env. The existing `keyStore` abstraction already
exists so secrets live outside the code (AWS Secrets Manager / env keys); lean on it
rather than putting keys in DB rows.

## Goals

- Remove Expo from the runtime serving path (manifest/assets resolve routing locally).
- Own branch/channel registry and channel→branch mappings in the DB.
- Replace Expo-account publish auth with server-issued tokens.
- Move application configuration out of env vars into the DB.
- Keep update **bundles** in the storage bucket (no change to where bytes live).

## Non-goals

- Storing update bundles/blobs in the DB.
- Removing bootstrap/secret env vars.
- Forcing a DB on existing deployments (keep the no-DB mode viable — see tradeoff).

## Design

### Data model (minimal)

- `apps` — app identity (replaces global `EXPO_APP_ID`; also the registry the
  [multi-app plan](./multi-app-support.md) needs).
- `branches` — branches per app.
- `channels` — channels per app.
- `channel_branch_mappings` — the routing table that replaces
  `FetchExpoChannelMapping`.
- `users` / `api_tokens` — identity + server-issued publish tokens.
- `settings` (optional) — per-app/application configuration.

Update **metadata** can live in the DB; update **files** stay in the bucket. The DB is
for routing, registry, identity, and config — not bundle storage.

### Components touched

1. **New `internal/db` layer** — connection pooling, driver, query layer.
2. **Schema migrations** — a *second* migration concern. Note the naming collision:
   `internal/migration` today runs **bucket data** migrations. Either generalize it or
   add a clearly separated DB schema-migration system. Do not conflate the two.
3. **`internal/services/expo.go`** — demote from runtime authority to optional one-way
   *import/sync* from Expo (or remove entirely).
4. **Handlers** — `manifest`/`assets` read mappings from DB; `upload`/`rollback`/
   `republish` use new token auth; `dashboard_handler` does DB CRUD.
5. **Auth** — server-issued publish-token issuance/validation; rework
   `auth_middleware`. Replace `FetchSelfExpoUsername` JWT `sub` usage in local bucket.
6. **Config** — `config.LoadConfig()` reads bootstrap from env, then loads app config
   from DB.
7. **Caching** — keep it, but as a read-through cache over the DB instead of over Expo.
   Invalidation gets easier since we now control writes.
8. **`eoas` CLI** (`apps/eoas`) — switch from Expo-account auth to server tokens;
   branch creation hits our API, not Expo.
9. **Deployment** — Helm chart, `docker-compose.yml`, docs: add a DB service,
   connection config, and a schema-migration step.

## The core tradeoff

The current design's headline selling point is **"No database required — zero external
dependencies beyond your storage"** (README). Adding a DB reverses that. Three postures:

- **Optional DB (recommended).** Keep Expo/bucket mode working; add DB as an
  alternative backend behind an interface, mirroring the existing
  `bucket`/`cache`/`keyStore` factories. Most work, preserves the value prop, opt-in.
- **DB metadata + bucket bundles.** DB owns routing/auth/config; bucket still stores
  files. Clean separation, moderate work. Recommended *shape* regardless of posture.
- **Full DB.** Also store bundle metadata/blobs in DB. Loses the "your cloud storage"
  benefit; not advised.

## Risks / open questions

- **Reverses the zero-dependency value proposition.** The optional-backend posture
  mitigates this but adds an abstraction layer to maintain.
- **Migration-system naming collision** (bucket migrations vs. DB schema migrations).
- **Auth migration is security-critical** — server-issued publish tokens must be at
  least as strong as the Expo-account check they replace.
- **Secret placement** — keep signing keys/`JWT_SECRET`/cloud creds in `keyStore`/env,
  not DB rows.
- **Data migration path** — existing deployments need a one-time import of current
  Expo channel/branch mappings into the DB.
- **DB driver/engine choice** (Postgres vs. SQLite for the no-extra-service case).
  SQLite could preserve much of the low-dependency ethos for small deployments.

## Relationship to multi-app support

This plan and [Multi-App Support](./multi-app-support.md) converge: an `apps` table is
exactly the registry multi-tenancy needs, and a DB is the natural substrate for
per-tenant routing/auth/config. If both are desired, design the schema once with the
app dimension built in from the start.

## Testing

- Extend `test/` integration suites to cover DB-backed routing/auth with the
  `Reset*Instance()` + `httpmock` patterns; add a DB fixture/teardown helper.
- Assert manifest/assets resolve routing with **no** outbound Expo calls in DB mode.
- Cross-tenant isolation tests if combined with multi-app.

## Recommendation

Adopt the **optional-DB, metadata-in-DB/bundles-in-bucket** posture. Sequence:

1. Introduce `internal/db` + schema migrations behind a backend interface.
2. Move channel→branch routing to DB (biggest runtime win; removes Expo hot-path).
3. Move branch/channel registry + dashboard CRUD to DB.
4. Replace publish auth with server-issued tokens.
5. Move application config out of env into DB `settings`.
6. Reduce `expo.go` to optional import/sync.

Design jointly with the multi-app plan if both are on the roadmap.
