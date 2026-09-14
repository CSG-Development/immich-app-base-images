# Changelog

Append-only iteration history for Immich DHI base images (`server/`, `postgres/`, workflows).

### Iteration 2026-09-14 15:27 UTC

- **Context**
  - GitHub Actions concurrency was canceling long docker builds (`Error: The operation was canceled` while writing layers).
- **Target**
  - Keep one named concurrency group per workflow+ref, but do not abort an in-flight image build when another run starts.
- **Actions log**
  - Set `concurrency.cancel-in-progress` from `true` to `false` in `build-postgres.yml` and `build-server-base.yml`; left `group:` unchanged.
- **Validation**
  - YAML reviewed: both files still use `group: ${{ github.workflow }}-${{ github.ref }}` with `cancel-in-progress: false`.
- **Problems**
  - Queued runs for the same ref now wait instead of canceling; CI wall time may grow if pushes overlap. Status: **mitigated**.
- **To Be Done**
  - Push `fix/ci-build-after-upstream-merge` and confirm Actions complete without cancel-in-progress aborts.
- **References**
  - `CHANGELOG.md`, `.github/workflows/build-postgres.yml`, `.github/workflows/build-server-base.yml`

### Iteration 2026-09-14 12:35 UTC

- **Context**
  - Local linux/amd64 size check of uncommitted DHI Dockerfiles vs GHCR tags `202606260459` / `14-vectorchord0.4.3-pgvectors0.2.0` / `17-vectorchord0.4.3-pgvectors0.3.0`.
- **Target**
  - Build `local/base-server-{dev,prod}:sizecheck` and `local/postgres:{14,17}-sizecheck`; confirm `/opt` is not a second copy of vector extensions.
- **Actions log**
  - Logged in via existing `~/.docker/config.json` `dhi.io` auth; pulled `dhi.io/postgres:{14,17}-debian13-dev` and `dhi.io/node:24.14-debian13[-dev]`.
  - First postgres RUN failed: `ln: .../opt/postgresql/14/lib/vector.so: File exists` because current DHI already has `/opt/postgresql/N/{lib,share}` → `/usr/lib/postgresql/N/{lib,share}`.
  - Adapted postgres RUN to skip per-file `.so` links when dest exists; link Debian `/usr/share/.../extension` into DHI sharedir.
  - `docker buildx build --platform linux/amd64 --load` postgres 14 (`PGVECTORS_SUFFIX` empty) and 17 (`_vectors`).
  - Rootless server build failed in `configure-apt.sh` writing `/etc/apt/preferences.d/preferences` (`Value too large for defined data type`).
  - Rebuilt server on rootful `unix:///var/run/docker.sock` with `--network=host` (bridge DNS/IPv6 could not reach `deb.debian.org`).
- **Validation**
  - Postgres 14: `docker inspect` 810990202 (~811 MB) vs GHCR 1090300618 (~1.09 GB); RUN layer 322 MB vs 645 MB; `/opt/postgresql` 12K vs GHCR 363M.
  - Postgres 17: 1072861370 (~1.07 GB) vs GHCR 1599673880 (~1.60 GB); RUN layer 573 MB vs 1.15 GB; `/opt/postgresql` 12K.
  - `/opt/postgresql/N/lib` is a DHI directory symlink; `vchord.so`/`vector.so`/`vectors.so` share inodes with `/usr/lib/...` (regular files, not per-file symlinks); extension SQL in sharedir is `ln -s` to `/usr/share`.
  - Server-dev: 1643701498 (~1.64 GB) vs GHCR 1635791387 (~1.64 GB); server-prod: 1728520189 (~1.73 GB) vs GHCR 1678392072 (~1.68 GB).
- **Problems**
  - File-level `ln -s` into `/opt/.../lib` is incompatible with current DHI dir-level `/opt`→`/usr` maps. Status: **resolved** (skip-if-exists).
  - Rootless overlay + DHI `gid=nobody` apt dirs: EOVERFLOW; GHA/rootful OK. Status: **mitigated**.
  - `ADD` `.deb` layers remain after `rm` (~44 MB PG14, ~48 MB PG17). Status: **open**.
- **To Be Done**
  - Push `fix/ci-build-after-upstream-merge` and confirm Actions on rootful runners; pin published postgres digest downstream.
