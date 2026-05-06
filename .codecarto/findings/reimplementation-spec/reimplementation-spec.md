# Reimplementation Spec — pi-mono (language-agnostic)

<!--
  Phase: reimplementation-spec
  Pipeline: workflow/pipeline-full-with-deep-audit.yaml
  Strategic Alignment Hook: language-agnostic (locked at plan approval)
  Synthesizes: architecture-map + behavioral-contracts + protocols-and-state + reverse-engineering-bundle + both defect scans
  Evidence: observed fact / strong inference / portability hazard / open question / design decision
-->

## System Summary

The reimplementation target is an **interactive coding-agent product**: a
program that takes a user's natural-language instruction, talks to one
of many LLM providers through a unified streaming protocol, optionally
invokes file-system / shell / search tools on the user's behalf, and
records the conversation as a resumable transcript. The flagship
delivery is a terminal CLI; a secondary delivery is a browser-embeddable
chat component that bypasses the agent runtime and talks directly to
the provider layer.

The new build's load-bearing design choice — the one that must be
preserved even if every other detail changes — is **provider neutrality
behind a stable streaming event union**: every LLM provider collapses
into a single 12-variant async event stream so consumers (the agent
loop, the renderer, the session store) can reason about behavior
generically. A successful port preserves this contract; everything
else is implementation choice.

The new build is in scope to **fix the data-loss surfaces** the
reverse-engineering pass identified (non-atomic file writes, deferred
flush losing pre-first-assistant prompts, edit/write tools without
temp-write+rename), to **default to denying untrusted-project code-exec**
(the existing `<cwd>/.pi/` 5-channel auto-load is not preserved as-is),
and to **gate file-mutating and shell tools behind explicit
per-invocation user consent**. These three policy positions are
documented in §Required Behaviors below with citation back to the
defect that triggered the decision.

## Conceptual Module Model

Eight concept-level modules, expressed without language-specific
package names. Map back to current source layers in §Layer Split.

### M1 — Provider Adapter

| Field | Value |
|---|---|
| **Responsibility** | Adapt one LLM-vendor HTTPS API into the unified streaming event union. |
| **Public inputs** | `{model, messages, tools?, signal?, providerOpts?}` |
| **Public outputs** | An async event stream of the 12-variant `AssistantMessageEvent` union (terminal `done` or `error`). |
| **Owned state** | None per call (stateless). |
| **Invariants** | Every call emits exactly one terminal event; events are FIFO; abort emits `error{reason:"aborted"}`; provider errors are emitted as `error` events, **NOT thrown**. |
| **Collaborators** | M2 (Provider Registry); the underlying vendor SDK / HTTP client. |

### M2 — Provider Registry

| Field | Value |
|---|---|
| **Responsibility** | Look up a provider adapter (M1) by `model.api` discriminator; provide model catalog. |
| **Public inputs** | `model.api: KnownApi`; model discovery requests. |
| **Public outputs** | Provider stream function; model catalog entries. |
| **Owned state** | Static catalog of registered providers + their `KnownApi` discriminators. |
| **Invariants** | Unknown `api` produces a structured error (NOT a throw on registry-miss); catalog is read-only at runtime. |
| **Collaborators** | All M1 instances; M3 (catalog used by agent loop for token-budgeting). |

### M3 — Agent Loop (Core Runtime)

| Field | Value |
|---|---|
| **Responsibility** | Drive an interactive turn: compose prompt → invoke provider → process events → dispatch tool calls → emit to subscribers; repeat until terminal event. |
| **Public inputs** | `prompt(message)`, `continue()`, `subscribe(callback)`, `abort()`. |
| **Public outputs** | Promise resolving to `RunResult`; subscriber events. |
| **Owned state** | Single `AbortController` per run; subscriber list (registration order); current turn's partial assistant message. |
| **Invariants** | Listener fan-out is **bounded** (a throwing listener is logged and skipped; **does NOT stop subsequent listeners** — design decision diverging from current source defect 1.13/5.10); abort cascades immediately to provider call and current tool dispatch; sessionId is opaque caller-provided string forwarded to providers. |
| **Collaborators** | M1/M2; M4 (Tool Dispatcher); M5 (Session Store). |

### M4 — Tool Dispatcher

| Field | Value |
|---|---|
| **Responsibility** | Execute tool calls requested by the LLM, with explicit user consent for dangerous operations. |
| **Public inputs** | Tool name + arguments; consent decision channel. |
| **Public outputs** | Tool result (success or error). |
| **Owned state** | Tool registry; per-realpath mutation queue; consent-decision policy. |
| **Invariants** | (a) Filesystem-mutating tools (`edit`, `write`) use **temp-write + rename** for atomicity; (b) `bash`, `edit`, `write` require **per-invocation user consent** (or non-interactive `--yes-tools` flag); (c) tool subprocess exit codes are **never coerced** (a killed process surfaces as `null`/signal, not 0); (d) auto-downloaded helper binaries are **checksum-verified** against pinned hashes. |
| **Collaborators** | M3; M5; M6 (TUI); the host filesystem and shell. |

