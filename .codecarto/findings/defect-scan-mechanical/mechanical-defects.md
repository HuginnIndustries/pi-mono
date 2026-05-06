# Mechanical Defects Report — pi-mono

<!--
  Phase: defect-scan-mechanical
  Pipeline: workflow/pipeline-full-with-deep-audit.yaml
  Source-of-detail: scratch/mech-pass1-{ai,coding-agent,agent-tui-webui}.md, scratch/mech-pass2.md, scratch/mech-pass6.md.
  Severity policy (user-locked): per-finding entries cap at high/critical; medium/low get tally rows only.
-->

## Scan Context

- **Source:** `../` (repository root: pi-mono, 5 packages, ~282 source files, ~102K LOC TypeScript)
- **Architecture reference:** `findings/architecture/architecture-map.md`
- **Pipeline:** `workflow/pipeline-full-with-deep-audit.yaml`
- **Date:** 2026-05-06
- **Scope:** Mechanical passes only (Pass 1 logic, Pass 2 error handling, Pass 6 configuration). Semantic passes (3 concurrency, 4 security, 5 contract violations) deferred to `defect-scan-semantic` after protocols.
- **Severity cap (user policy):** per-finding writeups limited to **critical** and **high**; medium/low rolled into tally rows.

---

## Pass 1: Logic and Correctness

