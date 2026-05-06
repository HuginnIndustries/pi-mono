# Semantic Defects Report — pi-mono

<!--
  Phase: defect-scan-semantic
  Pipeline: workflow/pipeline-full-with-deep-audit.yaml
  Source-of-detail: scratch/sem-pass{3,4,5}.md
  Severity policy (user-locked): per-finding entries cap at high/critical; medium/low get tally rows only.
-->

## Scan Context

- **Source:** `../` (repository root: pi-mono, 5 packages, ~282 source files, ~102K LOC TypeScript)
- **Architecture reference:** `findings/architecture/architecture-map.md`
- **Contracts reference:** `findings/contracts/behavioral-contracts.md`
- **Protocols reference:** `findings/protocols/protocols-and-state.md`
- **Mechanical defects reference:** `findings/defect-scan-mechanical/mechanical-defects.md`
- **Pipeline:** `workflow/pipeline-full-with-deep-audit.yaml`
- **Date:** 2026-05-06
- **Scope:** Semantic passes only (Pass 3 concurrency, Pass 4 security, Pass 5 contract violations). Mechanical passes (1 logic, 2 error handling, 6 configuration) covered in `findings/defect-scan-mechanical/mechanical-defects.md`.
- **Severity cap (user policy):** per-finding writeups limited to **critical** and **high**; medium/low rolled into tally rows.

---

## Pass 3: Concurrency and Resource Management

Sort: critical → high. Source-of-detail: `scratch/sem-pass3.md`.

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| 3.1 | `packages/coding-agent/src/core/auth-storage.ts:86-89` (also `settings-manager.ts:183-186`) | **(resolves dsm-RT2)** Synchronous busy-wait spin between proper-lockfile retries blocks the event loop up to 200ms per failed attempt. Starves the agent loop, TUI render queue, and any background timers under multi-instance contention. | high | observed fact | fix before porting |
| 3.2 | `packages/coding-agent/src/core/migrations.ts` (`migrateAuthToAuthJson`) | **(resolves dsm-RT3)** Unlocked read-modify-write on `settings.json` races SettingsManager's proper-lockfile-protected writes. A migration concurrent with a settings save can lose either edit. | high | observed fact | fix before porting |
| 3.3 | `packages/ai/src/providers/openai-codex-responses.ts:106-308` (also `agent-session.ts:2462`) | **(resolves dsm-RT10)** Codex retry loop has no jitter (`BASE_DELAY_MS * 2**attempt`); deterministic backoff causes thundering-herd under multi-instance retry storms after a provider outage. | high | observed fact | fix before porting |
| 3.4 | `packages/coding-agent/src/core/session-manager.ts` (`_persist` deferred-flush; SM2 in protocols-and-state.md) | Check-then-act on `flushed` flag interleaves under async listener fan-out — under specific timing, the same event can be persisted twice into the JSONL file. | high | strong inference | fix before porting |
| 3.5 | `packages/coding-agent/src/core/agent-session.ts` (auto-compaction restart) | `setTimeout(() => agent.continue().catch(()=>{}), 100)` race window between auto-compaction restart and user abort: cancellation in the 100ms window is silently dropped. | high | observed fact | fix before porting |
| 3.6 | `packages/coding-agent/src/core/agent-session.ts` (`_agentEventQueue` chain) | `.catch(()=>{})` on the persistence-promise chain silently drops `_persist` failures (e.g., disk full). Combined with 3.4, persistence reliability is unobservable. | high | observed fact | fix before porting |
| 3.7 | `packages/ai/src/providers/register-builtins.ts:129-136` (`forwardStream` IIFE) | No try/catch around the IIFE; mid-stream rejection is unhandled and the consumer hangs (`target.end()` never called, `result()` deadlocks). Already cited as mech 2.4 — reframed here as concurrency hang. | high | observed fact | fix before porting |
| 3.8 | `packages/coding-agent/src/core/tools/bash.ts` (`tempFileStream`) | No `'error'` listener on the temp-file stream; `.end()` is never awaited. Disk-full mid-bash escalates to uncaughtException; consumers can read truncated file. (Reframes mech 2.6 + 1.9 with concurrency lens.) | high | observed fact | fix before porting |
| 3.9 | `packages/coding-agent/src/utils/tools-manager.ts` | Concurrent `fd`/`rg` downloads race on the final `renameSync` to shared `binaryPath`. Two pi instances first-running concurrently can produce a corrupt binary. | high | observed fact | fix before porting |
| 3.10 | `packages/coding-agent/src/core/auth-storage.ts` (OAuth refresh + proper-lockfile) | OAuth refresh holds `proper-lockfile` across `await fetch(TOKEN_URL)` with `stale: 30000`. A slow OAuth endpoint (>30s) lets another instance steal the lock mid-fetch; both instances may write to `auth.json`. | high | observed fact | fix before porting |

