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

---

## 2026-05-06 — protocols phase (append)

### Session JSONL v3 Schema Catalog (resolves arch-CF4 + con-CF3)

10 record types in the `SessionEntry` discriminated union (`packages/coding-agent/src/core/session-manager.ts`):

1. `message` (role: user/assistant/tool)
2. `compaction` (summary that semantically replaces older entries)
3. `branch_summary` (marks a branch fork point)
4. `thinking_level_change` (mid-session thinking toggle)
5. `model_change` (mid-session model swap)
6. `custom` (extension-driven)
7. `custom_message` (extension-driven)
8. `label` (named bookmark)
9. `session_info` (per-file header — first record; carries `sessionId`, `cwd`, `parentSessionId?`, `version: 3`)
10. (base) `SessionEntryBase = { type, id, parentId, timestamp }`

**Identifiers**:
- `sessionId`: uuidv7
- Per-entry `id`: 8 hex chars from `randomUUID()` with up-to-100-retry collision check (`session-manager.ts:206-213`)
- `timestamp`: ISO-8601 string (no ns precision)

**Branching**: 3 distinct mechanisms — `branch()` in-place leaf move, `branchWithSummary()` adds a `branch_summary` entry, `createBranchedSession()` writes a new file with only the path-to-leaf and `parentSessionId` in header.

### Deferred-Flush Behavior (resolves dsm-RT6)

`_persist` (`session-manager.ts:801-819`) buffers in memory until first assistant message arrives, then flushes the whole buffer. **Crash before first assistant ⇒ user prompt lost** — the file may not exist on disk yet. `dispose()` and `abort()` do not flush.

### Non-Atomic Rewrite Confirmed (Defect 2.1)

`_rewriteFile` (`session-manager.ts:775-779`) is `writeFileSync(file, content)` with no temp+rename. Triggered on migration, corrupt-file recovery, and `createBranchedSession` paths.

### Encoded-cwd Lossy

`--${cwd.replace(/^[/\\]/,"").replace(/[/\\:]/g,"-")}--` (`session-manager.ts:429`):
- `/a/b` and `/a-b` collide
- Case sensitivity inherits the host filesystem
- **Cross-OS sync of `~/.pi/agent/sessions/` is unsafe**

### Concurrent-Reader Behavior (open)

No file locking, no inotify, no atomic publish. Mid-rewrite reader behavior is undefined; `pro-OQ2` requires runtime test.

### `auth.json` Schema Refinement

Per-provider entries with shape `{ type: "api_key" | "oauth", api_key?: string, oauth?: {...} }`. Implicit schema in `auth-storage.ts` TS types. Mode 0600 enforced on every write (Pass 6 confirmed).

### Lock Semantics Family

| File | Lock | Concurrent-write semantics |
|---|---|---|
| `auth.json` | proper-lockfile during OAuth refresh | Single-flight refresh |
| `settings.json` | proper-lockfile via SettingsManager | Defect P6.6 / dsm-RT3: `migrateAuthToAuthJson` does unlocked read-modify-write that races SettingsManager |
| `models.json` | None | Reader's tolerance for concurrent rewrite open |
| Sessions JSONL | None | Concurrent readers see partial writes |

---

## 2026-05-06 — porting phase (append)

The porting bundle's §Persistent Schema Notes consolidates the 4 file-format catalogs (auth.json, settings.json, models.json, session JSONL v3) and the 3 branching modes into a port-ready view. Three porting-policy decisions documented:

1. **Session JSONL atomicity** (defect 5.8 / mech 2.1): a port must implement temp-write + rename. Recommendation: **fix before porting**.
2. **Deferred-flush** (defect 5.7 / dsm-RT6): port-CF5 to reimpl-spec — preserve optimization (with documented contract) or flush on every event.
3. **Encoded-cwd lossy collisions** (protocols H16): a port should hash the cwd, not lossy string-replace.

### Cross-Phase Correction Applied (sem-CR2)

The per-file session JSONL header's `type` discriminator is **`"session"`** (per `packages/coding-agent/src/core/session-manager.ts:31`), not `"session_info"` as protocols PS1 wrote. `"session_info"` is a separate user-defined display-name marker. Porting bundle §Persistent Schema Notes reflects the corrected view; this append makes the correction durable in the catalog secondary.
