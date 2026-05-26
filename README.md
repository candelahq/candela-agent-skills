# 🕯️ Candela Agent Skills

Agent skills for AI-assisted development across the Candela ecosystem.

These skills teach AI coding agents (Gemini, Claude, etc.) the architecture, conventions, and workflows used across Candela repositories — so they produce correct code on the first try instead of guessing.

## Repositories Covered

| Skill | Repository | Stack |
|-------|-----------|-------|
| `candela` | [candelahq/candela](https://github.com/candelahq/candela) | Go · Rust · Next.js · DuckDB · BigQuery |
| `candela-desktop` | [candelahq/candela-desktop](https://github.com/candelahq/candela-desktop) | Flutter · Dart · Riverpod · ConnectRPC |
| `candela-jetbrains` | [candelahq/candela-jetbrains](https://github.com/candelahq/candela-jetbrains) | Kotlin · IntelliJ Platform SDK · Gradle |
| `candela-vscode` | [candelahq/candela-vscode](https://github.com/candelahq/candela-vscode) | TypeScript · VS Code API · Open VSX |

## Installation

### Antigravity (Gemini)

Copy or symlink skill directories into `~/.gemini/antigravity/skills/`:

```bash
# Clone this repo
git clone https://github.com/candelahq/candela-agent-skills.git

# Symlink into Antigravity skills
ln -s $(pwd)/candela-agent-skills/candela ~/.gemini/antigravity/skills/candela
ln -s $(pwd)/candela-agent-skills/candela-desktop ~/.gemini/antigravity/skills/candela-desktop
ln -s $(pwd)/candela-agent-skills/candela-jetbrains ~/.gemini/antigravity/skills/candela-jetbrains
ln -s $(pwd)/candela-agent-skills/candela-vscode ~/.gemini/antigravity/skills/candela-vscode
```

### Other Agents

Copy the `SKILL.md` files into the appropriate configuration directory for your agent.

## Structure

```
candela-agent-skills/
├── candela/
│   └── SKILL.md          # Go backend, Rust, Next.js UI, storage, proxy
├── candela-desktop/
│   └── SKILL.md          # Flutter desktop app, Riverpod, ConnectRPC
├── candela-jetbrains/
│   └── SKILL.md          # JetBrains IDE plugin, Kotlin, Gradle
├── candela-vscode/
│   └── SKILL.md          # VS Code extension, TypeScript, Open VSX
└── README.md
```

## Contributing

When architectural patterns change in the upstream repos, update the corresponding skill to keep agents current. Skills should encode **conventions and patterns** — not documentation that belongs in `docs/`.

## License

Apache-2.0
