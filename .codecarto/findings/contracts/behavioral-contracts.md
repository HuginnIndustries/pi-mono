# Behavioral Contracts — pi-mono

<!--
  Phase: contracts
  Pipeline: workflow/pipeline-full-with-deep-audit.yaml
  Source-of-detail: scratch/contracts-{cli,tools,extensions,pi-ai,pi-agent-core,tui-webui,config-auth}.md
  Evidence levels: observed fact / strong inference / portability hazard / open question
-->

## Surfaces Covered

| Surface | Audience | Owner |
|---|---|---|
| CLI (`pi`) | Operator | `packages/coding-agent` |
| Interactive TUI | End-user (terminal) | `packages/coding-agent` modes/interactive + `packages/tui` |
| Non-interactive print mode (`--print`) | Scripted / piped | `packages/coding-agent/src/modes/print-mode.ts` |
| Non-interactive RPC mode | Programmatic harness over stdio | `packages/coding-agent/src/modes/rpc-mode.ts` |
| Library — `@mariozechner/pi-ai` | Embedders (Node + browser) | `packages/ai` |
| Library — `@mariozechner/pi-agent-core` | Embedders (Node) | `packages/agent` |
| Library — `@mariozechner/pi-tui` | Embedders (Node terminal) | `packages/tui` |
| Library — `@mariozechner/pi-web-ui` | Embedders (browser, Lit) | `packages/web-ui` |
| Storage / file formats | All | `~/.pi/agent/` (auth.json, settings.json, models.json, sessions JSONL v3) + `<repo>/.pi/` |
| Extension API | Extension authors | `packages/coding-agent/src/extensions` |
| Slash commands / prompt templates | End-user (interactive) | `packages/coding-agent/src/core/{slash-commands,prompt-templates}.ts` |
| Browser web components | Browser embedders | `packages/web-ui` (~30 custom elements; flagship `<pi-chat-panel>`) |

## Feature Contracts

### CLI

#### F-CLI-1: Mode Dispatch

| Field | Value |
|---|---|
| Feature | `pi` mode selection |
| Trigger / input | argv parsing in `cli.ts` → `main.ts:98-109` `resolveAppMode` → `main.ts:673-726` dispatcher |
| Defaults | Interactive TUI when stdin is a TTY; auto-falls back to print mode when stdin is piped (`main.ts:634`) |
| Observable output | TTY paint (interactive); stdout text + exit code (print); JSON-RPC over stdio (rpc) |
| Side effects | All modes write session JSONL under `~/.pi/agent/sessions/--<encoded-cwd>--/` |
| Persisted state | Session JSONL v3 (append-only with full-rewrite for branch edits — see protocols phase, arch-CF4) |
| Error behavior | Subcommands (`install`, `update`, `config`) intercepted before `parseArgs` (`main.ts:431-437`); RPC mode rejects `@file` args (`main.ts:475-478`); `--fork` mutually exclusive with `--session`/`-c`/`-r`/`--no-session` (`main.ts:188-202`) |
| Retry / recovery | None at mode level |
| Owner | `packages/coding-agent/src/main.ts`, `cli.ts`, `cli/args.ts` |

*(observed fact)*

#### F-CLI-2: Session Resume / Fork

| Field | Value |
|---|---|
| Feature | `--session <id>`, `--fork <id>`, `-c` (continue), `-r` (resume), `--no-session` |
| Trigger / input | Session-id prefix |
| Defaults | New session (no flag); continue most-recent (`-c`) |
| Observable output | Either resumed transcript loaded into a new conversation, or fork creates a branch in the JSONL tree |
| Side effects | Reads `~/.pi/agent/sessions/`; resume creates a new JSONL file or appends to existing; fork rewrites |
| Persisted state | Session JSONL |
| Error behavior | **Defect 1.6**: ambiguous prefix silently picks `[0]` from match arrays (`main.ts:155-167` `resolveSessionPath`) — non-deterministic |
| Retry / recovery | None |
| Owner | `packages/coding-agent/src/main.ts`, `core/session-jsonl.ts` |

*(observed fact, known defect)*

#### F-CLI-3: Cancellation Contract

| Field | Value |
|---|---|
| Feature | Turn cancel / abort / exit |
| Trigger / input | Ctrl+C (double-tap, 500ms), Esc (overloaded), Ctrl+D, SIGTERM, SIGHUP |
| Defaults | Esc semantics: stream-abort / bash-abort / bash-mode-exit / double-Esc → `/tree` or `/fork` (`agent-session.ts:2355-2381`) |
| Observable output | TUI redraw; queued messages restored to editor on Esc-during-stream (`restoreQueuedMessagesToEditor({abort:true})` at `:2357`) |
| Side effects | 5 AbortControllers in `agent-session.ts:264-278` (`_compaction`, `_autoCompaction`, `_branchSummary`, `_retry`, `_bash`); main turn signal is `session.agent.signal` |
| Persisted state | Session JSONL state always survives cancel |
| Error behavior | SIGTERM → exit 143, SIGHUP → exit 129 (replicated across all three modes); detached child processes reaped via `killTrackedDetachedChildren()` |
| Retry / recovery | `session.abort()` aborts retry+agent |
| Owner | `agent-session.ts`, `modes/{interactive,print-mode,rpc-mode}.ts` |

