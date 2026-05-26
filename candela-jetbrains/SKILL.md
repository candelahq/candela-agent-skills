---
name: candela-jetbrains
description: |
  Development conventions for the candelahq/candela-jetbrains repository — the
  JetBrains IDE plugin (IntelliJ, GoLand, PyCharm, WebStorm, etc.) for Candela
  LLM cost tracking and observability. Covers the Kotlin codebase, IntelliJ
  Platform Plugin SDK, Gradle build, settings, status bar widget, and CI/CD.
  Use this skill when developing or contributing to the candela-jetbrains plugin.
license: Apache-2.0
metadata:
  version: v2
  publisher: candelahq
---

# Candela JetBrains Plugin Development Skill

## Repository Overview

`candela-jetbrains` is a JetBrains IDE plugin providing real-time LLM cost tracking,
budget warnings, and observability for the Candela platform. It works with any
JetBrains IDE (IntelliJ IDEA, WebStorm, PyCharm, GoLand, etc.).

- **Language**: Kotlin
- **SDK**: IntelliJ Platform Plugin SDK 2.6 (`org.jetbrains.intellij.platform:2.6.0`)
- **Build**: Gradle with Kotlin DSL
- **JDK**: 21 (required — `jvmToolchain(21)`)
- **Min IDE**: 2024.3 (`sinceBuild = "243"`, no `untilBuild` cap)

---

## Environment & Toolchain

### Nix-First Development

The repository has a `flake.nix` that pins JDK 21 (Temurin), Gradle, Kotlin, and
Lefthook. **All terminal commands must be prefixed with `nix develop -c`**, including
`git commit` (which triggers lefthook hooks that need tools from the nix shell).

```bash
nix develop -c ./gradlew buildPlugin
nix develop -c ./gradlew test
nix develop -c git commit -m "feat: your message"
```

### Pre-Commit Hooks (lefthook)

Lefthook runs on commit and push:

**Pre-commit** (parallel):
- Trailing whitespace check
- Private key detection
- Merge conflict marker check
- JSON validation
- End-of-file newline check
- Kotlin format check (`ktlintCheck`)

**Pre-push**:
- Full Gradle build: `nix develop -c ./gradlew buildPlugin`
- Tag-version check: validates `v*` tag matches `build.gradle.kts` version

**Never use `--no-verify`** — always fix the hook failure.

---

## Project Structure

```
candela-jetbrains/
├── build.gradle.kts                          # Plugin config, deps, publishing
├── settings.gradle.kts                       # Root project name
├── flake.nix                                 # Nix dev shell (JDK 21, Gradle, Kotlin)
├── lefthook.yaml                             # Pre-commit + pre-push hooks
├── src/main/
│   ├── kotlin/com/candelahq/candela/
│   │   ├── client/
│   │   │   ├── CandelaClient.kt              # HTTP client for Candela server
│   │   │   └── Models.kt                     # Data classes (DashboardData, etc.)
│   │   ├── actions/
│   │   │   └── Actions.kt                    # Menu actions (cost summary, budget, etc.)
│   │   ├── settings/
│   │   │   ├── CandleSettings.kt             # Persistent settings (PersistentStateComponent)
│   │   │   └── CandleSettingsConfigurable.kt # Settings UI panel
│   │   ├── CandleStatusBarWidget.kt          # Status bar widget + factory
│   │   ├── CandleNotifications.kt            # Budget warning notifications
│   │   └── CandleStartupActivity.kt          # Post-startup health check
│   └── resources/META-INF/
│       └── plugin.xml                        # Plugin descriptor
└── .github/workflows/
    ├── ci.yml                                # Build + test on push/PR
    └── release.yml                           # Tag-triggered publish
```

---

## Key Source Files

### CandelaClient.kt (`client/`)

HTTP client that communicates with the Candela server via ConnectRPC-compatible
JSON endpoints. Uses `java.net.http.HttpClient` with Gson for serialization.

- Calls `GetDashboardData` (consolidated RPC) with `include_budget=true`
- Falls back to legacy `GetUsageSummary` + `GetMyBudget` for older servers
- Health check via `/healthz`
- All methods are null-safe for offline scenarios

### CandleStatusBarWidget.kt

The main user-facing component — a status bar widget showing live token count,
cost, and budget percentage. Features:
- Auto-refresh on configurable interval (`refreshIntervalSeconds`)
- Budget urgency colors (green/yellow/red based on threshold)
- Click to show cost summary popup
- Disposal-safe with coroutine scope management

### CandleNotifications.kt

Budget warning notifications using IntelliJ's notification system:
- Triggers when budget exceeds `budgetWarningThreshold`
- Grant expiry warnings
- Uses `NotificationGroup` registered in `plugin.xml`

### Actions.kt (`actions/`)

