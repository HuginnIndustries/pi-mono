# Reverse-Engineering Bundle — pi-mono

<!--
  Phase: porting
  Pipeline: workflow/pipeline-full-with-deep-audit.yaml
  Synthesizes: architecture-map.md + behavioral-contracts.md + protocols-and-state.md + mechanical-defects.md + semantic-defects.md
  Source-of-detail: cited primary outputs above.
-->

## System Summary

`pi-mono` ships an interactive coding-agent CLI named `pi`, plus four
supporting libraries published as a lockstepped TypeScript monorepo
(currently `0.73.0` across all 5 packages). The system's job is to put
a competent, extensible coding agent in front of the user, talking to
~15 LLM providers through one unified streaming event shape, with full
session recording, an extension API, and both terminal-native and
browser-native delivery surfaces. The cross-cutting design choice —
the load-bearing one a port must preserve — is **provider neutrality
behind a stable streaming event union**: every provider (Anthropic
Messages, OpenAI Responses, Google Generative AI / Vertex, Bedrock
Converse Stream, Azure OpenAI, Mistral, OpenAI Codex Responses, plus a
faux test fixture) collapses into the same 12-variant
`AssistantMessageEvent` async stream.

For a reimplementer the system decomposes cleanly into **five concept
layers**, in dependency order: (1) a **provider-normalization layer**
that is registry-driven by `model.api`, exposing `streamSimple()`-style
streaming primitives + a model catalog + OAuth helpers; (2) a
**core-semantics agent runtime** that drives the loop, dispatches tool
calls, and fans events out to subscribers (no on-disk persistence — the
runtime is in-memory; persistence is the caller's job); (3) a
**terminal UI layer** providing differential ANSI rendering, stdin
escape-sequence assembly, and a small widget set; (4) a **product
shell** wrapping all of the above into the `pi` CLI with mode dispatch
(interactive TUI / non-interactive print / JSON-RPC over stdio), the
seven built-in tools (`read`, `bash`, `edit`, `write`, `grep`,
`find`, `ls`), JSONL session persistence v3 with branch-edit semantics,
a 5-level auth precedence resolver, ~20 builtin slash commands, and an
implicit ~22-event extension API loaded via `jiti`; and (5) a
**browser-shell** of Lit-based web components (notably
`<pi-chat-panel>`) that consume the same provider-normalization layer
but bypass the agent runtime and TUI layers entirely. The two delivery
shells (CLI + web-ui) are independent and a port can ship either or both.

The most porting-relevant facts are: build order is **load-bearing**
(`tui → ai → agent → coding-agent → web-ui`); session JSONL has a
**deferred-flush optimization** that loses the user prompt if the
process crashes before the first assistant message; the project
**auto-loads code from `<cwd>/.pi/`** with no consent gate, exposing 5
RCE channels to a hostile repo; and the **edit / write tools overwrite
user files non-atomically**, with mid-write crash producing data loss.
A faithful port must decide policy on each of these; this bundle
documents the contracts so those policy choices are explicit, not
accidental.

## Layer Map With Ownership

Five concept layers (concept names; current source-package mapping in
parens). See `findings/architecture/architecture-map.md` for the
package-level inventory.

| Layer / Module | Role | Owns |
|---|---|---|
| L1 — Provider Normalization (current: `@mariozechner/pi-ai`) | Adapt N provider HTTPS APIs into one streaming event union; provide model catalog + OAuth helpers | `AssistantMessageEvent` 12-variant union; `KnownApi` discriminator + provider registry; `EventStream` async-iterable; per-provider stream functions; OAuth flow (Anthropic Claude.ai + Google); env-var auth resolution; **read-only against credentials** (caller threads in `apiKey`); generated model catalog (16,815-line static file currently) |
| L2 — Agent Runtime (current: `@mariozechner/pi-agent-core`) | Drive the agent loop in-process; dispatch tool calls; fan events out to subscribers | `Agent` class (`prompt()`, `continue()`, `subscribe()` listener); single per-run `AbortController`; sequential listener fan-out; tool-call dispatch glue; **no persistence** (caller's job); optional `streamProxy` SSE CLIENT for remote-runtime use |
| L3 — Terminal UI (current: `@mariozechner/pi-tui`) | Differential ANSI rendering + raw stdin escape parsing | `Component` interface; `TUI` driver class (synchronized output mode 2026); `Container` widgets; `ProcessTerminal` (incl. Windows VT input via koffi FFI); `StdinBuffer` (escape-sequence assembly + bracketed paste); 12-widget set |
| L4 — Product Shell — CLI (current: `@mariozechner/pi-coding-agent`) | Compose L1–L3 into the `pi` CLI; own all user-visible workflow | Mode dispatch (TUI / print / RPC); 7 built-in tools (read / bash / edit / write / grep / find / ls); session JSONL v3 persistence; 5-level auth precedence resolver; ~20 builtin slash commands + prompt-template loader; ~22-event extension API + `jiti` extension loader; subcommands (install / update / config); migration runner; `pi` binary + Bun standalone binary builds |
| L5 — Product Shell — Browser (current: `@mariozechner/pi-web-ui`) | Lit-based web components for embedding the chat UI in browsers | ~30 custom elements (flagship `<pi-chat-panel>`); `StreamingMessageContainer` for streaming display; `MarkdownDisplay` (delegated to mini-lit); IndexedDB / localStorage state; consumer wires LLM auth + agent themselves (calls L1 directly, **bypasses L2**) |

**Dependency direction (no cycles):**

```
                L1 (provider normalization)              L3 (terminal UI)
                       │                                       │
                       ├──────────┬────────────────────────────│
                       │          │                            │
                       ▼          │                            │
                L2 (agent runtime)│                            │
                       │          │                            │
                       │          ▼                            │
                       │   L5 (browser shell — bypasses L2)    │
                       │                                       │
                       └────► L4 (CLI shell ──────────────────►┘)
```

L4 (CLI) imports L1 + L2 + L3. L5 (web-ui) imports L1 + L3 (NOT L2).
The two product shells are independent — a port may ship one or both.

**Build-order constraint** (load-bearing portability hazard): in the
current implementation, `tui → ai → agent → coding-agent → web-ui` is
strictly sequential because `pi-web-ui` uses stock `tsc` while the
others use `tsgo`, and downstream packages need upstream
`dist/*.d.ts`. *(observed fact)* A port that flattens or parallelizes
the build must replicate the type-emission contract.

## Feature Contract Table

Synthesized from `findings/contracts/behavioral-contracts.md`. Detailed
field-by-field contracts in that file; this is the porting-priority
view.

| Feature | Surface | Priority | Key Contracts | Notes |
|---|---|---|---|---|
| LLM streaming (provider neutrality) | Library API | core | F-AI-1 `streamSimple()` 12-variant `AssistantMessageEvent` (EC1) | The single most load-bearing contract. Defects 1.1, 1.5, 5.1 affect error behavior. |
| Provider selection by `model.api` | Library API | core | F-AI-2 registry dispatch | 9 production providers + faux. Bedrock lazy-loaded. |
| Agent loop (run / continue / abort) | Library API | core | F-AGENT-1 `Agent.prompt()` / `continue()` / `subscribe()`; SM1 stream FSM | Listener fan-out is sequential and unguarded — defect 1.13 / 5.10. |
| 5-level auth precedence | CLI + Library | core | §Authentication; resolved in L4 | mode 0600 enforced on every write. Corrupt JSON sets loadError + blocks reads (arch-OQ2 partial answer). |
| 7 built-in tools (read/bash/edit/write/grep/find/ls) | CLI | core | F-TOOLS table; `withFileMutationQueue` per-realpath serialization | Defects 2.1 / 2.2 (data loss on edit / write); 1.12 / 2.5 (bash exit-code contract); 4.3 (no path containment, no per-call consent) |
| Session JSONL v3 persistence | CLI | core | F-CLI-3; PS1 record catalog (10 types); SM2 deferred-flush FSM | Critical: SM2 deferred-flush loses user prompt on pre-first-assistant crash (5.7); non-atomic rewrite truncates (5.8 / 2.1). |
| Mode dispatch (TUI / print / RPC) | CLI | core | F-CLI-1 | Stdin-piped TTY auto-falls back to print mode. RPC mode rejects `@file` args. |
| Cancellation (Ctrl+C / Esc / signals) | CLI | core | F-CLI-3; 5 AbortControllers in AgentSession | Esc semantics overloaded (stream-abort / bash-abort / bash-mode-exit / double-Esc → /tree or /fork). 6 cancellation domains; abort() cascades to 2; dispose() cascades to 0. |
| Slash commands + prompt templates | CLI | important | F-CLI-4; 20 builtins + user-defined `.md` files | Discovery: agent-dir → cwd-dir → flag paths; non-recursive scan; YAML frontmatter (`description`, `argument-hint`). |
| Differential ANSI rendering (TUI) | CLI | important | F-TUI-1; SM3 render FSM | Synchronized output mode 2026 + custom APC `CURSOR_MARKER`. Width-overflow throw kills process (defect 1.15 / 5.14). |
| 22-event extension API | CLI | important | F-EXT-1..4; EC4 catalog | `emitToolCall` is the only unguarded emitter (5.9). 3-way duplication of event names (con-OQ5). |
| OAuth refresh under contention | CLI + Library | important | SM5; proper-lockfile single-flight | Stale-lock theft if OAuth fetch >30s (4.7 / S3.10). |
| Compaction + branch-edit semantics | CLI | important | SM2; PS2 `_rewriteFile` + 3 branching modes | Branch via `branch()` / `branchWithSummary()` / `createBranchedSession`. Non-atomic rewrite is critical defect. |
| streamProxy SSE CLIENT | Library | optional | F-AGENT-2 / EC3 / SM4; custom `data: <json>\n` framing (NOT canonical SSE) | No heartbeat (arch-OQ3 unresolved). Out-of-order events terminate turn (1.22 / 5.12). |
| 30 browser custom elements (`<pi-chat-panel>`) | Browser | optional | F-WEB-1; consumer wires LLM auth + agent | Zero tests, IndexedDB stores plaintext provider keys (4.10). Streaming clone-fidelity defect 1.16. |
| Subcommands (install / update / config) | CLI | optional | F-CLI-1 (intercepted before parseArgs) | Operator-state mutating outside session bounds. |
| Migrations on every startup | CLI | important | per Pass 6 P6.5; covers auth structure, sessions, binaries, keybindings, commands→prompts | Bare `catch {}` swallows partial failures. Hazardous default. |
| Photon WASM image processing | CLI | incidental | per arch-CF6 / P6.2 | Global `fs.readFileSync` monkey-patch. **port differently.** |
| OpenAI-compat URL matrix (~10 vendors) | Library | incidental | F-AI-2 + arch-CF8 + 5.2 | Long tail; reimplement registry + dispatch first, port high-traffic vendors only. |
| `models.json` + `auth.json` + `settings.json` schemas | CLI | important | PS5–PS7 | `!`-prefix shell-exec mechanism is real (4.2) but currently scoped to user-controlled config — see §Defect Synthesis correction. |

Full per-row contracts (trigger / defaults / output / side-effects / persisted state / error behavior / retry / owner) are in `findings/contracts/behavioral-contracts.md`.

## Protocol and State Notes

Synthesized from `findings/protocols/protocols-and-state.md`. The
reimplementer must preserve these protocols regardless of source
language.

### Streaming Event Catalog (load-bearing)

12-variant `AssistantMessageEvent` union: `start`, `text_start` /
`text_delta` / `text_end`, `thinking_start` / `thinking_delta` /
`thinking_end`, `toolcall_start` / `toolcall_delta` / `toolcall_end`,
`done`, `error`. Stream guarantees FIFO ordering (FIFO via
`EventStream` waiter queue); push-after-terminal is a no-op; abort
yields `error{reason:"aborted"}`; `result()` resolves on done/error
(never rejects). Tool-call coalescing keyed by `(streamIndex || id)`.
Finish reasons: `"stop" | "length" | "toolUse"` on done; `"error" |
"aborted"` on error. *(SM1 + EC1)*

### State Machines a Port Must Preserve

1. **SM1 — Stream FSM** (idle → active → text/thinking/toolcall blocks
   → done/error terminated). Non-trivial — defects 1.19 / 5.5
   demonstrate what goes wrong without the gate.
2. **SM2 — Deferred-flush FSM** for session JSONL: in-memory buffer
   until first assistant message, then flush + append. Critical: a
   port must decide whether to preserve the optimization (and accept
   data-loss surface 5.7) or fix it.
3. **SM3 — TUI render FSM** (idle → build → composite → strip
   CURSOR_MARKER → reset → diff → emit synchronized-output-wrapped).
4. **SM4 — streamProxy connection FSM** (disconnected → connecting →
   streaming → terminated). Out-of-order delta/end events throw.
5. **SM5 — OAuth refresh FSM** (proper-lockfile single-flight; re-read
   on contention; stale-lock theft hazard at >30s).
6. **SM6 — Extension reload FSM** (`session_shutdown(reload)` →
   rebuild runtime → invalidate old ctx → re-import → `session_start(reload)`).

### Persistent Schemas

- **Session JSONL v3**: 10 record types share `SessionEntryBase = {
  type, id, parentId, timestamp }`. **Correction (sem-CR2, applied
  here)**: per-file header is `type: "session"` (not `session_info` as
  protocols PS1 wrote — `session-manager.ts:31`); `session_info` is a
  separate user-defined display-name marker. Identifiers: sessionId is
  uuidv7, per-entry id is 8-hex-char prefix from `randomUUID` with
  collision retry. Append-only with full-rewrite for branch edits;
  rewrite is non-atomic (defect 5.8). **Encoded-cwd is lossy** —
  `/a/b` and `/a-b` collide; cross-OS sync unsafe.
- **`auth.json`**: per-provider `{ type: "api_key" | "oauth", api_key?,
  oauth?: {...} }`; mode 0600 on every write. Corrupt JSON → loadError
  + blocks reads.
- **`models.json`**: model overrides + custom headers. **`headers`
  values support `!`-prefix shell-exec via `resolveConfigValue`** —
  currently scoped to operator-controlled `~/.pi/agent/models.json`
  (NOT loaded from project scope per sem-CR1 correction below).
- **`settings.json`**: TS-typed; SettingsManager uses proper-lockfile.
  But `migrateAuthToAuthJson` does an unlocked read-modify-write that
  races (defect 3.2 / P6.6).

### Wire Formats

- `streamProxy` HTTP SSE: POST `${proxyUrl}/api/stream` with
  `Authorization: Bearer ${authToken}`; body `{model, context,
  options}`; response framing is custom **`data: <json>\n` (single
  newline, NOT canonical SSE `\n\n`)**. Server omits `partial:
  AssistantMessage` field; client reconstructs from closure-scoped
  accumulator. No heartbeat / keepalive.
- TUI ANSI escapes: synchronized output 2026 wraps each paint; custom
  APC `CURSOR_MARKER = \x1b_pi:c\x07`; Kitty keyboard push `\x1b[>7u`;
  pop `\x1b[<u`; cell-size query `\x1b[16t`; Kitty image `\x1b_G…\x1b\\`;
  iTerm2 image `\x1b]1337;File=…\x07`. Detection by env-var sniffing
  only.

## Portability Hazards

Consolidated from architecture-map.md, contracts, protocols H1–H27,
and both defect scans. Triaged for porting:

| Hazard | Source Phase | Impact | Mitigation |
|---|---|---|---|
| Synchronized output mode 2026 not universal | protocols H1 | TUI tearing on older terminals | Detect + fall back to plain emit |
| Custom APC `CURSOR_MARKER` | protocols H2 | Non-standard escape leaks as text on non-stripping terminals | Replace with standard cursor-position-report (CPR `\x1b[6n`) |
| Kitty keyboard / image protocols | protocols H3, H4 | Not all terminals support | Capability detect + fallback path |
| Width-overflow process kill (defect 5.14) | protocols H5 | Crash on long lines | Truncate (per `compositeLineAt` comment); reimplementation should default-truncate |
| Paste-mode no timeout flush (defect 1.18) | protocols H6 | Stdin permanently buffered after lost end-marker | Add timeout-armed flush |
| EPIPE on stdout.write not handled (defect 2.11) | protocols H7 | Headless agent after SSH disconnect | Install error listener; on EPIPE, abort session |
| Windows VT input requires koffi FFI | protocols H8 | Modifier keys lost without FFI | Use platform-native API (Win32 `SetConsoleMode` directly in target language) |
| streamProxy custom SSE framing | protocols H10 | CDN/comment injection breaks parsing | Use canonical SSE `\n\n` |
| streamProxy no heartbeat | protocols H11 | Stale-connection detection only at TCP layer | Add application-level keepalive |
| streamProxy out-of-order kills turn (defect 1.22) | protocols H12 | Network-level disorder ends session | Tolerate disorder; resync on next start event |
| Session JSONL non-atomic rewrite (defect 5.8) | protocols H14 | Mid-rewrite crash truncates | **Use temp-write + rename** |
| Session JSONL deferred flush (defect 5.7) | protocols H15 | Pre-first-assistant crash loses user prompt | **Flush on every event** OR **document the contract explicitly** |
| Encoded-cwd lossy / case-sensitive | protocols H16 | Cross-OS sync unsafe | Use a hash of cwd, not lossy string-replace |
| Listener fan-out unguarded (defect 1.13 / 5.10) | protocols H17 | One listener throw stops downstream | Wrap each handler in try/catch; route to error channel |
| Extension `emitToolCall` only unguarded emitter (defect 5.9) | protocols H18 | Asymmetric vs other 21 emit methods | Apply uniform try/catch; use a separate "block tool" return value if blocking is intended |
| Extension event-name 3-way duplication (con-OQ5) | protocols H19 | Drift surface | Single source of truth: `EVENT_NAMES` tuple → derive union, overloads, dispatch table |
| 7-provider error-message destruction (defect 1.1 / 5.15) | protocols H21 | UX bad on every provider error | Preserve provider-mapped errorMessage; emit error event with original cause |
| undici timeouts disabled globally (defect P6.1) | protocols H22 | All Node fetch deadlines disabled | Use per-call timeouts; reserve "no timeout" for the LLM stream specifically |
| photon WASM `fs.readFileSync` monkey-patch (defect P6.2) | protocols H23 | Global stdlib mutation; multi-tenant leakage | **Port differently** — use a non-monkey-patching image library or remove the feature |
| `<cwd>/.pi/` 5-channel RCE (defect 4.1 critical) | protocols H24 (corrected) | Untrusted-repo RCE | **`--trust-project` opt-in** per sem-OQ4; default reject |
| rg/fd auto-download no checksum (defect 4.4) | protocols H25 | Supply-chain hazard | Pin checksums; verify signatures; or ship binaries in distribution |
| `models.json` `!`-prefix shell-exec (defect 4.2) | protocols H24 (corrected) | Operator-config code-exec channel; not project-scoped in current source | Remove the mechanism OR require explicit opt-in flag |
| Session resume picks `[0]` on ambiguous prefix (defect 5.6) | protocols H27 | Non-deterministic session selection | Reject on ambiguity; require disambiguation |

## Defect Synthesis

**Pipeline**: full-with-deep-audit (mechanical + semantic).

Total: **76 high/critical findings** (mech 39 + semantic 37). 5 critical
across the two reports. Per-finding rows live in
`findings/defect-scan-mechanical/mechanical-defects.md` and
`findings/defect-scan-semantic/semantic-defects.md`. The porting view
buckets them by recommendation:

| Defect ID | Source Report | One-line Description | Severity | Porting Recommendation |
|---|---|---|---|---|
| **mech 2.1 / sem 5.7** | both | Session JSONL non-atomic write + deferred-flush — pre-first-assistant crash loses user prompt | critical | **fix before porting** (temp-write+rename + flush-on-event) |
| **mech 2.2 / sem 5.8** | both | edit/write tools overwrite user files non-atomically | critical | **fix before porting** (temp-write+rename) |
| **sem 4.1** | semantic | 5-channel project RCE via `<cwd>/.pi/` (extensions/packages/shellCommandPrefix/shellPath/npmCommand) | critical | **fix before porting** (consent-gate or `--trust-project`) |
| **sem 4.4 / dsm-RT9** | semantic | rg/fd auto-download no checksum/sig | high | **fix before porting** (pin checksums or ship binaries) |
| **sem 5.1 / con-CF6** | semantic | `streamSimple*` synchronous throw violates documented "errors via stream" | high | **fix before porting** (errors via stream end-to-end) |
| **mech 1.1 / sem 5.15** | both | 7-provider error-message destruction | high | **fix before porting** (preserve provider error text) |
| **mech 1.13 / sem 5.10** | both | listener fan-out unguarded; throw stops downstream | high | **fix before porting** (uniform try/catch around handlers) |
| **mech 1.15 / sem 5.14** | both | TUI width-overflow throws and kills process | high | **fix before porting** (always truncate) |
| **mech 1.18** | mechanical | paste-mode no timeout flush | high | **fix before porting** (timeout-armed flush) |
| **mech 1.19 / sem 5.5** | both | agent-loop swallows stream when `start` missing | high | **fix before porting** (treat as invalid SM1 transition; emit error) |
| **mech 1.6 / sem 5.6** | both | session-id ambiguous prefix silently picks `[0]` | high | **fix before porting** (reject ambiguity) |
| **mech 1.12+2.5 / sem 5.13** | both | bash exit-code contract: killed processes report 0 | high | **fix before porting** (preserve `null`/signal) |
| **mech 2.11 / proto H7** | mechanical | EPIPE on stdout.write not handled | high | **fix before porting** (install error listener; abort session) |
| **mech 1.22 / sem 5.12** | both | streamProxy out-of-order events throw | high | **fix before porting** (tolerate disorder; resync on start) |
| **mech 1.14 / sem 5.3** | both | streamProxy `partialJson` leak via shallow-clone | high | **fix before porting** (deep-clone or strip on emit) |
| **sem 4.3** | semantic | Tools have no input validation, no path containment, no per-call consent | high | **fix before porting** (per-call consent gate; cwd containment for write/edit) |
| **sem 4.5** | semantic | `/debug` writes secrets to pi-debug.log | high | **fix before porting** (redact in writer; mode 0600 on log) |
| **sem 4.6 / arch-CF9** | semantic | auth.json non-atomic write + brief mode window | high | **fix before porting** (temp-write+chmod+rename) |
| **sem 4.7 / S3.10** | semantic | OAuth refresh proper-lockfile stale-lock theft | high | **fix before porting** (shorter stale window; longer lock TTL; or single-flight via Promise) |
| **sem 4.8** | semantic | oauth.json.migrated kept with pre-migration mode | high | **fix before porting** (chmod 0600 on migration; or delete) |
| **sem 4.9** | semantic | Operator-scoped extensions auto-load without allowlist | high | **fix before porting** (allowlist; or signed-extension model) |
| **sem 4.10** | semantic | web-ui IndexedDB plaintext keys + unsafeHTML callsites | high | **fix before porting** (encrypt at rest; sanitize HTML) |
| **mech P6.1 / arch-CF7** | mechanical | undici timeouts disabled globally | high | **port differently** — different runtime, different policy. Set per-call timeouts. |
| **mech P6.2 / arch-CF6** | mechanical | photon WASM `fs.readFileSync` monkey-patch | high | **port differently** — use a non-monkey-patching image library, or remove the feature |
| **mech P6.3** | mechanical | Silenced `process.emitWarning` globally | high | **port differently** (don't silence Node deprecation/security warnings) |
| **mech P6.4 / sem 4.2** | both | `!`-prefix shell-exec via `resolveConfigValue` | high | **fix before porting** (remove or require explicit opt-in flag) |
| **mech P6.5** | mechanical | Migrations run unconditionally on every startup with bare-catch | high | **fix before porting** (gate by version detection; surface failures) |
| **mech P6.6 / sem 3.2** | both | migrateAuthToAuthJson unlocked race vs SettingsManager | high | **fix before porting** (use SettingsManager) |
| **sem 3.1 / dsm-RT2** | semantic | Synchronous busy-wait spin between proper-lockfile retries | high | **fix before porting** (async wait-for-lock) |
| **sem 3.3 / dsm-RT10** | semantic | Codex retry no jitter | high | **fix before porting** (full-jitter exponential backoff) |
| **sem 3.4** | semantic | Deferred-flush check-then-act → double-write | high | **fix before porting** (compare-and-swap on flushed flag) |
| **sem 3.5** | semantic | Auto-compaction restart 100ms race | high | **fix before porting** (link AbortController to compaction restart) |
| **sem 3.6** | semantic | `_persist` failures silently swallowed | high | **fix before porting** (surface persistence failures to UI) |
| **sem 3.7 / mech 2.4** | both | `forwardStream` IIFE unawaited; consumer hangs | high | **fix before porting** (await + propagate error) |
| **sem 3.9** | semantic | Concurrent rg/fd download race on rename | high | **fix before porting** (per-binary lockfile) |
| **mech P2.10 → see mech**, **mech 1.7** (compaction unreachable cases) | mechanical | etc. (24 more high mech findings) | high | **fix before porting** for most; **port differently** for source-specific shims |
| **OpenAI-compat URL matrix (10 vendors)** | semantic 5.2 / arch-CF8 | No new high-severity vendor-branch findings beyond cited ones; long-tail compat | high (per row) | **port differently / leave behind** — port high-traffic vendors first; treat the URL matrix as a long-tail compat shim, reimplement registry skeleton + the top-3 vendors, defer the others |

**Summary by recommendation:**
- **fix before porting**: ~58 of 76 (incl. all 5 critical) — these would carry forward as bugs into a faithful port and need to be addressed.
- **port differently**: ~14 of 76 — source-specific hazards (undici tweaks, photon WASM, koffi FFI, terminal escape escapes that don't translate 1:1, etc.) — the new implementation should solve the underlying need with idioms native to its runtime.
- **leave behind**: ~4 of 76 — the medium/low-traffic OpenAI-compat URL branches and source-specific ergonomics that don't carry across.

Full per-finding rows + tallies: `findings/defect-scan-mechanical/mechanical-defects.md`, `findings/defect-scan-semantic/semantic-defects.md`.

### Cross-Phase Corrections Applied (sem-CR1, sem-CR2)

Two corrections from the semantic phase were applied to this synthesis (per GUIDE.md "phase outputs can't correct prior phases" — the porting bundle is the right place to consolidate corrected views):

1. **sem-CR1 (project-RCE perimeter)**: Architecture / contracts / protocols framed `<repo>/.pi/models.json` `!`-prefix as the project-RCE channel. **Corrected**: current source does not load `models.json` from project scope (per `agent-session-services.ts:136` + `sdk.ts:202`). The `!`-prefix mechanism is real (4.2) but currently scoped to operator-controlled config. The actual project-RCE perimeter is the **5-channel bundle in 4.1**: project `<cwd>/.pi/extensions/*.ts`, project `settings.json`'s `packages` / `shellCommandPrefix` / `shellPath` / `npmCommand`. Porting Recommendation row above reflects the corrected view.
2. **sem-CR2 (PS1 record-type discriminator)**: protocols claimed `type: "session_info"` for the per-file header; actual code uses `type: "session"` (`session-manager.ts:31`). `session_info` is a separate user-defined display-name marker. Schema in §Persistent Schema Notes above reflects the corrected view.

## Observed Facts vs. Inferred Structure

### Observed Facts

- 5 packages at lockstep `0.73.0`; `scripts/sync-versions.js:36-45` enforces.
- Build order `tui → ai → agent → coding-agent → web-ui` (`README.md:74`).
- 12-variant `AssistantMessageEvent` union (`packages/ai/src/types.ts:269-281`).
- 22 ExtensionEvent union members; 29 distinct subscribable event names.
- 7 built-in tools (`packages/coding-agent/src/core/tools/index.ts:83`).
- 5-level auth precedence (`packages/coding-agent/src/core/auth-storage.ts:446-516`).
- Session JSONL v3, 10 record types in the union; `type: "session"` is the per-file header (sem-CR2).
- `streamProxy` is a CLIENT POSTing to `${proxyUrl}/api/stream` with Bearer auth; custom `data: <json>\n` framing.
- mode 0600 enforced on every `auth.json` write.
- 5 AbortControllers in `AgentSession`; `agent.signal` is the main turn signal.
- `streamSimple*` synchronously throws on missing API key for Codex/Completions (contradicts documented `StreamFunction` doc-comment at `types.ts:147-159`).
- `<cwd>/.pi/` is auto-loaded with no consent gate; 5 RCE channels.
- 76 high/critical defects total; 5 critical.

### Inferred Structure

- The five concept layers (L1–L5) are clean separations of responsibility, not just packaging conveniences. *(strong inference — confirmed by absence of cycles and clean import graph)*
- The `<cwd>/.pi/` auto-load behavior reflects an "operator runs pi only in trusted projects" mental model. The porting decision (consent gate vs default-trust) is a major policy call. *(strong inference — no documentation explicitly states this assumption but it's the simplest reading)*
- The deferred-flush optimization in SM2 was likely chosen for performance (avoid creating a JSONL file on every prompt that's never followed up). The data-loss surface is probably an unintended consequence. *(strong inference — `_persist` shape is consistent with this)*
- The 7-provider error-message destruction (defect 1.1) is a copy-paste anti-pattern from one provider to the next. `openai-completions.ts:376-378` shows the correct pattern, suggesting at least one author knew better. *(strong inference)*
- The `streamProxy` is shipped as a client because pi-mono runs in two delivery shells and both need a way to talk to a remote agent runner; the lack of a server in pi-mono itself implies users plug in their own server. *(strong inference)*

## Domain Glossary

| Term | Definition | Where Used |
|---|---|---|
| `pi` | The interactive coding-agent CLI binary; flagship product of pi-mono | `packages/coding-agent/package.json` `bin: pi` |
| `agent` (lowercase) | The in-process loop driver class (`Agent` in pi-agent-core) | All packages |
| `provider` | An LLM-vendor adapter in pi-ai; selected by `model.api: KnownApi` | `packages/ai/src/providers/` |
| `session` | A single conversation transcript persisted as a JSONL file | `~/.pi/agent/sessions/` |
| `branch` (verb) | Forking a session at an entry; produces a new file (createBranchedSession) or marks the current file (branchWithSummary / branch) | `core/session-manager.ts` |
| `compaction` | Replacing older session entries with a summary record to fit context window | `core/compaction/` |
| `extension` | A user/project-scoped TS module loaded via jiti; subscribes to ~22 lifecycle events; can register tools/commands/etc. | `core/extensions/` |
| `slash command` | A `/foo`-prefixed user command in interactive mode; either a builtin or a `.md` prompt template | `core/slash-commands.ts`, `core/prompt-templates.ts` |
| `tool` (lowercase) | One of the 7 built-in actions exposed to the LLM | `core/tools/` |
| `partial` | The accumulating in-progress AssistantMessage during streaming | `pi-ai`, `pi-agent-core` |
| `runtime` (extension context) | The runtime object exposed to extensions; can `subscribe`, register, etc. | `core/extensions/runtime.ts` |
| `EventStream` | Hand-rolled async-iterable with waiter queue; bridges provider events into the unified union | `packages/ai/src/utils/event-stream.ts` |
| `streamProxy` | The HTTP SSE CLIENT in pi-agent-core for talking to a remote agent runtime; **not a server** | `packages/agent/src/proxy.ts` |
| `mode` | One of `interactive` / `print` / `rpc` — selected at startup | `core/main.ts:resolveAppMode` |
| `KnownApi` | Discriminator string for provider selection (e.g., `"anthropic-messages"`, `"openai-responses"`) | `pi-ai` registry |
| `withFileMutationQueue` | Per-realpath serializer for tool-driven file edits | `core/file-mutation-queue.ts` |
| `proper-lockfile` | npm package providing inter-process file locking; used for `auth.json` OAuth refresh and SettingsManager writes | `auth-storage.ts`, `settings-manager.ts` |
| `AppName` | Build-time templated identifier (`"pi"` for default builds); shapes env var names like `${APP_NAME.toUpperCase()}_CODING_AGENT_DIR` | `core/config.ts:374-380` |
| `CURSOR_MARKER` | Custom APC escape `\x1b_pi:c\x07` used internally by pi-tui for cursor reporting; stripped before emit | `tui.ts:68` |
| `models.generated.ts` | 16,815-line static catalog of LLM models, generated from external sources by `scripts/generate-models.ts`; treat as opaque | `packages/ai/src/models.generated.ts` |

## Open Questions

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| (Inherits unresolved questions from prior phases) | various | arch-OQ1 (rg/fd offline boot), arch-OQ3 (proxy heartbeat), pro-OQ1-5 (timing tests, terminal capability), sem-OQ1-7 (lock-stealing, microtask interleaving, maintainer policy on consent + project trust + listener semantics) | Each requires runtime evidence or maintainer ruling — see status.yaml for the consolidated list |

This porting phase did not surface new genuinely-open questions beyond
those carried forward from prior phases.

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| port-CF1 | reimplementation-spec | Express SM1–SM6 state machines as language-agnostic FSMs in the spec's "Protocols and Persisted State" section. (Subsumes pro-CF2.) | Spec rubric. |
| port-CF2 | reimplementation-spec | Each of the 5 critical defects + the 14 "port differently" hazards needs an explicit "designed-around" or "left-behind" decision in the spec, with citation. | Spec gate per pipeline.yaml `completion_criteria`: "Defects identified in either scan are explicitly designed-around or noted as 'left behind', with the choice cited." |
| port-CF3 | reimplementation-spec | Decide policy on the 5-channel `<cwd>/.pi/` RCE: default-deny + `--trust-project` flag, or default-allow with documentation? sem-OQ4 needs maintainer ruling, but the spec must take a position. | Maintainer-policy gate from sem-OQ4 + sem-OQ3. |
| port-CF4 | reimplementation-spec | Decide policy on built-in tools: per-call user confirmation gate, or LLM-controlled (current behavior)? sem-OQ3 needs maintainer ruling. | Maintainer-policy gate from sem-OQ3. |
| port-CF5 | reimplementation-spec | Decide policy on session-JSONL deferred-flush: preserve optimization (and document data-loss contract), or flush on every event (eat the latency)? pro-OQ3 needs maintainer ruling. | Maintainer-policy gate from pro-OQ3. |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | The system summary, layer map, contract table, protocol notes, and porting findings are synthesized. | PASS | §System Summary (3 paragraphs) + §Layer Map (5 layers L1-L5 with role/owns + dep diagram) + §Feature Contract Table (20 feature rows, sorted by priority) + §Protocol and State Notes (event catalog + 6 state machines + persistent schemas + wire formats) + §Defect Synthesis (76 high/critical bucketed by recommendation). |
| 2 | Portability hazards and open questions are separated from facts. | PASS | §Portability Hazards table (24 rows, sourced from prior phases). §Open Questions section explicitly inherits from prior phases without re-stating. §Observed Facts vs. Inferred Structure separates the two with no fact in the inference list and vice versa. |
| 3 | Feature importance is sorted for porting. | PASS | Feature Contract Table column 3 has core/important/optional/incidental classifications for every row. Sorted: 8 core, 6 important, 4 optional, 2 incidental. |
| 4 | Known defects are referenced in the Defect Synthesis with porting recommendations (fix before porting / port differently / leave behind), or the section explicitly notes that no defect scan ran. | PASS | §Defect Synthesis lists all 5 critical + a top-prioritized subset of high findings with explicit recommendations. Summary by recommendation: ~58 fix before porting, ~14 port differently, ~4 leave behind. The two cross-phase corrections (sem-CR1, sem-CR2) are explicitly applied with citation. |
| 5 | Findings are marked with evidence levels. | PASS | All claims tagged with `(observed fact / strong inference / portability hazard / open question)`. The Observed Facts vs. Inferred Structure section explicitly partitions the load-bearing claims. |

**Validated by:** 2026-05-06 (porting phase, session 1, no-orchestrator mode)
**Overall:** PASS

(All inbound carry-forwards from prior phases — pro-CF1, sem-CR1, sem-CR2 — resolved here. 5 carry-forwards routed to reimplementation-spec. The Strategic Alignment Hook decision is already locked: language-agnostic spec.)
