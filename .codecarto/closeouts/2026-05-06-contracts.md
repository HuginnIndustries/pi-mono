# Closeout — 2026-05-06 — contracts

## Summary

Documented user-visible behavior across 12 surfaces (CLI, 3 modes, 4 library
APIs, 4 storage formats including project + operator scopes, extension API,
slash commands, browser custom elements). Resolved 4 carry-forwards
(arch-CF1 7-tool contracts, arch-CF2 auth model, dsm-RT5 extension API,
dsm-RT7 Codex retry). Surfaced 6 important corrections to the architecture
phase (`streamProxy` is a CLIENT not server; method names; event count;
state-anchor env var name; tool source filenames; ~25 vs 22 events).
Captured 18 black-box acceptance scenarios. Surfaced 1 new open question
(`con-OQ3`: `streamSimple*` sync-throw vs documented stream-error
contract) routed forward to Pass 5.

5 carry-forwards deferred to protocols phase (12-variant event catalog,
session JSONL schema, TUI render protocol, 22-event extension catalog,
streamProxy SSE framing).

Next session starting point: protocols phase. 5 carry-forwards
pre-routed: `arch-CF3`, `arch-CF4`, `arch-CF5`, `con-CF2`, `con-CF4`.

## Files Touched

- **Added:**
  - `findings/contracts/behavioral-contracts.md` (primary)
  - `scratch/contracts-{cli,tools,extensions,pi-ai,pi-agent-core,tui-webui,config-auth}.md` (7 extraction notes; gitignored)
- **Modified:**
  - `findings/public-surfaces/public-surfaces.md` (append: contracts-phase corrections + CLI flag/slash command/element inventory)
  - `findings/runtime-lifecycle/runtime-lifecycle.md` (append: mode dispatch refinement, 5-AbortController cancellation table, signal table, extension lifecycle sub-events, streamProxy CLIENT correction, migrations)
  - `findings/state-and-storage/state-and-storage.md` (append: migrations, models.json !-prefix channel, 0600 enforcement update, extension state)
  - `findings/config-model/config-model.md` (append: PI_CODING_AGENT_DIR correction, per-provider env var table, file schema authoritative reference, project-scoped layer, migration layer, trust-boundary reminder)
  - `workflow/status.yaml` (contracts → complete; current_phase → protocols; carry_forward populated for protocols + defect-scan-semantic)
  - `THREAD_LOG.md` (index entry appended)
- **Deleted:** none

## Tests / Gates

| Gate | Result | Notes |
|---|---|---|
| Validation block PASS | PASS | All 6 criteria PASS; no PARTIAL or FAIL. |
| Architecture carry-forwards resolved | PASS | arch-CF1 → 7-tool table; arch-CF2 → §Authentication 5-level table. |
| Mechanical-phase carry-forwards resolved | PASS | dsm-RT5 → F-EXT-1..4; dsm-RT7 → F-AI-1 retry contract. |
| Secondary outputs appended | PASS | All 4 declared appends written with dated 2026-05-06 contracts-phase sections. |

## Decisions Beyond Prompt

