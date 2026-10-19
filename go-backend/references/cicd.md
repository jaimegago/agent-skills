# CI/CD & Build Pipeline Reference

Extended guidance for Go project build pipelines, versioning, and release automation. Read this when setting up Makefiles, Docker builds, or CI workflows.

## Versioning

### Version Package Convention

Every Go project with binaries must have `internal/version/version.go`:

```go
package version

import "fmt"

var (
    Version    = "dev"
    CommitHash = "unknown"
    BuildTime  = "unknown"
)

func String(binaryName string) string {
    return fmt.Sprintf("%s %s (commit: %s, built: %s)", binaryName, Version, CommitHash, BuildTime)
}
```

### ldflags Injection

The Go linker's `-X` flag sets string variables at link time. Rules:

- Target must be a package-level `var` of type `string`
- Use full module path: `github.com/org/repo/internal/version.Version`
- Works with exported and unexported variables
- Cannot target `const` or function results

```bash
MODULE=$(go list -m)
VERSION=$(git describe --tags --always --dirty)
COMMIT=$(git rev-parse --short HEAD)
BUILD_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ)

go build -ldflags="-X '${MODULE}/internal/version.Version=${VERSION}' \
  -X '${MODULE}/internal/version.CommitHash=${COMMIT}' \
  -X '${MODULE}/internal/version.BuildTime=${BUILD_TIME}'" \
  -o bin/myservice ./cmd/myservice
```

### CLI Integration

For projects using plain `flag` package:

```go
func main() {
    showVersion := flag.Bool("version", false, "print version and exit")
    flag.BoolVar(showVersion, "v", false, "print version and exit (shorthand)")
    flag.Parse()

    if *showVersion {
        fmt.Println(version.String("myservice"))
        os.Exit(0)
    }
    // ...
}
```

For Cobra-based CLIs, use the built-in `Version` field on the root command.

## Makefile Conventions

Every Go project should have a Makefile. Minimum targets:

```makefile
MODULE   := $(shell go list -m)
VERSION  := $(shell git describe --tags --always --dirty 2>/dev/null || echo "dev")
COMMIT   := $(shell git rev-parse --short HEAD 2>/dev/null || echo "unknown")
BUILD_TIME := $(shell date -u +%Y-%m-%dT%H:%M:%SZ)

LDFLAGS := -X '$(MODULE)/internal/version.Version=$(VERSION)' \
           -X '$(MODULE)/internal/version.CommitHash=$(COMMIT)' \
           -X '$(MODULE)/internal/version.BuildTime=$(BUILD_TIME)'

# Build all binaries under cmd/
.PHONY: build
build:
	@for dir in cmd/*/; do \
		binary=$$(basename $$dir); \
		echo "Building $$binary..."; \
		go build -ldflags="$(LDFLAGS)" -o bin/$$binary ./$$dir; \
	done

# Build a single binary: make build-myservice
.PHONY: build-%
build-%:
	go build -ldflags="$(LDFLAGS)" -o bin/$* ./cmd/$*

.PHONY: version
version:
	@echo $(VERSION)

.PHONY: test
test:
	go test ./...

.PHONY: test-integration
test-integration:
	go test -tags=integration ./...

.PHONY: lint
lint:
	golangci-lint run

.PHONY: clean
clean:
	rm -rf bin/
```

### Key Principles

- `go list -m` auto-detects the module path — never hardcode it
- `git describe --tags --always --dirty` works with and without tags
- The `2>/dev/null || echo "dev"` fallback handles fresh repos with no commits
- Build all binaries by iterating `cmd/*/`
- Output binaries to `bin/` (add to `.gitignore`)

## Docker Builds

### Multi-Stage Pattern

```dockerfile
FROM golang:1.23-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .

ARG VERSION=dev
ARG COMMIT=unknown
ARG BUILD_TIME=unknown

RUN CGO_ENABLED=0 go build \
    -ldflags="-X '$(go list -m)/internal/version.Version=${VERSION}' \
              -X '$(go list -m)/internal/version.CommitHash=${COMMIT}' \
              -X '$(go list -m)/internal/version.BuildTime=${BUILD_TIME}'" \
    -o /app/bin/myservice ./cmd/myservice

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /app/bin/myservice /usr/local/bin/myservice
ENTRYPOINT ["myservice"]
```

Build with version injection:

```bash
docker build \
  --build-arg VERSION=$(git describe --tags --always --dirty) \
  --build-arg COMMIT=$(git rev-parse --short HEAD) \
  --build-arg BUILD_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  -t myservice:latest .
```

### Docker Rules

- Always use multi-stage builds — don't ship the Go toolchain
- Use `CGO_ENABLED=0` for static binaries (unless you need CGO)
- Use distroless or scratch as the runtime base
- Run as non-root (`nonroot` in distroless, or `USER` directive)
- Pass version via `ARG`, not by embedding git into the container

## GitHub Actions

### CI Workflow

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Required for git describe

      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod

      - name: Lint
        uses: golangci/golangci-lint-action@v6

      - name: Test
        run: go test ./...

      - name: Build
        run: make build
```

### Release Workflow

Tag-triggered releases with GoReleaser or manual ldflags:

```yaml
name: Release
on:
  push:
    tags: ['v*']

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod

      - name: Build release binaries
        run: make build

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          files: bin/*
          generate_release_notes: true
```

### Automatic Version Bumps on Main

To auto-increment version on each push to main, use lightweight tags:

```yaml
# In CI after successful build on main
- name: Tag build
  if: github.ref == 'refs/heads/main'
  run: |
    git tag -f build-latest
    git push -f origin build-latest
```

For semver, use explicit tagging (`git tag v1.2.3`) — don't auto-bump semver.

## Release Tagging Strategy

- Use semver tags: `v1.0.0`, `v1.1.0-rc.1`
- `git describe --tags --always --dirty` produces:
  - On tag: `v1.0.0`
  - After tag: `v1.0.0-3-g1a2b3c4` (3 commits past v1.0.0, at commit 1a2b3c4)
  - With uncommitted changes: `v1.0.0-3-g1a2b3c4-dirty`
  - No tags ever: `1a2b3c4` (just the commit hash)
- This format is always valid and always informative — no special handling needed

## GoReleaser (Optional)

For projects that distribute binaries publicly, GoReleaser handles cross-compilation, checksums, and GitHub releases:

```yaml
# .goreleaser.yaml
version: 2
builds:
  - main: ./cmd/myservice
    binary: myservice
    ldflags:
      - -X '{{ .ModulePath }}/internal/version.Version={{ .Version }}'
      - -X '{{ .ModulePath }}/internal/version.CommitHash={{ .ShortCommit }}'
      - -X '{{ .ModulePath }}/internal/version.BuildTime={{ .Date }}'
    goos: [linux, darwin]
    goarch: [amd64, arm64]
```

For internal tools, plain Makefile + CI is sufficient — don't over-engineer.
