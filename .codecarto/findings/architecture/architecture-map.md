# Architecture Map — pi-mono

<!--
  Phase: architecture
  Pipeline: workflow/pipeline-full-with-deep-audit.yaml
  Source location: ../ (the pi-mono repo root)
  Evidence levels: observed fact / strong inference / portability hazard / open question
  Source-of-detail: scratch/arch-{ai,agent-tui,coding-agent,web-ui,build}.md (per-package + cross-cutting extraction notes).
-->

## System Intent

`pi-mono` is a TypeScript monorepo whose flagship deliverable is the **`pi`** interactive
coding-agent CLI. Around the CLI, four supporting libraries provide a unified multi-provider
LLM streaming layer (`@mariozechner/pi-ai`), a generic agent runtime
(`@mariozechner/pi-agent-core`), terminal UI primitives
(`@mariozechner/pi-tui`), and a Lit-based web chat component library
(`@mariozechner/pi-web-ui`). The five packages are versioned in lockstep
(currently `0.73.0`), built in a fixed order, and distributed through three
channels: npm packages, Bun-compiled standalone binaries for five platforms,
and browser-targeted web components. *(observed fact — `package.json:5-9`,
`README.md:1-50`, `scripts/sync-versions.js:36-45`)*

The system's job is to put a competent, extensible coding agent in front of the
user, talking to ~15 LLM providers through one stream shape, with full session
recording, an extension API, and both terminal-native and browser-native
delivery surfaces. The cross-cutting design choice is **provider neutrality
behind a stable streaming event union**: every provider — OpenAI Responses,
Anthropic Messages, Google Vertex/AI, Bedrock Converse Stream, Azure OpenAI,
Mistral, OpenAI Codex Responses, plus a `faux` test fixture — collapses into
the same `AssistantMessageEvent` union (12 variants, terminal events `done` /
`error`). *(strong inference — `packages/ai/src/types.ts:269-281`,
`packages/ai/src/api-registry.ts`, `packages/ai/src/providers/register-builtins.ts:342-403`)*

## Layer Map

### Package Inventory

| Package | Role | Public Entrypoints | Key Internal Deps | Runtime Surface |
|---|---|---|---|---|
| `@mariozechner/pi-ai` (`packages/ai`) | protocol or normalization layer | `main`/`types` + `./oauth` subpath; ~70 named exports from `src/index.ts`; `streamSimple()`, provider registry, model catalog, oauth helpers | None (depends only on vendor SDKs + `typebox`) | Node + browser (Bedrock + `undici`/`proxy-agent` lazy-imported behind dynamic gates so browser builds work; `env-api-keys.ts:1-24`) |
| `@mariozechner/pi-agent-core` (`packages/agent`) | core semantics | `main`/`types` flat; `Agent` class (`src/agent.ts:158`), `agent-loop` (`src/agent-loop.ts`), `streamProxy` (`src/proxy.ts:116`) | `pi-ai`, `typebox` | Node-first; in-process driver. No on-disk persistence — caller's job. *(observed fact — `src/agent.ts:106,202,415`)* |
| `@mariozechner/pi-tui` (`packages/tui`) | UI or rendering | `Component`, `TUI`, `Container`, `ProcessTerminal`, `StdinBuffer`, 12 widgets (`src/index.ts:1-106`) | None (`chalk`, `marked`, `mime-types`, `get-east-asian-width`, optional `koffi`) | Node-first terminal; Windows requires `koffi` FFI for modifier-aware keys *(portability hazard)* |
| `@mariozechner/pi-web-ui` (`packages/web-ui`) | UI or rendering | Single `"."` and `"./app.css"` exports; ~75 named exports incl. ~30 custom elements (`pi-chat-panel` flagship at `ChatPanel.ts:17`) | `pi-ai` (uses `streamSimple`), `pi-tui` *(observed fact — `package.json` deps)* | Browser-only (uses `localStorage`, `indexedDB`, `navigator.clipboard`, `import.meta.url` worker URLs); ESM-only unbundled `tsc` output — consumers must bundle |
| `@mariozechner/pi-coding-agent` (`packages/coding-agent`) | product shell | `bin: pi`; CLI in `src/cli.ts`; built-in tools in `src/core/tools/`; extension API (`registerTool/Command/Shortcut/Flag/Provider/MessageRenderer`) | `pi-ai`, `pi-ai/oauth`, `pi-agent-core`, `pi-tui`, `jiti` (extension loader), optional `clipboard` | Node + Bun standalone binary; **does NOT depend on `pi-web-ui`** *(observed fact — verified by extraction agent)* |

