# ci-pipelines

Centralized reusable Gitea Actions pipelines per language/stack. App repos
call these instead of duplicating build workflows — one canonical pipeline per
language, each free to evolve (tests, lint, hardening) without touching
consumers.

## Pipelines

| Workflow | For | Image pattern |
|---|---|---|
| `.gitea/workflows/build-go.yml` | Go services (stdlib or any) | multi-stage → `distroless/static:nonroot` |
| `.gitea/workflows/build-csharp.yml` | C#/.NET 8+ services | `sdk:8.0` publish → `aspnet:8.0` (nonroot UID 1654) |
| `.gitea/workflows/build-rust.yml` | Rust services (axum/tokio etc.) | `rust:1-alpine` musl release → `distroless/static:nonroot` |

All three: build + push `:latest` + `:<sha8>` to the Gitea container registry
(default `192.168.1.2:3000`), tagged by the pushed commit's short SHA.

## Usage (caller repo)

```yaml
name: build-image

on:
  push:
    branches: [main]

jobs:
  build:
    uses: reeds/ci-pipelines/.gitea/workflows/build-go.yml@main
    with:
      image: reeds/smoke-go        # required: registry path, no host prefix
      # dockerfile: Dockerfile     # optional
      # registry: 192.168.1.2:3000 # optional
    secrets: inherit               # caller repo MUST define REGISTRY_PUSH_TOKEN
```

The caller repo needs a `REGISTRY_PUSH_TOKEN` Actions secret (registry
credential for user `reeds`).

## Deploy note

Images are referenced by tag in `reeds/sutliff-cluster` manifests (Flux).
Pin the manifest to the `:<sha8>` tag; re-pushing the same `:latest` does NOT
re-pull (bump the tag to roll a new image).

## Requirements

- Gitea ≥ 1.19 with reusable workflows (`workflow_call` / `uses:`) — this
  homelab runs v1.28.0-dev (verified `modules/actions/jobparser/workflow_call.go`).
- Runner label `[self-hosted]` (DinD sidecar; the job container re-mounts
  `/var/run/docker.sock` from the runner pod's shared emptyDir).
