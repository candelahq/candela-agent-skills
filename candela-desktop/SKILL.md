---
name: candela-desktop
description: |
  Development conventions for the candelahq/candela-desktop repository (v0.5.1) — the
  cross-platform Flutter/Dart desktop client for Candela. Covers Riverpod state
  management, ConnectRPC integration, CandelaAuthService, CLI process
  management, and build workflows.
  Use this skill when developing or contributing to the candela-desktop repository.
license: Apache-2.0
metadata:
  version: v2
  publisher: candelahq
---

# Candela Desktop Development Skill

## Repository Overview

`candela-desktop` is a Flutter desktop application (macOS/Windows/Linux) providing a
GUI for the Candela LLM observability platform. It connects to either a local
`candela` CLI proxy or a remote Candela server.

### Architecture

```
┌─────────────┐     ConnectRPC      ┌──────────────────┐
│  Flutter UI  │ ←───(HTTP/2)────→  │  Candela Server   │
│  (Riverpod)  │                    │  (Go, port 8181)  │
└──────┬──────┘                    └──────────────────┘
       │
       │ Process.start()
       ▼
┌──────────────┐
│  candela CLI  │  ← local proxy mode
│  (installed)  │     (port 1234 + 8181)
└──────────────┘
```

---

## Environment & Toolchain

### Required Tools

- **Flutter SDK** ≥ 3.32 (Dart ≥ 3.8)
- **Lefthook** for pre-commit hooks

### Pre-Commit Hooks (lefthook)

Lefthook runs `dart format --set-exit-if-changed` and `dart analyze` on commit.
**Never use `--no-verify`**.

### Running Locally

```bash
flutter pub get
flutter run -d macos     # or: -d windows, -d linux
```

### Building

```bash
flutter build macos --release
```

---

## Proto-First Architecture

All service definitions come from `candelahq/candela-protos` (BSR:
`buf.build/candelahq/protos`).

### Generated Dart Stubs

Generated ConnectRPC Dart stubs are committed under `lib/gen/`:

```
lib/gen/
├── candela/v1/
│   ├── annotations.connect.client.dart
│   ├── annotations.pb.dart
│   ├── budget.connect.client.dart
│   ├── budget.pb.dart
│   ├── dashboard.connect.client.dart
│   ├── dashboard.pb.dart
│   ├── trace.connect.client.dart
│   └── trace.pb.dart
└── google/protobuf/
    └── timestamp.pb.dart
```

**Never hand-edit files in `lib/gen/`**. Regenerate with:

```bash
buf generate    # requires buf.gen.yaml pointing to candela-protos
```

---

## State Management (Riverpod 3.x)

The desktop app uses **Riverpod** with code-gen annotations for reactive state.

### Provider Hierarchy

```
┌───────────────────────────────────────────────┐
│  Configuration Providers (sync)               │
│  candelaHostProvider, baseChannelProvider      │
│  (settings → transport bootstrap)             │
└──────────────────┬────────────────────────────┘
                   │
┌──────────────────▼────────────────────────────┐
│  Service Providers (sync, derived)            │
│  connectApiServiceProvider, cliServiceProvider │
│  (depend on channel/config providers)         │
└──────────────────┬────────────────────────────┘
                   │
┌──────────────────▼────────────────────────────┐
│  State Providers (async, UI-facing)           │
│  DashboardController, TracesNotifier, etc.     │
│  (depend on service providers, drive UI)      │
└───────────────────────────────────────────────┘
```

### Key Conventions

1. **Providers are defined in `lib/providers.dart`** for services, and co-located
   with features for Notifiers.
2. **All providers are `@riverpod` annotated** — use code generation:
   ```bash
   dart run build_runner build
   ```
3. **`AsyncNotifier`** for any state that involves async operations (API calls)
4. **`ref.watch()`** for reactive dependencies, **`ref.read()`** for one-shot actions
5. Providers that depend on `candelaHostProvider` auto-rebuild when the server URL changes

### Configuration Provider Pattern

```dart
@riverpod
String candelaHost(Ref ref) {
  return Platform.environment['CANDELA_HOST'] ??
      'http://localhost:8181';
}

@riverpod
ClientChannel baseChannel(Ref ref) {
  final host = ref.watch(candelaHostProvider);
  return createConnectChannel(host);
}
```