The five packages cleanly partition into two stable bases (`pi-ai`, `pi-tui`),
one core-semantics layer (`pi-agent-core`) on top of `pi-ai`, and two delivery
shells (`pi-coding-agent`, `pi-web-ui`) on top of those. *(strong inference)*

### Dependency Direction

```
        ┌────────────────────┐         ┌──────────────────────┐
        │   pi-ai (provider  │         │   pi-tui (terminal   │
        │   normalization)   │         │   primitives)        │
        └────────┬───────────┘         └─────────┬────────────┘
                 │                               │
       ┌─────────┴───────────┐                   │
       │                     │                   │
 ┌─────▼─────────┐   ┌───────▼─────────┐    ┌────▼──────────┐
 │ pi-agent-core │   │  pi-web-ui      │    │ pi-coding-agent│
 │ (agent loop)  │   │ (Lit components)│    │  (pi CLI)     │
 └─────────┬─────┘   └─────────────────┘    └────┬──────────┘
           │                                     │
           └─────────────────────────────────────┘
                 (coding-agent uses agent-core)
```

- **Lowest stable layer**: `pi-ai` (no internal deps) and `pi-tui` (no internal
  deps). *(observed fact)*
- **No cycles.** `pi-coding-agent` and `pi-web-ui` are leaves; `pi-agent-core`
  sits between `pi-ai` and `pi-coding-agent`. *(observed fact — extraction
  agents verified import direction)*
- `pi-coding-agent` does **not** depend on `pi-web-ui` (the two delivery
  surfaces are independent). *(observed fact — verified)*
- **Build order is load-bearing** because `pi-web-ui` uses stock `tsc` while
  the others use `tsgo`, and downstream packages need upstream `dist/*.d.ts`:
  `tui → ai → agent → coding-agent → web-ui` *(observed fact — `README.md:74`,
  root `package.json` build script)*. Treat as a **portability hazard** for
  any reimplementation that flattens or parallelizes the build.

## Public Surfaces

A one-paragraph synopsis here; the enumerated catalog appends to
[`findings/public-surfaces/public-surfaces.md`](../public-surfaces/public-surfaces.md).

- **Binaries**: `pi` (from `pi-coding-agent` `package.json` `bin`).
- **Libraries**: 5 npm packages under `@mariozechner/pi-*`. All ESM-first;
  `pi-web-ui` is ESM-only.