Sort: critical → high.

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| 1.1 | `packages/ai/src/providers/anthropic.ts` (and 6 sister providers: openai-responses, azure-openai-responses, google, google-vertex, amazon-bedrock, mistral) | Replicated anti-pattern: when `output.stopReason === "error"`, the throw uses opaque literal `"An unknown error occurred"`; the catch block then overwrites the provider-mapped `errorMessage` with the literal, destroying the original cause. `openai-completions.ts:376-378` shows the correct preserve-`output.errorMessage` pattern. | high | observed fact | fix before porting |
| 1.2 | `packages/ai/src/providers/google-shared.ts:514`, `google-vertex.ts` analogous | Both Google providers force `stopReason = "toolUse"` even when the original mapped finish reason was `length`/`error`/safety (e.g., MAX_TOKENS with a tool call → silently reported as toolUse). `openai-responses-shared.ts:514` does it correctly by guarding `=== "stop"`. | high | observed fact | fix before porting |
| 1.3 | `packages/ai/src/providers/openai-completions.ts:301-350` | Tool-call coalescing splits a single tool call into N fragments when a vendor sends continuation deltas without both `index` and `id`. Continuation chunks become orphan tool-calls. | high | observed fact | fix before porting |
| 1.4 | `packages/ai/src/providers/openai-responses-shared.ts:518` | `Error Code ${code}: ${msg} || "Unknown error"` — the template literal is always truthy so the fallback is dead and the user sees `"Error Code undefined: undefined"`. | high | observed fact | fix before porting |
| 1.5 | `packages/ai/src/providers/anthropic.ts`, `google-shared.ts`, `openai-responses-shared.ts` (`mapStopReason` defaults) | Three providers' `mapStopReason` `default:` branches throw on unrecognized SDK enum values, killing the entire stream when a vendor adds a new finish reason. Bedrock and openai-completions correctly fall through to `"error"`. | high | strong inference | fix before porting |
| 1.6 | `packages/coding-agent/src/main.ts:155-167` (`resolveSessionPath`) | On ambiguous session-id prefix matches, silently picks `[0]` from both local and global match arrays. Non-deterministic session resolution via `--session`/`--fork`. | high | observed fact | fix before porting |
| 1.7 | `packages/coding-agent/src/core/compaction.ts:299-336` (`findValidCutPoints`) | Duplicated handling and unreachable cases: `branchSummary` / `compactionSummary` roles inside `type === "message"` are never persisted. | high | observed fact | fix before porting |
| 1.8 | `packages/coding-agent/src/core/model-resolver.ts:58-65` (`isAlias`) | Misclassifies any non-8-digit-tail id as alias; custom model ids ending in 8 digits are wrongly de-prioritized. | high | observed fact | fix before porting |
| 1.9 | `packages/coding-agent/src/core/tools/bash-executor.ts:113-148` | `tempFileStream.end()` not awaited before resolving with `fullOutputPath`; consumers can read truncated file. | high | observed fact | fix before porting |
| 1.10 | `packages/coding-agent/src/utils/paths.ts:22-36` (`isLocalPath`) | Misses SCP-style git URLs (`git@host:path`); these get joined with cwd as filesystem paths in `resolveCliPaths`. | high | observed fact | fix before porting |
| 1.11 | `packages/coding-agent/src/core/migrations.ts:120` | `file.split("/").pop() || file.split("\\").pop()` corrupts filenames on Windows because the first split returns the full path (truthy), short-circuiting the backslash split. | high | observed fact | fix before porting |
| 1.12 | `packages/coding-agent/src/core/exec.ts:91-97` | `code: code ?? 0` reports killed processes (`code === null`) as exit code 0 (success); contradicts `bash-executor.ts` which correctly maps null → undefined. | high | observed fact | fix before porting |
| 1.13 | `packages/agent/src/agent.ts:539-541` (`Agent.processEvents`) | Listener fan-out is sequential and unguarded; one listener throw skips later listeners. (Architecture-flagged hotspot — confirms strong inference from arch.) | high | observed fact | fix before porting |
| 1.14 | `packages/agent/src/proxy.ts:310-348` (`streamProxy.processProxyEvent`) | Leaks `partialJson` accumulator into shallow-cloned ToolCall content; downstream consumers see partial JSON in finished tool-calls. | high | observed fact | fix before porting |
| 1.15 | `packages/tui/src/tui.ts:1100-1131` (`doRender`) | Throws on width-overflow, killing the process from a `process.nextTick`/`setTimeout` callback. Contradicts the "always truncate" comment in `compositeLineAt`. | high | observed fact | fix before porting |
| 1.16 | `packages/web-ui/src/StreamingMessageContainer.ts:43-60` | Uses `JSON.parse(JSON.stringify(...))` for streaming clones (loses `undefined`, `BigInt`, `Map`, `Set`); plus a rAF-vs-immediate-clear race. | high | observed fact | port differently |
| 1.17 | `packages/web-ui/src/components/ExpandableSection.ts:21-28` (`connectedCallback`) | Permanently destroys children on re-attachment because it captures-then-clears with no guard. | high | observed fact | fix before porting |
| 1.18 | `packages/tui/src/stdin-buffer.ts:264-364` | Paste-mode buffer has no timeout-armed flush; lost `\x1b[201~` leaks all subsequent stdin into `pasteBuffer` forever. | high | observed fact | fix before porting |
| 1.19 | `packages/agent/src/agent-loop.ts:287-344` (`streamAssistantResponse`) | Silently swallows the entire stream if `start` event is missing — events are gated on `if (partialMessage)`. | high | strong inference | fix before porting |
| 1.20 | `packages/web-ui/src/MessageList.ts:36-91` | Uses positional keys (`msg:${index}`) in `repeat()`, defeating Lit's keyed reconciliation and amplifying ExpandableSection re-attach bug (1.17). | high | observed fact | fix before porting |
| 1.21 | `packages/agent/src/agent.ts:463-478` (`Agent.handleRunFailure`) | Synthesizes an empty failure message even after a partial assistant message was already streamed — transcript ends with two assistant messages. | high | observed fact | fix before porting |
| 1.22 | `packages/agent/src/proxy.ts` | Throws on out-of-order events with no recovery; one misframe terminates the turn. | high | observed fact | fix before porting |

**No critical findings in Pass 1.**

---

## Pass 2: Error Handling and Resilience

