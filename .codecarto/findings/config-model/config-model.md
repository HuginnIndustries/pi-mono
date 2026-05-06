# Config Model — pi-mono

Catalog of configuration sources, precedence, and inheritance. Append-only
across phases; each phase adds a dated section.

---

## 2026-05-06 — architecture phase

### Auth Resolution Precedence (5-Level)

Defined in `packages/coding-agent/src/core/auth-storage.ts:446-516`:

| Level | Source | Notes |
|---|---|---|
| 1 (highest) | Runtime override | Programmatic API call passing explicit `apiKey` |
| 2 | `auth.json` `api_key` | Per-provider key in `~/.pi/agent/auth.json` |
| 3 | `auth.json` `oauth` | OAuth token; refresh path uses `proper-lockfile` to prevent multi-instance refresh races |
| 4 | Environment variable | Provider-specific env var (e.g. `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`) |
| 5 (lowest) | Fallback resolver | Provider-specific fallback (e.g. Vertex Application Default Credentials) |

The **`pi-ai` package itself only reads env vars** (and Vertex ADC) directly
— the `auth.json` path is owned by `coding-agent`'s auth-storage layer
which threads the resolved key in via `options.apiKey`. *(observed fact —
`packages/ai/test/oauth.ts:14` is the only `pi-ai` reference to
`auth.json`; full reads happen in coding-agent)*

### State Anchor Override

| Variable | Effect | Default |
|---|---|---|
| `ENV_AGENT_DIR` | Overrides the operator-state root | `~/.pi/agent/` |

Defined at `packages/coding-agent/src/core/config.ts:402-407`.

### Project-Scoped vs Operator-Scoped

| Layer | Path | Notes |
|---|---|---|
| Operator | `~/.pi/agent/` (overridable via `ENV_AGENT_DIR`) | Per-user defaults |
| Project | `<repo>/.pi/` | Per-project overrides; `extensions/`, `prompts/`, `git/`, `npm/` |

Project-scoped state is expected to take precedence over operator-scoped
for extensions and prompts. *(strong inference — defer exact precedence to
contracts phase)*

### Test-Mode Config Scrubbing

`test.sh`:
1. Backs up `~/.pi/agent/auth.json`.
2. Unsets ~30 provider envvars before `npm test`.
3. Restores after.

This makes the test runner hermetic and prevents incidental real-provider
calls. *(observed fact)*

### Hardcoded / Compiled-In Config

| Setting | Location | Value | Why notable |
|---|---|---|---|
| undici `bodyTimeout` | `packages/coding-agent/src/cli.ts:20` | `0` | Disables built-in timeouts globally — defect-scan carry-forward `arch-CF7` |
| undici `headersTimeout` | `packages/coding-agent/src/cli.ts:20` | `0` | (same) |
| Vitest test timeout | `packages/ai/vitest.config.ts` (cited via extraction) | 30s | Real-provider e2e tests gated on auth |
| OpenAI-compat URL matrix | `packages/ai/src/providers/openai-completions.ts:1030+` | ~15 vendor branches | Behavioral switches per vendor URL |
| ANSI escape sequences | `packages/tui/src/tui.ts` | hard-coded (synchronized output 2026, bracketed paste 2004, Kitty keyboard, modifyOtherKeys, Kitty/iTerm2 image, OSC 9;4 progress) | Portability hazards |

### Configuration Open Questions

- Full `PI_*` env var inventory and per-package precedence rules — defer to
  the contracts phase (`arch-CF1`/`arch-CF2`).
- Effect of malformed `auth.json` on startup — defer to defect-scan-mechanical
  Pass 6 (`arch-OQ2`).
- Whether `~/.pi/agent/settings.json` honors a JSON-Schema / TypeScript
  type contract or accepts arbitrary keys — defer to contracts.
