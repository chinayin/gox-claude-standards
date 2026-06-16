---
inclusion: fileMatch
fileMatchPattern: "{Makefile,.gitignore,.editorconfig,.golangci-lint-version,.github/workflows/*.yml}"
---

# Project Scaffold Standards

Every new Go project must include these root-level files. Use these templates directly.

## .editorconfig

```editorconfig
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
indent_style = space
indent_size = 4

[*.go]
indent_style = tab

[*.{yaml,yml,toml}]
indent_size = 2

[*.md]
trim_trailing_whitespace = false

[Makefile]
indent_style = tab
```

## .gitignore

```gitignore
# Build output (includes the golangci-lint binary downloaded by `make lint`)
bin/

# Local config overrides
config/*.local.yaml
.env

# macOS
.DS_Store

# IDE
.idea/
.vscode/
*.swp
*.swo

# Kiro
.kiro/
```

## .golangci-lint-version

Pins the golangci-lint version used by this project, serving as the **single version source** (shared by local and CI). Contents are a single version line:

```
v2.12.2
```

- Local `make lint` downloads the pinned version to `./bin` accordingly, with no dependency on brew or a global install
- CI reads it via the `version-file` input of `golangci-lint-action`
- Upgrading the linter means editing only this one file; local and CI follow in sync
- Note: official golangci-lint prebuilt binaries are compiled against the Go version current at release time. A binary compiled with a Go version **lower** than the `go` directive in go.mod will refuse to run, so pin a release built with Go >= the go.mod version

## Makefile

Required targets (variable declarations are project-specific, not part of this standard):

```makefile
.PHONY: help build run clean test lint lint-fix fmt check install-tools ensure-lint

# Pinned golangci-lint version (travels with the project; upgrade by editing .golangci-lint-version)
GOLANGCI_VERSION := $(shell cat .golangci-lint-version)
GOLANGCI := ./bin/golangci-lint

help: ## Show help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "  make %-13s - %s\n", $$1, $$2}'

build: clean ## Compile
	@go build -ldflags "$(LDFLAGS)" -o bin/$(APP) ./cmd/$(APP)

run: build ## Run
	@./bin/$(APP) --help

clean: ## Clean build output
	@rm -rf bin/

ensure-lint:
	@if [ ! -x "$(GOLANGCI)" ] || ! "$(GOLANGCI)" version 2>/dev/null | grep -q "$(GOLANGCI_VERSION:v%=%)"; then \
		curl -sSfL https://golangci-lint.run/install.sh | sh -s -- -b ./bin "$(GOLANGCI_VERSION)"; \
	fi

install-tools: ensure-lint ## Install pinned golangci-lint to ./bin
	@echo "golangci-lint $(GOLANGCI_VERSION) ready at $(GOLANGCI)"

test: ## Run tests
	@go test -v -race -count=1 -timeout 120s ./...

lint: ensure-lint ## Run linter (pinned version)
	@$(GOLANGCI) run ./...

lint-fix: ensure-lint ## Run linter with auto-fix
	@$(GOLANGCI) run --fix ./...

fmt: lint-fix ## Format code (lint + fix)
	@echo "Code formatted!"

check: fmt lint test ## Full local check (run before commit)
	@echo "All checks passed!"
```

### Makefile Rules

- Default target is `help`, auto-generated from `## comments`
- Every target must have a `## comment` describing its purpose (internal targets like `ensure-lint` omit it to stay out of help)
- Prefix commands with `@` to suppress echo
- `build` depends on `clean` for clean output
- Tests must include `-race` and `-timeout`
- `check` is pre-commit full validation (format + lint + test)
- **Lint uses the project-pinned version**: `lint`/`lint-fix` depend on `ensure-lint`, which reads the version from `.golangci-lint-version` and downloads it to `./bin`. **Never invoke a global `golangci-lint` directly** (avoids version drift across team members)
- Unified linter: golangci-lint v2
- Optional additions: `ci-check` (CI pipeline, no fmt), `docker-up/down/logs`

## CI Workflows

CI shares version sources with local development — **the Go version is read from `go.mod`, the golangci-lint version from `.golangci-lint-version`, and no version is hardcoded inside the workflow**.

`.github/workflows/lint.yml`:

```yaml
name: Lint

on:
  push:
    branches:
      - main
    tags:
      - 'v*'
  pull_request:

permissions:
  checks: write

jobs:
  golangci:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v6

      - name: Set up Go
        uses: actions/setup-go@v6
        with:
          go-version-file: go.mod

      - name: Run golangci-lint
        uses: golangci/golangci-lint-action@v9
        with:
          version-file: .golangci-lint-version
```

`.github/workflows/test.yml`:

```yaml
name: Test

on:
  push:
    branches:
      - main
    tags:
      - 'v*'
  pull_request:

jobs:
  test:
    name: Test
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ ubuntu-latest ]
    steps:
      - name: Checkout code
        uses: actions/checkout@v6

      - name: Set up Go
        uses: actions/setup-go@v6
        with:
          go-version-file: go.mod

      - name: Run tests
        run: make test
```

`.github/workflows/build.yml`:

```yaml
name: Build

on:
  push:
    branches:
      - main
    tags:
      - 'v*'
  pull_request:

jobs:
  build:
    name: Build (${{ matrix.os }})
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ ubuntu-latest ]
    steps:
      - name: Checkout code
        uses: actions/checkout@v6

      - name: Set up Go
        uses: actions/setup-go@v6
        with:
          go-version-file: go.mod

      - name: Build
        run: go build -v ./...

      - name: Verify modules
        run: go mod verify
```

### CI Rules

- Listen on the repository's default branch (standardized as `main`); the workflow `branches` must match the default branch, otherwise pushes won't trigger
- Single source for the Go version (`go.mod`) and for the golangci-lint version (`.golangci-lint-version`)
- Pin actions to a major version (`@v6` / `@v9`); do not use floating refs