### M5 — Session Store

| Field | Value |
|---|---|
| **Responsibility** | Persist conversation transcripts; support resume and branch operations. |
| **Public inputs** | Append-record, branch-from-record, fork-to-new-file, load-by-id requests. |
| **Public outputs** | Recorded entries, durably stored. |
| **Owned state** | JSONL files keyed by hashed cwd + uuidv7 session id; in-memory indexes for branching. |
| **Invariants** | (a) Every record is **flushed to disk on every write** (no deferred-flush optimization — design decision diverging from current source dsm-RT6 / 5.7); (b) Rewrites use **temp-write + rename** for atomicity (design decision diverging from current source 2.1 / 5.8); (c) Ambiguous session-id prefix lookups **return an error**, never silently pick `[0]` (diverging from defect 1.6 / 5.6); (d) cwd encoding uses a **hash** (not lossy string-replace), eliminating cross-OS sync hazard H16. |
| **Collaborators** | M3 (writes per event); host filesystem. |

### M6 — Render Loop (TUI)

| Field | Value |
|---|---|
| **Responsibility** | Render a tree of components to the terminal with differential paint and capability-aware fallback. |
| **Public inputs** | Component tree; render trigger (state change, terminal resize, timer); raw stdin. |
| **Public outputs** | ANSI escape sequences on stdout; component event delivery on input. |
| **Owned state** | Previous-frame buffer; terminal capability map (synchronized output, image protocols, keyboard protocol). |
| **Invariants** | (a) Lines exceeding terminal width are **truncated**, never thrown (diverging from defect 1.15 / 5.14); (b) EPIPE on stdout.write **aborts the session** instead of silently continuing headless (diverging from defect 2.11 / H7); (c) Bracketed-paste mode has a **timeout-armed buffer flush** to recover from lost end-markers (diverging from defect 1.18 / H6); (d) When the terminal does not support a capability, the renderer **falls back gracefully** rather than emitting unsupported escapes as text. |
| **Collaborators** | Host terminal; M3 (subscribes to events); M7 (Slash Commander). |

### M7 — Slash Commander

| Field | Value |
|---|---|
| **Responsibility** | Discover, parse, and dispatch user-invoked `/<name>` commands; expand prompt templates. |
| **Public inputs** | User input lines starting with `/`. |
| **Public outputs** | Builtin command effects (e.g., `/compact`, `/debug`); template-expanded prompts forwarded to M3. |
| **Owned state** | Command registry (~20 builtins + discovered prompt templates). |
| **Invariants** | (a) Discovery order: agent-dir → cwd-dir → flag paths; non-recursive `.md` scan; (b) Frontmatter is YAML with `description` and `argument-hint` keys; (c) `/debug` redacts secret-like values **before** writing to log files (diverging from defect 4.5); (d) Log files use mode 0600, not umask-default. |
| **Collaborators** | M3; M5; host filesystem. |

### M8 — Extension Host

| Field | Value |
|---|---|
| **Responsibility** | Load user-defined extension modules from operator and (gated) project scope; emit ~22 lifecycle events to subscribers; offer registration APIs for tools / commands / providers / shortcuts / flags / message-renderers. |
| **Public inputs** | Module discovery paths; extension `setup(runtime)` calls; lifecycle-emit calls from M3/M4/M5/M6. |
| **Public outputs** | Extension callbacks invoked; registered surface additions. |
| **Owned state** | Loaded extension instances; subscription map. |
| **Invariants** | (a) **All emit paths are uniformly guarded** (a throwing handler is logged and skipped, including `tool_call` — diverging from defect 5.9); (b) Project-scope extensions (`<cwd>/.pi/extensions/`) require **explicit user consent** via `--trust-project` flag or interactive prompt (diverging from defect 4.1); (c) Operator-scope extensions (`~/.pi/agent/extensions/`) load by default but are **listed in `pi config`** for review; (d) On `reload()`, captured runtime references are invalidated; new instances are loaded fresh. |
| **Collaborators** | M3 (consumes lifecycle events); M4 (registerTool); M6 (registerMessageRenderer); M7 (registerCommand). |

### M9 — Auth Resolver

| Field | Value |
|---|---|
| **Responsibility** | Resolve LLM-provider credentials at provider-call time using a 5-level precedence; manage OAuth refresh under multi-instance contention. |
| **Public inputs** | Provider name; runtime override (optional); environment; ambient-credential providers (e.g., cloud SDK). |
| **Public outputs** | Resolved credential blob (api_key string or OAuth-bearer token). |
| **Owned state** | Cached credential file; refresh-lock state; in-memory token cache with expiry. |
| **Invariants** | (a) 5-level precedence: runtime override → file `api_key` → file `oauth` → env var → ambient fallback; (b) Credential file is **always written atomically** with mode 0600 (creation, sync, async); (c) OAuth refresh is **single-flight** across multiple instances (file-lock or distributed-lock); (d) **Lock is not held across the slow OAuth fetch** — fetch happens off-lock; lock guards only the file read-modify-write (diverging from defect 4.7 / S3.10); (e) Corrupt credential file produces a structured error and blocks credential reads until the user fixes it. |
| **Collaborators** | M1 (consumes resolved credential); host environment. |

