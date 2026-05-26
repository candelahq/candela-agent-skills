---
name: candela
description: |
  Development conventions for the candelahq/candela repository — the OTel-native
  LLM observability platform. Covers the Go backend, Rust sidecar, Next.js UI,
  storage layer, proxy architecture, proto codegen, and testing patterns.
  Use this skill when developing or contributing to the candela repository.
  Candela also has an editor plugin ecosystem: candela-vscode (VS Code / Open VSX),
  candela-jetbrains (IntelliJ/GoLand/PyCharm), opencode-candela (terminal),
  and candela-cline (Cline integration). These plugins share the same
  ConnectRPC/REST API surface documented here.
license: Apache-2.0
metadata:
  version: v2
  publisher: candelahq
---

# Candela Development Skill

## Repository Overview

Candela is a monorepo with four major components:

| Component | Path | Language | Purpose |
|-----------|------|----------|---------|
| Server | `cmd/candela-server/` | Go | Production backend (ConnectRPC + proxy) |
| CLI proxy | `cmd/candela/` | Go | Local dev proxy + runtime manager |
| Sidecar | `cmd/candela-sidecar/` | Go | Minimal container sidecar (Pub/Sub + OTLP) |
| Rust workspace | `rust/` | Rust | Incremental rewrite (proxy, processor, sidecar) |
| Web UI | `ui/` | TypeScript/Next.js 16 | Dashboard, traces, admin panel |
| SDKs | `sdks/` | Python, TS | Enrichment SDKs (simplified — Go/Kotlin/Rust SDKs removed) |

---

## Environment & Toolchain

### Nix-First Development

The repository uses a `flake.nix` that pins all toolchain versions. **Every terminal
command must be prefixed with `nix develop -c`**, including `git commit` (which triggers
lefthook pre-commit hooks that need tools from the nix shell).

```bash
nix develop -c go test ./...
nix develop -c go run ./cmd/candela-server
nix develop -c buf generate
nix develop -c git commit -m "feat: your message"
```

**Toolchain versions** (pinned in flake.nix):
- Go 1.26, Buf, Node.js 22, pnpm, Python 3.12, uv, Rust stable, Hurl

### Pre-Commit Hooks (lefthook)

Lefthook runs automatically on `git commit`. Hooks include:
- `golangci-lint` (5min timeout), `go vet`, `go mod tidy`, `gofmt`, `go test` (30s timeout)
- `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo test`
- Trailing whitespace, YAML/JSON validation, merge conflict detection, private key detection

**Never use `--no-verify`** — always fix the hook failure.

---

## Proto-First Architecture