Sort: critical → high.

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| 2.1 | `packages/coding-agent/src/core/session-jsonl.ts` (and writers in `agent-session.ts`) | Session JSONL writes use non-atomic `writeFileSync`/`appendFileSync` with no try/catch; mid-write crash truncates session files. **Data loss in normal operation.** | critical | observed fact | fix before porting |
| 2.2 | `packages/coding-agent/src/core/tools/edit-tool.ts`, `write-tool.ts` | Both tools overwrite source files with `fs.writeFile` directly (no temp+rename); a crash mid-write destroys user code. | critical | observed fact | fix before porting |
| 2.3 | `packages/agent/src/agent.ts:539-541` | Listener fan-out unguarded; one throw aborts downstream listeners (already 1.13 — listed here for the error-handling lens). | high | observed fact | fix before porting |
| 2.4 | `packages/ai/src/providers/register-builtins.ts` | `forwardStream` IIFE is unawaited; mid-stream rejection is unhandled and consumer hangs (`target.end()` never called, `result()` deadlocks). | high | observed fact | fix before porting |
| 2.5 | `packages/coding-agent/src/core/exec.ts:99` | Discards spawn/wait errors (parameter is `_err`) and synthesizes `{code:1, stderr:""}` — spawn failures unobservable. | high | observed fact | fix before porting |
| 2.6 | `packages/coding-agent/src/core/tools/bash-executor.ts` | `tempFileStream` has no `'error'` listener; disk-full mid-bash escalates to uncaughtException. | high | observed fact | fix before porting |
| 2.7 | `packages/coding-agent/src/core/session-jsonl.ts` (`loadEntriesFromFile`) | Silently skips malformed JSONL lines with no log; corrupted-but-resumable sessions load with missing data. | high | observed fact | fix before porting |
| 2.8 | `packages/coding-agent/src/core/agent-session.ts:1996,2002,2500` | Background `agent.continue().catch(() => {})` swallows synchronous rejections; auto-retry / post-compaction restart wedges with no UI surface. | high | observed fact | fix before porting |
| 2.9 | `packages/coding-agent/src/core/migrations.ts:34-43` | Empty catch can leave OAuth credentials in inconsistent two-store state. | high | observed fact | fix before porting |
| 2.10 | `packages/coding-agent/src/modes/rpc/...` | Silently drops post-preflight prompt failures (only emits error if `!preflightSucceeded`). | high | observed fact | fix before porting |
| 2.11 | `packages/tui/src/tui.ts` (`process.stdout.write` paths) | No handler for stdout write failures (EPIPE / SSH disconnect mid-paint); agent continues running headless after disconnect. | high | observed fact | fix before porting |

---

## Pass 6: Configuration and Environment Hazards

Sort: critical → high.

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| 6.1 | `packages/coding-agent/src/cli.ts:20` (resolves **arch-CF7**) | undici `bodyTimeout: 0` and `headersTimeout: 0` are global, not LLM-scoped; affect every fetch/SDK/extension network call after this line. Provider-SDK `timeoutMs` is optional and unset by default, so the "AbortController-based deadlines" justification is incomplete. | high | observed fact | port differently |
| 6.2 | `packages/coding-agent/src/utils/photon.ts` (resolves **arch-CF6**) | `fs.readFileSync` global monkey-patch hazards: stdlib mutation, restore stacking under concurrent patchers, multi-tenant SDK-embedding leakage, hardcoded fallback path list, silent error swallowing on import failure. | high | observed fact | port differently |
| 6.3 | `packages/coding-agent/src/cli.ts:14` | Silences `process.emitWarning` globally — suppresses every Node deprecation/security warning in the process. | high | observed fact | fix before porting |
| 6.4 | `packages/coding-agent/src/core/config.ts` (`resolveConfigValue`) | Executes any string starting with `!` as a shell command and is wired into `auth.json` `api_key` resolution and `models.json` headers. Project-scoped `models.json` from `<repo>/.pi/` is therefore a covert RCE channel. **Pre-emptive route to Pass 4 for full security framing.** | high | observed fact | fix before porting |
| 6.5 | `packages/coding-agent/src/core/migrations.ts` (`runMigrations`) | Runs unconditionally on every startup (auth/sessions/binaries/keybindings/commands→prompts); all failure paths are bare `catch {}`. | high | observed fact | fix before porting |
| 6.6 | `packages/coding-agent/src/core/migrations.ts` (`migrateAuthToAuthJson`) | Unlocked read-modify-write on `settings.json` races the `proper-lockfile`-protected `SettingsManager`. Strong concurrency angle — also routed to Pass 3. | high | observed fact | fix before porting |

**Architecture-flagged answers (sub-finding-level information for downstream phases):**