Four menu actions registered under Tools → Candela:
- `ShowCostSummaryAction` — dialog with today's usage breakdown
- `CheckBudgetAction` — budget status with grants
- `OpenDashboardAction` — opens web dashboard in browser
- `RefreshStatusAction` — force-refresh status bar

### CandleSettings.kt (`settings/`)

Persistent settings via `PersistentStateComponent`, stored in `candela.xml`:

| Setting | Default | Description |
|---------|---------|-------------|
| `serverUrl` | `http://localhost:8181` | Candela server URL |
| `statusBarEnabled` | `true` | Show/hide status bar widget |
| `autoRefreshIntervalSeconds` | `60` | Polling interval (seconds) |
| `budgetWarningThreshold` | `80` | Budget warning at N% usage |

Access settings via:
```kotlin
val settings = CandleSettings.getInstance()
val url = settings.state.serverUrl
```

---

## plugin.xml Structure

The plugin descriptor at `src/main/resources/META-INF/plugin.xml` registers:

1. **Extensions** (`com.intellij` namespace):
   - `statusBarWidgetFactory` → `CandleStatusBarWidgetFactory`
   - `applicationConfigurable` → `CandleSettingsConfigurable` (under Tools)
   - `applicationService` → `CandleSettings`
   - `postStartupActivity` → `CandleStartupActivity`
   - `notificationGroup` → `"Candela"` (BALLOON display type)

2. **Actions** (under `Candela.Menu` group, added to `ToolsMenu`):
   - `Candela.ShowCostSummary`
   - `Candela.CheckBudget`
   - `Candela.OpenDashboard`
   - `Candela.RefreshStatus`

**Depends on**: `com.intellij.modules.platform` (works with all JetBrains IDEs)

---

## Build & Dependencies

### build.gradle.kts

Key configuration:

```kotlin
plugins {
    id("java")
    id("org.jetbrains.kotlin.jvm") version "2.1.20"
    id("org.jetbrains.intellij.platform") version "2.6.0"
}

group = "com.candelahq"
// version is updated per release

dependencies {
    intellijPlatform {
        intellijIdeaCommunity("2024.3")
        pluginVerifier()
    }
    implementation("com.google.code.gson:gson:2.13.1")
}

kotlin {
    jvmToolchain(21)
}
```

### Building

```bash
nix develop -c ./gradlew buildPlugin     # Build plugin ZIP
nix develop -c ./gradlew test             # Run tests (JUnit 5)
nix develop -c ./gradlew runIde           # Launch sandboxed IDE for testing
nix develop -c ./gradlew verifyPlugin     # Run JetBrains Plugin Verifier
```

---

## Testing

```bash
nix develop -c ./gradlew test
```

- Tests use JUnit 5 (`useJUnitPlatform()`)
- `kotlinx-coroutines-test` for async testing
- Test fixtures create mock `CandelaClient` responses

---

## CI/CD

### CI (`ci.yml`)

Runs on push/PR to `main`:
1. Checkout
2. Setup JDK 21 (Temurin)
3. Setup Gradle
4. `./gradlew buildPlugin`
5. `./gradlew test`

### Release (`release.yml`)

Triggered by `v*` tags:
1. Build + test
2. Check if `JETBRAINS_TOKEN` secret is set
3. If set: `./gradlew publishPlugin` (with `continue-on-error: true`)
4. Create GitHub Release with plugin ZIP from `build/distributions/`

The `publishPlugin` task uses `continue-on-error: true` so that GitHub releases
are created even if the Marketplace publish fails (e.g., pending approval).

### Publishing Token

The `JETBRAINS_TOKEN` environment variable is read by the `publishing` block
in `build.gradle.kts` via `providers.environmentVariable("JETBRAINS_TOKEN")`.

---

## Common Workflows

### Adding a New Menu Action

1. Create a new `AnAction` subclass in `actions/Actions.kt`
2. Register it in `plugin.xml` under the `Candela.Menu` group:
   ```xml
   <action id="Candela.NewAction"
           class="com.candelahq.candela.actions.NewAction"
           text="New Action"
           description="Description"/>
   ```
3. Access `CandleSettings` and `CandelaClient` from the action

### Adding a New Setting

1. Add the field to `CandleSettings.State` data class
2. Add the UI control in `CandleSettingsConfigurable`
3. Read it wherever needed via `CandleSettings.getInstance().state`

### Updating the Status Bar

The status bar widget polls `CandelaClient.getDashboardData()` on the configured
interval. To add new data to the display:
1. Ensure the data is returned by `GetDashboardData` RPC
2. Parse it in `CandelaClient.kt` / `Models.kt`
3. Update `CandleStatusBarWidget.kt` to render it

### Bumping Version for Release

1. Update `version` in `build.gradle.kts`
2. Commit
3. Tag: `git tag v0.X.Y && git push origin v0.X.Y`
4. The pre-push hook validates the tag matches `build.gradle.kts`
5. CI builds and publishes automatically
