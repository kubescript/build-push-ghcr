# Build and Push Docker Image to GHCR

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Build%20Push%20GHCR-blue.svg?colorA=24292e&colorB=0366d6&style=flat&longCache=true&logo=github)](https://github.com/marketplace/actions/build-and-push-docker-image-to-ghcr)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

A GitHub Action to build and push Docker images to GitHub Container Registry (GHCR) using the built-in `GITHUB_TOKEN` (no extra credentials needed).

## Features

- **Zero-credential auth**: Uses the workflow's built-in `GITHUB_TOKEN` to push to GHCR
- **BuildKit caching**: Fast rebuilds with GitHub Actions cache integration
- **Smart tagging**: Auto-generated tags for branches, PRs, commits, semver releases, and `latest`
- **Multi-arch builds**: Optional support for `linux/amd64`, `linux/arm64`, and others
- **Multi-stage builds**: Pick a target stage in your Dockerfile
- **Validation runs**: Build without pushing for PR-only verification

## Automatic Tagging

By default this action delegates tag generation to [`docker/metadata-action`](https://github.com/docker/metadata-action). The defaults produce:

| Trigger | Tags generated |
|---------|----------------|
| Push to default branch (e.g., `main`) | `main`, `sha-<short>`, `latest` |
| Push to non-default branch (e.g., `staging`) | `staging`, `sha-<short>` |
| Pull request | `pr-<number>`, `sha-<short>` |
| Git tag `v1.2.3` | `1.2.3`, `sha-<short>` |

You can fully override the tag rules via the `tags` input.

## Prerequisites

### GitHub Actions Permissions

Your workflow must declare:

```yaml
permissions:
  contents: read
  packages: write
```

`packages: write` is required so the built-in `GITHUB_TOKEN` can push to GHCR.

### Checkout

This action does **not** check out your repository — call `actions/checkout` before using it so the build context exists on disk.

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `image-name` | Image name under `ghcr.io/<owner>/` (e.g., `my-service`) | Yes | - |
| `token` | GitHub token with `packages:write`. Defaults to `github.token`. | No | `${{ github.token }}` |
| `docker-file-path` | Path to Dockerfile relative to repo root | No | `Dockerfile` |
| `docker-build-context` | Docker build context directory | No | `.` |
| `docker-build-target` | Multi-stage build target | No | `''` |
| `docker-build-args` | Build args, one `KEY=VALUE` per line | No | `''` |
| `tags` | Tag rules for `docker/metadata-action`, one per line | No | _see action.yml_ |
| `labels` | OCI image labels, one `KEY=VALUE` per line | No | OCI source/revision |
| `platforms` | Comma-separated target platforms (e.g., `linux/amd64,linux/arm64`) | No | `''` (runner native) |
| `cache-from` | BuildKit cache source | No | `type=gha` |
| `cache-to` | BuildKit cache destination (ignored when `push=false`) | No | `type=gha,mode=max` |
| `push` | `'true'` to push, `'false'` for build-only validation | No | `'true'` |

## Outputs

| Output | Description | Example |
|--------|-------------|---------|
| `version` | Version resolved by `docker/metadata-action` | `1.2.3` |
| `image-tags` | Newline-separated list of all tags pushed | `ghcr.io/org/svc:main\nghcr.io/org/svc:sha-abc123` |
| `image-repo` | Full image repository without tag | `ghcr.io/my-org/my-service` |
| `image-digest` | Manifest digest of the pushed image | `sha256:abc123...` |

## Usage

### Basic Example

```yaml
name: Build and Push

on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - name: Build and push to GHCR
        uses: kubescript/build-push-ghcr@v1
        with:
          image-name: my-service
```

### Pull Request Validation (Build Only)

```yaml
name: PR Build Check

on:
  pull_request:

permissions:
  contents: read
  packages: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - uses: kubescript/build-push-ghcr@v1
        with:
          image-name: my-service
          push: 'false'
```

### Semantic Version Release

```yaml
name: Release

on:
  push:
    tags: ['v*']

permissions:
  contents: read
  packages: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - id: build
        uses: kubescript/build-push-ghcr@v1
        with:
          image-name: my-service

      - run: |
          echo "Pushed version: ${{ steps.build.outputs.version }}"
          echo "Digest: ${{ steps.build.outputs.image-digest }}"
```

### Multi-Arch Build with Build Args

```yaml
- uses: kubescript/build-push-ghcr@v1
  with:
    image-name: my-service
    platforms: linux/amd64,linux/arm64
    docker-build-args: |
      APP_ENV=production
      COMMIT_SHA=${{ github.sha }}
```

### Multi-Stage Build

```yaml
- uses: kubescript/build-push-ghcr@v1
  with:
    image-name: my-service
    docker-build-target: production
    docker-file-path: ./docker/Dockerfile
```

### Monorepo

```yaml
- uses: kubescript/build-push-ghcr@v1
  with:
    image-name: api-service
    docker-build-context: ./services/api
    docker-file-path: ./services/api/Dockerfile
```

### Custom Tag Rules

```yaml
- uses: kubescript/build-push-ghcr@v1
  with:
    image-name: my-service
    tags: |
      type=ref,event=branch
      type=sha,prefix=,format=long
      type=raw,value=stable,enable={{is_default_branch}}
```

### Using Outputs Across Jobs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      digest: ${{ steps.build.outputs.image-digest }}
      repo: ${{ steps.build.outputs.image-repo }}
    steps:
      - uses: actions/checkout@v6
      - id: build
        uses: kubescript/build-push-ghcr@v1
        with:
          image-name: my-service

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "Deploying ${{ needs.build.outputs.repo }}@${{ needs.build.outputs.digest }}"
```

## Troubleshooting

### Error: `denied: installation not allowed to Write organization package`

**Problem**: The `GITHUB_TOKEN` cannot push to a package owned by a different org.

**Solution**: Either move the package to the same org as the repo, or pass a Personal Access Token with `write:packages` scope via the `token` input.

### Error: `Dockerfile not found`

**Problem**: Dockerfile path is wrong relative to the build context.

**Solution**: `docker-file-path` is resolved relative to the repo root (not the context). Set it to the full path inside the repo, e.g. `./services/api/Dockerfile`.

### Slow builds

**Solution**:
- Confirm `cache-from` and `cache-to` are set (they are by default)
- Reorder Dockerfile layers so frequently changing instructions come last
- Use multi-stage builds to keep final images small

### Multi-arch builds time out

**Problem**: Cross-arch builds use QEMU emulation and are significantly slower.

**Solution**: Increase `timeout-minutes` on the calling job, or split per-arch jobs and merge manifests separately.

### Build args not being applied

**Solution**: Build args must match `ARG` declarations in your Dockerfile — undeclared args are silently ignored. Verify with:

```bash
grep -E '^ARG ' Dockerfile
```

## Acknowledgments

This action uses the following third-party GitHub Actions:

- **[docker/metadata-action](https://github.com/docker/metadata-action)** - Compute Docker image tags and labels
- **[docker/setup-buildx-action](https://github.com/docker/setup-buildx-action)** - Set up Docker Buildx
- **[docker/login-action](https://github.com/docker/login-action)** - Login to container registries
- **[docker/build-push-action](https://github.com/docker/build-push-action)** - Build and push Docker images

Thank you to all the maintainers and contributors of these projects!

## Contributing

Contributions are welcome! Please open an issue or submit a pull request.

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.

## Support

- Report issues: [GitHub Issues](https://github.com/kubescript/build-push-ghcr/issues)
- Documentation: [GitHub Marketplace](https://github.com/marketplace/actions/build-and-push-docker-image-to-ghcr)

---

Made with ❤️ by [KubeScript](https://github.com/kubescript)
