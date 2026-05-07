# Post-Pipeline OQ Triage — 2026-05-06

## Summary

The full-with-deep-audit pipeline left **30 open questions** in
status.yaml. This addendum reviews them post-pipeline against fresh
source reads and prior scratch evidence, and reports resolution status.

**Result**: 6 newly resolved or partially resolved from source reads,
4 already answered by prior phases (formally surfacing them here), and
20 remain genuinely open (runtime tests, maintainer rulings, per-port
spikes — all out of scope for source-only analysis).

Two source reads also surfaced **new mechanism-level details** that
refine prior defects 1.13 / 5.10 (listener fan-out is over a live `Set`
with no defensive snapshot) and protocols H19 (the `ExtensionEvent`
union has actual drift vs the `on()` overloads, not just nominal
3-way duplication). And a **new mechanical defect** worth recording:
streamProxy SSE parsing splits on `\n` only, leaving CR in the data
slice on CRLF-line-ending servers.

## Resolutions

### Newly Resolved from Source

#### arch-OQ1 — PARTIALLY RESOLVED

**Question**: Does `pi` startup `rg`/`fd` auto-download fail gracefully
when host is offline?

**Answer**: Yes, no crash. `tools-manager.ts` `ensureTool` (`:286-291`)
catches all errors and returns `undefined` (the agent continues without
rg/fd; grep/find tools degrade silently). However:

- **Network errors and HTTP non-200 are not distinguished**
  (`getLatestVersion` only checks `!response.ok` at `:108-120`).
- **Offline (DNS/connect failures) bubbles up from `fetch` as a
  different error class** but is caught by the same catch-all.
- **No checksum verification** anywhere in this file (separately filed
  as defect 4.4 / dsm-RT9).
- An explicit **opt-in offline mode via `PI_OFFLINE`** env var (`:15-19,
  :258-263`) skips the download with a different message; auto-detection
  of offline is not implemented.

*(observed fact)*

#### arch-OQ3 / pro-OQ1 — RESOLVED

**Question**: streamProxy SSE heartbeat / keepalive behavior?

**Answer**: **No keepalive parsing of any kind** (`packages/agent/src/proxy.ts:192-206`):

- Parser splits on `\n` and ONLY processes `line.startsWith("data: ")`.
- Comment lines (`:`-prefix), `event: ...`, `id:`, `retry:`, blank
  lines, and any other SSE-canonical shapes are silently dropped.
- The only liveness mechanism is the caller's `AbortSignal` at
  `:187/:209`.
- pi-mono ships only the client; server-side keepalive is
  out-of-scope for this codebase. *(observed fact)*

#### con-OQ4 — RESOLVED

**Question**: Listener removal during fan-out iteration in
`agent.ts:539-541` — safe or undefined?

**Answer**: **Defined ECMAScript `Set`-iteration semantics**, with
mutation effects:

- Listeners are stored as `private readonly listeners = new Set<...>()`
  (`agent.ts:160`).
- Iteration is a live `for...of` directly over the Set:
  `for (const listener of this.listeners) { await listener(event, signal); }`
  (`:539-541`) — **no defensive snapshot** (no `[...this.listeners]`,
  no `Array.from`).
- Per ECMAScript Set iteration semantics:
  - A listener that unsubscribes itself **will not re-appear** (its
    `delete` at `:221` updates the live iterator).
  - A listener removed by an earlier sibling within the same fan-out
    pass **WILL be skipped** in this same pass.
  - Additions during iteration **WILL be visited** in this same pass.
- This is a **new mechanism-level detail** that refines defects
  1.13 / 5.10. *(observed fact)*

#### con-OQ5 — RESOLVED with NEW DRIFT SURFACED

**Question**: Extension API single-source-of-truth for the 22 events?

**Answer**: Confirmed **3-way duplication WITH actual drift**:

- `ExtensionEvent` union (`types.ts:950-972`) lists 22 event-type
  aliases, but **8 session_* events** (`SessionStartEvent`,
  `SessionBeforeSwitchEvent`, `SessionBeforeForkEvent`,
  `SessionBeforeCompactEvent`, `SessionCompactEvent`,
  `SessionShutdownEvent`, `SessionBeforeTreeEvent`,
  `SessionTreeEvent`) are reachable via `on()` overloads but **absent
  from the union**.
- `on()` overloads (`types.ts:1089-1126`) list **30 distinct event
  strings** — the 22 union members + the 8 session_* names.
- Runner has 9 dedicated `emit*` methods plus a generic `emit()` for
  the `RunnerEmitEvent = Exclude<...>` carve-out at `:115-126`.
