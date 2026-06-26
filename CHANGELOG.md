# Changelog

Append-only iteration history for Immich DHI base images (`server/`, `postgres/`, workflows).

### Iteration 2026-06-26 04:58 UTC

- **Context**
  - Sync DHI fork to upstream `immich-app/base-images` commit `171518b` (#319), source of `ghcr.io/immich-app/base-server-dev:202603251709`; fork diverged at `a639f69` (#311).
- **Target**
  - Port upstream delta: postgres `postgresql.override.conf` include, DHI Node pin to 24.14, CI action bumps in `build-postgres.yml`; rebuild and publish images.
- **Actions log**
  - Added `include_if_exists 'postgresql.override.conf'` to `postgres/postgresql.hdd.conf` and `postgresql.ssd.conf`.
  - Pinned `server/Dockerfile` DHI bases to `24.14-debian13[-dev]` (base, prod-build, prod).
  - Bumped `build-postgres.yml` docker actions: qemu/buildx/login v4, metadata v6, build-push v7.
- **Validation**
  - `docker pull dhi.io/node:24.14-debian13-dev` and `dhi.io/node:24.14-debian13` succeeded.
- **Problems**
  - None. Status: **resolved**.
- **To Be Done**
  - Push to `move-to-dhi` to trigger CI rebuild; verify published GHCR tags.
- **References**
  - `CHANGELOG.md`, `server/Dockerfile`, `postgres/postgresql.*.conf`, `.github/workflows/build-postgres.yml`

### Iteration 2026-04-01 14:00 UTC

- **Context**
  - Consolidate former standalone notes (DON-223 scope, image result structure) into repo logs.
- **Target**
  - Record DHI bases, multi-stage server build, Postgres extensions matrix, workflows, and known fixes (bash/ln, `LD_LIBRARY_PATH`, Valkey).
- **Validation**
  - `CHANGELOG.md` + `PROJECT_STATE.md` created at repo root.
- **Problems**
  - None new. Status: **resolved**.
- **To Be Done**
  - Rebuild/publish when DHI tags change; keep `versions.yaml` sets valid per PG major.
- **References**
  - `CHANGELOG.md`

### Iteration (historical) — DHI server image (Node 24)

- **Context**
  - `server/Dockerfile`: prod DHI runtime has no `/bin/sh`; all `RUN` in `prod-build` from `dhi.io/node:24-debian13-dev`; final stage `COPY` only.
- **Target**
  - Images `base-server-dev` / `base-server-prod` published to `ghcr.io/csg-development/`; workflow `build-server-base.yml` on `main` / `move-to-dhi`.
- **Validation**
  - Build succeeds; non-root `ln` fix: `USER root` for symlink step; `base-server-prod` includes `/bin/bash` for Immich entrypoint.
- **Problems**
  - VIPS warnings if `LD_LIBRARY_PATH` omits `/usr/local/lib` — fixed in prod stage env. Status: **resolved**.
- **To Be Done**
  - Consumer `immich-app-immich` Dockerfile to use published GHCR bases.
- **References**
  - `server/Dockerfile`, `CHANGELOG.md`

### Iteration (historical) — DHI Postgres + extensions

- **Context**
  - Base `dhi.io/postgres:${PG_MAJOR}-debian13-dev`; extensions in `/opt/postgresql/${PG_MAJOR}/` per DHI; pgvector, VectorChord, optional pgvecto.rs.
- **Target**
  - `postgres/versions.yaml` uses explicit `sets` (PG 14–17) to avoid invalid pgvector/pgvectors combos; PG17 uses pgvectors 0.3.0.
- **Validation**
  - `docker build` with documented build-args; workflow `build-postgres.yml`.
- **Problems**
  - Full minimal runtime blocked: entrypoint is bash — stays on `-dev`. Status: **open** (accepted trade-off).
- **To Be Done**
  - Optional rewrite entrypoint to posix + busybox for minimal image.
- **References**
  - `postgres/Dockerfile`, `postgres/versions.yaml`, `CHANGELOG.md`

### Iteration (historical) — CI and registries

- **Context**
  - DHI login in CI; GHCR push; lowercase `repository_owner`; attestations disabled where needed.
- **Target**
  - Reproducible builds; GHCR 403 mitigations (org permissions, PAT).
- **Validation**
  - Documented in historical DON-223 notes (GHCR troubleshooting).
- **Problems**
  - GHCR 403 — ops-dependent. Status: **mitigated** (doc).
- **To Be Done**
  - Pin critical images by digest where policy requires (e.g. Valkey).
- **References**
  - `.github/workflows/`, `CHANGELOG.md`

### Iteration 2026-04-01 15:00 UTC

- **Context**
  - Seagate shared docs `../docs/DON-223.md` and `../docs/immich-base-images-result.md` removed after consolidation.
- **Target**
  - Single source: this `CHANGELOG.md`; no broken links.
- **Validation**
  - Files deleted; `PROJECT_STATE.md` updated.
- **Problems**
  - None. Status: **resolved**.
- **To Be Done**
  - None.
- **References**
  - `CHANGELOG.md`, `PROJECT_STATE.md`
