# Protocols and State — pi-mono

<!--
  Phase: protocols
  Pipeline: workflow/pipeline-full-with-deep-audit.yaml
  Source-of-detail: scratch/proto-{events,sessions,tui,proxy,extension-events}.md
  Evidence: observed fact / strong inference / portability hazard / open question
-->

## Boundaries Identified

| # | Boundary | Carrier | Direction | Owner |
|---|---|---|---|---|
| B1 | Provider HTTPS API → pi-ai → caller | HTTPS / SSE-ish | inbound | `packages/ai/src/providers/*` |
| B2 | pi-ai stream union → pi-agent-core | In-process async iterable (`EventStream`) | inbound | `packages/ai/src/utils/event-stream.ts:4-66` |
| B3 | pi-agent-core ↔ caller (listeners + run results) | Function calls + Promises + AbortSignal | bidirectional | `packages/agent/src/agent.ts` |
| B4 | streamProxy CLIENT ↔ remote agent server | HTTP POST + custom `data: <json>\n` SSE | bidirectional | `packages/agent/src/proxy.ts` |
| B5 | coding-agent tools ↔ subprocesses | spawn + stdio + signals | bidirectional | `packages/coding-agent/src/core/tools/{bash,grep,find}.ts`, `core/exec.ts` |
| B6 | coding-agent ↔ session JSONL v3 | Filesystem append + full-rewrite for branch | bidirectional | `packages/coding-agent/src/core/session-manager.ts` |
| B7 | coding-agent ↔ `~/.pi/agent/auth.json` | Filesystem JSON with proper-lockfile | bidirectional | `packages/coding-agent/src/core/auth-storage.ts` |
| B8 | Extension runtime ↔ host coding-agent | In-process function calls + lifecycle event emission | bidirectional | `packages/coding-agent/src/core/extensions/{runner,loader,types}.ts` |
| B9 | pi-tui ↔ terminal | ANSI/VT escapes on stdout + raw stdin | bidirectional | `packages/tui/src/{tui,terminal,stdin-buffer}.ts` |
| B10 | pi-web-ui ↔ browser DOM/IndexedDB/fetch | Lit reactive + browser APIs | bidirectional | `packages/web-ui/src/*` |
| B11 | OAuth refresh ↔ provider | HTTPS + `proper-lockfile` for multi-instance single-flight | bidirectional | `packages/coding-agent/src/core/auth-storage.ts` |

## Event Catalog

### EC1 — `AssistantMessageEvent` (12-variant union — resolves arch-CF3, con-CF1)

**Source-of-truth**: `packages/ai/src/types.ts:269-281`. Per-variant detail in `scratch/proto-events.md`.

| Field | Value |
|---|---|
| Producer | Each provider in `packages/ai/src/providers/*` (Anthropic, OpenAI Responses, OpenAI Completions, Azure OpenAI Responses, OpenAI Codex Responses, Google Generative AI, Google Vertex, Mistral Conversations, AWS Bedrock Converse Stream, plus `faux` test fixture) |
| Consumer | pi-agent-core's `streamAssistantResponse` (`agent-loop.ts`); pi-coding-agent's session persister; pi-web-ui's `StreamingMessageContainer`; any embedder |
| Transport / carrier | In-process async iterable via `EventStream` (`packages/ai/src/utils/event-stream.ts:4-66`); over-the-wire variant via streamProxy (see EC4) |
| Ordering guarantees | FIFO (`event-stream.ts:20-35`); push-after-terminal is no-op (`:20-22`) |
| Required fields | Discriminator `type ∈ {start, text_start, text_delta, text_end, thinking_start, thinking_delta, thinking_end, toolcall_start, toolcall_delta, toolcall_end, done, error}` |
| Optional fields | Per-variant: text deltas carry `text`; tool-call events carry `id`/`streamIndex`/`name`/`arguments`; `done` carries `finishReason ∈ {stop, length, toolUse}` and usage stats; `error` carries `error.error` field (NOT `error.message`) and optional `reason` (e.g., `"aborted"`) |
| Identifiers / timestamps | Tool-call coalescing key: `(streamIndex \|\| id)` per `openai-completions.ts:303-307`; Anthropic uses server-supplied `index` per `anthropic.ts:551,575,596`; no timestamps on individual events (consumers stamp at receive time if needed) |
| Error cases | `error` event is terminal. **Defect 1.1** (7-provider opaque-literal destruction); **Defect 1.5** (3 providers throw on unrecognized `mapStopReason`); **Defect 1.2** (Google force-toolUse); **Defect 1.3** (openai-completions tool-call fragmentation when continuation deltas lack both index+id); **streamSimple* sometimes throws synchronously** instead of emitting `error` event — contract drift (con-OQ3, routed Pass 5 as con-CF6). |
| Restart / resume behavior | Stream is single-shot per call; no resume protocol. Caller's `AbortSignal` aborts; emits final `error{reason:"aborted"}`; `result()` resolves on `done`/`error`, never rejects. **`end()` without a terminal event hangs `result()` forever** (mech medium-tally). |