### Feature Notifier Pattern

```dart
@riverpod
class DashboardController extends _$DashboardController {
  bool _disposed = false;

  @override
  Future<DashboardState> build() async {
    ref.onDispose(() => _disposed = true);
    final api = ref.watch(connectApiServiceProvider);
    final userId = ref.watch(currentUserIdProvider);
    return _fetchDashboard(api, userId);
  }

  /// Bridge for imperative listeners (e.g. tray-menu refresh).
  void onStateChanged(void Function(DashboardState) callback) {
    ref.listenSelf((_, next) {
      if (!_disposed && next is AsyncData<DashboardState>) {
        callback(next.value);
      }
    });
  }

  Future<void> refresh() async {
    if (_disposed) return;
    state = const AsyncLoading();
    state = await AsyncValue.guard(() => build());
  }
}
```

---

## Service Layer (`lib/services/`)

### ConnectApiService

`lib/services/connect_api_service.dart` is the **single RPC client** for all dashboard,
trace, budget, and model data. It:

1. Creates typed ConnectRPC service clients from generated stubs
2. Converts proto response objects to domain models
3. Handles error mapping (ConnectException → user-friendly messages)

**Key pattern — Proto → Domain conversion:**

```dart
List<TraceSummary> _mapTraces(ListTracesResponse response) {
  return response.traces.map((t) => TraceSummary(
    traceId: t.traceId,
    rootSpanName: t.rootSpanName,
    startTime: t.startTime.toDateTime(),
    totalCost: t.totalCost,
    tokenCount: t.totalTokens.toInt(),
    modelId: t.modelId,
  )).toList();
}
```

### CliService

`lib/services/cli_service.dart` manages the local `candela` CLI process:

1. **Starts** the proxy: `Process.start('candela', ['start', '--mode=solo'])`
2. **Monitors** stdout/stderr for status updates
3. **Stops** on app shutdown: sends SIGTERM, waits, then SIGKILL

### TelemetryService

`lib/services/telemetry_service.dart` provides app-level telemetry and health checks:
- Server connectivity checks
- Connection state streaming
- Anonymous usage reporting

---

## Domain Models

Domain models are plain Dart classes in `lib/models/`. They are **not** generated —
they represent the UI-layer data shape, decoupled from proto messages.

Key models:
- `DashboardData` — aggregated usage, costs, top models
- `TraceSummary` / `TraceDetail` — trace list vs detail view
- `ModelInfo` — model metadata with pricing
- `BudgetStatus` — user's current spend vs limits
- `ConnectionState` — enum for server connectivity

---

## ADC & Authentication

### CandelaAuthService (replaces gcloud CLI — PR #78)

`CandelaAuthService` now handles all authentication natively, replacing the previous
`gcloud auth application-default login` dependency:

- OAuth 2.0 PKCE flow for user login
- Automatic token refresh and persistence
- No external CLI tools required

The `candela` CLI proxy still handles Vertex AI token forwarding, but the desktop app
no longer requires `gcloud` to be installed.

### Server Authentication

When connecting to a remote Candela server, the desktop app may send:
- API key via `x-api-key` header
- Bearer token from ADC for Vertex AI proxying

---

## UI Structure

### Page Layout

```
lib/
├── main.dart                    # App entry, ProviderScope
├── providers.dart               # Core provider definitions
├── providers.g.dart             # Generated Riverpod code
├── app.dart                     # MaterialApp, routing
├── gen/                         # Generated ConnectRPC stubs (DO NOT EDIT)
├── models/                      # Domain models
├── services/                    # API clients, CLI management
├── screens/
│   ├── dashboard_screen.dart    # Main dashboard
│   ├── traces_screen.dart       # Trace list + detail
│   ├── models_screen.dart       # Model catalog
│   ├── settings_screen.dart     # Config, connection status
│   └── onboarding_screen.dart   # First-run setup
└── widgets/
    ├── trace_detail_panel.dart
    ├── cost_chart.dart
    └── ...
```

### Theming

The app uses a dark-first Material 3 theme:
- **Primary**: Candela orange/amber
- **Surface**: Dark greys
- **Typography**: System fonts, compact for data-dense views