- The 9 carve-outs MUST match the 9 dedicated emit* methods — they
  currently do, **but the type doesn't enforce it**.
- **New finding**: an extension subscribing to `session_start` is
  correctly typed via the overload, but the central union understates
  what can flow through the runner. The union appears to have been
  written before session_* events were added. *(strong inference)*

This **strengthens protocols H19** with a specific drift instance, not
just a 3-way duplication concern.

#### pro-OQ4 — RESOLVED

**Question**: Fallback when terminal does not support `\x1b[?2026`
(synchronized output mode)?

**Answer**: **No fallback whatsoever.** `tui.ts:920, :926, :1019, :1048,
:1071, :1155` — all 6 emission sites unconditionally emit the
sequence. No DA1/DA2/XTGETTCAP capability query, no `TERM` env-name
check, no opt-out env var. The only terminal-branching nearby is
`isTermuxSession()` at `:111-112`, which only affects the height-change
full-redraw decision (`:969`), not synchronized-output emission.
**Terminals that don't grok the sequence will display literal text
or ignore it; either way pi assumes silent passthrough.**
*(observed fact, portability hazard)*

#### dsm-OQ1 / pro-OQ5 — RESOLVED

**Question**: EPIPE on `process.stdout.write` mid-paint — escalates to
uncaughtException, or silent absorb?

**Answer**: **Hard process kill, no mitigation.**

- `Terminal.write` at `terminal.ts:318-327` is a bare
  `process.stdout.write(data)` — no callback, no try/catch, no error
  listener.
- The only listener on `process.stdout` anywhere in pi-mono is
  `process.stdout.on("resize", ...)` at `terminal.ts:107`.
- A grep across **all of `/home/user/pi-mono/packages/`** for
  `uncaughtException` and `unhandledRejection` returns **zero matches**
  — no process-level handler exists in any package.
- Per Node semantics: EPIPE on stdout with no `'error'` listener is
  emitted as an unhandled `'error'` event on the writable stream →
  `uncaughtException` → default action **terminates the process with
  exit code 1 and stack trace to stderr**.
- Pi does nothing to mitigate this. **A closed pipe mid-paint
  hard-crashes the agent.** *(observed fact)*

This **strengthens protocols H7 / mech 2.11** with the precise
crash-path mechanism.

### Already Answered by Prior Phases (formal surface-up)

#### arch-OQ2 — PARTIALLY ANSWERED in mechanical Pass 6

Corrupt `auth.json`: process starts, sets `loadError`, blocks
credential reads until user fixes file.
*(observed fact — see `defect-scan-mechanical/mechanical-defects.md`
Pass 6 architecture-flagged answers)*

#### con-OQ1 — ANSWERED in semantic Pass 4 finding 4.5

`/debug` writes the **full conversation including all tool outputs and
any pasted secrets** to `pi-debug.log` with no redaction and umask-default
permissions (typically 0644). High-severity finding filed.

#### con-OQ3 — RESOLVED as semantic Pass 5 finding 5.1

`streamSimple*` synchronous-throw violates the documented `StreamFunction`
"failures encoded in stream" contract at `types.ts:147-159`. 9 of 9
production providers' `streamSimple*` variants violate it. Cascades
through `handleRunFailure` (defect 1.21) producing a 0-token assistant
message in the transcript.

#### dsm-OQ4 — ANSWERED in contracts F-EXT-1 / Pass 5 finding 5.9

`emitToolCall` is the only unguarded extension emitter. Throw blocks
the tool AND skips subsequent `tool_call` handlers. The reimpl-spec
takes the divergent position (M8 invariants): all emitters uniformly
guarded, throwing handlers logged + skipped. The current source's
divergent semantics are filed as a Pass 5 finding.

## New Defects Surfaced by This Triage

These are **mechanism-level refinements** of existing defects, not
genuinely new defects. Filed here for citation:

### NF1 — Listener fan-out uses live Set iteration (refines 1.13 / 5.10)

`packages/agent/src/agent.ts:160` declares
`private readonly listeners = new Set<...>()`; the fan-out at `:539-541`
is `for...of` over the live Set with no defensive snapshot. This means
a listener removed by an earlier sibling in the same fan-out pass is
skipped silently in this pass; an addition is visited. The reimpl-spec
M3 invariant ("listener fan-out is bounded; throws are logged + skipped")
should also pin **defensive snapshot semantics** — capture
`[...this.listeners]` before iteration to make per-pass behavior
predictable. *(observed fact)*

### NF2 — `ExtensionEvent` union missing 8 session_* events (refines H19 / 5.9)