*(observed fact — variants verified against `types.ts` and provider source)*

### EC2 — Tool-Call Events Sub-Protocol

Within EC1, the `toolcall_start` / `toolcall_delta` / `toolcall_end` triple is a sub-protocol consumers must handle:

| Stage | Producer behavior | Consumer obligation |
|---|---|---|
| `toolcall_start` | Emits when a new tool call begins; carries `id`, `name`, `streamIndex` | Allocate slot keyed by `(streamIndex \|\| id)` |
| `toolcall_delta` | Emits incremental `partialJson` for arguments | Append to slot; **defect 1.3**: openai-completions splits a single tool-call into N fragments when continuation deltas lack both `index`+`id` |
| `toolcall_end` | Emits when tool call complete | Parse accumulated `partialJson` as JSON; **streamProxy defect 1.14**: `partial.content[]` shallow-clone retains `partialJson` after `delete` on live ref |

*(observed fact)*

### EC3 — `ProxyAssistantMessageEvent` (streamProxy SSE — resolves con-CF4)

| Field | Value |
|---|---|
| Producer | streamProxy server (NOT shipped by pi-mono — caller's responsibility); `ProxyAssistantMessageEvent` union at `packages/agent/src/proxy.ts:36-57` |
| Consumer | `streamProxy({ proxyUrl, ... })` client at `proxy.ts:116`; reconstructs `AssistantMessageEvent` |
| Transport / carrier | HTTP `POST ${proxyUrl}/api/stream` with `Authorization: Bearer ${authToken}`; body `{model, context, options}` (whitelist via `buildProxyRequestOptions` at `:101-114`); response body parsed via `buffer.split("\n")` (`:192`), filters `data: ` prefix only (`:196`) |
| Ordering guarantees | Sequential; **out-of-order events throw** (defect 1.22, sites `:261/:275/:293/:307/:333`); `toolcall_end` mismatch silently returns `undefined` (`:347`) — asymmetry |
| Required fields | Same discriminator as EC1 minus the `partial: AssistantMessage` field, which the server **strips on the wire** to save bandwidth |
| Optional fields | Per-variant subset of EC1; client reconstructs `partial` from closure-scoped accumulator (`:121-137`) |
| Identifiers / timestamps | None framed; relies on EC1 invariants |
| Error cases | Network failure surfaced as single `{type:"error"}` event (`:214-224`), not thrown out of `streamProxy`; **no timeout, no heartbeat parsing** (arch-OQ3 unresolved); 401 not refresh-handled |
| Restart / resume behavior | None; closes on terminal event |

**Wire-framing note**: pi uses custom `data: <json>\n` (single newline). Canonical SSE uses `data: <json>\n\n` (double newline). Mid-frame proxies/CDNs that inject SSE comments (`:`-prefix lines) may break parsing. *(portability hazard)*

### EC4 — Extension Lifecycle Events (resolves con-CF2)

**Source-of-truth**: `ExtensionEvent` union at `packages/coding-agent/src/core/extensions/types.ts:950-972` (22 union members). Subscribable surface at `:1089-1126` exposes **29 distinct event names** because `SessionEvent` is itself an 8-member union. Per-event handler-input/return semantics catalog in `scratch/proto-extension-events.md`.

| Field | Value |
|---|---|
| Producer | coding-agent runner emits via `emit*` methods in `runner.ts`; sites scattered through `agent-session.ts` |
| Consumer | Extensions registered via `runtime.subscribe(...)` (or the `on()` typed overload) in extension modules |
| Transport / carrier | In-process function call; per-handler `try/catch` for **all emitters EXCEPT `emitToolCall`** at `runner.ts:806-827` (the unguarded path; throw propagates to `agent-session.ts:388-399` and re-thrown as `Extension failed, blocking execution: ...`) |
| Ordering guarantees | Handlers fire in registration order. For guarded emits, errors route to `emitError` listeners (handler continues, tool continues). For `emitToolCall`, throw blocks the tool **and** skips subsequent `tool_call` handlers |
| Required fields | Per event; load-bearing examples: `session_start{ reason: "fresh" \| "reload" }`, `tool_call{ name, args, callId }`, `before_provider_request{ model, messages, ... }`, `before_compact{ ... }` |
| Optional fields | Per event |
| Identifiers / timestamps | None at event level; compose with session JSONL timestamps for correlation |
| Error cases | Guarded: handler throw → `emitError`. Unguarded (`tool_call` only): handler throw → `Extension failed, blocking execution`. **`emitToolCall` is the only unguarded emitter — semantic divergence flagged as portability hazard** |
| Restart / resume behavior | `ctx.reload()` (`agent-session.ts:2383`) emits `session_shutdown(reload)` (`:2385`) → rebuild runtime (`:2389-2393`) → conditional `session_start(reload)` (`:2401`). `runtime.invalidate()` (`loader.ts:164`) marks captured `pi`/`ctx` stale; `assertActive` (`loader.ts:139`) throws on use-after-invalidation |

*(observed fact — corrects architecture phase's "~25 events" to 22 union members / 29 subscribable names)*

**Single-source-of-truth drift** (con-OQ5 still open): the 22 events are duplicated across the union type, the `on()` overloads, and the runner's emit methods. A drift-prevention recommendation is to derive all three from a single `EVENT_NAMES` tuple. *(open question — needs maintainer ruling)*

### EC5 — Session JSONL v3 Records (catalog → §Persistent Schema Notes)

10 record types (catalog enumerated below in §Persistent Schema Notes — record types share the persistence boundary, not the event-stream boundary). Detail in `scratch/proto-sessions.md`.

### EC6 — TUI Render Protocol (catalog → §State Machine)

Pixel-painting protocol; modeled as a render FSM below in §State Machine because the diff/emit cycle is more state-machine-shaped than event-catalog-shaped.

## State Machine

### SM1 — `AssistantMessageEvent` Stream FSM

```
              start                 text_start                                text_end                done
   (idle) ───────────► (active) ──────────────► (text_block) ─── text_delta ── (text_block)  ────► (text_done) ───► (terminated)
                          │                                                              │
                          │ thinking_start                                                │ toolcall_start
                          ▼                                                              ▼
                 (thinking_block) ── thinking_delta ── (thinking_block) ── thinking_end ─►(active)
                                                                                         │
                          ┌──────────────────────────────────────────────────────────────┘
                          ▼
                 (toolcall_block) ── toolcall_delta ── (toolcall_block) ── toolcall_end ─►(active)

                                                 ANY STATE ── error ──────────► (terminated)
                                                 ANY STATE ── abort ──────────► (terminated, error{reason:"aborted"})
```

| State | Trigger | Guard | Next State | Side Effects |
|---|---|---|---|---|
| idle | `start` | First event | active | Caller sees a new turn began |
| active | `text_start` | — | text_block | Caller allocates message content slot |
| active | `thinking_start` | — | thinking_block | Caller allocates thinking slot (Anthropic only typically) |
| active | `toolcall_start` | — | toolcall_block | Allocate slot keyed by `(streamIndex \|\| id)` |
| any block | `*_delta` | Same key | self | Append to slot |
| any block | `*_end` | Same key | active | Slot finalized; tool-calls parsed JSON |
| any | `done` | Terminal | terminated | `EventStream.result()` resolves with full message |
| any | `error` | Terminal | terminated | `EventStream.result()` resolves with error event (not thrown) |
| any | abort signal | Cancellation | terminated | Emits `error{reason:"aborted"}` |

*(observed fact, with defects: `start` missing → defect 1.19 silently swallows entire stream)*

### SM2 — Session JSONL Deferred-Flush FSM (resolves dsm-RT6)

```
       (in-memory buffer)
            │
            │ message events accumulate
            │ (user, tool calls, etc.)
            ▼
     (buffer holds all)
            │
            │ FIRST assistant message arrives
            │
            ▼
     [ FLUSH BUFFER TO DISK ]   ←── critical transition
            │
            ▼
       (file open, append per event)
            │
            ▼
       (close on dispose() or abort())
```

| State | Trigger | Guard | Next State | Side Effects |
|---|---|---|---|---|
| in-memory buffer | message event | not yet first-assistant | self (append to buffer) | Memory grows; **disk file may not exist** |
| in-memory buffer | first assistant arrives | — | flushed | `_persist` (`session-manager.ts:801-819`) flushes whole buffer; file created |
| flushed | message event | — | self | `appendFileSync` per record |
| flushed | dispose / abort | — | closed | **No flush on dispose/abort** |

**Critical implication for dsm-RT6**: a process crash (or `abort()`) **before the first assistant message arrives loses the user's prompt entirely** — the JSONL file may not even exist on disk. *(observed fact, contract gap, fix-before-porting recommended)*

### SM3 — TUI Render FSM (resolves arch-CF5)

```
   ┌───────────────┐
   │ idle / wait   │
   └───────┬───────┘
           │ render() requested (re-render trigger: state change, SIGWINCH, timer)
           ▼
   ┌───────────────────┐
   │ build lines       │  ← rootComponent.render()
   └───────┬───────────┘
           │
           ▼
   ┌───────────────────┐
   │ composite overlays│  ← compositeLineAt (tui.ts:810-858) — strict last-resort width clamp
   └───────┬───────────┘
           │
           ▼
   ┌───────────────────┐
   │ strip CURSOR_MARKER  ← `\x1b_pi:c\x07` (tui.ts:68)
   └───────┬───────────┘
           │
           ▼
   ┌───────────────────┐
   │ apply line resets │  ← SGR + OSC 8 terminator (tui.ts:796)
   └───────┬───────────┘
           │
           ▼
   ┌───────────────────┐
   │ diff vs previous  │
   └───────┬───────────┘
           │
           ▼ (if width changed: full-clear; if height changed AND not Termux: full-clear)
   ┌───────────────────────────────────┐
   │ emit firstChanged..lastChanged    │
   │ wrapped in `\x1b[?2026h ... \x1b[?2026l` (synchronized output)
   └───────┬───────────────────────────┘
           │
           ▼
   (back to idle)
```

| State | Trigger | Guard | Next State | Side Effects |
|---|---|---|---|---|
| idle | render request | — | building | — |
| building | (sync) | line-overflow detected | **process exit** | **Defect 1.15** — `tui.ts:1100-1131` throws inside setTimeout callback after writing crash log to `~/.pi/agent/pi-crash.log`. |
| building | (sync) | width changed | full-clear | Emit clear-screen sequence |
| building | (sync) | height changed AND `!isTermuxSession` | full-clear | Same |
| building | (sync) | normal | diff-and-emit | Compute changed line range |
| diff-and-emit | (sync) | — | idle | Wrap output in `\x1b[?2026h ... \x1b[?2026l` |
| any | EPIPE on stdout.write | — | (silent — defect 2.11) | Process continues running headless after disconnect |

*(observed fact, with defects 1.15 and 2.11 + portability hazard for synchronized-output-mode-2026 support)*

### SM4 — streamProxy Connection FSM

| State | Trigger | Guard | Next State | Side Effects |
|---|---|---|---|---|
| disconnected | `streamProxy({...})` invoked | — | connecting | HTTP POST to `${proxyUrl}/api/stream` |
| connecting | response 200 | — | streaming | SSE-like body parsing begins |
| connecting | response non-200 | — | terminated (error event) | Emit `{type:"error"}`; close |
| streaming | `data: <json>\n` line | — | self | Parse and `processProxyEvent` |
| streaming | terminal `done`/`error` | — | terminated | Caller gets terminal AssistantMessageEvent |
| streaming | out-of-order delta/end | **Defect 1.22** | thrown (turn dies) | Throws at `proxy.ts:261/275/293/307/333` |
| streaming | abort signal | — | terminated | (caller's responsibility — no proxy-side hook) |
| streaming | (no heartbeat) | stale connection | OS-level timeout only | **arch-OQ3 unresolved** |

*(observed fact + portability hazard — no application-level keepalive)*

### SM5 — OAuth Refresh FSM

| State | Trigger | Guard | Next State | Side Effects |
|---|---|---|---|---|
| valid | provider 401 | token age > soft-expiry | refresh-needed | — |
| refresh-needed | acquire `proper-lockfile` on `auth.json` | lock free | refreshing | Lock acquired |
| refresh-needed | acquire `proper-lockfile` | lock held | wait | Block until released |
| wait | lock released | — | re-read | Re-load `auth.json` (other instance refreshed) |
| re-read | new token present | — | valid | Use it |
| refreshing | provider grants new token | — | write+release | `auth.json` rewritten with new token; mode 0600 enforced |
| refreshing | provider denies | — | error → user | Surface failure to user |

*(observed fact — proper-lockfile single-flight prevents multi-instance refresh race; **partial answer to arch-CF9 — Pass 4 still owes the trust-boundary deep audit**)*

### SM6 — Extension Reload FSM

| State | Trigger | Guard | Next State | Side Effects |
|---|---|---|---|---|
| running | `ctx.reload()` | — | shutting-down | Emit `session_shutdown(reason:"reload")` (`:2385`) |
| shutting-down | all listeners notified | — | rebuilding | Tear down old runtime |
| rebuilding | settings/providers/resources reloaded | — | invalidated-old-ctx | `runtime.invalidate()` at `loader.ts:164` (old `pi`/`ctx` references throw on use) |
| invalidated-old-ctx | re-import extensions (jiti `moduleCache:false`) | — | starting | Fresh module instances, no stale closures |
| starting | new runtime ready | — | running | Emit `session_start(reason:"reload")` (`:2401`) |

*(observed fact, resolves con-CF2 reload semantics)*

## Persistent Schema Notes

### PS1 — Session JSONL v3 Record Catalog (resolves arch-CF4, con-CF3, dsm-RT6)

**Source-of-truth**: `packages/coding-agent/src/core/session-manager.ts` (no separate `session-jsonl.ts`). Catalog detail in `scratch/proto-sessions.md`.

All entries share the base shape `SessionEntryBase = { type, id, parentId, timestamp }`. Discriminated union by `type`:

| # | Type discriminator | Purpose | Special fields |
|---|---|---|---|
| 1 | `message` | User / assistant / tool messages | `role: "user" \| "assistant" \| "tool"`, `content: ...`, plus role-specific shape |
| 2 | `compaction` | Marks a compaction summary (replaces older entries semantically) | `summary` content |
| 3 | `branch_summary` | Marks a branch fork point | `branchedFrom` |
| 4 | `thinking_level_change` | Records mid-session thinking-level toggle | `level` |
| 5 | `model_change` | Records mid-session model swap | `from`, `to` |
| 6 | `custom` | Extension-driven custom record | extension-defined |
| 7 | `custom_message` | Extension-driven custom message | extension-defined |
| 8 | `label` | Named bookmark within session | `label` string |
| 9 | `session_info` | Header (per-file, first record) | `sessionId`, `cwd`, `parentSessionId?`, `version: 3` |
| (base) | `SessionEntryBase` | All entries | `{type, id, parentId, timestamp}` |

**Identifiers**:
- `sessionId`: uuidv7 (`session-manager.ts`).
- Per-entry `id`: first 8 hex chars of `randomUUID()` with up-to-100-retry collision check (`:206-213`).
- `parentId`: links into the parent entry (forms the conversation tree; non-message entries can also have parents).
- `timestamp`: ISO-8601 string, no nanoseconds.

### PS2 — Append vs Rewrite Semantics

- **Default mode**: append-only (`appendFileSync` per record).
- **Rewrite triggers**:
  - Migration (v2 → v3 transition).
  - Corrupt-file recovery.
  - `createBranchedSession` (new file with only the path-to-leaf as chosen by `findValidCutPoints`).
- **Rewrite mechanism**: `_rewriteFile` (`session-manager.ts:775-779`) — `writeFileSync(file, content)` with **no temp+rename** (defect 2.1 critical).
- **Defect 2.7** (silent malformed-line skip) confirmed at `loadEntriesFromFile:447-452` and `parseSessionEntries:293-295`.
- **Defect 1.7** confirmed at `compaction.ts:309-310` — `branchSummary`/`compactionSummary` roles inside `type:"message"` are inert because `appendMessage` forbids them (`:828-833`); only valid as top-level entries.

### PS3 — Branching Modes

3 distinct branching mechanisms:

| Mode | Mechanism | File-level behavior |
|---|---|---|
| In-place leaf move | `branch()` | Append-only; leaf pointer changes |
| Branch with summary | `branchWithSummary()` | Adds a `branch_summary` entry to current file |
| New-file branch | `createBranchedSession()` | New file written with only the path-to-leaf; `parentSessionId` in header |

### PS4 — Encoded-cwd & Cross-OS Concerns

- Path encoding: `--${cwd.replace(/^[/\\]/,"").replace(/[/\\:]/g,"-")}--` (`:429`).
- **Lossy encoding**: `/a/b` and `/a-b` collide.
- Case sensitivity inherits the host filesystem. **Cross-OS sync of `~/.pi/agent/sessions/` is unsafe.** *(portability hazard)*
- No file locking; no inotify; no atomic publish. Concurrent-reader detection is unsupported.

### PS5 — `auth.json` Schema

- JSON file at `~/.pi/agent/auth.json`, mode 0600 enforced on every write (Pass 6 confirmed).
- Per-provider entries with shape `{ type: "api_key" | "oauth", api_key?: string, oauth?: {...} }`.
- Implicit schema (TS types in `auth-storage.ts`).
- Corrupt JSON → `loadError` set; all reads blocked until fixed (per Pass 6 partial answer to arch-OQ2).
- Windows ACL behavior is open question (`con-OQ2`).

### PS6 — `models.json` Schema (active config — RCE channel for untrusted projects)

- JSON file at `~/.pi/agent/models.json` and project-scoped `<repo>/.pi/models.json`.
- `headers` values support `!`-prefix shell-exec via `resolveConfigValue` (`packages/coding-agent/src/core/config.ts`). **Trust-boundary violation when sourced from untrusted repo** — `dsm-RT1` Pass 4 finding.

### PS7 — `~/.pi/agent/` Family

| File | Locking | Concurrent-write semantics |
|---|---|---|
| `auth.json` | `proper-lockfile` during OAuth refresh | Single-flight refresh |
| `settings.json` | `SettingsManager` uses `proper-lockfile` | But `migrateAuthToAuthJson` does an unlocked read-modify-write that races (defect P6.6 / dsm-RT3) |
| `models.json` | None | Reader's tolerance for concurrent rewrite is open question |
| Sessions JSONL | None | No locking — concurrent readers see partial writes; mid-rewrite state is undefined |

## Compatibility Hazards

| ID | Hazard | Where | Severity | Notes / Implication for porting |
|---|---|---|---|---|
| H1 | Synchronized output mode 2026 not universal | `packages/tui/src/tui.ts` — wraps every paint | high | Older terminals will tear; pi has no fallback (assumes support) |
| H2 | Custom APC `CURSOR_MARKER = \x1b_pi:c\x07` | `tui.ts:68` | high | Non-standard; if a third-party terminal doesn't strip APC properly, leaks into terminal as text |
| H3 | Kitty keyboard protocol push `\x1b[>7u` (not `>1u`) | `terminal.ts:155` | medium | Specific protocol level; older Kitty terminals support fewer flags |
| H4 | Kitty image protocol `\x1b_G…\x1b\\` and iTerm2 `\x1b]1337;File=…\x07` | `terminal-image.ts:106-107` | medium | Detection by env-var sniffing only (no OSC query); tmux/screen/cmux force `images: null` |
| H5 | Width-overflow process kill | `tui.ts:1100-1131` | high | **Defect 1.15** — render-loop throws kill the process from setTimeout callback. Crash log to `~/.pi/agent/pi-crash.log`. |
| H6 | Paste-mode no timeout-armed flush | `stdin-buffer.ts:292-311` | high | **Defect 1.18** — lost `\x1b[201~` leaks all subsequent stdin into pasteBuffer forever |
| H7 | EPIPE on stdout.write not handled | `terminal.ts:318-327` | high | **Defect 2.11** — agent continues running headless after SSH disconnect |
| H8 | Windows VT input requires koffi FFI | `terminal.ts:216` | high | Silent degradation if koffi unavailable; modifier keys lost |
| H9 | Tmux/screen/cmux force `images: null` | `terminal-image.ts:40-88` | medium | Embedded terminals lose image support |
| H10 | streamProxy custom `\n` framing vs canonical SSE `\n\n` | `proxy.ts:192-196` | high | CDN/proxies that inject SSE comments (`:`-prefix) break parsing |
| H11 | streamProxy no heartbeat / keepalive | `proxy.ts` | high | Stale-connection detection is OS-TCP only; arch-OQ3 unresolved |
| H12 | streamProxy out-of-order events terminate turn | `proxy.ts:261/275/293/307/333` | high | **Defect 1.22** — one misframe kills the turn; asymmetric vs `toolcall_end` mismatch which silently returns undefined (`:347`) |
| H13 | streamProxy `partialJson` leak via shallow-clone | `proxy.ts:325`, `:339` | medium | **Defect 1.14** — finished tool calls retain `partialJson` in earlier snapshots |
| H14 | Session JSONL non-atomic rewrite | `session-manager.ts:775-779` | critical | **Defect 2.1** — mid-rewrite crash truncates; data loss in normal operation |
| H15 | Session JSONL deferred flush — buffer until first assistant message | `session-manager.ts:801-819` | critical | **dsm-RT6** — crash before first assistant loses user prompt entirely |
| H16 | Session encoded-cwd lossy + case-sensitivity inherits FS | `session-manager.ts:429` | medium | Cross-OS sync unsafe; `/a/b` and `/a-b` collide |
| H17 | Async fan-out is sequential, unguarded | `agent.ts:539-541` | high | **Defect 1.13** — listener throw stops subsequent listeners |
| H18 | Extension `emitToolCall` is the only unguarded emitter | `runner.ts:806-827` | high | Semantic divergence vs other 21 emit methods which catch and route to `emitError` |
| H19 | Extension event-name 3-way duplication | `types.ts:950-972`, `:1089-1126`, runner emit methods | medium | **con-OQ5** — drift surface; recommend single-source `EVENT_NAMES` tuple |
| H20 | OpenAI-compat URL matrix tool-call delta variation | `openai-completions.ts:1029-1088` | high | **arch-CF8** — Pass 5 audit pending; ~10 vendor branches with subtle behavior differences |
| H21 | Provider error-message destruction (7 of 9 providers) | `anthropic.ts`, openai-responses-shared, azure-openai-responses, google, google-vertex, amazon-bedrock, mistral | high | **Defect 1.1** — replicated anti-pattern destroys provider-mapped `errorMessage` via opaque `"An unknown error occurred"` literal |
| H22 | undici `bodyTimeout`/`headersTimeout` set to 0 globally | `cli.ts:20` | high | **Defect P6.1 / arch-CF7** — disables every Node-fetch deadline in the process |
| H23 | photon WASM `fs.readFileSync` global monkey-patch | `coding-agent/src/utils/photon.ts` | high | **Defect P6.2 / arch-CF6** — multi-tenant SDK embedding hazard, restore stacking under concurrent patchers, hardcoded fallback path list |
| H24 | `models.json` `!`-prefix shell-exec | `coding-agent/src/core/config.ts` `resolveConfigValue` | critical | **dsm-RT1** — covert RCE channel via project-scoped `<repo>/.pi/models.json` |
| H25 | `rg`/`fd` auto-download with no checksum/sig verification | `coding-agent/src/utils/tools-manager.ts` | high | **dsm-RT9** — supply-chain hazard; binaries execute as part of grep/find tools |
| H26 | OAuth refresh + filesystem locking via proper-lockfile | `auth-storage.ts` | medium | Multi-instance correct; **windows lock semantics open** (con-OQ2) |
| H27 | Session resume picks `[0]` on ambiguous prefix silently | `main.ts:155-167` | high | **Defect 1.6** — non-deterministic session selection |

## Open Questions

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| pro-OQ1 | needs-fixture-capture | streamProxy SSE heartbeat / keepalive behavior — does the server send any keepalive at all, or rely purely on TCP? Continues arch-OQ3. | Live capture against running proxy required. |
| pro-OQ2 | needs-runtime-test | Concurrent-reader detection on session JSONL during mid-rewrite — does a reader observe a truncated file, an EBUSY, or an inconsistent JSONL stream? | Requires concurrent process test. |
| pro-OQ3 | needs-spec-ruling | Should the deferred-flush behavior in `_persist` be considered an intentional optimization or a contract gap (dsm-RT6)? Crash before first assistant loses user prompt. | Maintainer ruling on whether the user prompt is durable. |
| pro-OQ4 | needs-runtime-test | Behavior when `\x1b[?2026` is not supported by the terminal (no synchronized output) — does pi-tui detect and fall back, or does the terminal interpret the bracketed sequence as text? | Terminal-capability test required. |
| pro-OQ5 | needs-runtime-test | EPIPE on `process.stdout.write` mid-paint (defect 2.11 / H7) — does Node's default-warning-on-uncaught-error fire, or does it silently absorb? | Disconnect-mid-paint test required. |

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| pro-CF1 | porting | Defect-synthesis row consolidation: H1–H27 hazards table is the protocol-phase view; porting needs to integrate with the mechanical+semantic defect catalogs into a single porting-oriented table. | Synthesis is porting phase's rubric. |
| pro-CF2 | reimplementation-spec | The 6 state machines (SM1–SM6) need to be re-expressed as language-agnostic FSMs in the reimpl spec — particularly SM2 (deferred-flush), SM4 (proxy connection), SM5 (OAuth refresh), and SM6 (extension reload), which are non-trivial to port faithfully. | Reimpl-spec rubric for protocols and state. |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | An event catalog is documented. | PASS | §Event Catalog has 6 catalog entries (EC1 AssistantMessageEvent 12-variant union, EC2 tool-call sub-protocol, EC3 ProxyAssistantMessageEvent, EC4 22-event extension catalog, EC5 ↗§Persistent Schema, EC6 ↗§State Machine). All resolved carry-forwards (arch-CF3, con-CF1, con-CF2, con-CF4) confirmed RESOLVED. |
| 2 | A state machine is documented. | PASS | §State Machine has 6 FSMs (SM1 stream FSM with ASCII diagram + transition table; SM2 deferred-flush FSM; SM3 TUI render FSM; SM4 streamProxy connection FSM; SM5 OAuth refresh FSM; SM6 extension reload FSM). |
| 3 | Persistent schema notes are documented. | PASS | §Persistent Schema Notes has 7 sub-sections (PS1 JSONL v3 record catalog with all 10 entry types; PS2 append-vs-rewrite incl. defects 2.1, 2.7, 1.7; PS3 3 branching modes; PS4 encoded-cwd; PS5 auth.json; PS6 models.json RCE channel; PS7 lock semantics across the family). |
| 4 | Compatibility hazards are documented. | PASS | §Compatibility Hazards table H1–H27 (27 hazards) covering ANSI/terminal, async ordering, OAuth refresh, filesystem locking, encoding, web-fetch timeouts, supply chain, RCE channel. |
| 5 | Findings are marked with evidence levels. | PASS | Every load-bearing claim followed by `(observed fact / strong inference / portability hazard / open question)`. Hazard table column "Severity" + "Notes" cite defect IDs (1.x, 2.x, 6.x, dsm-RT*, arch-CF*) for round-trip auditability. |

**Validated by:** 2026-05-06 (protocols phase, session 1, no-orchestrator mode)
**Overall:** PASS

(arch-CF3, arch-CF4, arch-CF5, con-CF1, con-CF2, con-CF3, con-CF4, dsm-RT6 all RESOLVED. 5 new open questions surfaced — none of them resolvable from source alone. 2 carry-forwards routed forward to porting and reimplementation-spec.)