*(observed fact)*

#### F-CLI-4: Slash Commands & Prompt Templates

| Field | Value |
|---|---|
| Feature | `/command` dispatch in interactive mode |
| Trigger / input | User types `/<name>` followed by optional args |
| Defaults | 20 builtins enumerated in `core/slash-commands.ts:18-40` |
| Observable output | Builtin behavior, or prompt-template expansion to a prepared prompt that is sent to the LLM |
| Side effects | None at dispatch; expansion may load files from prompt template's @file refs |
| Persisted state | None |
| Error behavior | Unknown command surfaces as user error in TUI |
| Retry / recovery | n/a |
| Owner | `packages/coding-agent/src/core/{slash-commands,prompt-templates}.ts` |

**Discovery order (observed fact)**: agent-dir `~/.pi/agent/prompts/` → `<cwd>/.pi/prompts/` → `--prompt-template` paths; non-recursive `.md` scan (`prompt-templates.ts:248-273`). Frontmatter is `---`-fenced YAML supporting `description` and `argument-hint` keys (`prompt-templates.ts:107,112-125`). Filename minus `.md` becomes the command name.

### Library — pi-ai

#### F-AI-1: `streamSimple()` and Variants

| Field | Value |
|---|---|
| Feature | LLM streaming entrypoint (4 variants: `stream` / `complete` / `streamSimple` / `completeSimple`) |
| Trigger / input | Caller passes `{ model, messages, tools?, signal?, ... }`; provider selected by `model.api: KnownApi` |
| Defaults | `EventStream` async-iterable; `AbortSignal` plumbed through every provider |
| Observable output | 12-variant `AssistantMessageEvent` stream (terminal `done` or `error`) |
| Side effects | HTTPS calls to provider endpoint; reads env vars + Vertex ADC for credentials; **no filesystem writes** (caller's job to thread `apiKey`) |
| Persisted state | None |
| Error behavior | **Inconsistent**: `streamSimple*` throws synchronously on missing API key for Codex/Completions, contradicting the documented "failures encoded in stream" contract (per scratch/contracts-pi-ai). Otherwise emits `error` event in stream. **Defect 1.5**: 3 providers' `mapStopReason` defaults throw on unrecognized SDK enum values, killing the entire stream. **Defect 1.1**: 7 providers destroy provider-mapped `errorMessage` via opaque `"An unknown error occurred"` literal. |
| Retry / recovery | **Codex only** (per dsm-RT7) — `openai-codex-responses.ts:106` regex `/rate.?limit\|overloaded\|service.?unavailable\|upstream.?connect\|connection.?refused/i`, MAX_RETRIES=3, BASE_DELAY=1000ms, exponential, **no jitter** (dsm-RT10). All other providers fail-fast. |
| Owner | `packages/ai/src/index.ts`, `src/api-registry.ts`, `src/providers/` |

*(observed fact, known defects 1.1, 1.5; resolves dsm-RT7)*

#### F-AI-2: Provider Selection

| Field | Value |
|---|---|
| Feature | Provider dispatch by `model.api: KnownApi` |
| Trigger / input | `getApiProvider(api)` in `src/api-registry.ts` |
| Defaults | 9 production providers + 1 `faux` test fixture; Bedrock lazy-loaded via `importNodeOnlyProvider` |
| Observable output | Provider's stream function |
| Side effects | First call to a Node-only provider triggers dynamic import |
| Persisted state | None |
| Error behavior | Throws `"No API provider registered for api: <id>"` on unknown id |
| Retry / recovery | n/a |
| Owner | `packages/ai/src/api-registry.ts`, `src/providers/register-builtins.ts:342-403` |

The OpenAI-compat URL-substring matrix at `openai-completions.ts:1029-1088` recognizes ~10 vendor families (z.ai, Moonshot, Cloudflare ×2, Cerebras, Grok, Chutes, DeepSeek, Opencode, OpenRouter). Deep audit deferred to defect-scan-semantic Pass 5 per arch-CF8. *(observed fact, deferred for full coverage)*

### Library — pi-agent-core

#### F-AGENT-1: `Agent` Class API

| Field | Value |
|---|---|
| Feature | In-process agent loop driver |
| Trigger / input | `new Agent({ streamFn, ... })`, then `agent.prompt(message)` or `agent.continue()` (note: methods are `prompt()` / `continue()` — **not** `run()`) |
| Defaults | Single `AbortController` per run; exposed as `agent.signal` |
| Observable output | Promise resolving to `RunResult`; events delivered to listeners registered via `agent.subscribe(callback)` (note: not `addListener`/`removeListener`) |
| Side effects | None within pi-agent-core (`fs` is not imported anywhere in the package's `src/`); side effects come from the `streamFn` provider call and listener-side actions |
| Persisted state | **None.** `sessionId` is opaque caller-provided string forwarded to `streamFn` only; `agent.reset()` is in-memory; persistence is the caller's job. |
| Error behavior | **Defect 1.13/2.3**: listener fan-out is sequential `for...of await` at `agent.ts:539-541` with no try/catch — a throw aborts subsequent listeners AND propagates to `handleRunFailure`. **Defect 1.21**: `handleRunFailure` synthesizes empty assistant message at `:475` even when partial assistant message was already streamed (transcript ends with two assistant messages). **Defect 1.19**: `streamAssistantResponse` silently swallows the entire stream if `start` event is missing (`agent-loop.ts:308-316`). |
| Retry / recovery | None at `Agent` level; caller controls via `streamFn` retry policy or higher-level controllers |
| Owner | `packages/agent/src/agent.ts:158`, `agent-loop.ts` |

*(observed fact, known defects)*

#### F-AGENT-2: `streamProxy` SSE Client (NOT a Server)

| Field | Value |
|---|---|
| Feature | HTTP SSE client to a remote agent execution endpoint |
| Trigger / input | `streamProxy({ proxyUrl, ... })` (`proxy.ts:116`); POST to `${proxyUrl}/api/stream` (`:152`) |
| Defaults | Standard fetch + Server-Sent Events |
| Observable output | Reconstructs an `AssistantMessageEvent`-shaped stream from `data: {ProxyAssistantMessageEvent}\n` lines (`:195-205`) — the proxy server strips `partial` to save bandwidth and the client reconstructs |
| Side effects | HTTPS POST to caller-supplied URL |
| Persisted state | None |
| Error behavior | **Defect 1.22**: throws on out-of-order events at `proxy.ts:261/275/293/307/333` — one misframe terminates the turn with no recovery. No heartbeat / keepalive parsing (arch-OQ3 unresolved). **Defect 1.14**: `processProxyEvent` shallow-clones at `:325` leaks `partialJson` into supposedly-finished tool calls in partial snapshots. |
| Retry / recovery | None |
| Owner | `packages/agent/src/proxy.ts:116` |

**Important correction to architecture phase**: `streamProxy` is a **CLIENT only**. The package does not ship a listening server. *(observed fact)*

#### F-AGENT-3: Listener / Hook Contract

`agent.subscribe(callback)` registers a listener. **Listener contract** (per `agent.ts:539-541`, observed fact):
- Listeners run in registration order, sequentially via `for...of await`.
- A throwing listener **stops subsequent listeners** and **propagates the error** to `handleRunFailure`.
- Listener removal during iteration: open question (`dsm-OQ4` — needs maintainer decision).

### Library — pi-tui

#### F-TUI-1: `Component` Interface + `TUI` Driver

| Field | Value |
|---|---|
| Feature | Differential terminal renderer + widget API |
| Trigger / input | `new TUI(rootComponent)`, then `.start()` / `.stop()` |
| Defaults | Synchronized output mode 2026 wraps each paint; bracketed paste 2004 enabled; Kitty / iTerm2 image escapes opportunistic |
| Observable output | Diff-only line writes to stdout; SIGWINCH self-kick on width change |
| Side effects | Mutates terminal (alt-screen NOT used); spawns `ProcessTerminal` for raw stdin; uses `koffi` FFI for Windows VT input |
| Persisted state | In-memory `previousLines` buffer for diff |
| Error behavior | **Defect 1.15**: throws on width-overflow at `tui.ts:1100-1131`, killing process from `process.nextTick`/`setTimeout` callback (contradicts the "always truncate" comment in `compositeLineAt`). **Defect 2.11**: no handler for stdout-write failures (EPIPE / SSH disconnect mid-paint) — agent continues running headless after disconnect. |
| Retry / recovery | None |
| Owner | `packages/tui/src/tui.ts`, `terminal.ts`, `stdin-buffer.ts` |

*(observed fact, portability hazards: terminal-protocol assumptions throughout)*

#### F-TUI-2: Paste Buffering

`StdinBuffer` (`stdin-buffer.ts`) assembles partial escape sequences and bracketed-paste content. **Defect 1.18**: paste-mode buffer has no timeout-armed flush; lost `\x1b[201~` leaks all subsequent stdin into `pasteBuffer` forever. *(observed fact, known defect)*

### Library — pi-web-ui

#### F-WEB-1: `<pi-chat-panel>` Custom Element

| Field | Value |
|---|---|
| Feature | Lit web component packaging the chat UI for browser consumers |
| Trigger / input | Embed `<pi-chat-panel>`; configure via `setAgent(agent, config)` (no HTML attributes — `ChatPanel` uses only `@state()`, no `@property()` decorators) |
| Defaults | Reactive over `@state()`; rAF-batched streaming display |
| Observable output | DOM updates, no custom events emitted from `pi-chat-panel` itself |
| Side effects | `localStorage`, `IndexedDB`, `navigator.clipboard`, `import.meta.url` worker URLs (pdfjs), `fetch` (model discovery, attachment, document extraction); CORS-proxy URL rewriter (`proxy-utils.ts`) |
| Persisted state | Browser `localStorage` + `IndexedDB` |
| Error behavior | **Defect 1.16**: `JSON.parse(JSON.stringify(...))` clone in `StreamingMessageContainer.ts:43-60` loses `undefined`, `BigInt`, `Map`, `Set`, `Date`, `Function`, cycles. **Defect 1.17**: `ExpandableSection.connectedCallback` permanently destroys children on re-attachment (amplified by `MessageList.ts:36-91` positional keys defeating Lit reconciliation). |
| Retry / recovery | n/a |
| Owner | `packages/web-ui/src/ChatPanel.ts:17` + `src/index.ts` (~30 element tags) |

*(observed fact, known defects)*

**Backend coupling**: pi-web-ui has zero backend coupling beyond CORS-proxy URL rewrite + direct fetch — LLM streaming delegates to `pi-ai`'s `streamSimple`. **Consumers must wire LLM auth and `Agent` themselves.** *(observed fact)*

### Built-in Tools (resolves arch-CF1)

The `pi` coding-agent exposes exactly **7 built-in tools** to the LLM. Canonical registration in `packages/coding-agent/src/core/tools/index.ts:83`. Per-tool detail in `scratch/contracts-tools.md`.

| Tool | Source (current names — note source-filename drift from mech-defects citations) | Side-effect category | Critical contract notes |
|---|---|---|---|
| `read` | `core/tools/read.ts` | filesystem read | Standard read; no buffering hazards. |
| `bash` | `core/tools/bash.ts` (mech-defects cite legacy `bash-executor.ts`) | shell exec + filesystem (temp pipes) | **Exit code defect 1.12+2.5**: `code: code ?? 0` reports killed processes (`code === null`) as exit 0. Spawn errors discarded as `_err`. **Defect 1.9**: `tempFileStream.end()` not awaited before resolving — consumers can read truncated file. **No `'error'` listener on tempFileStream** (defect 2.6) — disk-full mid-bash escalates to uncaughtException. |
| `edit` | `core/tools/edit.ts` (mech cites `edit-tool.ts`) | filesystem write | **Defect 2.2 critical (data loss)**: `ops.writeFile(absolutePath, ...)` direct, **no temp+rename**. Crash mid-write destroys user code. Serialization via `withFileMutationQueue` (`file-mutation-queue.ts:19`) per-realpath (`realpathSync.native` symlink resolution). |
| `write` | `core/tools/write.ts` (mech cites `write-tool.ts`) | filesystem write | Same defect 2.2 as `edit` — no temp+rename. |
| `grep` | `core/tools/grep.ts` | shell exec (rg subprocess) | Auto-downloads `rg` binary on first use via `tools-manager.ts` — **no checksum/sig verification** (dsm-RT9 supply-chain hazard, deferred to Pass 4). |
| `find` | `core/tools/find.ts` | shell exec (fd subprocess) | Auto-downloads `fd` binary on first use; same supply-chain hazard. |
| `ls` | `core/tools/ls.ts` | filesystem read | Standard. |

**Tool-level contract aggregates (observed fact unless noted):**
- **edit/write data-loss surface**: a port of `pi` MUST implement temp-write+rename or accept this contract.
- **bash exit-code contract**: callers cannot reliably distinguish "tool succeeded" from "tool was killed."
- **grep/find first-run contract**: network side-effect on first use; offline behavior open question (arch-OQ1).
- **withFileMutationQueue serialization**: parallel calls to the same file are queued, not parallelized.
- **Source-filename drift**: `mechanical-defects.md` cited legacy filenames (`bash-executor.ts`, `edit-tool.ts`, `write-tool.ts`); current source uses `bash.ts`, `edit.ts`, `write.ts`. Defects still apply.

### Extension API (resolves dsm-RT5)

#### F-EXT-1: Loader

| Field | Value |
|---|---|
| Feature | Project + operator extension loading |
| Trigger / input | Discovered at session start; `<cwd>/.pi/extensions/` first, `~/.pi/agent/extensions/` second; explicit paths last. **Dedup is by absolute resolved path** (first-seen wins) — same name in different dirs both load. |
| Defaults | `@mariozechner/jiti` fork with `moduleCache: false` |
| Observable output | Extensions registered against the runtime; ready for hook firing |
| Side effects | `import.meta.url` detection: in Bun-binary mode (`$bunfs`/`~BUN`) uses `virtualModules` + `tryNative: false` for static bundle of `typebox`, `@mariozechner/pi-*`, `@sinclair/typebox*`. In Node/dev mode, uses path aliases. |
| Persisted state | None (caches disabled) |
| Error behavior | Per-event `try/catch` for all `emitXxx` paths route handler errors to `emitError` listeners — **EXCEPT `emitToolCall`** at `runner.ts:806-827` which does not catch. Throw propagates out, caught at `agent-session.ts:394-399`, re-thrown as `Extension failed, blocking execution`. The tool is blocked AND subsequent extension `tool_call` handlers are skipped. |
| Retry / recovery | User-driven via `ctx.reload()` (`agent-session.ts:2383`); no fs watcher |
| Owner | `packages/coding-agent/src/extensions/loader.ts`, `runner.ts` |

*(observed fact)*

#### F-EXT-2: Lifecycle Events

22 distinct events (corrected from architecture phase's "~25"). Catalog deferred to **protocols phase** (per arch-CF3). Source-of-truth: `types.ts:950-972` (the `ExtensionEvent` union) and the typed `on()` overloads at `types.ts:1089-1126`. Notably the architecture phase's "before_provider_response" / "after_provider_request" do **not** exist — only `before_provider_request` and `after_provider_response`. *(observed fact, correction)*

#### F-EXT-3: Stale Runtime + Reload Contract

`runtime.invalidate()` (`loader.ts:164`) marks captured `pi`/`ctx` stale after `newSession`/`fork`/`switchSession`/`reload`. Every method calls `assertActive` and throws a long explanatory error message if called after invalidation. Hot reload sequence: emit `session_shutdown(reload)` → reload settings/providers/resources → re-import extensions fresh (`moduleCache: false` ensures no stale closures) → emit `session_start(reload)`. *(observed fact, resolves dsm-RT5)*

#### F-EXT-4: Drift Risks (dsm-RT5 audit)

- Extension API is **implicit** — no separate SDK package; authors discover from `.pi/extensions/` examples. *(portability hazard)*
- The 22 lifecycle events are duplicated across three locations (union type, `on` overloads, runner emit methods) — drift surface. *(open question — needs documented single-source-of-truth)*
- `emitToolCall`'s unguarded behavior is undocumented at the API surface. *(observed fact)*

## High-Value Behaviors

### Cancellation & Abort

- **5 AbortControllers** in `AgentSession` (`agent-session.ts:264-278`): `_compaction`, `_autoCompaction`, `_branchSummary`, `_retry`, `_bash`. Main turn signal is `session.agent.signal`.
- **Esc semantics** are overloaded (interactive mode): stream-abort / bash-abort / bash-mode-exit / double-Esc → `/tree` or `/fork`.
- **Ctrl+C double-tap** within 500ms exits.
- **Esc-during-stream** preserves queued messages by restoring them to the editor.
- **SIGTERM** → exit 143; **SIGHUP** → exit 129 (replicated across all three modes); detached child processes reaped via `killTrackedDetachedChildren()`.
- **Library-level**: `pi-agent-core` exposes a single `agent.signal`. Caller plumbs further AbortControllers if needed.

### Streaming & Partial Output

- 12-variant `AssistantMessageEvent` union (catalog deferred to protocols per arch-CF3).
- Stream guarantees: ordered; must terminate with `done` or `error`; `EventStream.push` is no-op after first terminal event.
- **Tool-call coalescing** keyed by `(streamIndex || id)`. **Defect 1.3**: openai-completions splits a single tool call into N fragments when a vendor sends continuation deltas without both `index` and `id`.
- Abort yields `error` event with `reason: "aborted"`; does **not** reject `result()`.
- Mid-stream rejection in `forwardStream` (defect 2.4) is unhandled; consumer hangs (`target.end()` never called, `result()` deadlocks).

### Tool Execution & Validation

- 7 built-in tools (above table).
- **withFileMutationQueue** per-realpath serialization for filesystem-mutating tools.
- **Critical data-loss contract** (P2.1, P2.2): session JSONL writes and edit/write tool writes are non-atomic. Mid-write crash truncates / destroys.

### Compaction & Summarization

- Auto-compaction has its own AbortController; `before_compact` extension hook fires before context compaction; branch-summary AbortController distinct.
- **Defect 1.7**: `findValidCutPoints` (`compaction.ts:299-336`) has duplicated handling and unreachable cases.

### Persistence & Resume

- Session JSONL v3 under `~/.pi/agent/sessions/--<encoded-cwd>--/<ts>_<sessionId>.jsonl`. Append-only with full-rewrite for branch edits. (Schema → protocols, arch-CF4.)
- **Defect 1.6**: ambiguous session-id prefix silently picks `[0]`.
- **Defect 2.7**: malformed JSONL lines silently skipped on load.

## Security and Authorization

### Authentication (resolves arch-CF2)

5-level precedence resolved in `packages/coding-agent/src/core/auth-storage.ts:446-516`:

| Level | Source | Read mechanism | Failure mode |
|---|---|---|---|
| 1 (highest) | Runtime override | Explicit `apiKey` to `streamSimple()` | Caller's responsibility |
| 2 | `auth.json` `api_key` per-provider | proper-lockfile read | Corrupt JSON → `loadError` set, all reads blocked (partial answer to arch-OQ2) |
| 3 | `auth.json` `oauth` | proper-lockfile read; refresh via `proper-lockfile` to prevent multi-instance race | Refresh failure → stale token, eventual 401 |
| 4 | Env var | `packages/ai/src/utils/env-api-keys.ts:91-209` per-provider table; `<authenticated>` sentinel for Vertex/Bedrock ambient credentials | Absent → fall through to level 5 |
| 5 (lowest) | Fallback resolver | Provider-specific (Vertex ADC, Bedrock) | Provider-specific |

**Important note**: `pi-ai` itself only reads env vars + Vertex ADC. The `auth.json` path is owned by `coding-agent`'s auth-storage layer which threads the resolved key in via `options.apiKey`. *(observed fact)*

### Authorization

- pi-mono itself does **no per-user authorization** beyond OS-level filesystem permissions on `~/.pi/agent/`. *(strong inference)*
- **`streamProxy` is a client, not a server** — there is no `/api/stream` server in pi-mono. Any HTTP-server side has no authn/authz contract from this package. *(observed fact, correction to arch)*
- Extension API: extensions run with full process privileges. `emitToolCall` failure blocks the tool but does not isolate the extension. *(observed fact)*

### Trust Boundaries (resolves arch-CF9 framing; full security treatment in defect-scan-semantic Pass 4)

| Trusted | Untrusted |
|---|---|
| `~/.pi/agent/` content | LLM responses |
| Env vars set by user | Downloaded `rg`/`fd` binaries (no checksum/sig — dsm-RT9) |
| `<repo>/.pi/` if user trusts the repo | Shell command output |
| User-installed extensions | LLM-suggested shell commands (subject to user confirmation per slash-command UX) |

**Trust-boundary violation flagged**: `resolveConfigValue` `!`-prefix shell-exec when `models.json` is project-scoped from an untrusted repo. **Covert RCE channel** if a user opens a hostile repo. (Pre-routed to Pass 4 as `dsm-RT1`; mechanical row P6.4 documents the mechanism.) *(observed fact)*

### Secret Management

- Storage: `auth.json` mode 0600 enforced **on every write** (creation, sync, async — confirmed by Pass 6, closes part of arch-CF9). Windows ACL behavior is **open question**.
- Rotation: OAuth refresh automatic with `proper-lockfile`-protected single-flight; API keys are user-managed.
- Logging: `pi-debug.log` — secret-leak audit is open question.

### Session Lifecycle

- Sessions persist forever in `~/.pi/agent/sessions/`. No expiration.
- No invalidation surface beyond filesystem deletion.

## Configuration Model

See [`findings/config-model/config-model.md`](../config-model/config-model.md) for full enumerated detail (this phase appends an authoritative section there).

**Summary** (load-bearing):

- **5-level precedence**: CLI flags > project `<repo>/.pi/` > operator `~/.pi/agent/` > env vars > compiled defaults.
- **State anchor override**: `PI_CODING_AGENT_DIR` (note: the architecture phase had `ENV_AGENT_DIR` — that is the *constant name* in source; the resolved env var on a default `pi` build is `PI_CODING_AGENT_DIR`, computed from `${APP_NAME.toUpperCase()}_CODING_AGENT_DIR` at `packages/coding-agent/src/config.ts:374-380`. A fork rebrands every `PI_*` var.) *(observed fact, correction)*
- **File schemas** (load-bearing):
  - `auth.json`: per-provider `api_key` and/or `oauth` blob; mode 0600 enforced on every write.
  - `settings.json`: user settings; partial schema validation (full schema audit deferred to defect-scan-semantic).
  - `models.json`: model overrides + custom headers; **headers values can be `!`-prefix shell strings** (P6.4 / dsm-RT1).
- **Migrations contract** (per P6.5): `runMigrations` runs **unconditionally on every startup** (auth/sessions/binaries/keybindings/commands→prompts) with bare-`catch {}` failure handling. Every `pi` startup performs schema migrations regardless of source-version semantics. *(observed fact, hazard)*
- **Validation timing**: corrupt `auth.json` → process starts, sets `loadError`, blocks credential reads until user fixes file (per Pass 6 partial answer to arch-OQ2). *(observed fact)*

## Doc/Test Conflicts

| Conflict | Source-of-truth | Notes |
|---|---|---|
| Architecture phase named methods `addListener`/`removeListener`/`run()` | Source: `agent.subscribe(callback)` / `prompt()` / `continue()` (`packages/agent/src/agent.ts`) | Source wins; arch corrected. |
| Architecture phase named events `before_provider_response` / `after_provider_request` | Source: only `before_provider_request` / `after_provider_response` exist (`types.ts:950-972`) | Source wins; arch corrected. |
| Architecture phase characterized `streamProxy` as a server with `/api/stream` endpoint | Source: `streamProxy` is a CLIENT only; pi-mono ships no listening server | Source wins; arch corrected. |
| Architecture phase claimed ~25 lifecycle events | Source: 22 distinct events from emit sites + ExtensionEvent union | Source wins; minor correction. |
| Architecture phase env var named `ENV_AGENT_DIR` | Source: `ENV_AGENT_DIR` is the constant name; resolved env var is `PI_CODING_AGENT_DIR` (templated `${APP_NAME.toUpperCase()}_CODING_AGENT_DIR`) | Both names usable, but the user-facing one is `PI_CODING_AGENT_DIR`. |
| Mechanical-defects.md cited tool source filenames `bash-executor.ts`, `edit-tool.ts`, `write-tool.ts` | Source: `bash.ts`, `edit.ts`, `write.ts` | Source wins; line numbers from mech-defects do not match current source but defects still apply. |
| pi-web-ui has **zero tests**; pi-agent-core's mech findings 1.13/1.19/1.21/1.22 have **no test coverage** | n/a | Documented contract gaps — tests would catch the listed defects. |
| `streamSimple*` synchronous throw on missing API key for Codex/Completions | pi-ai's documented "failures encoded in stream" contract | **Conflict**: documented behavior says all errors flow through the stream; observed behavior is sometimes a synchronous throw. Open question whether this is intentional API surface for pre-flight validation, or a defect. |

## Black-Box Acceptance List

| # | Scenario | Precondition | Action | Expected Outcome |
|---|---|---|---|---|
| 1 | Interactive session start | TTY stdin, no `~/.pi/agent/auth.json` (or absent provider key) | Run `pi` | Process starts; prompts/auth flow appears; no crash |
| 2 | Print mode auto-fallback | stdin is piped (not TTY) | `echo "hello" | pi` | Auto-falls back to print mode (no TUI) |
| 3 | Corrupt auth.json startup | `auth.json` exists but invalid JSON | Run `pi` | Process starts; `loadError` set; credential reads blocked until file fixed |
| 4 | Session resume | Existing session id `abc123` | `pi --session abc123` | Transcript loaded; new turn appended (or full-rewrite on branch) |
| 5 | Ambiguous session prefix | Two sessions match prefix `ab` | `pi --session ab` | **Currently**: silently picks `[0]` (defect 1.6). Spec requires error. |
| 6 | Esc-during-stream | Active LLM stream, queued messages | Press Esc | Stream aborted; queued messages restored to editor |
| 7 | Double-Esc | After single Esc | Press Esc again within window | Triggers `/tree` or `/fork` |
| 8 | SIGTERM exit | Running session | `kill -TERM $pid` | Exit 143; child processes reaped |
| 9 | edit-tool data-loss surface | File `foo.ts` exists | LLM tool-call `edit` writes; process killed mid-write | **Currently**: file may be truncated/destroyed (defect 2.2). Spec requires temp+rename. |
| 10 | bash-tool killed exit code | Subprocess receives SIGKILL | Run a bash tool that gets killed | **Currently**: returns exit 0 (defect 1.12+2.5). Spec requires non-success indicator. |
| 11 | OAuth refresh under contention | 2 `pi` instances; stale OAuth token | Both call provider | proper-lockfile serializes refresh; one refreshes, other reads new token |
| 12 | models.json `!`-prefix shell | `<repo>/.pi/models.json` `headers` has `!`-prefix string | Run `pi` in that repo | Shell command executes during header resolution. **Trust-boundary violation; Pass 4 finding dsm-RT1.** |
| 13 | Extension throws in `tool_call` hook | An extension hook throws | LLM invokes a tool | Tool blocked; subsequent `tool_call` handlers skipped (per F-EXT-1 + defect 1.13) |
| 14 | Project-scoped extension | `<repo>/.pi/extensions/foo.ts` and `~/.pi/agent/extensions/foo.ts` exist | Run `pi` | Both load (dedup by absolute path, not name) |
| 15 | streamProxy out-of-order event | proxy server sends events out of order | Client reading | Throws; turn terminates (defect 1.22) |
| 16 | Width-overflow render | TUI line exceeds terminal width | Render | **Currently**: throws; process killed (defect 1.15). Spec requires truncate. |
| 17 | Slash command discovery | `/foo.md` in `~/.pi/agent/prompts/` AND `<cwd>/.pi/prompts/` | `/foo` | First-found wins per discovery order (agent-dir → cwd-dir → flag paths) |
| 18 | `streamSimple` missing API key (Codex/Completions) | No env var, no auth.json, no runtime override | `streamSimple({ model: codex, ... })` | **Currently**: synchronous throw. **Open question** whether this is the intended pre-flight contract or should emit `error` event. |

## Open Questions

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| con-OQ1 | needs-runtime-test | `pi-debug.log` content audit — does it leak secrets / OAuth tokens? | Live runtime test required. |
| con-OQ2 | needs-runtime-test | `auth.json` mode 0600 on Windows — does NTFS ACL behavior match POSIX intent? | Windows test required. |
| con-OQ3 | needs-spec-ruling | Is `streamSimple*` synchronous throw on missing API key intentional pre-flight validation, or a contract drift from "failures encoded in stream"? | Maintainer ruling. |
| con-OQ4 | needs-maintainer-decision | Listener removal during fan-out iteration in `agent.ts:539-541` — safe or undefined? | Maintainer policy. |
| con-OQ5 | needs-spec-ruling | Extension API single-source-of-truth — should the 22 events be exported from one location instead of three (union type, `on` overloads, runner emit methods)? | Drift surface needing maintainer call. |

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| con-CF1 | protocols | 12-variant `AssistantMessageEvent` field-level schema (the contract names variants; protocols owns the wire-format detail). Subsumes part of arch-CF3. | Event-catalog production is protocols phase rubric. |
| con-CF2 | protocols | 22-event extension catalog with handler-input-shape + return-semantics per event. | Event catalog rubric. |
| con-CF3 | protocols | Session JSONL v3 record schema (architecture had this as arch-CF4; contracts confirms it but defers schema details). | Persistent format rubric. |
| con-CF4 | protocols | streamProxy SSE wire framing (heartbeat/keepalive open question; `data: <json>\n` vs canonical SSE `\n\n`). | Protocols phase wire-format rubric. |
| con-CF5 | defect-scan-semantic | dsm-RT1 (covert-RCE via `models.json` `!`-prefix shell) — Pass 4 framing now formally declared in contracts. | Already routed; reaffirmed here. |
| con-CF6 | defect-scan-semantic | con-OQ3 — `streamSimple*` sync-throw vs stream-error contract drift; needs Pass 5 framing. | Pass 5 (contract violations) rubric. |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | User-facing surfaces are split by surface type. | PASS | §Surfaces Covered table partitions 12 surface types; §Feature Contracts groups contracts by surface. |
| 2 | Feature contracts record trigger, defaults, outputs, side effects, persisted state, error behavior, and recovery behavior. | PASS | All 8 feature-contract tables (F-CLI-1..4, F-AI-1..2, F-AGENT-1..3, F-TUI-1..2, F-WEB-1, F-EXT-1..4) have all 8 fields populated. The 7-tool table has all 5 columns + critical-contracts summary. |
| 3 | Security and authorization model is documented (if applicable). | PASS | §Security and Authorization with sub-sections Authentication (5-level precedence resolves arch-CF2), Authorization, Trust Boundaries (resolves arch-CF9 framing), Secret Management, Session Lifecycle. |
| 4 | Contract ownership is mapped back to a layer or package. | PASS | Every contract row has `Owner` field citing `packages/<pkg>/...` paths. Surfaces table maps each surface to its owning package. |
| 5 | A black-box acceptance list is included. | PASS | 18 acceptance scenarios spanning interactive/print/RPC modes, session resume, cancellation, data-loss surfaces, security boundaries, extension API, streamProxy, slash-command discovery. |
| 6 | Findings are marked with evidence levels. | PASS | Every load-bearing claim is followed by `(observed fact / strong inference / portability hazard / open question)`. Defect cross-references cite mechanical-defects rows (1.x, 2.x, 6.x). |

**Validated by:** 2026-05-06 (contracts phase, session 1, no-orchestrator mode)
**Overall:** PASS

(arch-CF1 resolved as the 7-tool catalog. arch-CF2 resolved as the 5-level auth precedence + Authentication section. dsm-RT5 resolved as F-EXT-1..4. dsm-RT7 resolved as F-AI-1 retry contract. arch-CF9 partial-resolved — full Pass 4 trust-boundary deferred. Architecture phase's incorrect characterizations of `streamProxy` as a server, the 25-event count, and the `addListener`/`run` method names corrected here in §Doc/Test Conflicts.)
