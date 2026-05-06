# Closeout — 2026-05-06 — defect-scan-mechanical

## Summary

Ran Pass 1 (logic), Pass 2 (error handling), and Pass 6 (config/environment)
against pi-mono's five packages with five parallel extraction agents. Surfaced
**39 high/critical per-finding entries** (22 Pass 1, 11 Pass 2 incl. 2
critical, 6 Pass 6); medium/low rolled into per-package tallies per
user-locked severity-cap policy. Resolved both architecture-routed
carry-forwards (arch-CF6 photon WASM monkey-patch as P6.2; arch-CF7 undici
global timeout-disable as P6.1). Surfaced two **critical data-loss
findings**: non-atomic JSONL session writes (P2.1) and `edit`/`write` tools
overwriting user code without temp+rename (P2.2). Routed 11 items forward to
the semantic phase (5 to Pass 4 security incl. a covert-RCE channel via
`models.json` `!`-prefix shell exec, 3 to Pass 3 concurrency, 1 to Pass 5
contract violation, plus arch-CF8 / arch-CF9 already in flight).

Next session starting point: contracts phase. Carry-forward IDs
`arch-CF1`, `arch-CF2`, `dsm-RT5`, `dsm-RT7` are pre-routed to that phase.

## Files Touched

- **Added:**
  - `findings/defect-scan-mechanical/mechanical-defects.md` (primary output)
  - `scratch/mech-pass1-ai.md`, `mech-pass1-coding-agent.md`, `mech-pass1-agent-tui-webui.md`, `mech-pass2.md`, `mech-pass6.md` (extraction notes; gitignored)
- **Modified:**
  - `workflow/status.yaml` (defect-scan-mechanical → complete; current_phase → contracts; carry_forward populated for contracts + defect-scan-semantic)
  - `THREAD_LOG.md` (index entry appended)
- **Deleted:** none

## Tests / Gates

| Gate | Result | Notes |
|---|---|---|
| Validation block PASS | PASS WITH GAPS | All 7 criteria PASS or PASS WITH GAPS; gap is presentation-only on Pass 2 medium tally (bare-catch site count rather than per-defect-category — honest but not category-partitioned). No FAIL. |
| Severity-cap policy applied | PASS | Per-finding rows for high/critical only; medium/low aggregated to tallies. |
| Architecture carry-forwards resolved | PASS | arch-CF6 → P6.2; arch-CF7 → P6.1. arch-OQ2 partially answered (corrupt auth.json: process starts, sets loadError, blocks credential reads). |
| Routed-to-semantic populated | PASS | 11 entries with explicit `target_phase: defect-scan-semantic` in status.yaml. |

## Decisions Beyond Prompt

- **D003** | Pass 2 bare-catch site count chosen over per-defect-category tally
  | The Pass 2 extraction agent surveyed catch-block hygiene at the call
  site, producing per-package counts of bare-catch sites (118 in
  coding-agent, 21 in pi-ai, 12 in pi-tui, 9 in pi-web-ui, 1 in
  pi-agent-core). Re-categorizing those 161 sites into the rubric's
  defect-categories (swallowed-error / inconsistent-propagation /
  missing-cleanup / etc.) was judged not worth the agent budget given the
  severity cap discards them anyway. The presentation gap is documented in
  the validation block.
- **D004** | Pre-emptive routing of P6.4 (`!`-prefix shell exec) to Pass 4
  via `dsm-RT1` even though it lives in the mechanical Pass 6 table | The
  finding meets Pass 6's "dangerous default" rubric (the mechanism is
  documented and intentional in the source) but its impact is only legible
  in Pass 4's trust-boundary framing. Listing it in both places is
  honest — the mechanical row says "what" and the semantic carry-forward
  says "why this is a security finding."

## Proposed Conventions

### C3 Severity-Cap Policy Gets Documented in the Output Header

**Why:** A reader scanning a defect-scan output should be able to tell at a
glance whether the absence of medium/low findings means "none found" or
"capped per policy." The mechanical-defects.md "Scan Context" section now
explicitly records the user-locked policy.

**How to apply:** Any defect-scan output (mechanical or semantic) should
state the severity-cap policy in its Scan Context section so future
readers (and the porting / reimpl-spec phases) don't conflate
"cap-suppressed" with "absent."

### C4 "Resolved arch-CFn" Citation in the Location Column

**Why:** When a mechanical-pass finding directly closes an architecture
carry-forward, citing the carry-forward ID in the same row makes the
status.yaml ↔ findings round-trip easy to audit.

**How to apply:** Use `(resolves arch-CFn)` immediately after the
file:line citation in the Location column. The orchestrator (or the
post-pipeline auditor) can grep for unresolved CF IDs cheaply.

## Open Questions Left Behind

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| dsm-OQ1 | needs-runtime-test | EPIPE escalation on stdout.write mid-paint | Live runtime test required |
| dsm-OQ2 | needs-runtime-test | appendFileSync ENOSPC mid-record short-write vs full-fail | Disk-pressure test required |
| dsm-OQ3 | needs-runtime-test | Bun-binary unhandled-rejection default | Bun runtime confirmation required |
| dsm-OQ4 | needs-maintainer-decision | Extension-hook-throw policy | Maintainer call |

## Carry-Forward Routed

| ID | Target Phase | Description |
|---|---|---|
| dsm-RT1 | defect-scan-semantic | resolveConfigValue !-prefix shell exec → Pass 4 covert-RCE framing |
| dsm-RT2 | defect-scan-semantic | auth-storage busy-wait between lock retries → Pass 3 |
| dsm-RT3 | defect-scan-semantic | migrateAuthToAuthJson unlocked read-modify-write race → Pass 3 |
| dsm-RT4 | defect-scan-semantic | getAttributionHeaders privacy review → Pass 4 |
| dsm-RT5 | contracts | Extension stale-runtime contract (jiti reload semantics) |
| dsm-RT6 | protocols (subsumed by arch-CF4) | _persist deferred-flush behavior on shutdown |
| dsm-RT7 | contracts | Retry regex coupling to provider error message strings |
| dsm-RT8 | defect-scan-semantic (= arch-CF8) | OpenAI-compat URL matrix Pass 5 audit |
| dsm-RT9 | defect-scan-semantic | rg/fd auto-download no checksum/signature → Pass 4 supply-chain |
| dsm-RT10 | defect-scan-semantic | Codex retry no jitter → Pass 3 thundering-herd |
| dsm-RT11 | defect-scan-semantic | streamProxy partialJson leak → Pass 5 contract violation |

## Framework Feedback (optional)

The pipeline-yaml's `defect-scan-mechanical` completion criteria duplicate
"Findings are marked with evidence levels." (it's listed twice in the
phase YAML's `completion_criteria`). The validation table reflects the
duplication. Minor — consider deduping in the framework.

## Next Session Pointer

Run **contracts** phase. Read
`findings/contracts/SKILL.md` and `templates/behavioral-contracts.md`.
Required reads: `findings/architecture/architecture-map.md` and
`findings/defect-scan-mechanical/mechanical-defects.md`. Resolve carry-forwards
**arch-CF1** (7 built-in tool contracts: read/bash/edit/write/grep/find/ls
— inputs, defaults, outputs, side-effects, error behavior),
**arch-CF2** (5-level auth precedence as a contract row + security/auth
section), **dsm-RT5** (extension stale-runtime contract), **dsm-RT7**
(retry regex coupling). The contract phase is **not severity-capped** —
contracts are descriptive, not findings. Append to the secondary outputs
that the contracts phase declares: `public-surfaces.md`,
`runtime-lifecycle.md`, `state-and-storage.md`, `config-model.md`.
