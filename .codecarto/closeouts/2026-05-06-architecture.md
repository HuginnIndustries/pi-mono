# Closeout — 2026-05-06 — architecture

## Summary

Mapped pi-mono into 5 packages with two stable bases (`pi-ai` provider
normalization, `pi-tui` terminal primitives), a core-semantics layer
(`pi-agent-core`), and two independent delivery shells
(`pi-coding-agent` shipping the `pi` CLI; `pi-web-ui` shipping Lit web
components). Recorded the load-bearing build order
(tui→ai→agent→coding-agent→web-ui), the 5-level auth precedence resolved
in coding-agent (pi-ai itself only reads env + Vertex ADC), JSONL session
persistence v3 under `~/.pi/agent/sessions/`, and 6 explicit portability
hazards (synchronized output mode 2026, koffi FFI, photon WASM
monkey-patch, undici timeouts disabled, rg/fd auto-download, generated
`models.generated.ts`). Routed 9 carry-forward items to downstream phases
(2 to contracts, 3 to protocols, 2 to defect-scan-mechanical, 2 to
defect-scan-semantic) and 3 open questions that need runtime evidence.

Next session starting point: defect-scan-mechanical (passes 1, 2, 6) with
the high/critical severity cap. arch-CF6 (photon monkey-patch) and arch-CF7
(undici timeouts) are pre-routed to that phase.

## Files Touched

- **Added:**
  - `findings/architecture/architecture-map.md` (primary output)
  - `findings/public-surfaces/public-surfaces.md` (secondary, dated section appended)
  - `findings/runtime-lifecycle/runtime-lifecycle.md` (secondary, dated section appended)
  - `findings/state-and-storage/state-and-storage.md` (secondary, dated section appended)
  - `findings/build-and-deploy/build-and-deploy.md` (secondary, dated section appended)
  - `findings/config-model/config-model.md` (secondary, dated section appended)
  - `scratch/arch-ai.md`, `scratch/arch-agent-tui.md`, `scratch/arch-coding-agent.md`, `scratch/arch-web-ui.md`, `scratch/arch-build.md` (extraction notes; gitignored)
- **Modified:**
  - `workflow/status.yaml` (architecture phase → complete; current_phase → defect-scan-mechanical; carry_forward items routed)
  - `THREAD_LOG.md` (index entry appended)
- **Deleted:** none

## Tests / Gates

Source-only analysis phase; no pi-mono test suite was run (out of scope per
plan). The framework's only "gate" for this phase is the validation block
at the bottom of `findings/architecture/architecture-map.md`.

| Gate | Result | Notes |
|---|---|---|
| Validation block PASS | PASS | All 5 completion criteria PASS; no FAIL or PARTIAL. |
| Secondary outputs present | PASS | All 5 dated sections appended. |
| Carry-forward routed | PASS | 9 entries with explicit `target_phase`. |
| Open questions structured | PASS | 3 entries with `kind` and `deferred_reason`. |

## Decisions Beyond Prompt

- **D001** | Web-ui correctly classified as Lit, not React | The original
  task brief assumed React; extraction agent verified zero React imports
  in `packages/web-ui/src/`. All architecture-map and secondary outputs
  reflect Lit `LitElement` / `@customElement` reality. Worth pinning
  because downstream phases will rely on this classification.
- **D002** | Severity-cap policy for defect scans documented in
  status.yaml `next_actions` | Per user preference selected during plan
  approval, defect scans cap per-finding writeups at high/critical;
  medium/low get tally-only entries. Recorded so subsequent phases can
  honor the contract.

## Proposed Conventions

### C1 Per-finding Evidence Level Tags

**Why:** The architecture-map's section-by-section discipline of tagging
each claim `(observed fact / strong inference / portability hazard / open
question)` materially improved review hygiene — reviewers could scan
without tracing every cite.

