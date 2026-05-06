# Runtime Lifecycle — pi-mono

Catalog of boot, run, shutdown, and background tasks. Append-only across phases;
each phase adds a dated section.

---

## 2026-05-06 — architecture phase

### `pi` CLI Boot Sequence

1. `packages/coding-agent/src/cli.ts` entry. *(observed fact)*
2. **Global undici tweak**: `bodyTimeout` and `headersTimeout` set to `0` at
   startup (`cli.ts:20`) so long local-LLM stalls don't trigger built-in
   client timeouts. *(observed fact, portability hazard)*
3. Argument parse → mode select (interactive TUI / non-interactive RPC /
   headless). *(strong inference — defer mode boundaries to contracts phase)*
4. Auth resolution via 5-level precedence in `src/core/auth-storage.ts:446-516`:
   `runtime override → api_key → oauth → env → fallback resolver`. *(observed fact)*
5. `AgentSession` construction with **5 distinct `AbortController`s** (run,
   per-request, per-tool, etc.). *(observed fact — coding-agent extraction note)*
6. Built-in tool registration: `read`, `bash`, `edit`, `write`, `grep`, `find`,
   `ls` (`packages/coding-agent/src/core/tools/`). *(observed fact)*
7. `rg` / `fd` shell-out targets verified; binaries auto-downloaded by
   `utils/tools-manager.ts` if absent. **Network side effect at startup.**
   *(portability hazard)*
8. Extension loading via `jiti` with bundled `virtualModules` (so extensions
   work even inside the Bun-compiled standalone binary). *(observed fact)*
9. TUI initialization: `pi-tui` `TUI` class spawns process terminal,
   raw stdin via `StdinBuffer`, sets up synchronized output mode 2026.
   *(observed fact)*

### Main Loop

Per iteration:

1. Compose user message + context → `Agent.run(...)`.
2. Provider stream produced via `pi-ai`'s `streamSimple()` → registry
   dispatches by `model.api` (the `KnownApi` discriminator). *(observed fact)*
3. Provider emits raw events; `EventStream` async-iterable bridges them into
   the unified `AssistantMessageEvent` union (12 variants). *(observed fact —
   `packages/ai/src/utils/event-stream.ts:4-66`, `src/types.ts:269-281`)*
4. `Agent` listener fan-out is **sequential** — listeners run in order, not
   in parallel. *(observed fact — `packages/agent/src/agent.ts:443`)*
5. On `tool-call` event, dispatch to registered tool. File-mutating tools
   serialize via `withFileMutationQueue` to prevent concurrent writes to
   the same path. *(observed fact)*
6. Tool result appended to transcript; transcript writes are append-only
   JSONL with full-rewrite for branch edits. *(observed fact)*
7. `_agentEventQueue` serializes UI delivery so the TUI never receives
   out-of-order events. *(observed fact)*
8. Repeat until terminal `done` or `error` event.

### Shutdown / Cleanup

- `AgentSession`'s `AbortController`s are aborted on shutdown signal.
  *(strong inference — defer exact handler chain to defect-scan)*
- TUI restores terminal state (synchronized output mode off, alt-screen not
  used so no restore needed there). *(strong inference)*
- OAuth refresh holds a `proper-lockfile` lock during refresh — released
  on completion or process exit. *(observed fact — coding-agent extraction)*
- Open question: what happens to in-flight streams when `pi` is killed mid-tool-call? Defer to defect-scan-semantic Pass 3 (concurrency).

### Background / Scheduled Tasks

- **None identified at the architecture level.** No cron, no setInterval
  beyond TUI redraw / debounce. Defer detailed concurrency audit to
  defect-scan-semantic Pass 3.

### `pi-agent-core` `streamProxy` Lifecycle

When `streamProxy` is enabled (`packages/agent/src/proxy.ts:116`):

1. Server accepts HTTP POST at `/api/stream`.
2. Each request opens an SSE stream (`Content-Type: text/event-stream`).
3. Each event framed as `data: {ProxyAssistantMessageEvent}\n`. Event
   union at `src/proxy.ts:36-57`. *(observed fact)*
