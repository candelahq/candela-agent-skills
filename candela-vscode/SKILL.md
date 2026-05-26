---
name: candela-vscode
description: |
  Development conventions for the candelahq/candela-vscode repository — the
  VS Code extension for Candela LLM cost tracking and observability. Covers
  the TypeScript codebase, status bar integration, budget warnings, auto-discovery,
  and Open VSX publishing workflow.
  Use this skill when developing or contributing to the candela-vscode extension.
license: Apache-2.0
metadata:
  version: v2
  publisher: candelahq
---

# Candela VS Code Extension Development Skill

## Repository Overview

`candela-vscode` is a VS Code extension providing real-time LLM cost tracking,
budget warnings, and observability for the Candela platform.

- **Language**: TypeScript
- **Engine**: VS Code ≥ 1.100.0
- **Published on**: [Open VSX Registry](https://open-vsx.org/) (NOT the VS Code Marketplace)
- **Categories**: AI, Other

---

## Environment & Toolchain

### Required Tools

- **Node.js** ≥ 22
- **npm** (lockfile committed)

### Building

```bash
npm ci                  # Install dependencies
npm run compile         # TypeScript compile (tsc -p ./)
npm run watch           # Watch mode for development
```

### Packaging

```bash
npx @vscode/vsce package   # Creates .vsix file
```

---

## Project Structure

```
candela-vscode/
├── package.json                # Extension manifest (contributes, config, scripts)
├── tsconfig.json               # TypeScript configuration
├── src/
│   ├── extension.ts            # Extension entry point (activate/deactivate)
│   ├── candela-client.ts       # HTTP client for Candela server API
│   └── discover.ts             # Auto-discovery of Candela server URL
├── dist/                       # Compiled output (extension.js)
├── .github/workflows/
│   ├── ci.yml                  # Build + type-check on push/PR
│   └── release.yml             # Tag-triggered Open VSX publish
└── .vscodeignore               # Files excluded from VSIX package
```

---

## Key Source Files

### extension.ts

The extension entry point. On activation (`onStartupFinished`):

1. Discovers Candela server URL (config → env → config files → default)
2. Creates a `CandelaClient` with 30s cache TTL
3. Creates a status bar item (right-aligned, priority 50)
4. Registers four commands
5. Sets up auto-refresh polling with failure backoff

**Commands registered:**

| Command | Title | Description |
|---------|-------|-------------|
| `candela.showDashboard` | Show Dashboard | Opens web dashboard in browser |
| `candela.showCostSummary` | Show Cost Summary | Information message with usage breakdown |
| `candela.checkBudget` | Check Budget | Budget status with progress bar and grants |
| `candela.refreshStatus` | Refresh Status | Force refresh + cache invalidation + health reset |

**Failure backoff**: After 3 consecutive failures, polling interval increases to
5 minutes (`BACKOFF_INTERVAL_S = 300`). Resets to normal on success.

### candela-client.ts

The HTTP client — the core API layer shared with the Cline plugin.

- Primary: `getDashboardData(hours)` — calls the consolidated
  `GetDashboardData` RPC with `include_budget=true`
- Fallback: `legacyFanout()` — parallel `GetUsageSummary` + `GetMyBudget`
  for servers that haven't upgraded
- Health check: `isAlive()` via `/healthz` (2s timeout)
- Built-in response cache with configurable TTL
- Handles both camelCase and snake_case proto JSON field names

**Key types exported:**
- `DashboardData` — consolidated usage + budget + grants
- `UsageSummary` — token counts and cost
- `BudgetInfo` — spend vs limit with reset countdown
- `GrantInfo` — bonus budget grants with expiry
- `ModelUsage` — per-model breakdown with cache token stats

### discover.ts

Auto-discovers the Candela server URL. Resolution order:

1. `CANDELA_PROXY_URL` env var
2. `CANDELA_CONFIG` env var → parse port from YAML
3. `./config.yaml` (project-local)
4. `~/.config/candela/config.yaml` (global)
5. Default: `http://localhost:8181`

Uses regex-based YAML parsing to avoid a dependency.

---

## Extension Settings (package.json)

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `candela.serverUrl` | string | `http://localhost:8181` | Candela server URL |
| `candela.statusBar.enabled` | boolean | `true` | Show status bar item |
| `candela.statusBar.showBudget` | boolean | `true` | Show budget % in status bar |
| `candela.budgetWarning.threshold` | number | `80` | Warning at N% budget usage |
| `candela.autoRefresh.intervalSeconds` | number | `60` | Polling interval (0 = disabled) |

Settings changes are watched via `onDidChangeConfiguration` — the client is
recreated and polling is rescheduled automatically.

---

## Status Bar Widget

The status bar shows: `$(flame) 12.5K · $1.23 · 🟡65%`

- **Tokens**: Formatted with K/M suffixes
- **Cost**: USD with adaptive precision ($0.0012, $0.123, $1.23)
- **Budget**: Emoji indicator (🟢 <60%, 🟡 60-90%, 🔴 >90%) + percentage
- **Warning background**: Applied when budget exceeds threshold
- **Click action**: Opens cost summary dialog
- **Offline state**: `$(circle-slash) Candela: offline`

---

## CI/CD

### CI (`ci.yml`)

Runs on push/PR to `main`, matrix: Node.js 22 + 24:
1. `npm ci`
2. `npx tsc --noEmit` (type check)
3. `npx tsc -p ./` (compile)

### Release (`release.yml`)

Triggered by `v*` tags:
1. `npm ci` + `npm run compile`
2. Validate `OVSX_TOKEN` via `npx ovsx verify-pat`
3. Package: `npx @vscode/vsce package`
4. Publish: `npx ovsx publish *.vsix -p "$OVSX_TOKEN"`
5. Create GitHub Release with VSIX artifact

If the `OVSX_TOKEN` is invalid/expired, the workflow:
- Creates a commit comment with fix instructions
- Fails with a clear error message
- Does NOT create a GitHub Release

### Publishing Token

The `OVSX_TOKEN` secret must be a valid Open VSX personal access token.
Generate at https://open-vsx.org/user-settings/tokens.

---

## Common Workflows

### Adding a New Command

1. Add the command definition to `package.json` → `contributes.commands`
2. Implement the handler function in `extension.ts`
3. Register with `vscode.commands.registerCommand()` in `activate()`
4. Add to `context.subscriptions` for cleanup

### Adding a New Setting

1. Add to `package.json` → `contributes.configuration.properties`
2. Read via `vscode.workspace.getConfiguration("candela").get<T>("key")`
3. If it affects polling/client, handle in the `onDidChangeConfiguration` watcher

### Updating the Status Bar Display

1. Modify `updateStatusBar()` in `extension.ts`
2. Data comes from `client.getDashboardData(24)`
3. Format with `formatCost()` / `formatTokens()` helpers
4. Set `statusBarItem.text`, `.tooltip`, `.backgroundColor`

### Testing Locally

1. Open the repo in VS Code
2. Press F5 → launches Extension Development Host
3. The extension activates on startup (check Output → Candela)
4. Ensure a Candela server is running on localhost:8181

### Publishing a New Version

1. Update `version` in `package.json`
2. Commit and push
3. Tag: `git tag v0.X.Y && git push origin v0.X.Y`
4. CI builds and publishes to Open VSX automatically

---

## Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `@types/vscode` | ^1.100.0 | VS Code API typings |
| `@types/node` | ^22.0.0 | Node.js typings |
| `typescript` | ^5.8.0 | TypeScript compiler |

No runtime dependencies — the extension uses built-in `fetch` (Node.js 22+)
and the VS Code API exclusively.
