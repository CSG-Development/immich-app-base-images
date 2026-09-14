# Project state — immich-app-base-images

Structured companion to `CHANGELOG.md`.

## CHANGELOG vs this file

- `CHANGELOG.md` is append-only iteration history.
- This file is the living snapshot: wiring, current goals, open risks, short backlog.

## Context

- Builds **base-server-dev**, **base-server-prod**, and **postgres** (PG 14–17 with vector extensions) from **Docker Hardened Images** (`dhi.io/node:24.14-debian13[-dev]`, `dhi.io/postgres`), publishes to `ghcr.io/csg-development/`.
- Upstream sync baseline: tag `202607211135` (`cf578f0` #359) merged to `move-to-dhi` via PR #2; DHI `FROM`/`sets`/`PGDATA` kept. VectorChord stays `0.4.3` + `pgvectors` (not upstream `1.1.1`).
- Image layout (folders, `LD_LIBRARY_PATH`, minimal vs dev): summarized in `CHANGELOG.md` historical iterations.

## Target

- Server: multi-stage — no shell in final prod stage except copied **bash** for Immich `tini` entrypoint; VIPS stack with `/usr/local/lib` on `LD_LIBRARY_PATH`.
- Postgres: real extension files under `/usr/lib` and `/usr/share`; DHI runtime `/opt/postgresql/${PG_MAJOR}/…` is absolute `ln -s` to those files; entrypoint remains bash-based on `-dev` image; `ENV PGDATA=/var/lib/postgresql/data` overrides DHI versioned default so Immich volume mounts work.
- GitHub Actions: build/push on relevant branches; DHI registry auth via secrets.

## Validation

- Build matrices and workflows as in repo `.github/workflows/`; Postgres build-args in `CHANGELOG.md` (DHI Postgres iteration).
- Local linux/amd64 size check (2026-09-14): see `CHANGELOG.md` iteration 12:35 UTC (`local/*:sizecheck` vs GHCR).

## Problems

- **Valkey protected mode**: not fixed in this repo — consumer compose must `valkey-server --protected-mode no` on private networks. Status: **mitigated** (downstream).
- **Postgres minimal image**: blocked by bash entrypoint. Status: **open** (accepted).
- **Postgres PGDATA vs Immich mount**: Dockerfile override is in the image; GHCR republish + consumer digest/compose still pending. Status: **open**.
- **CI after upstream 3.1.0 merge**: server `libexpat` pin and postgres DHI wget failed on `move-to-dhi`; fix is on `fix/ci-build-after-upstream-merge`. Status: **open**.
- **Postgres extension duplication**: current DHI already directory-symlinks `/opt/postgresql/N/{lib,share}` → `/usr/lib/...`; local PG14 811 MB / PG17 1.07 GB vs GHCR 1.09 / 1.60 GB; `/opt` is 12K. Status: **mitigated** (CI publish still pending).
- **Rootless Docker + DHI apt dirs**: writing `/etc/apt/preferences.d` fails with EOVERFLOW (`userxattr`); use rootful or GHA. Status: **mitigated**.

## To Be Done

1. Land `fix/ci-build-after-upstream-merge` on `move-to-dhi` after Actions pass.
2. Rebuild/publish postgres with `PGDATA=/var/lib/postgresql/data`; pin digest in `immich-app-immich` compose.prod.
3. Defer VectorChord `1.1.1` / drop `pgvectors` until `immich-app-immich` syncs.

## References

- `CHANGELOG.md` (mandatory)
- `server/Dockerfile`, `postgres/Dockerfile`, `postgres/versions.yaml`