### M10 — Browser Chat Component

| Field | Value |
|---|---|
| **Responsibility** | Embed the conversation UI in a browser host; consume M1 directly (no agent loop). |
| **Public outputs** | Custom-element registrations (~30); rendered chat UI. |
| **Owned state** | Browser-local conversation history (encrypted at rest if storing keys). |
| **Invariants** | (a) Custom elements communicate via documented properties / events / slots — no global state coupling; (b) Provider keys stored in browser state are **encrypted at rest** (diverging from defect 4.10); (c) HTML rendering goes through a sanitizer or trusted-types policy; (d) Streaming display preserves type fidelity (no `JSON.parse(JSON.stringify(...))` clones — diverging from defect 1.16). |
| **Collaborators** | M1 (called directly); host browser; consumer-supplied LLM-auth wiring. |

## Layer Split

| Module | Layer | Notes |
|---|---|---|
| M1 — Provider Adapter | core semantics | The streaming event-union contract is load-bearing across the system. |
| M2 — Provider Registry | core semantics | Dispatch table; minimal logic. |
| M3 — Agent Loop | core semantics | The interactive coding workflow IS the agent loop. |
| M4 — Tool Dispatcher | adapters | Filesystem + shell + search adapters with explicit consent gating. |
| M5 — Session Store | adapters | Filesystem persistence adapter (could swap for SQLite or other). |
| M6 — Render Loop (TUI) | delivery surfaces | Terminal-specific. |
| M7 — Slash Commander | delivery surfaces | TUI- and print-mode-specific. |
| M8 — Extension Host | delivery surfaces (with consent gate) | Treat as part of the CLI shell, not core. |
| M9 — Auth Resolver | adapters | Filesystem + env + cloud-SDK adapters. |
| M10 — Browser Chat Component | delivery surfaces | Independent of M3–M8. |

The `core semantics` layer is what survives the port unchanged: the
provider event union (M1) and the agent loop (M3) are the contract a
faithful port preserves. The `adapters` layer is where target-language
idioms take over — a Go port might use `context.Context` for
cancellation, a Rust port might use `tokio::sync::CancellationToken`,
etc. The `delivery surfaces` layer is freely re-shape-able.

## Required Behaviors

Derived from `findings/contracts/behavioral-contracts.md` §Feature
Contracts. The reimplementation must:

1. **LLM streaming**: produce a 12-variant async event stream from
   any supported provider, with FIFO ordering, terminal `done`/`error`
   events, abort yielding `error{reason:"aborted"}`, and tool-call
   coalescing keyed by `(streamIndex || id)`. Errors flow through the
   stream — `streamSimple()` does NOT throw on missing API key
   (diverging from defect 5.1).
2. **Provider selection**: dispatch by `model.api: KnownApi`
   discriminator; unknown api yields structured error.
3. **Agent run**: support `prompt(message)`, `continue()`, `subscribe`,
   `abort()`. Listeners run in registration order with bounded throw
   semantics.
4. **5-level auth precedence**: runtime override → stored api_key →
   stored oauth → env var → ambient fallback. Atomic file writes with
   mode 0600.
5. **7 built-in tools**: `read`, `bash`, `edit`, `write`, `grep`,
   `find`, `ls`. **Per-invocation consent gate** for `bash`, `edit`,
   `write` (overridable with `--yes-tools` flag). Tool subprocess exit
   codes never coerced. Auto-downloaded helper binaries
   checksum-verified.
6. **Session persistence**: append-only JSONL with full-rewrite for
   branch edits; atomic rewrites (temp + rename); flush on every
   event (no deferred-flush optimization). Hashed cwd. Ambiguous
   session-id prefixes rejected.
7. **Mode dispatch**: interactive TUI / non-interactive print /
   JSON-RPC over stdio. Stdin-piped TTY auto-falls-back to print
   mode.
8. **Cancellation**: Ctrl+C double-tap to exit; Esc to abort current
   stream; SIGTERM/SIGHUP exit cleanly with detached-child reap.
9. **Slash commands**: ~20 builtins + user-defined `.md` prompt
   templates (YAML frontmatter); discovery order agent-dir → cwd-dir →
   flag paths.
10. **TUI rendering**: differential paint, synchronized output mode 2026
    where supported; falls back gracefully when not. Width-overflow
    truncates rather than throws. EPIPE aborts session.
11. **Extension API**: ~22 lifecycle events; uniform try/catch on all
    emitters; project-scope extensions gated by `--trust-project`.
12. **OAuth refresh**: single-flight across multiple instances; lock
    not held across slow fetch.
