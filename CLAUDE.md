# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes. It follows a microservices architecture with a single binary that acts as different components based on invocation (similar to BusyBox).

## Build and Development Commands

### Building Components

```bash
# Build all (cli + image)
make all

# Build CLI only (local)
make cli-local

# Build all Go code
make build-local

# Build specific components
make server           # API server
make controller       # Application controller
make repo-server      # Repository server
make applicationset-controller
```

### Testing

```bash
# Run unit tests (all tests except e2e)
make test-local

# Run unit tests for a specific module
make test-local TEST_MODULE=github.com/argoproj/argo-cd/v3/server

# Run tests with race detection
make test-race-local

# Run e2e tests (requires test environment running)
make test-e2e-local

# Start local e2e test environment
make start-e2e-local
```

### Running Locally

```bash
# Start all services locally (requires kubectl access)
make start-local

# Run specific services with goreman
make run  # Use with EXCLUDE env var to exclude services

# Start development server (via goreman directly)
goreman -f Procfile start
```

### Code Generation

```bash
# Full codegen (mocks, proto, client, openapi, docs, manifests)
make codegen-local

# Fast codegen (skips vendor)
make codegen-local-fast

# Individual generators
make mockgen              # Generate mocks
make protogen-fast        # Generate protobuf code
make clientgen            # Generate Kubernetes clients
make openapigen           # Generate OpenAPI specs
make clidocsgen           # Generate CLI documentation
make manifests-local      # Generate Kubernetes manifests
```

### Linting

```bash
# Lint Go code
make lint-local

# Lint UI code
cd ui && yarn lint
```

### UI Development

```bash
# Install UI dependencies
cd ui && yarn install

# Build UI
make build-ui

# Start UI dev server
cd ui && yarn start
```

## Architecture

### Core Components

**Single Binary, Multiple Roles**: The `cmd/main.go` entry point determines which component to run based on the binary name (ARGOCD_BINARY_NAME):

1. **Application Controller** (`controller/`)
   - Reconciliation engine that monitors applications and syncs their desired state with live state
   - Handles resource caching, health assessment, and sync operations
   - Key files: `appcontroller.go`, `state.go`, `sync.go`

2. **API Server** (`server/`)
   - REST and gRPC API server providing the interface for UI and CLI
   - Organized by resource types: `application/`, `cluster/`, `project/`, `repository/`, etc.
   - Handles authentication, RBAC authorization, and session management

3. **Repository Server** (`reposerver/`)
   - Manages Git operations and manifest generation
   - Handles caching, GPG verification, and repository access
   - Supports Helm, Kustomize, and custom config management plugins

4. **ApplicationSet Controller** (`applicationset/`)
   - Manages ApplicationSet resources for multi-cluster/multi-tenant scenarios
   - Contains generators: Git, Cluster, List, Matrix, etc.

5. **Commit Server** (`commitserver/`)
   - Handles commit operations to Git repositories

6. **CMP Server** (`cmpserver/`)
   - Config Management Plugin server for custom tooling integration

7. **Notification Controller** (`notification_controller/`)
   - Sends notifications and alerts for application events

### Key Directories

- **`cmd/`**: Main entry point - single binary that becomes different components
- **`pkg/`**: Shared packages (apiclient, apis, client libraries)
- **`util/`**: 66 utility packages (app, argo, cache, db, helm, kustomize, git, grpc, settings, etc.)
- **`common/`**: Shared constants, version management, and common functionality
- **`gitops-engine/`**: Core GitOps library (resource cache, reconciliation, sync planning)
- **`ui/`**: React + TypeScript web application
- **`test/`**: E2E tests, fixtures, and test utilities
- **`resource_customizations/`**: Custom resource health checks and actions
- **`manifests/`**: Kubernetes manifests for deploying Argo CD

### Service Communication

- Services communicate internally via gRPC
- External API exposed via REST and gRPC-web
- Redis used for caching and inter-service communication
- Component ports (defaults):
  - API Server: 8080
  - Repo Server: 8081
  - Redis: 6379
  - Dex (Auth): 5556
  - Commit Server: 8086

### Testing Architecture

- Unit tests co-located with source files (`*_test.go`)
- E2E tests in `test/e2e/`
- Test fixtures in `test/fixture/`
- Docker-based test environment (see `test/container/`)
- Use `goreman` to run all services for e2e tests

## Development Workflow

### Making Changes

1. Make code changes
2. Run code generation if needed: `make codegen-local-fast`
3. Build: `make build-local`
4. Run tests: `make test-local`
5. Lint: `make lint-local`

### Local Testing

```bash
# Terminal 1: Start local Argo CD
make start-local

# Terminal 2: Use CLI
./dist/argocd login localhost:8080 --insecure
./dist/argocd app list
```

### Pre-commit

```bash
# Run full pre-commit validation (codegen, build, lint, test)
make pre-commit-local
```

## Important Implementation Details

### Go Module

- Module path: `github.com/argoproj/argo-cd/v3`
- Uses Go 1.25.0
- Kubernetes client-go version: v0.34.0

### Environment Variables

Key environment variables for development:
- `ARGOCD_BINARY_NAME`: Determines which component to run
- `ARGOCD_FAKE_IN_CLUSTER`: Allows running outside cluster
- `ARGOCD_E2E_TEST`: Enables e2e test mode
- `ARGOCD_APPLICATION_NAMESPACES`: Multi-namespace support
- `BIN_MODE`: Run from compiled binary vs `go run`

### Docker

```bash
# Build image
make image

# Build dev image (faster, uses local binaries)
make image DEV_IMAGE=true

# Use podman instead of docker
make image DOCKER=podman
```

## Key Files

- **`Makefile`**: All build, test, and development commands
- **`Procfile`**: Service definitions for local development (used by goreman)
- **`go.mod`**: Go module dependencies
- **`VERSION`**: Current version number
- **`.golangci.yaml`**: Linter configuration
- **`.mockery.yaml`**: Mock generation configuration

## Notes

- The repository uses a consolidated binary approach - all components are built into a single binary
- Code generation is required after modifying protobuf, APIs, or CRDs
- The UI is built separately and embedded into the final image
- Most development can be done locally without Kubernetes cluster access (uses `ARGOCD_FAKE_IN_CLUSTER`)
- The test environment requires Docker and kubectl
