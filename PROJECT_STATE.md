# Project state — immich-app-base-images

Structured companion to `CHANGELOG.md`.

## Context

- Builds **base-server-dev**, **base-server-prod**, and **postgres** (PG 14–17 with vector extensions) from **Docker Hardened Images** (`dhi.io/node:24.14-debian13[-dev]`, `dhi.io/postgres`), publishes to `ghcr.io/csg-development/`.
- Upstream sync baseline: `immich-app/base-images` `171518b` (#319) = `base-server-dev:202603251709` (fork point `a639f69` #311).
- Image layout (folders, `LD_LIBRARY_PATH`, minimal vs dev): summarized in `CHANGELOG.md` historical iterations (former standalone result doc).

## Target

- Server: multi-stage — no shell in final prod stage except copied **bash** for Immich `tini` entrypoint; VIPS stack with `/usr/local/lib` on `LD_LIBRARY_PATH`.
- Postgres: extension layout under `/opt/postgresql/`; entrypoint remains bash-based on `-dev` image.
- GitHub Actions: build/push on relevant branches; DHI registry auth via secrets.

## Validation

- Build matrices and workflows as in repo `.github/workflows/`; Postgres build-args in `CHANGELOG.md` (DHI Postgres iteration).

## Problems

- **Valkey protected mode**: not fixed in this repo — consumer compose must `valkey-server --protected-mode no` on private networks. Status: **mitigated** (downstream).
- **Postgres minimal image**: blocked by bash entrypoint. Status: **open** (accepted).

## To Be Done

1. Keep `postgres/versions.yaml` in sync with upstream .deb availability (pgvectors PG17, etc.).
2. Verify CI-published GHCR tags after upstream sync rebuild on `move-to-dhi`.

## References

- `CHANGELOG.md` (mandatory)
- `server/Dockerfile`, `postgres/Dockerfile`, `postgres/versions.yaml`