13. **Compaction**: replace older entries with summary records when
    context-budget threshold reached; preserve branch-edit semantics.
14. **Browser chat component** (optional surface): embed the UI in a
    browser host; consume provider layer directly; encrypt provider
    keys at rest; sanitize HTML.

Acceptance scenarios in §Acceptance Scenarios below pin these
behaviors as black-box checks.

## Protocols and Persisted State

Resolves **port-CF1**. SM1–SM6 from
`findings/protocols/protocols-and-state.md` re-expressed as
language-agnostic FSMs.

### SM1' — Stream FSM (must preserve)

States: `idle`, `active`, `text_block`, `thinking_block`,
`toolcall_block`, `terminated`.

Transitions:
- `idle → active` on first `start` event.
- `active → text_block` on `text_start`; `text_block → text_block` on
  `text_delta`; `text_block → active` on `text_end`.
- Same shape for `thinking_block` and `toolcall_block`.
- Any state → `terminated` on `done` or `error`.
- Any state → `terminated` on abort (emits `error{reason:"aborted"}`).

A port must pin: terminal events resolve any pending `await`; events
after `terminated` are no-ops; tool-call coalescing keyed by
`(streamIndex || id)`.

### SM2' — Session Persistence FSM (design decision: no deferred flush)

States: `closed`, `open`, `closed`.

Transitions:
- `closed → open` on first event of a session.
- `open → open` on each event (write to file synchronously, atomically).
- `open → closed` on session dispose.

**Diverges from current source SM2** (which had an in-memory buffer
state pre-flush). The reimpl-spec position: **flush on every event**.
The latency cost is small at single-line append granularity; the
data-loss surface (5.7) is not acceptable.

### SM3' — Render FSM (must preserve)

States: `idle`, `building`, `compositing`, `diffing`, `emitting`,
`idle`.

Transitions on render-trigger (state change / SIGWINCH / timer).
Width-overflow → truncate (NOT throw). Width-change → full clear.
Height-change → full clear except in environments where it tears
(currently Termux; reimpl detects host environment).

### SM4' — Remote Agent Connection FSM (if remote-agent feature is
ported)

States: `disconnected`, `connecting`, `streaming`, `terminated`.