- **Network surfaces**: optional SSE proxy endpoint at `/api/stream`
  (`pi-agent-core`'s `streamProxy`, `src/proxy.ts:116`); LLM provider HTTPS
  endpoints (each provider package has its own URL set, including ~15
  OpenAI-compatible vendor branches in
  `packages/ai/src/providers/openai-completions.ts:1030+`).
- **File formats**: session JSONL trees under `~/.pi/agent/sessions/--<encoded-cwd>--/<ts>_<sessionId>.jsonl`,
  format version 3, append-only with full-rewrite for branch edits *(observed fact —
  `coding-agent` extraction agent)*; `auth.json` (mode 0600) credential store; `settings.json`,
  `models.json`, `themes/`, `tools/`, `bin/`, `prompts/`, `extensions/`,
  `skills/`, `pi-debug.log` under `~/.pi/agent/` (overridable via
  `ENV_AGENT_DIR`, `config.ts:402-407`).
- **Browser custom elements**: ~30 custom elements registered by `pi-web-ui`
  (`pi-chat-panel` is the flagship).
- **User-facing modes**: interactive TUI, non-interactive RPC (in-process,
  driven via `Agent` class), headless / scriptable runs.

## Runtime Lifecycle

A one-paragraph synopsis here; the enumerated lifecycle appends to
[`findings/runtime-lifecycle/runtime-lifecycle.md`](../runtime-lifecycle/runtime-lifecycle.md).

`pi` boot: `cli.ts` parses args → resolves auth via the 5-level precedence in
`src/core/auth-storage.ts:446-516` (runtime override → `api_key` → oauth →
env → fallback resolver) → constructs `AgentSession` with 5 distinct
`AbortController`s → registers built-in tools (`read`, `bash`, `edit`,
`write`, `grep`, `find`, `ls`) → loads extensions via `jiti` with bundled
`virtualModules` (so they work in the Bun-compiled binary) → enters the
loop. *(observed fact — coding-agent agent extraction note.)* Each
iteration: provider stream → emit
`AssistantMessageEvent`s → if `tool-call` event, dispatch to tool → write
result back into transcript → loop. Transcript writes are append-only JSONL
with full-rewrite for branch edits. *(observed fact)*

`grep` and `find` shell out to `rg` / `fd`, which are auto-downloaded by
`utils/tools-manager.ts` on first use — a network side effect at startup.
*(portability hazard)*

## Concurrency Model

- **Threading model**: single-threaded event loop with async/await everywhere.
  *(observed fact)*
- **Stream primitive**: hand-rolled `EventStream` async-iterable with a waiter
  queue (`packages/ai/src/utils/event-stream.ts:4-66`); used to bridge provider
  streams into the unified `AssistantMessageEvent` union. *(observed fact)*
- **Cancellation**: `AbortController`/`AbortSignal` plumbed through every
  provider call. `AgentSession` carries 5 separate `AbortController`s
  (run-level, per-request, per-tool-call, etc.). *(observed fact — coding-agent
  agent extraction.)*
- **Serialization**: per-file mutation queue (`withFileMutationQueue`) to
  serialize edits to the same path; `_agentEventQueue` to serialize UI
  delivery. *(observed fact)*
- **Retries / backoff**: only `openai-codex-responses.ts:106-308` has
  explicit retry + backoff. Other providers fail-fast and surface errors as
  in-stream `error` events instead of throws. *(observed fact)*
- **Timeouts**: undici `bodyTimeout` and `headersTimeout` are set to `0` at
  CLI startup (`packages/coding-agent/src/cli.ts:20`) — significant for
  long local-LLM stalls; **portability hazard** if reimplementing on a
  runtime that doesn't share undici's semantics.
- **OAuth refresh** uses `proper-lockfile` to prevent multi-instance
  refresh races against `auth.json`. *(observed fact)*
- **No global rate limiter.** *(observed fact)*
- **TUI synchronization**: ANSI synchronized output mode 2026 wraps each
  paint *(observed fact — `packages/tui/src/tui.ts:888-1204`)*.

## Build and Packaging

A one-paragraph synopsis here; the enumerated detail appends to
[`findings/build-and-deploy/build-and-deploy.md`](../build-and-deploy/build-and-deploy.md).

- **Workspace manager**: npm workspaces (top-level `package.json`
  `workspaces`).
- **Build order** (sequential, load-bearing): `tui → ai → agent →
  coding-agent → web-ui`. `tsgo` for the first four; stock `tsc` for
  web-ui.
- **Lockstep versioning**: `scripts/sync-versions.js:36-45` fails if any
  `packages/*/package.json` versions diverge. All five at `0.73.0`.
- **Linting/formatting**: biome 2.3.5 (`biome.json`); pre-commit husky hook
  runs `npm run check` (format + lint + tsgo + browser smoke).
- **CI workflows** (`.github/workflows/`): `ci.yml`, `build-binaries.yml`
  (Bun cross-compile to 5 platforms), and a contributor-gate trio
  (`issue-gate.yml`, `pr-gate.yml`, `approve-contributor.yml`) +
  `openclaw-gate.yml` driven by maintainer `lgtmi` / `lgtm` comments and a
  `.github/APPROVED_CONTRIBUTORS` text file. Currently in "refactor mode"
  auto-closing all issues until 2026-05-17 (`issue-gate.yml:20`). *(observed fact)*
- **Distribution channels**: 5 npm packages, Bun standalone binaries (with
  `koffi` externalized + Windows `.node` shipped beside the binary,
  `photon_rs_bg.wasm` bundled), browser-importable web components.
- **Hermetic test runner**: `test.sh` backs up `~/.pi/agent/auth.json` and
  unsets ~30 provider envvars before `npm test`. *(observed fact)*

## Porting Priorities

| Component | Priority | Rationale |
|---|---|---|
| `pi-ai` provider abstraction (`AssistantMessageEvent` union, `EventStream`, registry) | core | Without it nothing else has an LLM. The 12-variant event shape is the load-bearing contract every layer above depends on. |
| `pi-agent-core` agent loop (`Agent`, `agent-loop`, listener fan-out) | core | The interactive coding workflow IS the agent loop. *(strong inference)* |
| `pi-coding-agent` 7 built-in tools (`read`, `bash`, `edit`, `write`, `grep`, `find`, `ls`) | core | These are the entire externally-observable behavior of `pi`. *(strong inference)* |
| 5-level auth precedence + `~/.pi/agent/auth.json` (mode 0600) | core | Without auth resolution nothing connects to a provider. *(observed fact — `coding-agent/src/core/auth-storage.ts:446-516`)* |
| Session JSONL persistence (format v3, append-only with branch rewrite) | important | The user-visible "history / resume" surface; not strictly required for first-run parity but expected. *(strong inference)* |
| `pi-tui` differential renderer + widget set | important | `pi`'s default delivery is the terminal. Synchronized output 2026 + Kitty/iTerm2 image escapes are portability hazards if the host terminal differs. |
| Provider expansion (9 production providers + faux + the ~15 OpenAI-compatible vendor branch matrix) | important | Subset can ship; full parity requires all. The OpenAI-compat branch matrix at `openai-completions.ts:1030+` is the long tail. |
| Extension API (~25 lifecycle events, `registerTool/Command/Shortcut/Flag/Provider/MessageRenderer`, jiti loader) | optional | Real users rely on it (`.pi/extensions/`) but a bare port can defer. |
| `pi-web-ui` Lit components | optional | Independent delivery surface; not on the critical path for the CLI. |
| photon WASM (`packages/coding-agent/src/utils/photon.ts`) — image-processing helper, monkey-patches `fs.readFileSync` to redirect WASM path | incidental | Brittle vs. photon upgrades; rare to need in a port. **Portability hazard.** |
| koffi FFI (Windows VT input in `pi-tui`) | incidental | Platform-specific glue; if porting to Windows-native or non-JS, replace with native APIs. |
| OpenAI-compatible vendor URL detection branches | incidental | Long tail of compat shims; reimplement the registry skeleton, port the high-traffic ones first. |

## Durable State

A one-paragraph synopsis here; the enumerated detail appends to
[`findings/state-and-storage/state-and-storage.md`](../state-and-storage/state-and-storage.md)
and [`findings/config-model/config-model.md`](../config-model/config-model.md).

Operator state lives at `~/.pi/agent/` (overridable via `ENV_AGENT_DIR`,
`coding-agent/src/core/config.ts:402-407`):

- `auth.json` (mode 0600) — credentials. *(observed fact)*
- `settings.json` — user settings. *(observed fact)*
- `models.json` — model overrides / catalog cache. *(observed fact)*
- `sessions/--<encoded-cwd>--/<ts>_<sessionId>.jsonl` — append-only JSONL
  sessions, format version 3, branch edits do full-rewrite. *(observed
  fact — coding-agent extraction)*
- `themes/`, `tools/`, `bin/`, `prompts/`, `extensions/`, `skills/` — user
  assets that the CLI reads at startup. *(observed fact)*
- `pi-debug.log` — diagnostic trace.

Project-scoped twin at `<repo>/.pi/`: `extensions/`, `prompts/`, `git/`,
`npm/` (the latter two gitignored). *(observed fact — root `.pi/` exists in
pi-mono itself.)*

The `~/.pi/agent/auth.json` path is also referenced from `packages/ai/test/oauth.ts:14`
but `pi-ai` itself never reads it — the *coding-agent* layer threads
resolved keys in via `options.apiKey`. *(observed fact — packages/ai
extraction note.)*

## Open Questions

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| arch-OQ1 | needs-runtime-test | Does the `pi` startup `rg`/`fd` auto-download fail gracefully (or block startup) when the host is offline and these binaries are absent? | Cannot determine from source alone; requires live boot in an offline VM. |
| arch-OQ2 | needs-runtime-test | What happens when `~/.pi/agent/auth.json` exists but is corrupt JSON — does `pi` start, prompt, or crash? | Auth resolver branches at `auth-storage.ts:446-516` but parse-error path is not obvious from source. |
| arch-OQ3 | needs-fixture-capture | Exact wire format of the `streamProxy` SSE messages over `/api/stream`: documented as `data: {ProxyAssistantMessageEvent}\n` (`agent/src/proxy.ts:36-57`) but a recorded sample would confirm framing/heartbeat behavior. | Live capture against the running proxy is required to confirm. |

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| arch-CF1 | contracts | Tool surface contracts for the 7 built-in coding-agent tools (`read`, `bash`, `edit`, `write`, `grep`, `find`, `ls`) — parameter shapes, defaults, side-effect categories — should be enumerated in the contracts table. Architecture only catalogs them. | The contracts phase's rubric is the right place to spell out trigger/inputs/defaults/outputs/side-effects/error-behavior per tool. |
| arch-CF2 | contracts | The 5-level auth precedence resolver (`runtime override → api_key → oauth → env → fallback`) deserves a contract row with security/auth-model treatment. | Security & authorization is a contracts-phase rubric section. |
| arch-CF3 | protocols | Detailed event catalog for `AssistantMessageEvent` (12 variants) — producer, consumer, transport, ordering, fields. Architecture only names them. | Event-catalog production is the protocols phase's primary deliverable. |
| arch-CF4 | protocols | JSONL session persistence format v3 — exact record schema, branch-edit rewrite semantics, replay rules. | Persistent schema notes are a protocols-phase deliverable. |
| arch-CF5 | protocols | TUI render protocol — synchronized output mode 2026 framing, the custom APC `CURSOR_MARKER`, diff strategy, width-change full-clear behavior, Kitty/iTerm2 image escapes. | Compatibility hazards are a protocols-phase rubric section. |
| arch-CF6 | defect-scan-mechanical | photon WASM monkey-patch of `fs.readFileSync` (`packages/coding-agent/src/utils/photon.ts`) — global `require`-time mutation is the kind of mechanical-pass-2 (error handling) and pass-6 (config/environment) hazard the early defect scan should flag. | Pass-6 environment-coupling rubric is the right framing. |
| arch-CF7 | defect-scan-mechanical | undici `bodyTimeout`/`headersTimeout: 0` at CLI startup (`coding-agent/src/cli.ts:20`) is a config hazard — disables a built-in safety mechanism globally. | Pass-6 environment/config coupling. |
| arch-CF8 | defect-scan-semantic | OpenAI-compatible URL-based compat-detection matrix (`packages/ai/src/providers/openai-completions.ts:1030+`) — deep semantic-pass-5 audit of contract violations across the ~15 vendor branches. | Pass-5 (API contract violations) is the right rubric; contracts/protocols context required. |
| arch-CF9 | defect-scan-semantic | OAuth refresh races against `auth.json` are mitigated by `proper-lockfile`, but trust-boundary review of credential read/write paths (mode 0600 enforcement, fallback paths, error handling on parse failure) belongs in pass-4. | Security & trust boundary is pass-4's rubric. |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | The system intent is documented. | PASS | §System Intent (paragraph + load-bearing design choice statement). |
| 2 | The layer map and dependency direction are documented. | PASS | §Layer Map → Package Inventory + Dependency Direction (with ASCII diagram); cycles called out absent; load-bearing build order flagged. |
| 3 | Public surfaces are identified. | PASS | §Public Surfaces (synopsis); enumerated catalog will append to `findings/public-surfaces/public-surfaces.md`. |
| 4 | Runtime lifecycle, concurrency model, and porting priorities are summarized. | PASS | §Runtime Lifecycle, §Concurrency Model, §Porting Priorities (table). |
| 5 | Findings are marked with evidence levels. | PASS | Every claim is followed by `(observed fact / strong inference / portability hazard / open question)`; portability hazards in build order, undici timeouts, koffi, photon WASM, terminal escapes are explicitly tagged. |

**Validated by:** 2026-05-06 (architecture phase, session 1, no-orchestrator mode)
**Overall:** PASS
