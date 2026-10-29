# image-build

Container images built from upstream release tags and pushed to GHCR
(`ghcr.io/pyrogeen/`).

Each top-level directory is one app:

- `build.yaml` — upstream repo, pinned tag, target image name. **Renovate tracks the
  `tag:` line** (git-tags datasource via custom manager) and opens a PR per new
  upstream release.
- build inputs live in the upstream repo; the workflow clones upstream at the pinned
  tag and builds its own `Dockerfile` with docker on a GitHub-hosted runner.
- Local files in the app dir (e.g. a `Dockerfile`) override upstream build context.

On merge to `main`, `.github/workflows/build.yaml` builds the image and pushes it as
`<image>:<upstream-tag>` (no `latest`).

## Registry access

Builds authenticate with the default `GITHUB_TOKEN` (`packages: write`) — no
separate token needed.

## Adding a new app

1. Create `<app>/build.yaml` with `upstream:`, `tag:`, `image:`, `registry: ghcr.io`.
2. Push to `main` — the workflow builds and pushes the image.
3. Renovate picks up the file automatically (custom manager pattern).

## Consumers

Images are pulled by icenet HelmReleases (see `icenet/cluster/mcp/deploy/`). For
self-hosted or anonymous pulls the packages must be **public** on GHCR, or the
consumer namespaces need a `ghcr.io` pull secret.