Transitions: connect via HTTP POST + standard SSE (NOT custom `\n`
framing — design decision diverging from current source's H10);
streaming events parsed; out-of-order delta/end **resyncs on next
start** (rather than terminating turn — diverging from 1.22 / 5.12).
Add **application-level keepalive** every N seconds to detect stale
connections (diverging from H11 / arch-OQ3).

### SM5' — OAuth Refresh FSM (must preserve, with fix)

States: `valid`, `refresh-needed`, `refreshing`, `valid`.

Transitions: provider 401 OR token close to expiry → `refresh-needed`
→ acquire single-flight lock → if another instance refreshed in the
interim, re-read; else `refreshing` → fetch new token (lock NOT held
across fetch) → atomically write to credential store → release lock →
`valid`.

**Diverges from current source SM5** in: lock not held across the slow
fetch (diverging from 4.7).

### SM6' — Extension Reload FSM (must preserve)

States: `running`, `shutting-down`, `rebuilding`, `invalidated`,
`starting`, `running`.

Transitions: on `reload()` → emit `session_shutdown(reload)` →
teardown → invalidate captured runtime references → re-import (no
module cache) → emit `session_start(reload)`.

### Persistent Schemas

#### Credential Store (analog of `auth.json`)

JSON object `{ <provider>: { type: "api_key" | "oauth", api_key?:
string, oauth?: {...} } }`. Mode 0600 / OS-equivalent ACL. Atomic
writes (temp + rename + chmod). Corrupt JSON → `loadError` blocks
credential reads.

#### Session Store (analog of JSONL v3)

Line-delimited JSON with a discriminated union of record types:

| Record | Discriminator | Notes |
|---|---|---|
| header | `type: "session"` | First record per file; carries session id, hashed cwd, parent session id (if branched), schema version |
| message | `type: "message"` | role: user/assistant/tool; content body |
| compaction | `type: "compaction"` | summary of older entries |
| branch_summary | `type: "branch_summary"` | branch fork point marker |
| thinking_level_change | `type: "thinking_level_change"` | mid-session toggle |
| model_change | `type: "model_change"` | mid-session model swap |
| custom | `type: "custom"` | extension-driven |
| custom_message | `type: "custom_message"` | extension-driven |
| label | `type: "label"` | named bookmark |
| user-display-name marker | `type: "session_info"` | user-defined display name |

(Per sem-CR2: header is `type: "session"`; `session_info` is the
display-name marker.)

Identifiers: session id is **uuidv7**; per-record id is an
8-character collision-resistant string. Timestamps are ISO-8601 with
millisecond precision.

Path encoding: `${hash(cwd)}/<session_id>.jsonl` — hashed (NOT lossy
string-replace) to avoid encoded-cwd collisions across OSes.

Atomic rewrite: temp + rename for branch operations and migrations.

#### Settings Store (analog of `settings.json`)

JSON object; ACID writes via OS-level lock. Schema validated on load.

#### Models Catalog (analog of `models.json`)

JSON object with model overrides + custom HTTP headers. **`!`-prefix
shell-exec mechanism is NOT preserved** (diverging from defect 4.2 /
P6.4); replace with explicit "shell command" header value type that
requires user consent at config-write time.

## External Dependencies

| Dependency | Stance | Rationale |
|---|---|---|
| LLM provider HTTPS APIs (Anthropic, OpenAI, Google, Bedrock, Mistral, Azure, etc.) | **wrap** | The provider-adapter layer (M1) is where vendor differences are isolated. Each vendor SDK gets a thin adapter. |
| HTTP client | **replace** (target-language idiomatic) | The current undici tweaks (defect P6.1) don't translate; use the target language's stdlib HTTP client with per-call timeouts. |
| File-locking primitive (current: proper-lockfile) | **replace** | Use the target language's idiomatic inter-process lock (e.g., `flock` on Linux, `LockFileEx` on Windows; or distributed lock if running multi-host). |
| OAuth flows (Anthropic Claude.ai, Google) | **wrap** | Provider-specific OAuth flows are spec-driven; implement them per the provider's documented protocol. |
| Search binaries (`rg`, `fd`) | **postpone** initially; **replace** if performance demands | MVP can use the host shell's `grep`/`find`. Optimize later with bundled binaries (checksum-verified). |
| Image processing (current: photon WASM) | **replace** | Use a non-monkey-patching image library native to the target language. |
| Terminal capability detection | **emulate** | Re-derive the existing capability matrix (synchronized output, Kitty / iTerm2 escapes) per target language; fall back gracefully. |
| Markdown rendering | **wrap** (target-language idiomatic) | The current mini-lit MarkdownBlock has a third-party-audit open question (sem-OQ6). A port should pick an audited markdown library + sanitizer. |
| jiti extension loader | **replace** | Target-language module loaders are different. Use the target's idiomatic dynamic-loading or compiled extension format. |
| undici (HTTP client tweaks) | **replace** | (See HTTP client row.) |
| Bun standalone binary mechanics | **replace** | Use the target's native binary build (Go `go build`, Rust `cargo build`, etc.). |
| Photon WASM bundling | **replace** | (See Image processing.) |
| koffi FFI (Windows VT input) | **replace** | Use native FFI of the target language (or, if target is Windows-friendly enough, no FFI needed). |

## Portability Hazards

The reverse-engineering bundle's §Portability Hazards table lists 24
hazards across the original source. Hazards specific to the
reimplementation effort:

1. **Concurrency model translation**: the current source uses async
   iterables (`EventStream` waiter queue). A port to a non-JS runtime
   may use channels (Go), streams (Rust async), Reactive Streams (Java),
   or other primitives. The contract (FIFO, terminal events,
   push-after-terminal no-op) must be preserved; the underlying primitive
   is implementation choice.
2. **Listener fan-out semantics**: the spec **diverges from current
   source defect 1.13** — listener throws are logged and skipped, not
   propagated. A port must take this position consciously and document
   it.
3. **Atomic file writes**: every persistent write must be temp +
   rename. A port must verify the target language's stdlib supports
   this on every OS the port targets (Windows is the usual problem).
4. **OS-level credential file mode**: 0600 on POSIX; equivalent ACL on
   Windows. The port's auth-storage layer needs OS-conditional code.
5. **Terminal capability detection**: the current source detects
   capabilities by env-var sniffing only. A port may use OSC queries
   for higher accuracy, but must handle the case of a non-responding
   terminal (timeout the query, fall back).
6. **Bracketed-paste timeout flush**: the port must implement a
   timeout-armed flush (the current source doesn't — defect 1.18).
7. **Streaming clone fidelity** (browser delivery): use a structured
   clone primitive that preserves `undefined`, `BigInt`, `Map`, `Set`,
   `Date`, etc. — not `JSON.parse(JSON.stringify(...))`.
8. **Multi-instance OAuth refresh**: the port's lock primitive must
   support both the "wait for lock" and "stale lock detection" semantics.
9. **Hash-based path encoding**: the port must pick a hash function
   (SHA-256 with hex truncation is sufficient) for cwd encoding;
   document that re-hash on hash-fn change is required.

## Implementation Sequence

### Scope Tiers

**Minimum viable port:**

- M1 (single provider — pick the highest-traffic vendor, e.g.,
  Anthropic or OpenAI Responses)
- M2 (registry skeleton)
- M3 (agent loop with `prompt`/`continue`/`subscribe`/`abort`)
- M5 (session persistence: atomic flush-on-event JSONL)
- M9 (auth resolver: env-var path only initially)
- A minimal CLI: read line from stdin, run, print events to stdout

This is enough to be "a working coding agent": the user types a
prompt, the agent calls the LLM, results stream back, transcript
persists. Tools, TUI, slash commands, multi-provider all postponed.

**Major-workflow parity:**

- M4 (Tool Dispatcher with all 7 built-in tools and per-call
  consent gating)
- M6 (TUI render loop with capability detection and graceful fallback)
- M7 (Slash Commander with the 20 builtins + prompt-template loading)
- M9 (full 5-level precedence, OAuth refresh)
- Mode dispatch (interactive / print / RPC)
- 3+ providers in M2

This is "the original product, faithfully reimplemented." A user
familiar with the original could sit down at this and feel at home.

**Full parity:**

- M8 (Extension Host with consent-gated project scope)
- All 9 production providers in M2 + the OpenAI-compat URL matrix
- M10 (Browser Chat Component)
- Compaction with branch-summary records
- Subcommands (install / update / config)
- Bundled / checksum-verified search binaries
- Bun-binary-style standalone distribution

### Build Order Within a Tier

Work bottom-up: M1 + M2 + M9 (auth) before M3 (agent loop); M5
(session) before M3 needs durable persistence; M3 before M6 (TUI
needs an agent to render); M4 (tools) any time after M3.

## Acceptance Scenarios

Black-box, language-neutral.

| # | Scenario | Input | Expected Output / Side Effect |
|---|---|---|---|
| 1 | Streaming a simple prompt | TTY stdin; valid env-var API key for some provider; user types "hello" + enter | Agent emits a `start` event, then text-delta events, then `done`; full assistant message visible in stdout/TUI; one new JSONL session file created (or appended) under the operator state directory; session contains a header record + the user message + the assistant message. |
| 2 | Provider missing API key | No env-var, no stored credential, no runtime override | Agent emits an `error` event in the stream (NOT a synchronous throw); UI shows "missing API key for provider X". No partial assistant message in transcript. |
| 3 | Stdin piped (non-TTY) | `echo "summarize this file" \| pi --print path/to/file` | Mode auto-falls back to print; agent runs to completion; stdout contains assistant message; exit code 0 on success. |
| 4 | Corrupt credential file at startup | Credential file exists but is invalid JSON | Process starts; UI shows "credential file is corrupt; please fix"; credential reads return error until user fixes the file; no crash. |
| 5 | Ambiguous session-id prefix | Two existing sessions both start with `ab` | `pi --session ab` returns an error: "ambiguous prefix; matches sessions abc... and ab1...". (Diverges from defect 1.6.) |
| 6 | Esc during streaming | Active stream, queued messages | Pressing Esc aborts the stream; an `error{reason:"aborted"}` event is emitted; queued messages are restored to the editor (not lost). |
| 7 | Pre-first-assistant crash | User sends prompt; process killed before LLM responds | The user prompt **is in the session JSONL on disk** (diverges from defect 5.7). Resuming the session shows the prompt. |
| 8 | Edit-tool mid-write crash | LLM tool-call invokes `edit` on `foo.ts`; process killed mid-write | `foo.ts` is **either fully old or fully new** (no truncation). (Diverges from defect 2.2 / 5.8.) |
| 9 | Bash tool killed | Tool spawns subprocess; subprocess receives SIGKILL | Tool result reports a non-zero/signal exit indicator (NOT 0). (Diverges from defect 1.12 / 5.13.) |
| 10 | Bash tool consent | LLM invokes `bash` with command `rm -rf /tmp/foo` | UI shows consent prompt: "Run command? Y/n". User must accept; with `--yes-tools` flag, run unconditionally. (Resolves port-CF4.) |
| 11 | OAuth refresh under contention | Two instances, both with stale OAuth tokens | Both call provider; single-flight refresh; one refreshes, the other reads the new token; no duplicate token-write to file. |
| 12 | Project-scope extensions in untrusted repo | Hostile repo contains `<repo>/.pi/extensions/evil.ts` and `<repo>/.pi/settings.json` with `packages: ["evil-pkg"]` | `pi` does NOT auto-load. UI shows: "This project has extensions / packages. Trust this project? (Y/n)". With `--trust-project` flag, load. Without, default-deny. (Resolves port-CF3 + diverges from defect 4.1.) |
| 13 | Extension throws in `tool_call` hook | Project- or operator-scope extension's `tool_call` hook throws | Throw is logged; subsequent `tool_call` handlers in the chain still fire (uniform fan-out semantics — diverges from defect 5.9). Tool call **proceeds** (hook is not blocking). |
| 14 | Session resume | Existing session id `abc123` | `pi --session abc123` loads the transcript; new turn appends to the same JSONL file. |
| 15 | Session fork | Existing session id `abc123` | `pi --fork abc123` creates a new JSONL file with the path-to-leaf and a `parentSessionId` in the header. |
| 16 | Width-overflow render | TUI line exceeds terminal width (e.g., a long URL) | Line is **truncated** and rendered (NOT process kill). (Diverges from defect 1.15 / 5.14.) |
| 17 | EPIPE on stdout | TUI runs over SSH; SSH connection drops mid-paint | Session **aborts** (NOT silently continues headless). (Diverges from defect 2.11.) |
| 18 | rg/fd binary download | First-run; helper binaries absent | Download from a pinned URL; verify checksum against a pinned hash; **abort with error if checksum mismatches**. (Diverges from defect 4.4.) |
| 19 | `models.json` shell-exec attempted | User config has a `headers` value with a shell-exec syntax (e.g., legacy `!`-prefix) | Loader rejects the value: "shell-exec mechanism is not supported; use a static value or environment variable". (Diverges from defect 4.2 / P6.4.) |
| 20 | OAuth refresh slow endpoint | Provider's token endpoint takes 60s to respond | Fetch happens **off-lock**; another instance can read+write credential file during the 60s without lock conflict. (Diverges from defect 4.7 / S3.10.) |

## Deliberate Non-Goals

- **The OpenAI-compatible URL-detection matrix for the long-tail of
  ~10 vendors** is NOT preserved as-is. The port reimplements the
  registry skeleton and the top 3 vendors (OpenAI Responses, Anthropic
  Messages, Google Generative AI). Adding more vendors is a
  post-1.0 plugin opportunity.
- **The Bun standalone binary mechanism** (with `koffi` externalized
  + `photon_rs_bg.wasm` bundling + Windows `.node` shipping) is NOT
  preserved. The port uses target-language native binaries.
- **The custom APC `CURSOR_MARKER`** (`\x1b_pi:c\x07`) is NOT
  preserved; replace with the standard cursor-position-report (CPR
  `\x1b[6n`).
- **The streamProxy CLIENT (remote-agent feature)** is OPTIONAL — it
  ships in full-parity tier with canonical SSE framing (NOT the
  current `data: <json>\n` custom framing) and an application-level
  keepalive.
- **The contributor-gate workflow trio** (issue-gate / pr-gate /
  approve-contributor) is NOT preserved. Replace with the new repo's
  contribution model.
- **Photon WASM image processing** is NOT preserved. Either omit the
  feature entirely or replace with a non-monkey-patching native
  library.
- **The 16,815-line generated `models.generated.ts`** is NOT
  preserved as-is. The port either re-generates a similar catalog from
  the same upstream sources (models.dev / Vercel AI Gateway /
  Cloudflare AI Gateway) at build time, or fetches the catalog at
  runtime.
- **Lockstep versioning across 5 packages** is NOT preserved
  (depending on the port's repo strategy — single binary vs library
  set).

## Known Unknowns

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| reimpl-OQ1 | needs-spike | Best-fit concurrency primitive for the target language's `EventStream` analog (channel? async iterator? Reactive Streams?) — needs prototype to confirm waiter-queue semantics translate. | Target-language choice not yet locked (per Strategic Alignment Hook = language-agnostic); spike per target. |
| reimpl-OQ2 | needs-runtime-test | Acceptable latency cost of "flush on every event" (vs deferred-flush). The current source's deferred-flush is presumably for performance. The spec position is "fix the data-loss surface, accept latency cost"; quantify the cost via a spike. | Performance test required against the new build. |
| reimpl-OQ3 | needs-maintainer-decision | Per-call user consent for `bash`/`edit`/`write` will materially change the UX. Should the consent model be: (a) always prompt, (b) remember per-session, (c) remember per-tool-+-arg-pattern, (d) flag-controlled `--yes-tools`? | Maintainer policy on UX trade-off. |
| reimpl-OQ4 | needs-runtime-test | Terminal capability detection accuracy: env-var sniffing vs OSC query. Some terminals lie or don't respond. | Test against a matrix of terminals. |
| reimpl-OQ5 | needs-spike | Multi-host OAuth refresh — does the port need distributed-lock support, or is single-host (filesystem lock) sufficient? | Depends on port deployment model. |
| reimpl-OQ6 | needs-maintainer-decision | Should the port preserve `streamProxy` (remote-agent) feature? It's optional in scope tier; if shipped, must use canonical SSE + keepalive. | Product decision per port. |
| reimpl-OQ7 | needs-spike | Browser chat component: do we ship the same custom-element API, or shift to a different browser framework? Lit is a reasonable choice for a port; React/Vue/etc. are alternatives. | Target ecosystem per port. |

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| reimpl-PostCF1 | spike | Performance prototype of "flush on every event" vs "buffered with periodic flush" to quantify reimpl-OQ2. | Empirical measurement required to validate the spec position. |
| reimpl-PostCF2 | delta | If a future framework revision adds an explicit `correct-prior-phase` carry-forward kind (per closeout feedback), apply sem-CR1 / sem-CR2 with that mechanism. | Framework feedback already filed; awaiting framework update. |
| reimpl-PostCF3 | spike | Distributed-lock prototype for multi-host OAuth refresh (reimpl-OQ5). | Empirical validation. |

(Post-pipeline target_phase values per template: `spike`, `delta`,
`amendment`. The reimpl-spec is the terminal phase in this pipeline.)

## Spike List

1. **EventStream concurrency primitive spike**: prototype the 12-variant
   stream in 2-3 candidate target languages (Go, Rust, Python). Confirm
   FIFO + terminal-event + push-after-terminal-no-op + abort semantics
   translate. Target ≤ 200 LOC each.
2. **Atomic file write spike**: prototype temp-write + rename on
   Windows in the candidate languages. Confirm semantics work as
   expected (Windows historically had issues with rename atomicity
   under share locks). Target a small benchmark vs naive `writeFile`.
3. **Per-call tool consent UX spike**: build a TUI with `bash`/`edit`/
   `write` tools and a consent prompt; A/B test "always prompt" vs
   "remember per-session" vs "flag-controlled" with N users. Target a
   quick paper-prototype + log analysis from heavy users of the
   original.
4. **Terminal capability detection spike**: build a capability prober
   that combines env-var sniffing + OSC queries with timeouts. Test
   against ≥ 8 terminal emulators (xterm, iTerm2, Kitty, Alacritty,
   WezTerm, Windows Terminal, GNOME Terminal, VS Code integrated
   terminal). Target a capability matrix.
5. **Streaming clone fidelity spike** (browser delivery): benchmark
   structured-clone vs naive JSON-clone on real streaming workloads.
   Target a simple benchmark.
6. **OAuth single-flight spike**: benchmark file-lock-based
   single-flight under simulated multi-instance contention. Target a
   simple stress test that exercises the >30s slow-fetch case.
7. **`<cwd>/.pi/` consent UX spike**: A/B test default-deny + prompt
   vs `--trust-project` flag vs "interactive prompt with remember per-
   directory". Target heavy-user feedback.

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | Concept-level modules are defined. | PASS | §Conceptual Module Model defines 10 modules (M1–M10) with all 6 required fields populated for each. |
| 2 | Required behaviors are stated. | PASS | §Required Behaviors lists 14 numbered must-do items, each derived from contracts F-* IDs and protocols SM*/EC* IDs. |
| 3 | Protocol and persisted state expectations are stated. | PASS | §Protocols and Persisted State re-expresses SM1-SM6 as language-agnostic SM1'-SM6' (resolves port-CF1) plus 4 persistent schemas (credential store, session store, settings store, models catalog). |
| 4 | Acceptance scenarios and known unknowns are included. | PASS | §Acceptance Scenarios lists 20 black-box scenarios; §Known Unknowns lists 7 reimpl-OQs (the terminal phase's open-questions list). |
| 5 | Defects identified in either scan are explicitly designed-around or noted as "left behind", with the choice cited. | PASS | All 5 critical defects + 14 "port differently" hazards are addressed. Specific: 1.1 (M1 invariants), 1.6/5.6 (M5 invariants), 1.13/5.10 (M3 invariants), 1.15/5.14 (M6 invariants), 1.18 (M6 invariants), 1.19/5.5 (M3 invariants), 2.1/5.8 + 2.2 (M5 + M4 invariants), 2.11 (M6 invariants), 1.22/5.12 (SM4'), 4.1 (M8 invariants — port-CF3 resolved with default-deny + --trust-project), 4.2/P6.4 (Models Catalog: `!`-prefix NOT preserved), 4.4 (M4 invariants — checksum verification), 4.6 (M9 invariants), 4.7/S3.10 (M9 invariants — lock not held across fetch), 5.1 (M1 invariants — errors via stream), 5.7/dsm-RT6 (M5 + SM2': flush on every event — port-CF5 resolved), 5.9 (M8 invariants — uniform fan-out), 5.13 (M4 invariants), P6.1/P6.2 (External Dependencies stance: replace), 4.3 (M4 invariants — per-call consent — port-CF4 resolved). The 14 "port differently" hazards are addressed in §External Dependencies and §Deliberate Non-Goals. The 4 "leave behind" hazards are listed in §Deliberate Non-Goals. |
| 6 | Findings are marked with evidence levels. | PASS | The spec is itself a synthesis of prior phases' findings (each tagged); the spec annotates its design decisions with citation back to the defect or phase it diverges from (e.g., "diverging from defect 1.13/5.10", "per sem-CR2"). |

**Validated by:** 2026-05-06 (reimplementation-spec phase, session 1, no-orchestrator mode)
**Overall:** PASS

(All 5 inbound carry-forwards from porting resolved: port-CF1 SM1'-SM6'; port-CF2 designed-around / left-behind for all 5 critical + 14 'port differently' + 4 'leave behind'; port-CF3 default-deny + --trust-project; port-CF4 per-call consent for bash/edit/write; port-CF5 flush-on-every-event. All maintainer-policy positions taken with citation back to the defect that triggered the decision. Strategic Alignment Hook locked at language-agnostic — opinionated template not used.)