**How to apply:** All downstream primary outputs (contracts, protocols,
porting, reimpl-spec) should adopt the same in-line tag style. Tagging
should appear at the end of each load-bearing claim, not just in section
headers.

### C2 Scratch-File-Per-Subagent Convention

**Why:** With 5 parallel extraction agents writing
`scratch/arch-{ai,agent-tui,coding-agent,web-ui,build}.md`, the primary
output became a synthesis of named scratch artifacts — each cite-able by
the closeout and durable across phases. This let the primary session
keep context budget for the load-bearing reasoning.

**How to apply:** Subsequent phases requiring bulk extraction should use
the same `scratch/<phase-prefix>-<package-or-topic>.md` naming. The
primary output cites scratch as the source-of-detail in a header
comment.

## Open Questions Left Behind

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| arch-OQ1 | needs-runtime-test | Does pi startup rg/fd auto-download fail gracefully (or block) when offline? | Requires live offline-VM boot. |
| arch-OQ2 | needs-runtime-test | What happens when ~/.pi/agent/auth.json is corrupt JSON? | Parse-error path not obvious from source. |
| arch-OQ3 | needs-fixture-capture | streamProxy SSE heartbeat / keepalive framing? | Live capture required. |

## Carry-Forward Routed

| ID | Target Phase | Description |
|---|---|---|
| arch-CF1 | contracts | 7 built-in coding-agent tool contracts (parameters, defaults, side-effect categories) |
| arch-CF2 | contracts | 5-level auth precedence as a contract row + security/auth-model section |
| arch-CF3 | protocols | AssistantMessageEvent 12-variant catalog (producer/consumer/transport/ordering/fields) |
| arch-CF4 | protocols | JSONL session format v3 schema + branch-edit rewrite semantics |
| arch-CF5 | protocols | TUI render protocol (synchronized output 2026, APC CURSOR_MARKER, Kitty/iTerm2 escapes) |
| arch-CF6 | defect-scan-mechanical | photon WASM monkey-patch of fs.readFileSync |
| arch-CF7 | defect-scan-mechanical | undici bodyTimeout/headersTimeout: 0 at CLI startup |
| arch-CF8 | defect-scan-semantic | OpenAI-compat URL matrix (~15 vendor branches) Pass-5 audit |
| arch-CF9 | defect-scan-semantic | Credential read/write trust-boundary review (mode 0600, fallback paths, parse errors) |

## Framework Feedback (optional)

Two minor frictions worth noting:

1. The framework's `.codecarto/.gitignore` ships configured to ignore
   the actual analysis outputs (architecture-map.md, defect-report.md,
   etc.). That makes sense for the framework template repo (where the
   .codecarto/ directory is template-only) but actively defeats a
   "drop into target repo and commit findings" workflow. Pi-mono's
   .gitignore was reset to ignore only `scratch/*` and OS artifacts.
   Worth a thought upstream — perhaps shipping two .gitignore variants
   (`gitignore.template` + `gitignore.target`).
2. The `closeout-template.md` Tests / Gates table hardcodes
   `bun install` / `tsc --noEmit` / `oxlint` / `bun test`. These are
   irrelevant for source-only analysis phases; the gate that matters
   is the validation block. Renaming the section "Tests / Gates" to
   "Phase Gates" or making it free-form would reduce the slight
   awkwardness of N/A-ing those rows.

## Next Session Pointer

Run **defect-scan-mechanical** (passes 1, 2, 6). Read
`findings/defect-scan-mechanical/SKILL.md` and the three pass files
(`findings/defect-scan/passes/01-logic-and-correctness.md`,
`02-error-handling.md`, `06-config-and-environment.md`). Required reads
include `findings/architecture/architecture-map.md`. Resolve
carry-forward `arch-CF6` and `arch-CF7` in the output. Apply the
high/critical severity cap; medium/low get a per-pass tally row only.
Route any spotted Pass 3/4/5 items forward as
`target_phase: defect-scan-semantic` carry-forward entries.
