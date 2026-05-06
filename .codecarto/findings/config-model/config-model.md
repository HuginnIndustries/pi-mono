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

---

## 2026-05-06 — contracts phase (append)

Authoritative refinement of the architecture-phase config model with corrections.

### Corrected Env Var Anchor

- The state-anchor env var is **`PI_CODING_AGENT_DIR`** (templated `${APP_NAME.toUpperCase()}_CODING_AGENT_DIR` at `packages/coding-agent/src/config.ts:374-380`).
- `ENV_AGENT_DIR` is the source-internal constant name; the architecture phase mistakenly elevated the constant name to a user-facing variable.
- A fork rebrands every `PI_*` var by changing `APP_NAME`.

### Per-Provider API-Key Env Var Table

Authoritative per-provider env var table from `packages/ai/src/utils/env-api-keys.ts:91-209`:

| Provider | Env var(s) | Notes |
|---|---|---|
| Anthropic | `ANTHROPIC_API_KEY` | |
| OpenAI | `OPENAI_API_KEY` | |
| Azure OpenAI | `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_ENDPOINT` | Endpoint required separately |
| Google AI Studio | `GOOGLE_API_KEY` / `GOOGLE_GENERATIVE_AI_API_KEY` | |
| Google Vertex | `<authenticated>` sentinel | Uses Vertex Application Default Credentials |
| AWS Bedrock | `<authenticated>` sentinel | Uses AWS SDK ambient credentials |
| Mistral | `MISTRAL_API_KEY` | |
| OpenAI Codex | `OPENAI_API_KEY` (also OAuth) | |
| OpenAI-compatible (per URL family) | varies | See `openai-completions.ts:1029-1088` for the URL→key matrix |

Plus hidden `PI_CACHE_RETENTION` override (cache TTL), and a Bun `/proc/self/environ` Linux fallback for ambient env reads.

### File Schema Authoritative Reference

For `auth.json`, `settings.json`, `models.json`, sessions JSONL:

| File | Format | Schema location | Validation timing | Failure mode |
|---|---|---|---|---|
| `auth.json` | JSON | Implicit in `auth-storage.ts` types | On startup `reload()` | Corrupt → `loadError` set, all reads blocked until fixed (per Pass 6) |
| `settings.json` | JSON | TS type in `core/settings-manager.ts` | On read | Schema-validation depth: open question (con-OQ1+) |
| `models.json` | JSON | TS type in `core/config.ts` | On read | Headers values may be `!`-prefix shell strings — RCE channel for untrusted sources (dsm-RT1) |
| Sessions JSONL v3 | JSONL | TS type (defer schema to protocols, con-CF3) | On load via `loadEntriesFromFile` | Malformed lines silently skipped (defect 2.7) |

### Project-Scoped Layer (`<repo>/.pi/`)

| Path | Purpose | Discovery rule |
|---|---|---|
| `<repo>/.pi/extensions/` | Project-scoped extensions | Loaded FIRST, then operator. Dedup by absolute path (not by name). |
| `<repo>/.pi/prompts/` | Project-scoped slash commands | Loaded SECOND in discovery order (agent-dir → cwd-dir → flag paths) |
| `<repo>/.pi/models.json` | Project-scoped model overrides + headers | Active config — `!`-prefix shell strings execute |
| `<repo>/.pi/git/`, `<repo>/.pi/npm/` | Project-scoped credential storage | Gitignored; runtime credential file storage for in-project agent runs |

### CLI-Flag Layer

30+ user-facing flags parsed at `packages/coding-agent/src/cli/args.ts:67-189`. Unknown flags flow into `unknownFlags` map for extension consumption (`args.ts:167-180`) — extensions register their own flags.

### Migration Layer

Per Pass 6 and contracts phase: `runMigrations` runs unconditionally on every startup with bare-`catch {}` failure handling. Migrations affect:
- auth.json structure
- sessions path/format
- binaries (rg/fd) location
- keybindings
- commands → prompts

Hazard: silent partial migration on failure.

### Trust-Boundary Reminder

The `<repo>/.pi/` layer is **trusted only if the user trusts the repo**. Cloning a hostile repo and running `pi` in it is a Pass 4 attack surface (covert RCE via `models.json` `!`-prefix; arbitrary extensions auto-loaded into the agent process). Defer full Pass 4 treatment to defect-scan-semantic.