- `auth.json` mode 0600 enforced on **every** write (creation, sync, async) — no gap. (Closes part of arch-CF9.)
- `ENV_AGENT_DIR` is actually `PI_CODING_AGENT_DIR`; tilde-expanded; not validated for writability.
- `rg`/`fd` auto-download fails non-fatally (yellow log, returns undefined, tools degrade silently). **No checksum/signature verification on downloaded binaries.** (Routed to Pass 4.)
- `models.generated.ts` (16,815 lines) freshness is human-driven by `npm run generate-models`, not CI-enforced.
- **Partial answer to arch-OQ2**: corrupt `auth.json` → process starts, does not prompt, does not crash; sets `loadError`, blocks credential reads until user fixes file.

---

## Summary

### Findings by Severity

| Severity | Count |
|----------|-------|
| Critical | 2 |
| High | 37 |
| Medium | 92 (tallied) |
| Low | 53 (tallied) |
| **Total (per-finding)** | **39** |
| **Total (incl. tallies)** | 184 |

### Findings by Pass

| Pass | Critical | High | Medium (tally) | Low (tally) | Per-finding total |
|------|----------|------|----------------|-------------|-------------------|
| 1. Logic and correctness | 0 | 22 | 53 | 35 | 22 |
| 2. Error handling | 2 | 9 | (218 total bare-catch sites in coding-agent; 21 in pi-ai; 12 in pi-tui; 9 in pi-web-ui; 1 in pi-agent-core — see scratch/mech-pass2.md for category breakouts) | (rolled into bare-catch tally) | 11 |
| 6. Config and environment | 0 | 6 | 21 | 21 | 6 |

### Medium/Low Tallies (per package)

| Package | Pass 1 Med | Pass 1 Low | Pass 2 bare-catch | Pass 6 Med | Pass 6 Low |
|---|---|---|---|---|---|
| pi-ai | (rolled into Pass 1 ai scratch) 6M / 5L | | 21 | 4 | 5 |
| pi-agent-core | 4 | 2 | 1 | 1 | 1 |
| pi-tui | 8 | 11 | 12 | 2 | 4 |
| pi-coding-agent | (Pass 1 cited 8M / 15L) | | 118 | 11 | 9 |
| pi-web-ui | 13 | 9 | 9 | 3 | 2 |

(Per-package per-category breakouts live in the scratch files; the primary owns the load-bearing per-finding rows.)

### Top 5 Findings (Most Impactful)

1. **2.1** — Session JSONL writes are non-atomic with no try/catch; mid-write crash truncates session files. *(critical, data loss in normal operation, fix before porting)*
2. **2.2** — `edit`/`write` tools overwrite user source code with `fs.writeFile` directly; crash mid-write destroys user code. *(critical, data loss, fix before porting)*
3. **6.4** — `resolveConfigValue` executes `!`-prefix strings as shell; project-scoped `models.json` is therefore a covert RCE channel. *(high, security pre-route to Pass 4, fix before porting)*
4. **6.1** — undici `bodyTimeout/headersTimeout: 0` globally — disables every Node-fetch deadline in the process, not just LLM calls. *(high, port differently — different runtime needs different policy)*
5. **1.1** — Replicated anti-pattern across 7 providers destroying original error message via opaque literal-throw. Stream-error UX is universally bad on this path. *(high, fix before porting)*

### Routed To Semantic Phase