`packages/coding-agent/src/core/extensions/types.ts:950-972` (the union)
is missing all 8 session_* event aliases that `on()` overloads at
`:1089-1126` advertise. Result: an extension subscribing to e.g.
`session_start` is correctly typed via the overload, but the central
union understates what flows through the runner — runner-internal code
that filters or routes based on the union will miss session_* events.
*(observed fact, drift surface confirmed beyond just 3-way duplication)*

### NF3 — streamProxy SSE parser ignores CR in CRLF line endings (NEW)

`packages/agent/src/proxy.ts:192` does `buffer.split("\n")` only.
Servers using canonical CRLF line endings will leave a `\r` at the end
of each `data: <json>\r` payload. The `JSON.parse` on the slice
`line.slice(6)` will succeed if the JSON-end character is robust
(closing `}` followed by `\r`), but if a server emits `data: \r\n`
keepalive frames with a literal CR-only payload, behavior is
undefined. Worth a Pass 1 refinement: server CRLF compatibility is
a portability concern. *(observed fact, mechanical defect — refinement
of H10 / mech 1.22)*

## Still Genuinely Open

20 questions remain that cannot be resolved from source alone:

| ID | Why still open |
|---|---|
| con-OQ2 | Windows ACL behavior for auth.json mode 0600 — runtime test. |
| dsm-OQ2 | appendFileSync ENOSPC short-write vs full-fail — runtime test. |
| dsm-OQ3 | Bun-binary unhandled-rejection default — runtime test against Bun runtime. |
| pro-OQ2 | Concurrent-reader detection on session JSONL mid-rewrite — OS-dependent runtime test. |
| pro-OQ3 | Should `_persist` deferred-flush be considered intentional optimization or contract gap? — maintainer ruling. (Spec already took position: flush-on-event.) |
| sem-OQ1 | Lock-stealing semantics under proper-lockfile `stale: 30000` with slow OAuth — runtime test. |
| sem-OQ2 | Microtask-interleaving timing for deferred-flush check-then-act — runtime + theoretical. |
| sem-OQ3 | Per-call consent UX trade-off for bash/edit/write — maintainer policy. (Spec position: per-call consent.) |
| sem-OQ4 | `<cwd>/.pi/` default-trust posture — maintainer policy. (Spec position: default-deny + `--trust-project`.) |
| sem-OQ5 | Listener-fan-out unguarded — maintainer ruling on intent. (Spec position: log + skip; uniform.) |
| sem-OQ6 | mini-lit MarkdownBlock HTML-sanitization — third-party library audit. |
| sem-OQ7 | Slow-OAuth-token-endpoint exploitability bounds — runtime test. |
| reimpl-OQ1..7 (×7) | Per-port spikes / runtime tests / maintainer decisions — out of scope for source-only analysis on the current source. Belong with the implementation team that picks up the spec. |

Note that `sem-OQ3`, `sem-OQ4`, `sem-OQ5`, and `pro-OQ3` are now
"maintainer policy" only in the sense that the original maintainer's
intent is unknown; the **reimpl-spec already took explicit positions** on
each (per port-CF3 / port-CF4 / port-CF5). A future spike or maintainer
conversation could reclassify these as RESOLVED, but the spec's
position stands without that input.

## Updates to status.yaml

Mark resolutions for: arch-OQ1 (partial), arch-OQ3 (resolved), con-OQ1
(resolved via 4.5), con-OQ3 (resolved via 5.1), con-OQ4 (resolved),
con-OQ5 (resolved with NF2), dsm-OQ4 (resolved via 5.9), pro-OQ1
(resolved, same as arch-OQ3), pro-OQ4 (resolved), pro-OQ5 (resolved
with NF1-mechanism), arch-OQ2 (partial via Pass 6).

Net effect: 11 of 30 open questions now have a documented answer
(resolved or partially resolved). 19 still need runtime / maintainer /
spike evidence.

## Files Touched

- **Added:**
  - `closeouts/2026-05-06-post-pipeline-oq-triage.md` (this file)
- **Modified:**
  - `workflow/status.yaml` (close-out / partial-close annotations on the resolved OQs)
  - `THREAD_LOG.md` (index entry)

## No Re-amendment of Prior Outputs

Per GUIDE.md "phase outputs can't correct prior phases", this addendum
does not amend the architecture-map, contracts, protocols, mech-defects,
sem-defects, porting bundle, or reimpl-spec primaries. The resolutions
and the three new mechanism-level findings (NF1, NF2, NF3) are recorded
HERE; downstream work (a new pipeline run, or a delta phase per the
existing spec-delta-application skill) can fold them into refreshed
primaries if desired.