**No critical findings in Pass 3.** Critical data-loss races are filed as mechanical 2.1/2.2 (non-atomic JSONL/edit/write) — they are concurrency-shaped but already in the mechanical record.

**Cross-cutting observation**: `AgentSession` carries 6 concurrent cancellation domains (5 named AbortControllers + `agent.signal`). `abort()` cascades to 2 of them; `dispose()` cascades to none. Likely cause of any "stuck after Ctrl+C" UX reports. *(strong inference)*

---

## Pass 4: Security and Trust Boundaries

Sort: critical → high. Source-of-detail: `scratch/sem-pass4.md`.

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| 4.1 | `packages/coding-agent/src/core/extensions/loader.ts:340-352`, `core/agent-session-services.ts`, `core/settings-manager.ts` | **CRITICAL untrusted-repo trust-boundary breach.** `<cwd>/.pi/` exposes **5 concurrent RCE channels** when `pi` is run in a hostile repo: (1) auto-discovered extensions in `<cwd>/.pi/extensions/*.ts` loaded by jiti; (2) `packages: [...]` array in project `settings.json` drives `npm install` + load; (3) `shellCommandPrefix` in project settings is prepended to every bash call; (4) `shellPath` becomes the spawned shell binary; (5) `npmCommand` replaces the npm executable. Cloning a hostile repo and running `pi` triggers any of these. | critical | observed fact | fix before porting |
| 4.2 | `packages/coding-agent/src/core/config.ts` (`resolveConfigValue`) | The `!`-prefix shell-exec mechanism is real (this resolves the mechanism part of dsm-RT1) — but only currently invoked via user-controlled `~/.pi/agent/auth.json` `api_key` and `~/.pi/agent/models.json` headers. **Important correction to prior phases**: `<repo>/.pi/models.json` is NOT loaded from project scope in current source (per `agent-session-services.ts:136`, `sdk.ts:202`). The H24 hazard in protocols and the §Trust Boundaries row in contracts (`behavioral-contracts.md:329,388`) need correction in the porting phase. The actual project-scope RCE perimeter is 4.1 above. | high | observed fact | fix before porting |
| 4.3 | `packages/coding-agent/src/core/tools/{bash,edit,write,read,grep,find,ls}.ts` | Built-in tools have **zero input validation, no path containment (no chroot, no `cwd`-relative restriction), and no per-call user confirmation gate**. Contracts §F-CLI-4 implies "user confirmation per slash-command UX" but tool calls themselves bypass any such gate — once an LLM invokes `bash` or `write`, the tool runs unconditionally. | high | observed fact | fix before porting (or port differently with explicit consent gate) |
| 4.4 | `packages/coding-agent/src/utils/tools-manager.ts` | **(resolves dsm-RT9)** No checksum/signature verification on auto-downloaded `rg`/`fd` binaries. `~/.pi/agent/tools/` is PATH-prepended in tool dispatch, so a successful MITM during the download (or a compromised mirror) yields persistent code-exec privilege via every subsequent grep/find tool call. | high | observed fact | fix before porting |
| 4.5 | `packages/coding-agent/src/core/slash-commands.ts` (`/debug` command) | `/debug` writes the full conversation — including all tool outputs and any pasted secrets — to `pi-debug.log` with no redaction and umask-default permissions (typically 0644). **Closes con-OQ1.** | high | observed fact | fix before porting |
| 4.6 | `packages/coding-agent/src/core/auth-storage.ts` | Non-atomic `auth.json` write: a crash mid-write loses credentials. Plus a brief mode window where the file exists but is not yet `chmod`ed to 0600 (window between open+write and chmod). Combine with mech 2.1 (non-atomic write of session JSONL) — same data-loss surface for credentials. | high | observed fact | fix before porting |
| 4.7 | `packages/coding-agent/src/core/auth-storage.ts` (OAuth refresh, also S3.10) | proper-lockfile held across slow OAuth `fetch` >30s = stale-lock theft (lock declared stale after 30s; another instance acquires; both write). Trust-boundary view: a slow provider can be exploited by a co-tenant to inject an attacker-controlled refresh token. | high | strong inference | fix before porting |
| 4.8 | `packages/coding-agent/src/core/migrations.ts` | `oauth.json.migrated` (the legacy file post-migration) is left on disk with its **pre-migration mode** (typically 0644). Old OAuth refresh tokens remain readable to any process on the host. | high | observed fact | fix before porting |
| 4.9 | `packages/coding-agent/src/core/extensions/loader.ts` (project + operator extensions auto-loaded, dedup by absolute path) | Worse than 4.1: even when no project `.pi/extensions/` exists, a forked or installed pi build may have shipped operator extensions in `~/.pi/agent/extensions/` that the user did not explicitly trust. No allowlist. | high | strong inference | fix before porting |
| 4.10 | `packages/web-ui/src/CustomProviderManager.ts` (or similar) | web-ui stores custom-provider API keys in `IndexedDB` plaintext. Sandbox iframes correctly omit `allow-same-origin` so XSS-in-sandbox is contained, but `unsafeHTML` callsites in artifact components are defense-in-depth concerns; mini-lit `MarkdownBlock` for conversation rendering is a third-party-audit open question. | high | observed fact | fix before porting |

