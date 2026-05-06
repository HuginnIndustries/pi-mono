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