4. Stream closes when terminal `done` / `error` event sent.
5. **Heartbeat / keepalive behavior unverified from source.** *(arch-OQ3 — needs runtime fixture capture)*

### Web-UI Browser Lifecycle

1. Consumer mounts `pi-chat-panel` custom element.
2. Element opens IndexedDB / localStorage for session state.
3. LLM calls go via `pi-ai`'s `streamSimple` (browser-targeted build —
   Node-only providers like Bedrock are dynamic-imported behind gates).
4. `import.meta.url` worker URLs used for pdfjs document extraction.
5. Streaming display uses an rAF-batched deep-clone (`StreamingMessageContainer.ts:43-60`).
   *(observed fact — web-ui extraction)*

---

## 2026-05-06 — contracts phase (append)

Refinements to the architecture-phase boot sequence + Agent loop, with new contract-level detail.

### Mode Dispatch Refinement

`packages/coding-agent/src/main.ts:98-109` `resolveAppMode` selects mode before any session work. `main.ts:673-726` is the dispatcher. Stdin-piped interactive mode auto-falls back to print mode at `main.ts:634`. RPC mode rejects `@file` args (`main.ts:475-478`). Subcommands (`install`, `update`, `config`) are intercepted before `parseArgs` (`main.ts:431-437`).

### Cancellation Lifecycle (load-bearing)

5 AbortControllers in `core/agent-session.ts:264-278`:

| Controller | Aborts |
|---|---|
| `_compaction` | Manual `/compact` |
| `_autoCompaction` | Auto-compaction triggered by context-size threshold |
| `_branchSummary` | Branch-edit summary generation |
| `_retry` | In-flight retry attempt |
| `_bash` | Active bash subprocess |

Plus `session.agent.signal` (the main turn signal). `session.abort()` aborts retry+agent. `agent-loop.ts` listener fan-out is sequential and unguarded (defect 1.13). Esc-during-stream calls `restoreQueuedMessagesToEditor({abort:true})` at `agent-session.ts:2357` to preserve queued messages.

### Signal Handling

Replicated across all three modes (interactive `:3258-3287`, print `print-mode.ts:47-63`, rpc `rpc-mode.ts:351-365`):

| Signal | Effect | Exit code |
|---|---|---|
| Ctrl+C (single, within 500ms of next) | Double-tap to exit | n/a (waits for second tap) |
| Ctrl+C (after 500ms) | Re-arms timer | n/a |
| Esc | Stream-abort / bash-abort / bash-mode-exit / double-Esc → /tree or /fork (interactive only) | n/a |
| Ctrl+D | Exit interactive mode | 0 |
| SIGTERM | Reap detached child processes via `killTrackedDetachedChildren()`; exit | 143 |
| SIGHUP | Same as SIGTERM | 129 |

### Extension Lifecycle Sub-events

Per the Extension API contract (F-EXT-1..4):

```
session_start → (agent-loop iterations: before_provider_request → ... → after_provider_response → tool_call → tool_result → ...) → session_shutdown
                                                                                                         │
                                                                                                         └── (before_compact at threshold)
```

- `session_start(reload)` and `session_shutdown(reload)` fire on `ctx.reload()` (`agent-session.ts:2383`) without a process exit.
- `tool_call` is the **only unguarded** hook (per `runner.ts:806-827` — see contract F-EXT-1). Throw blocks the tool and skips subsequent `tool_call` handlers.

### `streamProxy` Lifecycle Correction

The architecture phase described `streamProxy` as a server lifecycle. **Correction**: `streamProxy` is a CLIENT lifecycle:

1. Caller invokes `streamProxy({ proxyUrl, ... })`.
2. Client POSTs to `${proxyUrl}/api/stream` (`packages/agent/src/proxy.ts:152`).
3. Parses `data: <json>\n` lines (`:195-205`); reconstructs `AssistantMessageEvent` with `partial` field added back (server strips it for bandwidth).
4. Stream closes when terminal `done` / `error` event arrives.
5. **No heartbeat / keepalive parsing** (arch-OQ3 unresolved; con-CF4 carry-forward to protocols).

### Migrations Lifecycle