All service boundaries are defined in the separate
[`candelahq/candela-protos`](https://github.com/candelahq/candela-protos) repo,
published to BSR as `buf.build/candelahq/protos`.

### Code Generation

```bash
nix develop -c buf generate   # from repo root
```

This generates:
- `gen/go/` — Go protobuf + ConnectRPC stubs
- `ui/src/gen/` — TypeScript ES protobuf + ConnectRPC stubs
- `gen/bq/` — BigQuery schema from proto (via `protoc-gen-bq-schema`)

**Never hand-edit files in `gen/`** — they are fully generated from BSR.

### Adding a New RPC

1. Define the proto in `candelahq/candela-protos` first
2. Run `buf generate` in the candela repo to regenerate stubs
3. Implement the handler in `pkg/connecthandlers/`
4. Wire the handler in `cmd/candela-server/main.go`
5. The UI gets TypeScript stubs automatically via the same `buf generate`

### ConnectRPC Transport

The backend serves ConnectRPC on port **8181**. The UI communicates via Connect protocol
(not gRPC-Web). The consolidated `GetDashboardData` RPC (with `include_budget=true`)
replaces the older `GetUsageSummary` + `GetMyBudget` fan-out — plugins use this single
call for usage, budget, and grant data. Service paths follow the pattern:
```
POST http://localhost:8181/candela.v1.TraceService/ListTraces
POST http://localhost:8181/candela.v1.DashboardService/GetDashboardData
```

---

## Storage Architecture (CQRS)

Candela uses Command Query Responsibility Segregation with well-defined interfaces
in `pkg/storage/store.go`:

| Interface | Role | Implementations |
|-----------|------|-----------------|
| `SpanWriter` | Write-only ingestion | DuckDB, SQLite, BigQuery, Pub/Sub, OTLP |
| `SpanReader` | Read-only queries | DuckDB, SQLite, BigQuery |
| `TraceStore` | Convenience (both) | DuckDB, SQLite, BigQuery |
| `CombinedUsageReader` | Optional: single-query usage+models | BigQuery |
| `UserStore` | User/budget/grant CRUD | Firestore |
| `ProjectStore` | Project + API key management | DuckDB, SQLite |
| `AnnotationStore` | Trace annotations | DuckDB, SQLite |
| `SyncStore` | Offline store-and-forward | DuckDB, SQLite |

### Adding a New Storage Backend

1. Create a new package under `pkg/storage/` (e.g., `pkg/storage/postgres/`)
2. Implement `SpanWriter` and/or `SpanReader` interfaces
3. Wire it into the backend factory in `cmd/candela-server/main.go`
4. Use `storage.ErrNotFound` for not-found errors
5. For integration tests, use `sqlitestore.New(":memory:")` as reference

### Domain Types

Core domain types live in `pkg/storage/store.go`:
- `Span` — a single span with `GenAIAttributes` for LLM-specific data
- `TraceSummary` / `Trace` — lightweight summary vs full trace with spans
- `UserRecord`, `BudgetRecord`, `GrantRecord` — multi-user management
- `BudgetCheckResult` — pre-flight budget enforcement result

### Fan-Out Processor

The `SpanProcessor` (in `pkg/processor/`) writes to **all configured writers** concurrently
with per-sink isolation. If one sink fails, others continue. Configuration:

```yaml
worker:
  batch_size: 100
  flush_interval: "2s"
```

---

## Proxy Architecture

The proxy (`pkg/proxy/`) is the core LLM API reverse proxy supporting:
- **OpenAI**, **Google/Gemini**, **Anthropic (Vertex AI)**, **Anthropic Direct**,
  **Anthropic Bedrock** (SigV4)
- SSE streaming capture without affecting user latency
- W3C Trace Context propagation (`Traceparent` / `Tracestate`)
- Automatic token extraction and cost calculation

### Proxy Routes

Routes are registered at `/proxy/{provider}/`:
```
/proxy/openai/v1         → OpenAI API
/proxy/google/           → Vertex AI (Gemini)
/proxy/gemini-oai/v1     → Gemini OpenAI-compat
/proxy/anthropic/        → Anthropic via Vertex AI
/proxy/anthropic-direct  → Anthropic direct (client API key)
/proxy/anthropic-bedrock → Claude via AWS Bedrock
```

### Adding a New Provider

1. Add the route registration in the proxy setup (follow existing provider patterns)
2. Implement any provider-specific request/response translation
3. Add cost calculation entries in `pkg/costcalc/`
4. Add functional tests in `test/functional/`

> **Note**: OpenAI models were purged from `pkg/costcalc/` — only Google and
> Anthropic models are priced. Users routing through the OpenAI proxy endpoint
> will see `$0.00` cost unless they add entries back.

---

## CLI Proxy (`cmd/candela/`)

The `candela` CLI operates in three modes:
- **Solo Mode** — local models only, SQLite traces, no cloud
- **Solo + Cloud** — local + Vertex AI direct (via ADC)
- **Team Mode** — connects to shared Candela server

### LM-Compatible Listener

Port `:1234` serves a unified `/v1/models` + `/v1/chat/completions` endpoint that
merges local, cloud, and remote models. Smart routing sends requests to the right backend.

### Embedded UI

The `/_local/` path serves an embedded vanilla JS management UI for runtime control,
model management, and local traces. ConnectRPC services: `RuntimeService`.

---

## Web UI (`ui/`)

Next.js 16 app with App Router, dark theme, ConnectRPC v2 transport.

### Key Conventions

- **Pages**: `ui/src/app/{route}/page.tsx` — App Router convention
- **Generated stubs**: `ui/src/gen/` — never edit, generated by `buf generate`
- **Hooks**: `ui/src/hooks/` — custom React hooks (`useCurrentUser`, `useProtoValidation`)
- **Transport**: `ui/src/lib/` — ConnectRPC transport config
- **E2E tests**: `ui/e2e/` — Playwright (route interception to mock backend)

### Running the UI

```bash
cd ui && pnpm install && pnpm run dev   # → http://localhost:3000
pnpm run build                           # TypeScript type-check + production build
pnpm run test:e2e                        # Playwright (27+ tests)
```

### E2E Test Pattern

Tests mock the backend via `page.route()`:
```typescript
await page.route("**/candela.v1.DashboardService/GetModelBreakdown", (route) => {
    route.fulfill({ body: JSON.stringify({ models: [...] }) });
});
```

Use semantic selectors (`getByText`, `getByRole`) over CSS selectors.

---

## Rust Workspace (`rust/`)

Incremental rewrite of performance-critical components:

| Crate | Purpose |
|-------|---------|
| `candela-core` | Domain types, `SpanWriter` trait |
| `candela-proxy` | LLM reverse proxy engine |
| `candela-processor` | Batched span processor + cost calc |
| `candela-storage` | Pub/Sub + OTLP sinks |
| `bins/candela-sidecar` | Production sidecar binary |

Rust components are tested via:
```bash
cd rust && cargo test --workspace
```

And linted via lefthook: `cargo fmt --check` + `cargo clippy -- -D warnings`.

---

## Testing Patterns

### Go Tests

- **Unit tests**: Same package, `_test.go` suffix, table-driven with `t.Run()`
- **Integration tests**: `pkg/connecthandlers/integration_test.go` — full service stack
  with in-memory SQLite
- **Race detector**: Always run with `-race -count=1` for correctness
- **Coverage**: `go test ./... -coverprofile=coverage.out`

```bash
nix develop -c go test ./... -v -race -count=1   # full suite
nix develop -c go test ./pkg/proxy -v              # specific package
```

### Functional Tests (Go/Rust Parity)

HTTP-level tests in `test/functional/` using Hurl ensure Go and Rust implementations
have identical behavior:

```bash
./test/functional/run.sh --go    # test Go binary
./test/functional/run.sh --rust  # test Rust binary
```

### Adding Tests

- For storage: use `sqlitestore.New(":memory:")` for ephemeral databases
- For proxy: create a test HTTP server, proxy instance, and submitter mock
- For E2E: add to the appropriate Playwright suite, mock backend via `page.route()`

---

## Configuration

Server config via `config.yaml` (or `$CANDELA_CONFIG`):

```yaml
server:
  host: "0.0.0.0"
  port: 8181
storage:
  backend: "duckdb"      # duckdb | sqlite | bigquery
sinks:
  pubsub:
    enabled: false
  otlp:
    enabled: true
    endpoint: "http://localhost:4318/v1/traces"
proxy:
  enabled: true
worker:
  batch_size: 100
  flush_interval: "2s"
```

CLI config at `~/.config/candela/config.yaml`:

```yaml
runtime_backend: ollama
remote: https://candela-xxx.a.run.app  # Team Mode
providers:
  - name: google
    models: [gemini-2.5-pro]
vertex_ai:
  project: my-gcp-project
  prompt_caching: true
  cache_ttl: 5m
```

---

## Release & CI

### CI Pipeline (GitHub Actions)

Three jobs: `build-and-test` (Go), `ui-build-and-lint` (Next.js), `ui-e2e` (Playwright).
Proto generation requires `BUF_TOKEN` secret.

### GoReleaser

Tagged releases (e.g., `v0.5.4`) trigger GoReleaser for multi-platform binaries.
Sidecar has its own tag prefix: `sidecar-v*`.

### Homebrew

Formula in `candelahq/homebrew-tap`:
```bash
brew install candelahq/tap/candela
brew install --cask candelahq/tap/candela-desktop
```

---

## Common Workflows

### Adding a New Dashboard Metric End-to-End

1. **Proto**: Add field to the response message in `candela-protos`
2. **Server**: Implement in `pkg/connecthandlers/` (read from storage)
3. **Storage**: Add query method to relevant `SpanReader` implementations
4. **UI**: Use generated TypeScript stub, add component to dashboard page
5. **Desktop**: Generated Dart stubs pick it up via `buf generate`

### Adding a New Model to Cost Calculator

1. Add pricing entry in `pkg/costcalc/` with input/output per-token rates
2. Add test cases to `pkg/costcalc/*_test.go`
3. The proxy auto-detects the model from API responses

### Firestore Emulator for Local Dev

```bash
nix develop -c gcloud emulators firestore start --host-port=localhost:8282
export FIRESTORE_EMULATOR_HOST=localhost:8282
nix develop -c go run ./cmd/candela-server
```