- **References**
  - `CHANGELOG.md`, `postgres/Dockerfile`, `server/Dockerfile`, `server/configure-apt.sh`

### Iteration 2026-09-14 12:05 UTC

- **Context**
  - Postgres images stored VectorChord/pgvector/pgvecto.rs `.so` and extension files twice (`/usr` and `/opt`); PG14 ~1.09 GB vs upstream 756 MB.
- **Target**
  - Keep one canonical copy under Debian `/usr` paths; DHI `/opt/postgresql/${PG_MAJOR}/…` should be absolute symlinks, not a second copy.
- **Actions log**
  - After extracting debs into `/usr/lib` and `/usr/share`, replaced `cp -a` into `/opt` with `ln -s` using absolute targets.
  - `vector*` glob also covers `vectors*` control/sql when `PGVECTORS_TAG` is set (all four `versions.yaml` rows).
- **Validation**
  - Dockerfile reviewed: PG 14–16 (`pgvectors` 0.2.0, empty suffix) and PG17 (0.3.0 + `_vectors`) still use the same `RUN`; DHI rebuild not run locally.
- **Problems**
  - Size win needs a published image; if Postgres/`dlopen` does not follow `/opt` symlinks, that is a follow-up. Status: **open**.
- **To Be Done**
  - Rebuild postgres on CI; confirm `/opt/.../vchord.so` is a symlink and image size drops; pin digest downstream.
- **References**
  - `CHANGELOG.md`, `postgres/Dockerfile`, `postgres/versions.yaml`

### Iteration 2026-09-14 11:41 UTC

- **Context**
  - `move-to-dhi` merged upstream tag `202607211135` (PR #2); GitHub Actions then failed on server, postgres, and empty leftover workflows.
- **Target**
  - Restore CI builds without bumping VectorChord past `0.4.3` or dropping `pgvectors`.
- **Actions log**
  - Dropped stale `libexpat1=2.7.1-2` pin from `server/Dockerfile` (Debian trixie-security is now `2.8.3-1~deb13u1`).
  - Fetch VectorChord/pgvecto.rs `.deb` files with Dockerfile `ADD` (BuildKit follows GitHub 301; no DHI apt/wget).
  - Deleted empty merge-conflict stubs `.github/workflows/{test,org-zizmor,tag-server-base}.yml`.
- **Validation**
  - VectorChord `0.4.3` download URL 301s to `supervc-stack` (~12 MB); pgvecto.rs `0.2.0` / `0.3.0_vectors` assets return 302. Last green server/postgres runs were 2026-07-17.
  - Full DHI image rebuild not run locally (needs `dhi.io` login + GHCR).
- **Problems**
  - GHCR republish and consumer digest pin still pending until this branch builds. Status: **open**.
- **To Be Done**
  - Push `fix/ci-build-after-upstream-merge`, confirm Actions, merge to `move-to-dhi`; pin published postgres digest downstream.
- **References**
  - `CHANGELOG.md`, `server/Dockerfile`, `postgres/Dockerfile`, `.github/workflows/`

### Iteration 2026-07-17 08:07 UTC

- **Context**
  - CSG DHI postgres inherited `PGDATA=/var/lib/postgresql/14/data`; Immich/homecloud mount `/var/lib/postgresql/data`, so initdb ran off the bind mount after compose pinned `ghcr.io/csg-development/postgres`.
- **Target**
  - Bake Immich-compatible `PGDATA=/var/lib/postgresql/data` into the postgres image on branch `fix/postgres-pgdata-path` (from `move-to-dhi`).
- **Actions log**
  - Set `ENV PGDATA=/var/lib/postgresql/data` in `postgres/Dockerfile` after the DHI `FROM`.
- **Validation**
  - Dockerfile diff reviewed; rebuild/publish and digest pin deferred to follow-up.
- **Problems**
  - Published GHCR digest and homecloud/test redeploy still pending. Status: **open**.
- **To Be Done**
  - Rebuild/publish `14-vectorchord0.4.3-pgvectors0.2.0`; pin new digest in `immich-app-immich` compose.prod; set `PGDATA` in homecloud compose; realign test volume.
- **References**
  - `CHANGELOG.md`, `postgres/Dockerfile`

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
