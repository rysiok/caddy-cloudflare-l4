# Copilot Instructions

## Project Overview

This repository produces custom Caddy Docker images with these modules compiled in via `xcaddy`:

- [caddy-dns/cloudflare](https://github.com/caddy-dns/cloudflare) — Cloudflare DNS-01 ACME challenge support
- [caddy-cloudflare-ip](https://github.com/WeidiDeng/caddy-cloudflare-ip) + [caddy-combine-ip-ranges](https://github.com/fvbommel/caddy-combine-ip-ranges) — Cloudflare proxy IP trust
- [caddy-l4](https://github.com/mholt/caddy-l4) — Layer 4 (TCP/UDP) routing

There is no application source code to build or test locally. The deliverables are Docker images and CI workflows.

## Architecture

**Two Dockerfiles, one pattern:** `Dockerfile` (Debian-based) and `Dockerfile.alpine` both follow the same multi-stage build — use the `caddy:VERSION-builder` image to run `xcaddy build` with the four modules, then copy the resulting binary into the final `caddy:VERSION` image. The `CADDY_VERSION` build arg controls which upstream Caddy version to target.

**Automated release pipeline:** A scheduled GitHub Actions workflow (`check-caddy-release.yml`) runs daily. It executes `scripts/check_caddy_status.py` which checks the latest Caddy GitHub release against what's already published on Docker Hub. If a new version is detected and the official Caddy image has all required platforms available, it dispatches a `caddy-release` event that triggers both build workflows (`build-docker-image-standard.yml` and `build-docker-image-alpine.yml`).

**Multi-platform builds:** Images are built for `linux/amd64`, `linux/arm64`, `linux/arm/v7`, `linux/ppc64le`, and `linux/s390x`. This platform list must stay in sync between `scripts/check_caddy_status.py` (`REQUIRED_PLATFORMS`) and the `platforms:` field in both build workflow files.

## Key Conventions

- Caddy versions use the bare semver format (`2.8.0`) throughout Docker tags and build args — the `v` prefix from GitHub release tags is always stripped.
- Both Dockerfiles must list the same set of `--with` modules. When adding or removing a module, update both files.
- `Caddyfile_global` and `Caddyfile_per_site` are reference examples for users, not used by the build.
- The check script (`scripts/check_caddy_status.py`) communicates with GitHub Actions via `GITHUB_OUTPUT` using `set_action_output()`. It outputs `NEEDS_BUILD` and `LATEST_VERSION`.

## CI Workflows

| Workflow | Trigger | Purpose |
|---|---|---|
| `check-caddy-release.yml` | Daily cron + manual | Detects new Caddy releases, dispatches build events |
| `build-docker-image-standard.yml` | `repository_dispatch` + manual | Builds and pushes Debian-based image |
| `build-docker-image-alpine.yml` | `repository_dispatch` + manual | Builds and pushes Alpine-based image |
| `stale.yml` | Daily cron | Auto-closes stale issues/PRs |

## Required Secrets/Variables

- `GITHUB_TOKEN` — automatic, used for GHCR push and API calls
- `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` — optional, for Docker Hub push
- `DOCKERHUB_REPOSITORY_NAME` — optional variable/secret to override the Docker Hub repo name
