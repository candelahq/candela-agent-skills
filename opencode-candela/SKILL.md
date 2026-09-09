---
name: opencode-candela
description: |
  Development conventions for the candelahq/opencode-candela repository — the
  official OpenCode plugin for Candela LLM observability, cost guardrails, and
  spending intelligence. Covers plugin lifecycle hooks (chat.headers, event),
  strict URL origin verification (CWE-200 prevention), attribution headers,
  local session analytics, and testing patterns.
  Use this skill when developing or contributing to the opencode-candela plugin.
license: Apache-2.0
metadata:
  version: v2
  publisher: candelahq
---

# Candela OpenCode Plugin Development Skill

## Repository Overview

`opencode-candela` is the official plugin for [OpenCode](https://github.com/opencode-ai/opencode) — a terminal-native AI coding agent. It injects real-time cost tracking, daily budget guardrails, and mission attribution into AI coding sessions.

- **Language**: TypeScript (ES2022)
- **Package**: `@candelahq/opencode` (published to npm)
- **Runtime**: Node.js ≥ 20
- **Integration Surface**: OpenCode Plugin API (`chat.headers` transform, `event` listener, `tool` contributions)

---

## Environment & Toolchain

### Required Tools

- **Node.js** ≥ 20 (Node 22 or 24 recommended)
- **npm**

### Building & Testing

```bash
npm ci                  # Install dependencies
npm run build           # Compile TypeScript to dist/
npm test                # Run Jest test suite
npm run lint            # ESLint check
```

---

## Plugin Lifecycle & Architecture

The plugin entry point is `src/index.ts`. It exports a default plugin factory function that registers hooks with the OpenCode runtime:

```typescript
export default async function candelaPlugin(client: OpenCodeClient) {
  // 1. Resolve Candela server URL (env var -> config -> default http://localhost:8181)
  // 2. Initialize Candela client with cache TTL
  // 3. Register lifecycle hooks:
  return {
    "chat.headers": async (input, output) => { /* header injection */ },
    event: async ({ event }) => { /* session lifecycle & cost alerts */ },
  };
}
```

### 1. `chat.headers` Hook

Executes before any outgoing LLM request. Injects tracing, context, and attribution headers into the HTTP request headers.

> [!IMPORTANT]
> **Strict Origin Matching (CWE-200 Prevention)**: Never inject sensitive tracing headers or internal IDs into requests targeting direct external third-party providers. All header injection MUST pass `isCandelaRequest(input, candelaUrl)`.

```typescript
if (!isCandelaRequest(input, candelaUrl)) {
  return; // Skip third-party endpoints
}
```

### 2. `event` Hook

Listens to agent session events:
- **`file.watcher.updated`**: Detects external configuration updates (`.opencode.json` or `~/.config/opencode/config.json`) and invalidates cached budgets via `candela.invalidateCache()`.
- **`session.start`**: Validates initial user budget and prints welcome banner/status.
- **`session.end`**: Logs session summary to local analytics and reports total token/USD consumption.

---

## Security & Origin Validation

To prevent leaking internal mission IDs, git hashes, or credentials to third-party endpoints (CWE-200), `opencode-candela` implements strict origin matching:

### `matchesOrigin(urlA, urlB)`

Compares protocol, hostname, and port with **loopback equivalence**:
* `http://localhost:8181` and `http://127.0.0.1:8181` are treated as equivalent origins.
* Disallows partial URL substring attacks (e.g., `attacker.com/?target=http://localhost:8181`).

```typescript
export function matchesOrigin(urlA?: string, urlB?: string): boolean {
  if (!urlA || !urlB) return false;
  try {
    const parsedA = new URL(urlA);
    const parsedB = new URL(urlB);
    if (parsedA.origin === parsedB.origin) return true;
    const isLocalA = parsedA.hostname === "localhost" || parsedA.hostname === "127.0.0.1";
    const isLocalB = parsedB.hostname === "localhost" || parsedB.hostname === "127.0.0.1";
    const portA = parsedA.port || (parsedA.protocol === "https:" ? "443" : "80");
    const portB = parsedB.port || (parsedB.protocol === "https:" ? "443" : "80");
    return isLocalA && isLocalB && parsedA.protocol === parsedB.protocol && portA === portB;
  } catch {
    return false;
  }
}
```

---

## Attribution & Tracing Headers

When talking to a validated Candela endpoint, the plugin attaches:

| Header | Source | Purpose |
|---|---|---|
| `X-Mission-Id` | `process.env.CANDELA_MISSION_ID` | Groups multi-step agent workflows across child sessions |
| `X-Candela-Job-Id` | `process.env.CANDELA_MISSION_ID` | Backend job attribution for task-scoped budget tracking |
| `X-Git-Commit` | Workspace git repository | Attaches active git commit SHA to generated traces |
| `X-Session-Tag` | `.opencode.json` settings | Arbitrary team/environment tags for filtering |

---

## Local Analytics

The plugin logs session metadata to `~/.candela/session_analytics.jsonl` for offline developer spend insights:
- Session ID, start/end timestamps
- Input/output tokens, total estimated USD cost
- Model and provider names
- Budget state (remaining, percent used)

---

## Key Conventions & Best Practices

1. **Best-Effort Operations**: Analytics logging and cache invalidations must be wrapped in `try/catch` blocks so disk or network glitches never break the developer's chat session.
2. **Deterministic Loopback Resolution**: When constructing default URLs, prefer `http://localhost:8181` or extract from `~/.config/candela/config.yaml`.
3. **Unit Testing**: All security checks (`matchesOrigin`, `isCandelaRequest`, header injection) must have 100% test coverage under `src/__tests__/`.