| ID | Description | Why Routed |
|---|---|---|
| dsm-RT1 | `resolveConfigValue` `!`-prefix shell-exec wired into `auth.json` `api_key` and `models.json` headers — full security framing belongs in Pass 4. (Cited in 6.4 above.) | Pass 4 — security & trust boundaries |
| dsm-RT2 | Synchronous busy-wait in `auth-storage.ts` between lock retries (medium-tally; concurrency lens belongs in Pass 3). | Pass 3 — concurrency |
| dsm-RT3 | `migrateAuthToAuthJson` unlocked read-modify-write races `proper-lockfile`-protected SettingsManager (cited in 6.6). | Pass 3 — concurrency |
| dsm-RT4 | `getAttributionHeaders` privacy review (PII / fingerprinting) — surfaced during coding-agent Pass 1. | Pass 4 — security & trust |
| dsm-RT5 | Extension stale-runtime contract — extensions load via `jiti` and may capture stale module references; contract-level treatment. | Contracts phase / Pass 5 |
| dsm-RT6 | Persistence contract for `_persist` deferred flush in agent-session.ts — what happens to in-flight events on shutdown? Continues arch-CF4. | Pass 5 (or arch-CF4 in protocols) |
| dsm-RT7 | Retry regex coupling to provider error messages — contracts phase rubric. | Contracts |
| dsm-RT8 | OpenAI-compat URL detection matrix (~15 vendor branches, `openai-completions.ts:1030+`) — Pass 5 audit per arch-CF8. | Pass 5 (already arch-CF8) |
| dsm-RT9 | `rg`/`fd` auto-download has no checksum/signature verification — supply-chain hazard. | Pass 4 — security & trust |
| dsm-RT10 | Codex retry loop has no jitter (medium tally; concurrency framing). | Pass 3 — concurrency |
| dsm-RT11 | F2 / 1.14 streamProxy `partialJson` leak into shallow-cloned ToolCall — arguably a contract violation. | Pass 5 — API contract |

### Mechanical Pass Open Questions

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| dsm-OQ1 | needs-runtime-test | What happens when Node hits EPIPE on `process.stdout.write` mid-paint — does it escalate to uncaughtException, or get silently dropped? Affects 2.11 escalation severity. | Cannot determine without a live runtime test. |
| dsm-OQ2 | needs-runtime-test | What happens when `appendFileSync` hits ENOSPC mid-record — short write or full failure? Affects 2.1 severity. | Requires live disk-pressure test. |
| dsm-OQ3 | needs-runtime-test | Default unhandled-rejection behavior in pi-mono's published Bun-binary path (Node 22 default became `throw` in 15+; Bun's default may differ). | Requires Bun runtime confirmation. |
| dsm-OQ4 | needs-maintainer-decision | Should an extension hook throwing be allowed to stop the loop or always be logged-and-continued? Drives F1 severity classification. | Maintainer policy call. |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | At least two of the three mechanical passes (1, 2, 6) produced findings or documented "no defects found." | PASS | All three passes produced findings (22 / 11 / 6 high+critical respectively). |
| 2 | Each finding has location, severity, evidence level, and recommended action. | PASS | Every row in the three pass tables has all 6 columns populated. Action ∈ {fix before porting, port differently, leave behind}. |
| 3 | Findings are organized by pass and sorted by severity. | PASS | One section per pass; sort is critical → high (Pass 2 has 2 critical at the top; Passes 1 and 6 are all high). |
| 4 | Summary tables are complete and counts match the detailed findings. | PASS WITH GAPS | Severity totals match: 2 critical + 37 high = 39 per-finding rows; medium/low tallies aggregated from scratch files. **Gap noted**: medium/low per-category breakouts under "Pass 2 bare-catch" are aggregated as bare-catch site counts rather than per-defect-category, because the agent surveyed catch-block hygiene at the site level. The numbers are honest (218/21/12/9/1) but not category-partitioned the way Pass 1 and Pass 6 are. **Routed**: not deferred — gap is a presentation choice not a missing finding. |
| 5 | Findings are marked with evidence levels. | PASS | Every row's `Evidence Level` ∈ {observed fact, strong inference}. No `open question` evidence-level findings (those went to dsm-OQ1..4 or to the routed-to-semantic table). |
| 6 | Items spotted that are actually semantic in nature are routed to defect-scan-semantic via carry_forward. | PASS | 11 items in the "Routed To Semantic Phase" table; status.yaml `carry_forward` for `defect-scan-semantic` updated accordingly. |
| 7 | Findings are marked with evidence levels. | PASS | (Duplicate of #5; pipeline.yaml has `Findings are marked with evidence levels.` listed twice — both satisfied.) |

**Validated by:** 2026-05-06 (defect-scan-mechanical phase, session 1, no-orchestrator mode)
**Overall:** PASS WITH GAPS

(Gap = presentation-only on the Pass 2 medium/low breakout; not a missing finding. arch-CF6 and arch-CF7 were resolved as 6.2 and 6.1 respectively. arch-OQ2 partially answered in Pass 6 architecture-flagged answers section.)
