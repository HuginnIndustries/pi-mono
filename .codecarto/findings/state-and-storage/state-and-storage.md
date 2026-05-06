# State and Storage — pi-mono

Catalog of durable state. Append-only across phases; each phase adds a dated section.

---

## 2026-05-06 — architecture phase

### Operator State Anchor

Default root: `~/.pi/agent/`. Overridable via `ENV_AGENT_DIR` env var
(`packages/coding-agent/src/core/config.ts:402-407`). *(observed fact)*

| Path | Purpose | Format | Mode | Owner |
|---|---|---|---|---|
| `~/.pi/agent/auth.json` | LLM provider credentials, OAuth tokens | JSON | 0600 (enforced) | `coding-agent`, written by `auth-storage.ts:446-516` |
| `~/.pi/agent/settings.json` | User settings | JSON | default | `coding-agent` |
| `~/.pi/agent/models.json` | Model overrides + catalog cache | JSON | default | `coding-agent` (defer schema to protocols) |
| `~/.pi/agent/sessions/--<encoded-cwd>--/<ts>_<sessionId>.jsonl` | Session transcripts | JSONL, format v3, append-only with full-rewrite for branch edits | default | `coding-agent` |
| `~/.pi/agent/themes/` | User themes | Directory tree | default | `coding-agent`/extensions |
| `~/.pi/agent/tools/` | User-installed tool binaries (`rg`, `fd` auto-downloaded here) | Binaries | exec | `coding-agent` `tools-manager.ts` |
| `~/.pi/agent/bin/` | User-installed CLI helpers | Binaries | exec | `coding-agent` |
| `~/.pi/agent/prompts/` | User-defined slash commands | Markdown files | default | `coding-agent` |
| `~/.pi/agent/extensions/` | User extensions | TS/JS modules loaded via `jiti` | default | `coding-agent` extension loader |
| `~/.pi/agent/skills/` | User skills (similar to prompts but more structured) | Markdown bundles | default | `coding-agent` |
| `~/.pi/agent/pi-debug.log` | Diagnostic trace | Text log | default | `coding-agent` |

### Project-Scoped State (`<repo>/.pi/`)

In pi-mono itself (and any project consuming `pi`):

| Path | Purpose |
|---|---|
| `.pi/extensions/` | Project-scoped TUI widgets / hooks (URL prompt analyzer, performance profiling hooks, redraw counters in pi-mono itself) |
| `.pi/prompts/` | Project-scoped slash commands (`cl.md`, `is.md`, `pr.md`, `wr.md` ship in pi-mono) |
| `.pi/git/`, `.pi/npm/` | Gitignored credential storage for in-project agent runs |

Project-scoped state takes precedence over operator-scoped state for
extensions and prompts. *(strong inference — typical layered config; defer
exact precedence to contracts/config-model)*

### Environment Variables (architecture-level)

The `test.sh` hermetic runner unsets ~30 provider envvars before `npm
test`. Authoritative list to be enumerated in
[`config-model.md`](../config-model/config-model.md) by the contracts
phase. The architecture-level callout: **`ENV_AGENT_DIR`** overrides the
state anchor; **`PI_*`** variables exist (defer enumeration to contracts).
*(observed fact — `test.sh`)*

### State Owned Inside Packages (transient)

- `pi-agent-core`: in-memory only. `sessionId` is an opaque string forwarded
  to providers; `agent.reset()` is in-memory. Persistence is the caller's
  job. *(observed fact — `packages/agent/src/agent.ts:106,202,415`)*
- `pi-tui`: `previousLines` buffer for differential render in the
  `TUI.doRender` loop. Memory-only. *(observed fact)*
- `pi-web-ui`: browser `localStorage` + `indexedDB`, model fixture at
  `src/utils/test-sessions.ts` (128 KB demo fixture, not runtime state).

### Generated Artifacts (Build Output)

- `packages/ai/src/models.generated.ts` — 16,815 lines, generated from
  models.dev / Vercel AI Gateway / Cloudflare AI Gateway by
  `packages/ai/scripts/generate-models.ts`. **Treat as opaque** for
  downstream phases. *(observed fact)*
- Per-package `dist/` directories from build.
- Bun-compiled binaries shipped via `build-binaries.yml` (5 platforms).
  `photon_rs_bg.wasm` bundled beside the binary; Windows `.node` shipped
  beside the binary; `koffi` externalized.

### Locks / Concurrency on State

- `proper-lockfile` on `auth.json` during OAuth refresh. *(observed fact)*
- `withFileMutationQueue` per-file serialization for tool-driven file edits.
- No SQLite, leveldb, or external database. *(observed fact)*

### Open Questions

- What is the exact replay semantics of session JSONL v3 — when a branch
  edit triggers a full-rewrite, are concurrent readers able to detect the
  rewrite mid-read? *(defer to protocols phase, see arch-CF4)*
- Does `pi` enforce `auth.json` mode 0600 on every write or only on creation?
  *(defer to defect-scan-semantic Pass 4)*

---

## 2026-05-06 — contracts phase (append)

### Migrations Run on Every Startup

`runMigrations` (`packages/coding-agent/src/core/migrations.ts:304-314`) runs unconditionally on every `pi` startup. Steps (each wrapped in bare `catch {}`):
1. Auth structure migration (auth.json layout)
2. Sessions migration (path/format updates)
3. Binaries migration (rg/fd location moves)
4. Keybindings migration
5. Commands → prompts migration

Failure handling silently swallows errors — partial migration state can persist. Documented as a contract: every `pi` startup performs schema migrations regardless of source-version semantics.

### `models.json` Headers as Active Channel

`models.json` headers values can begin with `!` and execute as shell commands during config resolution (`resolveConfigValue` in `packages/coding-agent/src/core/config.ts`). When `models.json` lives at `<repo>/.pi/models.json` and the project comes from an untrusted source, this is a **covert RCE channel**. (Pass 4 finding `dsm-RT1`; mechanical row P6.4.)

### Auth Persistence Update

`auth.json` mode 0600 enforced **on every write** (creation, sync, async paths) — confirmed by Pass 6, closes part of arch-CF9. Windows ACL behavior remains an open question (`con-OQ2`).

### `pi-debug.log` Audit (open)

Whether `pi-debug.log` leaks secrets / OAuth tokens is `con-OQ1` — needs live runtime test.

### Extension State

- Extension instances are loaded fresh on `ctx.reload()` (`agent-session.ts:2383`). `moduleCache: false` in the jiti loader ensures no stale closures.
- Project-scoped extensions (`<cwd>/.pi/extensions/`) are added FIRST; operator-scoped (`~/.pi/agent/extensions/`) SECOND. Dedup is by absolute resolved path (first-seen wins) — same name in different dirs both load.

### Subcommand-Owned State

`pi install`, `pi update`, `pi config` modify operator state outside session bounds. (Defer detail to porting/reimpl-spec.)