---

## Testing

### Unit Tests

```bash
flutter test
```

Tests are in `test/` mirroring the `lib/` structure. Key patterns:
- Mock services with `mocktail` or manual mocks
- Use `ProviderContainer` for testing Riverpod providers in isolation
- Proto fixtures: create proto objects directly for test data

### Widget Tests

```dart
testWidgets('dashboard shows loading then data', (tester) async {
  await tester.pumpWidget(
    ProviderScope(
      overrides: [
        connectApiServiceProvider.overrideWithValue(mockApi),
      ],
      child: const CandleApp(),
    ),
  );
  expect(find.byType(CircularProgressIndicator), findsOneWidget);
  await tester.pumpAndSettle();
  expect(find.text('Total Cost'), findsOneWidget);
});
```

### Integration Tests

```bash
flutter test integration_test/
```

---

## Common Workflows

### Adding a New Dashboard Card

1. **Proto**: If the data field doesn't exist, add it to `DashboardResponse` in
   `candela-protos` and regenerate stubs
2. **Service**: Add the mapping in `ConnectApiService._mapDashboard()`
3. **Model**: Add the field to `DashboardData` in `lib/models/`
4. **Controller**: `DashboardController` already fetches the full response — just expose
   the new field. Uses immutable `state.copyWith()` updates and disposal guards.
5. **Widget**: Create a card widget in `lib/widgets/`, add it to `dashboard_screen.dart`

### Adding a New Screen

1. Create `lib/screens/new_screen.dart`
2. Add the route in `lib/app.dart` (use `GoRouter` if configured, otherwise Navigator)
3. Create an `AsyncNotifier` provider for the screen's state
4. Wire the provider to `ConnectApiService` for data fetching
5. Add a navigation entry (sidebar, bottom nav, or drawer)

### Updating Proto Stubs

When `candela-protos` has new definitions:

```bash
# In candela-desktop repo
buf generate                          # regenerates lib/gen/
dart run build_runner build           # regenerates Riverpod code if providers changed
flutter pub get                       # ensure deps are up to date
```

### Debugging Connection Issues

1. Check `candela` CLI is running: `ps aux | grep candela`
2. Verify port 8181 is listening: `lsof -i :8181`
3. Check the Settings screen → Connection Status
4. Review `TelemetryService` logs for connectivity events
5. If ADC: run `gcloud auth application-default print-access-token` to verify

---

## Build & Release

### macOS

```bash
flutter build macos --release
# Output: build/macos/Build/Products/Release/Candela.app
```

### Homebrew Cask

Distributed via `candelahq/homebrew-tap`:
```bash
brew install --cask candelahq/tap/candela-desktop
```

### CI/CD

GitHub Actions runs:
1. `dart format --set-exit-if-changed .`
2. `dart analyze --fatal-infos`
3. `flutter test`
4. `flutter build macos` (on macOS runner)

---

## Recent Changes (v0.5.1)

Key changes from recent PRs:

| PR | Change | Details |
|----|--------|---------|
| #85 | Update button | Calls `performBrewUpgrade()` with confirmation dialog |
| #85 | maxSyntheticSpans clamp | Fixed span limit clamping for large traces |
| #85 | Model deduplication | Models deduplicated by model name in models page |
| #79 | Dynamic UserScope toggle | Admin views can toggle user scope filtering |
| #77 | Pricing columns | Models page shows pricing + cache efficiency badges |

---

## Dependencies (key packages)

| Package | Version | Purpose |
|---------|---------|---------|
| `flutter_riverpod` | ^3.0.0 | State management |
| `riverpod_annotation` | ^3.0.0 | Riverpod code-gen |
| `connectrpc` | ^0.4.0 | ConnectRPC Dart client |
| `protobuf` | ^4.0.0 | Protobuf runtime |
| `fl_chart` | ^0.70.0 | Charts & graphs |
| `window_manager` | ^0.4.0 | Desktop window control |
| `url_launcher` | ^6.3.0 | Opening URLs |
| `path_provider` | ^2.1.0 | Platform paths |
| `shared_preferences` | ^2.3.0 | Local config storage |
| `build_runner` | dev | Code generation |
| `riverpod_generator` | dev | Riverpod code-gen |