- **D005** | Acceptance scenarios cite known defects (e.g., #5 ambiguous
  prefix, #9 edit data-loss, #10 bash exit code) | Choice between writing
  spec-as-intended (no defects) and spec-as-observed (defects in place).
  Chose observed because the porting/reimpl-spec phases need both: the
  spec-as-intended for the new build, the spec-as-observed for parity
  testing against the existing pi binary. Each defect-related scenario
  marks "**Currently**: ..." and "Spec requires ...".
- **D006** | Recorded architecture-phase corrections in the contracts
  phase's "Doc/Test Conflicts" section rather than amending the
  architecture-map.md retroactively | Per GUIDE.md "phase outputs can't
  correct prior phases" guidance — corrections live where they are
  surfaced, with cross-references to status.yaml (which also still
  carries the original arch-CFn IDs).

## Proposed Conventions

### C5 Cross-Phase Correction-Routing Convention

**Why:** Contracts phase surfaced 6 architecture-phase inaccuracies
(streamProxy is a client; method names; event count; etc.). Without a
documented place to surface them, downstream phases (porting, reimpl)
might cite the architecture-map.md and propagate the errors.

**How to apply:** Each phase's "Doc/Test Conflicts" section should also
flag prior-phase corrections explicitly, with the prior phase named. The
porting / reimpl phases should read the corrections list as part of
their pre-flight, not just the prior primary outputs.

### C6 Defect Cross-References in Contract Tables

**Why:** Linking each contract row's "Error behavior" cell back to the
mechanical-defects row (e.g., "**Defect 1.13/2.3**:") makes the
contracts ↔ defects round-trip auditable and makes downstream phases
cite the defect-of-record.

**How to apply:** Where a contract row mentions error behavior
documented as a defect, cite the defect ID inline. Mech defects use
`<pass>.<num>` (e.g., `1.13`).

## Open Questions Left Behind

| ID | Kind | Description |
|---|---|---|
| con-OQ1 | needs-runtime-test | pi-debug.log secret-leak audit |
| con-OQ2 | needs-runtime-test | auth.json mode 0600 on Windows ACL |
| con-OQ3 | needs-spec-ruling | streamSimple* sync-throw contract drift |
| con-OQ4 | needs-maintainer-decision | Listener removal during fan-out iteration |
| con-OQ5 | needs-spec-ruling | Extension API single-source-of-truth for 22 events |

## Carry-Forward Routed

| ID | Target Phase | Description |
|---|---|---|
| con-CF1 | protocols (subsumed by arch-CF3) | 12-variant AssistantMessageEvent field-level schema |
| con-CF2 | protocols | 22-event extension catalog with handler-input-shape + return-semantics |
| con-CF3 | protocols (subsumed by arch-CF4) | Session JSONL v3 record schema |
| con-CF4 | protocols | streamProxy SSE wire framing (data: <json>\n vs canonical SSE) |
| con-CF5 | defect-scan-semantic | dsm-RT1 covert-RCE Pass 4 framing reaffirmed |
| con-CF6 | defect-scan-semantic | con-OQ3 streamSimple* sync-throw contract drift Pass 5 framing |

## Framework Feedback (optional)

Two contracts-phase agents timed out at full-scope (one twice). Splitting
into smaller per-topic agents (CLI / 7 tools / extensions; config + auth
focused retry) succeeded. Worth noting in the framework guidance: for
large-package contracts work, prefer 3-5 narrow agents over 1-2 wide
ones, especially when the package has thick mode-dispatch + tool
inventory + extension API surfaces all at once.

## Next Session Pointer

Run **protocols** phase. Read `findings/protocols/SKILL.md` and
`templates/protocols-and-state.md`. Required reads:
`findings/architecture/architecture-map.md`,
`findings/defect-scan-mechanical/mechanical-defects.md`,
`findings/contracts/behavioral-contracts.md`. Resolve 5 carry-forwards:

- **arch-CF3** + **con-CF1**: 12-variant AssistantMessageEvent field-level catalog (event names, fields per event, ordering guarantees, terminal events).
- **arch-CF4** + **con-CF3**: Session JSONL v3 record schema, branch-edit rewrite semantics, replay rules.
- **arch-CF5**: TUI render protocol — synchronized output mode 2026, custom APC CURSOR_MARKER, diff strategy, width-change full-clear, Kitty/iTerm2 image escapes, OSC 9;4 progress.
- **con-CF2**: 22-event extension catalog with handler-input-shape + return-semantics per event. Source-of-truth: `types.ts:950-972` (the `ExtensionEvent` union), `types.ts:1089-1126` (`on()` overloads), and the runner's emit methods.
- **con-CF4**: streamProxy SSE wire framing — `data: <json>\n` (custom) vs canonical SSE `\n\n`; no heartbeat/keepalive parsing (arch-OQ3); event union at `proxy.ts:36-57`; partial-stripping protocol between server and client.

Append to secondary outputs: `public-surfaces.md`, `runtime-lifecycle.md`, `state-and-storage.md`, `config-model.md`.
