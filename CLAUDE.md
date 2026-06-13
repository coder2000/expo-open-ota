# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A self-hosted Go server implementing the [Expo Updates protocol](https://docs.expo.dev/technical-specs/expo-updates-1/) for over-the-air (OTA) updates. It serves signed update manifests and assets to Expo apps, stores update bundles in pluggable cloud storage (S3/GCS/local), and ships with a React dashboard and an `eoas` CLI for publishing. There is **no database** — all state lives in the storage bucket.

## Commands

```bash
make build              # go build ./...
make build docker       # docker-compose build
make up                 # run locally with hot reload (requires `reflex`): reflex -r '\.go$' -- go run cmd/api/main.go
make up docker          # docker-compose up -d
make test_app           # run all Go tests with coverage (coverage.out), strips test/ and cmd/ lines
make test_app html      # same, plus generate coverage.html
make test_app docker    # run tests in the docker test profile
make test_app_watch     # re-run tests on file change (requires `entr`)

go test ./internal/branch/...                 # test a single package
go test -run TestName ./test/                 # run a single test by name
```

The dashboard (`apps/dashboard`, Vite + React) and CLI (`apps/eoas`, oclif/TypeScript) each have their own `package.json` with `build`/`lint`/`dev` scripts. The Go server serves the prebuilt dashboard from `apps/dashboard/dist` at `/dashboard`.

Running locally requires a `.env` file (CI creates an empty one if missing). `config.LoadConfig()` reads `.env` via godotenv and validates required vars at startup.

## Architecture

### Request entrypoint
`cmd/api/main.go` → `config.LoadConfig()` + `metrics.InitMetrics()` → `migration.RunMigrationsWithLock()` → `infrastructure.NewRouter()` (in `internal/router`). The router (gorilla/mux) defines all routes in one place — start there to find the handler for any endpoint.

Route groups:
- **Expo protocol (unauthenticated, called by client apps):** `/manifest`, `/assets`
- **Publishing (called by `eoas` CLI):** `/requestUploadUrl/{BRANCH}`, `/uploadLocalFile`, `/markUpdateAsUploaded/{BRANCH}`, `/rollback/{BRANCH}`, `/republish/{BRANCH}`
- **Dashboard auth:** `/auth/login`, `/auth/refreshToken`
- **Dashboard API (behind `AuthMiddleware`):** `/api/*`
- **Ops:** `/metrics` (Prometheus), `/hc` (health)

### Pluggable providers (the core design pattern)
Four subsystems each expose an interface + a factory that selects an implementation from an env var, with a cached singleton and a `Reset*Instance()` used by tests:

| Subsystem | Package | Env selector | Implementations |
|-----------|---------|--------------|-----------------|
| Storage (update bundles/assets) | `internal/bucket` | `STORAGE_MODE` | `s3` (also S3-compatible via `AWS_BASE_ENDPOINT`), `gcs`, `local` |
| Asset delivery | `internal/cdn` | derived from storage/env | CloudFront (`cloudfront.go`), GCS signed URLs (`gcs_direct.go`), generic/direct (`generic.go`) |
| Key storage (signing keys) | `internal/keyStore` | `KEYS_STORAGE_TYPE` | AWS Secrets Manager, environment vars, local files |
| Cache | `internal/cache` | `CACHE_MODE` | `local` (in-memory), `redis`, `redis-sentinel` |

When adding a provider, implement the interface, register it in the factory's `Resolve*`/`Get*` function, and add config validation in `config/config.go`.

### Update model & manifest flow
- `internal/update/updates.go` is the heart of the protocol: listing updates for a branch+runtimeVersion+platform, computing cache keys, building the signed `UpdateManifest`, and producing rollback / no-update-available directives. All response types live in `internal/types/types.go`.
- `internal/handlers/manifest_handler.go` builds the multipart, code-signed protocol response (`ManifestHandler`). Signing uses keys from `keyStore` via `internal/crypto`.
- Updates are identified by a timestamp-based UUID; metadata is persisted into the bucket (`StoreUpdateUUIDInMetadata`). An update directory in the bucket can also contain a rollback marker.
- `internal/services/expo.go` talks to Expo's GraphQL API to map channels↔branches and validate Expo auth tokens (`EXPO_ACCESS_TOKEN`, `EXPO_APP_ID`).

### Migrations (bucket data migrations, not DB)
`internal/migration` is a generic migration framework operating on the storage `Bucket`. Concrete migrations live in `internal/migrations/<timestamp>_<name>/` and self-register via a blank import in `internal/migrations/migrations.go` (which `main.go` blank-imports). `RunMigrationsWithLock()` runs them at startup with a lock to avoid concurrent runners. To add one: create a package under `internal/migrations/`, register it with `migration.Register`, and add its blank import.

### Caching
Manifest lookups, channel-branch mappings, and Expo auth are cached to avoid hitting storage/Expo on every request. Cache keys are computed by `Compute*CacheKey` helpers (in `internal/update` and `internal/services`). Keys are namespaced by `CACHE_KEY_PREFIX`. When changing what's stored or its shape, bump/adjust the relevant cache key helper so stale entries don't leak.

## Tests
Integration-style tests live in `test/` and use `internal/.../Reset*Instance()` to swap providers and `jarcoal/httpmock` to stub Expo's API. `test/helpers.go` and `test/expo_multipart_parser.go` provide shared setup. Snapshot-style assertions exist (see `make` snapshot test history) — when output format changes, expect snapshot updates. Coverage strips `test/` and `cmd/` lines (see Makefile).

## Key env vars
`STORAGE_MODE`, `BASE_URL`, `EXPO_ACCESS_TOKEN`, `EXPO_APP_ID`, `JWT_SECRET` are core. Storage: `S3_BUCKET_NAME`/`AWS_REGION`/`AWS_BASE_ENDPOINT`, `GCS_BUCKET_NAME`, `LOCAL_BUCKET_BASE_PATH`. Keys: `KEYS_STORAGE_TYPE` + the `AWSSM_*`, `PRIVATE_LOCAL_*`/`PUBLIC_LOCAL_*` paths, `CLOUDFRONT_*`. Cache: `CACHE_MODE`, `REDIS_*`, `CACHE_KEY_PREFIX`. Dashboard: `USE_DASHBOARD`, `ADMIN_PASSWORD`. Observability: `PROMETHEUS_ENABLED`. See `config/config.go` for validation logic and the docs site (`apps/docs/docs/server-configuration`) for the full reference.