`runMigrations` runs **unconditionally on every startup** (per Pass 6 P6.5; `core/migrations.ts:304-314`):
1. Auth migration (auth.json structure)
2. Sessions migration
3. Binaries migration (rg/fd location)
4. Keybindings migration
5. Commands → prompts migration

Every step wrapped in bare `catch {}` — partial migration state can persist with no telemetry.

---

## 2026-05-06 — protocols phase (append)

State-machine refinements to the lifecycle.

### Session JSONL Deferred-Flush Lifecycle (resolves dsm-RT6)

**Critical timing**: per `_persist` at `packages/coding-agent/src/core/session-manager.ts:801-819`, message events accumulate in an **in-memory buffer** until the **first assistant message** arrives, at which point the entire buffer is flushed in a single append loop. `dispose()` and `abort()` do **not** flush.

**Implication**: a process crash (or `abort()`) before the first assistant message arrives **loses the user's prompt entirely** — the JSONL file may not exist on disk yet. Documented as protocol hazard H15 + open question pro-OQ3 (needs maintainer ruling).

### Extension Reload Lifecycle (resolves con-CF2 reload semantics)

`ctx.reload()` (`agent-session.ts:2383`) sequence:
1. Emit `session_shutdown(reason:"reload")` (`:2385`)
2. Tear down old runtime
3. Reload settings/providers/resources (`:2389-2393`)
4. `runtime.invalidate()` at `loader.ts:164` — captured `pi`/`ctx` references throw on next use via `assertActive` (`loader.ts:139`)
5. Re-import extensions fresh (jiti `moduleCache:false` ensures no stale closures)
6. Emit `session_start(reason:"reload")` (`:2401`)

### streamProxy Connection Lifecycle (resolves con-CF4)

Per `packages/agent/src/proxy.ts`:
1. POST `${proxyUrl}/api/stream` with `Bearer ${authToken}`; body `{model, context, options}` (whitelist via `buildProxyRequestOptions` at `:101-114`).
2. SSE-like body parsing via `buffer.split("\n")` (`:192`); only `data: ` prefix lines processed (`:196`).
3. Per parsed event, `processProxyEvent` reconstructs `AssistantMessageEvent` (incl. mutating closure-scoped `partial: AssistantMessage` at `:121-137` — server stripped the field; client re-attaches it).
4. **Out-of-order `_delta`/`_end` events throw** at `:261/:275/:293/:307/:333` (defect 1.22). `toolcall_end` mismatch silently returns `undefined` (`:347`) — asymmetry.
5. Network failures surfaced as `{type:"error"}` event (`:214-224`), not thrown.
6. **No timeout**, **no heartbeat parsing** — stale-connection detection is OS-TCP only (arch-OQ3 unresolved).
7. Closes on terminal `done`/`error` event.

### TUI Render Lifecycle Refinement (resolves arch-CF5)

6-step pipeline in `tui.ts:888-1204`:
1. Build lines (rootComponent.render())
2. Composite overlays (`compositeLineAt` at `:810-858` — strict last-resort width clamper)
3. Strip CURSOR_MARKER (`\x1b_pi:c\x07` at `:68`)
4. Apply line resets (SGR + OSC 8 terminator at `:796`)
5. Diff against `previousLines`
6. Emit only `firstChanged..lastChanged` wrapped in `\x1b[?2026h ... \x1b[?2026l`

Width change always full-clears (`:960`); height change full-clears EXCEPT on Termux (`:969`, `isTermuxSession` at `:111`).

### Width-Overflow Crash Path

**Defect 1.15 / H5**: width-overflow throws inside the `setTimeout` callback after writing crash log to `~/.pi/agent/pi-crash.log` (`tui.ts:1100-1131`). Process killed; lifecycle ended.

### OAuth Refresh Lifecycle (state machine SM5)

Multi-instance refresh is single-flight via `proper-lockfile`:
1. Provider 401 + soft-expiry exceeded → enter refresh-needed.
2. Acquire lock on `auth.json`. If held by another instance, wait.
3. After acquiring, re-read `auth.json` — another instance may have refreshed.
4. If still stale, exchange refresh token for new access token.
5. Write new token to `auth.json` (mode 0600 enforced); release lock.

Provider 401 not handled by streamProxy CLIENT (no refresh path).
