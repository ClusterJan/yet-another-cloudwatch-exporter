# Ko Build Configuration

## Build Summary

The multi-architecture container image was successfully built and pushed.

**Image:** `ttl.sh/yace-test:0.63.0@sha256:ce8eb60d7a530eac17038e19d0a8374d4e74b7680b83620efb898189a6384591`

**Platforms:**
- linux/amd64
- linux/arm64

## Configuration

The `.ko.yaml` configuration file at the project root includes:
- **Base image:** `gcr.io/distroless/static:nonroot` (minimal, secure base)
- **Build optimizations:** `-s -w` ldflags to strip debug info
- **Version injection:** Prometheus version variables from environment

## Usage

### Push to Registry

```bash
# Set your registry
export KO_DOCKER_REPO=ghcr.io/your-username/yace
# or
export KO_DOCKER_REPO=docker.io/your-username/yace

# Set version info
export VERSION=$(cat VERSION)
export REVISION=$(git rev-parse --short HEAD)
export BRANCH=$(git rev-parse --abbrev-ref HEAD)
export BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Build and push multi-arch
ko build --platform=linux/amd64,linux/arm64 --bare --tags=${VERSION} ./cmd/yace
```

### Load to Local Docker

```bash
export KO_DOCKER_REPO=ko.local
ko build --local ./cmd/yace
```

## Notes

The image at `ttl.sh` is temporary (expires after ~24h). Use your own registry like Docker Hub, GHCR, or ECR for production deployments.