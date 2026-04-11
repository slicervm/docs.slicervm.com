# Publish Your Own Images

This page provides a GitHub Actions workflow for building and publishing a derived Slicer image to GHCR. For how to write the Dockerfile itself, see [Build a custom image](/platform/custom-images/).

Slicer images are not multi-arch. If you need arm64 images, add a separate job on an arm64 runner (`ubuntu-24.04-arm`) and tag accordingly (e.g. `aarch64-latest`).

## GitHub Actions workflow

Create `.github/workflows/publish-image.yaml`:

```yaml
name: publish-image

on:
  push:
    branches:
      - main
    paths:
      - "Dockerfile"
      - ".github/workflows/publish-image.yaml"

permissions:
  packages: write
  contents: read

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64
          push: true
          tags: |
            ghcr.io/${{ github.repository_owner }}/slicer-custom:x86_64-latest
```

After pushing, update your Slicer YAML to reference the new image:

```yaml
config:
  image: "ghcr.io/your-org/slicer-custom:x86_64-latest"
```

GHCR packages default to private. Make the package public in your GitHub package settings, or set `insecure_registry: true` in your Slicer YAML if you are using a private registry without TLS.

## See also

* [Build a custom image](/platform/custom-images/) - writing the Dockerfile
* [Images for microVMs](/reference/images/) - base image tags and availability