**Tally for medium/low** (per scratch/sem-pass4.md): 17 medium, 22 low.

**dsm-RT4 (`getAttributionHeaders` privacy review)**: resolved as **low** — only opt-in version+platform+arch reporting, no PII / install-id / fingerprint. Not promoted to per-finding entry.

---

## Pass 5: API Contract Violations

Sort: critical → high. **Spec Reference column REQUIRED.** Source-of-detail: `scratch/sem-pass5.md`.

| # | Location | Defect | Spec Reference | Severity | Evidence Level | Action |
|---|----------|--------|----------------|----------|----------------|--------|
| 5.1 | `packages/ai/src/index.ts` and per-provider `streamSimple*` paths | **(resolves con-CF6 / con-OQ3)** `streamSimple*` synchronously throws on missing API key for Codex/Completions, contradicting the documented "failures encoded in stream" contract. The `StreamFunction` doc-comment at `packages/ai/src/types.ts:147-159` explicitly states *"Once invoked, request/model/runtime failures should be encoded in the returned stream, not thrown."* 9 of 9 production providers' `streamSimple*` variants violate this for the missing-API-key precondition. | F-AI-1 (Error behavior); EC1 (terminal events). `types.ts:147-159` | high | observed fact | fix before porting |
| 5.2 | `packages/ai/src/providers/openai-completions.ts:1029-1088` (compat URL matrix) | **(resolves arch-CF8)** Per-vendor audit of all 10 vendors (z.ai, Moonshot, Cloudflare Workers AI, Cloudflare AI Gateway, Cerebras, Grok, Chutes, DeepSeek, Opencode, OpenRouter). No vendor produces invalid `AssistantMessageEvent` streams in the main path. **Documented divergences**: OpenRouter triggers EC2 tool-call fragmentation (defect 1.3 already filed), Cloudflare AI Gateway has a null-Authorization header path, 4 vendors silently drop `reasoning_effort`, and the `streamSimple*` sync-throw (5.1) applies to every vendor. **Audit conclusion**: no new high-severity vendor-branch findings beyond 5.1, 5.2 (this row), and existing defect 1.3. | EC1; arch-CF8 | high | observed fact | fix before porting (for each cited divergence) |
| 5.3 | `packages/agent/src/proxy.ts:325, :339` | **(resolves dsm-RT11)** `processProxyEvent` shallow-clones the slot at `:325` into `partial.content[]`, then `toolcall_end` at `:339` does `delete (content as any).partialJson` on the **live** ref — the earlier clone retains `partialJson`. A consumer reading `partial` snapshots after `toolcall_end` sees `partialJson` on a supposedly-finished tool call. Violates the AssistantMessageEvent contract: a `toolcall_end` event's resulting partial should not retain delta state. | EC1 (toolcall_end); EC2 (tool-call sub-protocol) | high | observed fact | fix before porting |
| 5.4 | `packages/agent/src/agent.ts:463-478` (`handleRunFailure`) — cascade with 5.1 | When 5.1 fires (sync throw) and the throw propagates to `handleRunFailure`, defect 1.21 then synthesizes an empty assistant message. Net: missing-API-key produces a 0-token "assistant message" in the transcript rather than an error UI. Cascading contract violation. | F-AGENT-1 (handleRunFailure); EC1 (terminal events) | high | strong inference | fix before porting |
| 5.5 | `packages/agent/src/agent-loop.ts:308-316` (`streamAssistantResponse` gate) | Defect 1.19 reframed: the agent-loop's gate `if (partialMessage)` silently swallows mid-stream events when `start` is missing, instead of treating it as an invalid SM1 transition. Should emit `error` event with explicit reason. | SM1 (stream FSM); F-AGENT-1 | high | observed fact | fix before porting |
| 5.6 | `packages/coding-agent/src/main.ts:155-167` (`resolveSessionPath`) | Defect 1.6 reframed: ambiguous `--session` prefix silently picks `[0]` from match arrays. Spec (F-CLI-2 + acceptance scenario #5) requires error-on-ambiguity. | F-CLI-2; Acceptance #5 | high | observed fact | fix before porting |
| 5.7 | `packages/coding-agent/src/core/session-manager.ts:801-819` (`_persist` deferred flush; SM2) | **CRITICAL data-loss contract violation.** SM2 deferred-flush JSONL: a process crash before the first assistant message arrives loses the user's prompt entirely — file may not exist on disk yet. F-CLI-3's claim *"Session JSONL state always survives cancel"* is **FALSE** in that window. (Reframes mech 2.1 / dsm-RT6 / arch-CF4 as a contract violation.) | F-CLI-3; SM2; PS2 | critical | observed fact | fix before porting |
| 5.8 | `packages/coding-agent/src/core/session-manager.ts:775-779` (`_rewriteFile`) | **CRITICAL data-loss contract violation.** Non-atomic `writeFileSync(file, content)` — mid-rewrite crash truncates entire session. F-CLI-3 + PS2 violations. (Reframes mech 2.1 / 2.2 with the contract lens.) | F-CLI-3; PS2 | critical | observed fact | fix before porting |
| 5.9 | `packages/coding-agent/src/core/extensions/runner.ts:806-827` (`emitToolCall` only unguarded emitter) | Confirmed: `emitToolCall` is the only emit method without per-handler `try/catch`. Throw blocks the tool **and** skips subsequent handlers, while every other emit catches and routes to `emitError`. Asymmetric semantics violate the implicit "all extension hooks behave the same on error" contract. | F-EXT-1; EC4 | high | observed fact | fix before porting |
| 5.10 | `packages/coding-agent/src/core/agent-session.ts:539-541` (listener fan-out) | Defect 1.13 reframed: listener fan-out is sequential and unguarded; one throw stops downstream listeners and propagates to handleRunFailure. F-AGENT-3 documents this as "current behavior" but doesn't pin whether it is the intended contract. Drift surface: ambiguous spec status. | F-AGENT-3; con-OQ4 | high | strong inference | fix before porting (post maintainer ruling con-OQ4) |
| 5.11 | `packages/coding-agent/src/core/session-manager.ts:31` | **Spec drift inside CodeCarto findings**: protocols PS1 record catalog claims `type: "session_info"` for the per-file header, but actual code uses `type: "session"`; `session_info` is a separate user-defined display-name marker. **Correct PS1 in porting phase consolidation.** | PS1 (protocols-and-state.md) | high | observed fact | fix before porting (correct findings) |
| 5.12 | `packages/agent/src/proxy.ts:261/275/293/307/333` | Out-of-order `_delta`/`_end` events throw — one misframe terminates the turn with no recovery. Asymmetric: `toolcall_end` mismatch silently returns `undefined` (`:347`). Violates SM4 connection FSM expectation that the server-client protocol is robust to network-level disorder. (Reframes defect 1.22.) | SM4; EC3 | high | observed fact | fix before porting |
| 5.13 | `packages/coding-agent/src/core/tools/bash.ts:392`, `core/exec.ts:99` | bash tool exit-code contract: `code: code ?? 0` reports killed processes (`code === null`) as exit 0 success; spawn errors are discarded. F-TOOLS bash row: "callers cannot reliably distinguish 'tool succeeded' from 'tool was killed.'" Documented but not specified — acceptance scenario #10 fails. (Reframes mech 1.12+2.5 as Pass 5 contract violation.) | F-TOOLS bash; Acceptance #10 | high | observed fact | fix before porting |
| 5.14 | `packages/tui/src/tui.ts:1100-1131` (width-overflow throw) | SM3 transition into "process exit" state on width-overflow. F-TUI-1's error behavior says throw; widget rendering's `compositeLineAt` "always truncate" comment contradicts. Violates the implicit "TUI is a robust output layer" contract. (Reframes mech 1.15 / H5.) | F-TUI-1; SM3 | high | observed fact | fix before porting |
| 5.15 | `packages/ai/src/providers/anthropic.ts` and 6 sister providers (mech 1.1) | 7 providers use opaque `"An unknown error occurred"` literal in error throws, destroying the provider-mapped `errorMessage`. Violates F-AI-1 error-behavior contract: error events should preserve provider-mapped error text. | F-AI-1; EC1 (error variant) | high | observed fact | fix before porting |
| 5.16 | `packages/ai/src/providers/google-shared.ts:514`, `google-vertex.ts` | Force `stopReason = "toolUse"` even when original mapped finish reason was `length`/`error`/safety. Violates F-AI-1 finish-reason contract: must preserve provider-actual finish reason. (Reframes mech 1.2.) | F-AI-1; EC1 (done) | high | observed fact | fix before porting |
| 5.17 | `packages/web-ui/src/MessageList.ts:36-91` + `ExpandableSection.ts:21-28` | Cascading: `MessageList` uses positional `repeat()` keys (defeating Lit reconciliation) + `ExpandableSection.connectedCallback` permanently destroys children on re-attachment. Net: scrolling/reordering messages destroys component state silently. Violates F-WEB-1 streaming-message contract that components remain stable across virtual-scroll updates. (Reframes mech 1.17 + 1.20.) | F-WEB-1 | high | strong inference | fix before porting |

**OpenAI-Compat URL Matrix Audit Conclusion (arch-CF8)**: Per-vendor sub-table in `scratch/sem-pass5.md`. No vendor produces invalid AssistantMessageEvent streams in the main path. Documented divergences are tracked in 5.2 above and existing mechanical defect 1.3.

**Black-Box Acceptance Failures**: 8 of 18 contract scenarios FAIL or PARTIAL FAIL on current source. Pass 5 covers 6 directly (5.1, 5.6, 5.7, 5.9, 5.12, 5.13); the rest map to mechanical 2.2 (file overwrite), 1.12+2.5 (bash exit), and Pass 4 finding 4.1 (covert-RCE channels).

---

## Summary

### Findings by Severity

| Severity | Count |
|----------|-------|
| Critical | 3 |
| High | 34 |
| Medium | 17 (Pass 4) + others tallied per scratch — see per-pass table |
| Low | 22 (Pass 4) + others tallied — see per-pass table |
| **Total (per-finding)** | **37** |

### Findings by Pass

| Pass | Critical | High | Medium (tally) | Low (tally) | Per-finding total |
|------|----------|------|----------------|-------------|-------------------|
| 3. Concurrency and resources | 0 | 10 | (per scratch — see file) | (per scratch) | 10 |
| 4. Security and trust | 1 | 9 | 17 | 22 | 10 |
| 5. API contract violations | 2 | 15 | (per scratch) | (per scratch) | 17 |

### Top Findings

1. **4.1** — CRITICAL untrusted-repo trust-boundary breach: 5 concurrent RCE channels via `<cwd>/.pi/` (extensions, packages, shellCommandPrefix, shellPath, npmCommand). Cloning a hostile repo and running `pi` triggers any of them. *(critical, fix before porting)*
2. **5.7** — CRITICAL data-loss contract violation: SM2 deferred-flush — crash before first assistant message loses user prompt. *(critical, fix before porting)*
3. **5.8** — CRITICAL data-loss contract violation: non-atomic `_rewriteFile` truncates session on mid-rewrite crash. *(critical, fix before porting)*
4. **4.4** — High supply-chain hazard: rg/fd auto-download with no checksum/signature; PATH-prepended → persistent code-exec privilege via grep/find. *(high, fix before porting)*
5. **5.1** — High contract drift: `streamSimple*` synchronous throw violates the explicitly-documented `StreamFunction` "errors via stream" contract; cascades through `handleRunFailure` to a 0-token assistant message in the transcript. *(high, fix before porting)*

### Carry-Forward Closure

| ID | Source Phase | Closed Because |
|---|---|---|
| dsm-RT1 | defect-scan-mechanical (P6.4) | Pass 4 finding 4.1 (real RCE perimeter is `<cwd>/.pi/` settings + extensions, NOT models.json — correction); Pass 4 finding 4.2 documents the `!`-prefix mechanism as currently scoped to user-controlled config |
| dsm-RT2 | defect-scan-mechanical | Pass 3 finding 3.1 (auth-storage busy-wait) |
| dsm-RT3 | defect-scan-mechanical | Pass 3 finding 3.2 (migrateAuthToAuthJson race) |
| dsm-RT4 | defect-scan-mechanical | resolved as low — only opt-in version/platform/arch reporting; no PII |
| dsm-RT9 | defect-scan-mechanical | Pass 4 finding 4.4 (rg/fd no checksum) |
| dsm-RT10 | defect-scan-mechanical | Pass 3 finding 3.3 (Codex retry no jitter) |
| dsm-RT11 | defect-scan-mechanical | Pass 5 finding 5.3 (streamProxy partialJson leak) |
| arch-CF8 | architecture | Pass 5 finding 5.2 (OpenAI-compat URL matrix audit) |
| arch-CF9 | architecture | Pass 4 findings 4.6 (auth.json non-atomic), 4.7 (lock-stealing on slow OAuth), 4.8 (oauth.json.migrated mode) — split into three findings |
| con-CF5 | contracts | Pass 4 finding 4.1 + 4.2 (RCE perimeter correction; the channel that originally triggered con-CF5 is the wider 4.1 bundle) |
| con-CF6 | contracts | Pass 5 finding 5.1 |

### Important Correction to Prior Phases

- **`<repo>/.pi/models.json` is NOT loaded from project scope** in current source (per `agent-session-services.ts:136`, `sdk.ts:202`). The architecture phase + protocols phase + contracts phase all framed `<repo>/.pi/models.json` `headers` `!`-prefix as the project-scope RCE channel. The `!`-prefix mechanism is real (4.2) but currently only invoked through user-controlled `~/.pi/agent/auth.json` and `~/.pi/agent/models.json`. The actual project-scope RCE perimeter is the **5-channel bundle in 4.1** (extensions + packages + shellCommandPrefix + shellPath + npmCommand from project `<cwd>/.pi/settings.json` + `<cwd>/.pi/extensions/`).
- **Routed to porting phase** to correct hazard H24 in `protocols-and-state.md` and the trust-boundary section in `behavioral-contracts.md` during synthesis.

### Open Questions Surfaced

| ID | Kind | Description |
|---|---|---|
| sem-OQ1 | needs-runtime-test | Lock-stealing semantics under proper-lockfile `stale: 30000` with slow OAuth fetch — both instances writing? Maintainer policy. |
| sem-OQ2 | needs-runtime-test | Microtask-interleaving timing for the deferred-flush check-then-act (3.4) — actual bug in practice or theoretical? |
| sem-OQ3 | needs-maintainer-decision | Should built-in tools (bash/edit/write) require explicit user confirmation per call, or remain LLM-controlled? |
| sem-OQ4 | needs-maintainer-decision | Should `<cwd>/.pi/` be loaded at all without explicit consent, or require a `--trust-project` flag? |
| sem-OQ5 | needs-spec-ruling | Listener-fan-out unguarded behavior (5.10): intended contract or defect? |
| sem-OQ6 | needs-runtime-test | mini-lit `MarkdownBlock` audit — does it sanitize HTML? Third-party. |
| sem-OQ7 | needs-runtime-test | Slow-OAuth-token-endpoint timing for 4.7 — practical exploitability bounds. |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | All three semantic passes (3, 4, 5) produced findings or documented "no defects found." | PASS | Pass 3: 10 high; Pass 4: 1 critical + 9 high + tallies; Pass 5: 2 critical + 15 high. |
| 2 | Each finding has location, severity, evidence level, and recommended action. | PASS | Every row in §Pass 3, §Pass 4, §Pass 5 tables has all 6 (or 7 for Pass 5) columns populated. Action ∈ {fix before porting, port differently, leave behind}. |
| 3 | Pass 5 findings cite the contract or protocol reference they violate. | PASS | Every Pass 5 row has `Spec Reference` column populated with at least one of: F-CLI-N / F-AI-N / F-AGENT-N / F-TUI-N / F-WEB-N / F-EXT-N / EC1-6 / SM1-6 / PS1-7 / Acceptance #N / con-OQ4. |
| 4 | Findings are organized by pass and sorted by severity; summary tables match the detailed findings. | PASS | One section per pass; sort is critical → high. Summary tables: 3 critical (1 in P4, 2 in P5) + 34 high = 37 per-finding rows. Counts match. |
| 5 | Findings are marked with evidence levels. | PASS | Every row's `Evidence Level` ∈ {observed fact, strong inference}. No `open question` evidence-level findings (those went to sem-OQ1..7 list). |
| 6 | Any carry_forward entries that targeted defect-scan-semantic have been resolved or explicitly re-routed. | PASS | All 11 inbound carry-forwards resolved (table above): dsm-RT1/2/3/4/9/10/11, arch-CF8, arch-CF9, con-CF5, con-CF6. None re-routed (porting phase will consolidate). |
| 7 | Findings are marked with evidence levels. | PASS | (Duplicate of #5; pipeline yaml has the criterion listed twice — both satisfied.) |

**Validated by:** 2026-05-06 (defect-scan-semantic phase, session 1, no-orchestrator mode)
**Overall:** PASS

(11 inbound carry-forwards CLOSED. 7 new open questions surfaced — all needing runtime evidence or maintainer policy. **One critical correction** to architecture/contracts/protocols framing recorded in §Important Correction — porting phase must consolidate the corrected RCE perimeter view.)
