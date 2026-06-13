# Feature Plan: Multi-App Support

**Status:** Draft
**Author:** (tbd)
**Date:** 2026-06-13

## Summary

Today a single `expo-open-ota` instance serves exactly one Expo app. This document
analyzes why, confirms that the constraint is a server/deployment choice (not a
protocol limitation), and proposes how to support multiple apps — either by running
multiple instances (available now) or by adding true path-based multi-tenancy to a
single instance (proposed).

## Background: how app identity works today

### The server is single-app by design

- `EXPO_APP_ID` is a **global, required** env var, read once at startup
  (`config/config.go:90`). The process fails to boot without it.
- Every Expo GraphQL call hardcodes that single `appId` — channel↔branch mapping,
  branch creation, auth validation (`internal/services/expo.go`, e.g.
  `GetExpoAppId()` at line 72 and its callers throughout the file).
- The storage layout has **no app dimension**. Bucket keys are
  `branch/runtimeVersion/updateId` (`internal/bucket/localBucket.go:33`,
  `internal/bucket/s3Bucket.go`).
- The protocol endpoints (`/manifest`, `/assets`) resolve purely by
  branch + runtimeVersion + platform (+ channel). No app identifier is accepted.
- Cache keys are computed from branch/runtimeVersion/updateId/platform only
  (`Compute*CacheKey` helpers in `internal/update` and `internal/services`).

Net: one running server == one Expo app.

### The OTA protocol is app-agnostic by intent

The [Expo Updates protocol](https://docs.expo.dev/technical-specs/expo-updates-1/)
carries **no app identifier** in the request. The manifest request headers
(`internal/handlers/manifest_handler.go`) are:

- `expo-protocol-version`
- `expo-platform`
- `expo-runtime-version`
- `expo-channel-name`
- `expo-current-update-id` / `expo-embedded-update-id`
- `expo-expect-signature`, `EAS-Client-ID`, `expo-fatal-error`, …

The `/assets` request is similarly scoped (channel + asset + runtimeVersion +
platform). The app's identity **is the endpoint URL**: each app sets `updates.url`
in its config to the manifest endpoint it should talk to, and the server behind that
URL is assumed to speak for one app. Multi-app routing is explicitly *outside* the
spec — it's a URL/infrastructure concern.

Expo's own hosted EAS Update serves many apps from `u.expo.dev` by embedding a
**project ID in the URL path** (`u.expo.dev/<project-id>`), not via a protocol
field. So even there, the multi-app dimension lives in the URL, leaving the protocol
headers untouched. Any multi-tenant design we build should follow the same grain.

## Option A — Multiple instances (available today)

Run one instance per app. No code changes required.

| Setting | Per-app value |
|---|---|
| `EXPO_APP_ID` | required, different per app |
| `S3_KEY_PREFIX` | recommended if sharing one S3 bucket (`internal/bucket/bucket.go:68`) |
| `LOCAL_BUCKET_BASE_PATH` / `GCS_BUCKET_NAME` | separate per app (no prefix support) |
| signing keys, `JWT_SECRET`, `BASE_URL` | per instance |

Each app's client points `updates.url` at that instance's `BASE_URL`.

**Pros:** zero development, strong isolation (separate keys, separate blast radius),
already protocol-aligned.
**Cons:** operational overhead scales linearly with apps (N deployments, N configs,
N dashboards); shared-bucket isolation only available for S3 (via `S3_KEY_PREFIX`),
not local/GCS.

## Option B — Path-based multi-tenancy (proposed)

Carry an app identifier in the URL path (e.g. `/<appId>/manifest`,
`/<appId>/assets`) — mirroring `u.expo.dev/<project-id>` — and thread it through the
server's layers. The protocol messages themselves stay unchanged, so it remains
fully spec-compliant.

### Scope of change

1. **Routing** (`internal/router`)
   - Add an app segment to protocol + publishing routes, or resolve app from
     hostname. Recommend explicit path segment for clarity and parity with EAS.
   - Extract the app id and place it on the request context.

2. **Config** (`config/config.go`)
   - Replace the single global `EXPO_APP_ID` with a per-app registry: a map of
     `appId -> { expoAppId, signing keys / key-store config, expo access token,
     storage namespace }`. Source from env (JSON blob) or a config file.
   - Keep single-app mode working as the degenerate case (one entry) for backward
     compatibility.

3. **Expo service** (`internal/services/expo.go`)
   - Make every GraphQL call take the app id / access token as a parameter instead
     of reading the global. `GetExpoAppId()` becomes per-request.

4. **Storage** (`internal/bucket`)
   - Prefix all bucket paths with the app id: `<appId>/branch/runtimeVersion/...`.
   - For S3, compose with the existing `S3_KEY_PREFIX`. Add equivalent namespacing
     for local and GCS so all three back-ends isolate per app.

5. **Cache** (`internal/cache`, `Compute*CacheKey` helpers)
   - Add app id to every cache key (or fold into `CACHE_KEY_PREFIX` per app) so
     tenants can't read each other's cached manifests/mappings.

6. **Key store & signing** (`internal/keyStore`, `internal/crypto`)
   - Resolve signing keys per app. Each tenant must sign with its own key so
     `expo-expect-signature` validation passes on the client.

7. **Dashboard & API** (`internal/handlers`, `apps/dashboard`)
   - Scope `/api/*` views and the dashboard to a selected app; auth tokens carry the
     app scope.

8. **Migrations** (`internal/migration`)
   - Bucket migrations must iterate per app namespace, or be parameterized by app.

9. **`eoas` CLI** (`apps/eoas`)
   - Publishing endpoints gain the app segment; CLI config selects which app to
     publish to.

### Backward compatibility

- Treat the absence of an app segment as the existing single-app behavior, or
  provide a default app id, so current deployments keep working without config
  changes.

### Risks / open questions

- **Key isolation is security-critical** — a bug that crosses tenant signing keys or
  storage namespaces is a serious leak. Needs dedicated tests.
- **Auth model** — how dashboard users map to apps (one admin for all vs. per-app
  credentials). Current auth is a single `ADMIN_PASSWORD`.
- **Config ergonomics** — env-var-only config gets unwieldy with many apps; a config
  file may be warranted.
- **Per-app Expo access tokens** vs. one token with access to multiple Expo projects.

### Testing

- Extend `test/` integration suites to spin up ≥2 apps and assert isolation across
  storage, cache, and signing. Reuse `Reset*Instance()` + `httpmock` patterns.
- Add a path-traversal/cross-tenant test analogous to the existing
  `test/dashboard_path_traversal_test.go`.

## Recommendation

- **Short term:** use Option A (multiple instances). It is protocol-aligned and
  needs no code. Use `S3_KEY_PREFIX` if sharing an S3 bucket.
- **Long term:** if operating many apps becomes painful, implement Option B. The
  clean provider abstractions (bucket/cdn/cache/keyStore factories) make the change
  tractable, but it touches routing, config, storage, caching, signing, dashboard,
  and the CLI — plan it as a deliberate, well-tested effort given the security
  surface.
