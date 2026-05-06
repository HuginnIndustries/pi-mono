# Closeout — 2026-05-06 — reimplementation-spec

## Summary

Wrote the language-agnostic reimplementation spec — the terminal
deliverable of the `pipeline-full-with-deep-audit` pipeline. 10
conceptual modules (M1 Provider Adapter through M10 Browser Chat
Component), partitioned across the three layers (core semantics /
adapters / delivery surfaces). 14 required behaviors derived from
contracts. 6 state machines re-expressed as language-agnostic FSMs
(SM1'–SM6'). 4 persistent schemas (credential store, session store,
settings store, models catalog). 20 black-box acceptance scenarios.
External-dependency stance taken for each non-trivial dependency
(replace / wrap / emulate / postpone). Implementation sequence with 3
scope tiers (MVP / major-workflow parity / full parity).

All 5 inbound carry-forwards from porting resolved with explicit
maintainer-policy positions:
- **port-CF1**: SM1'–SM6' as language-agnostic FSMs (in spec).
- **port-CF2**: every defect from either scan is designed-around or
  left-behind with citation in §Validation row 5.
- **port-CF3**: project-scope `<cwd>/.pi/` is **default-deny** with
  `--trust-project` flag (or interactive prompt). The 5-channel RCE
  surface (defect 4.1) is closed.
- **port-CF4**: bash/edit/write tools require **per-invocation user
  consent** (or `--yes-tools` flag). The "no consent gate" surface
  (defect 4.3) is closed.
- **port-CF5**: session JSONL persistence **flushes on every event**.
  The deferred-flush data-loss surface (defect 5.7 / dsm-RT6) is
  closed; latency cost quantified by reimpl-PostCF1 spike.

7 reimpl open questions (`reimpl-OQ1..7`) for spikes / runtime tests /
maintainer decisions per port. 7-item spike list. 3 post-pipeline
carry-forwards (`PostCF1..3`) for performance/distributed-lock spikes
and a future framework correction-mechanism delta.

**Pipeline complete.** All 7 phases of full-with-deep-audit at status
`complete`. status.yaml `current_phase: complete`.

## Files Touched

- **Added:**
  - `findings/reimplementation-spec/reimplementation-spec.md` (primary, terminal deliverable)
- **Modified:**
  - `workflow/status.yaml` (reimplementation-spec → complete; current_phase → complete; carry_forward updated to PostCF1-3 with target_phase ∈ {spike, delta})
  - `THREAD_LOG.md` (final index entry)
- **Deleted:** none

## Tests / Gates

| Gate | Result | Notes |
|---|---|---|
| Validation block PASS | PASS | All 6 criteria PASS; explicit defect-design-around table in row 5 maps every critical + every "port differently" hazard to a module invariant. |
| Strategic Alignment Hook honored | PASS | Language-agnostic spec; `templates/reimplementation-spec.md` used (not the opinionated variant). |
| All inbound carry-forwards closed | PASS | 5 of 5 from porting (port-CF1 through port-CF5). |
| Pipeline status | COMPLETE | `current_phase: complete` set in status.yaml. |

## Decisions Beyond Prompt

- **D015** | The spec **diverges from the source** on 5 specific
  invariants (listed in M3, M4, M5, M6, M8 owned-state /
  invariants tables) | Each divergence is a maintainer-policy decision
  the porting phase routed forward. Rather than punting them as "open
  to the porter," the spec takes a position with citation. The four
  positions: (a) listener fan-out is bounded (one throw is logged + skipped, NOT propagated); (b) session persistence flushes on every event (no deferred buffer); (c) edit/write/bash require per-call consent; (d) project-scope extensions default-deny with `--trust-project` opt-in. (e) all extension emit paths are uniformly guarded. A different port can choose differently, but the spec position is documented and citation-linked back to the defects.
- **D016** | Acceptance scenarios reference defects by ID with
  "(diverges from defect X.Y)" annotation | This makes it easy to
  cross-check that the spec actually addresses every defect. Reviewer
  can scan §Acceptance Scenarios, see "diverges from defect ...",
  trace back to the original mech/sem report.
- **D017** | "Deliberate Non-Goals" section explicitly lists what the
  port doesn't carry over | Per the rubric, this is essential; without
  it, a faithful port might over-engineer the long-tail OpenAI-compat
  matrix or replicate the Bun-binary mechanism unnecessarily.

## Proposed Conventions

### C13 Spec Annotates Divergences from Defects

**Why:** The spec changes behavior on ~5 invariants relative to the
source (defaults around consent, listener fan-out, deferred-flush, path
encoding, etc.). Without explicit annotation, a reviewer can't tell
"intentional design choice" from "didn't notice the source." Annotating
each invariant with `(diverging from defect X.Y)` makes the choice
auditable.

**How to apply:** When a spec invariant intentionally differs from
observed source behavior, cite the defect ID inline. The pipeline.yaml
completion-criteria row 5 ("Defects identified in either scan are
explicitly designed-around or noted as left behind, with the choice
cited") is the test for this.

### C14 Three-Tier Scope Bucket Maps to Defect Severity

**Why:** Pairing scope tiers (MVP / major-workflow parity / full parity)
with the defect-severity bucketing makes the implementation sequence
prioritization actionable. MVP focuses on the load-bearing behavior;
major-workflow includes the consent gates and atomicity fixes that
close the critical defects; full parity adds the delivery surfaces
that don't carry critical risk.

**How to apply:** The Implementation Sequence section's tier
descriptions should explicitly note which defects each tier closes,
not just which features each tier ships.

## Open Questions Left Behind

| ID | Kind | Description |
|---|---|---|
| reimpl-OQ1 | needs-spike | Concurrency primitive choice per target language |
| reimpl-OQ2 | needs-runtime-test | Latency cost of flush-on-every-event |
| reimpl-OQ3 | needs-maintainer-decision | Per-call consent UX (always / remember / flag-controlled) |
| reimpl-OQ4 | needs-runtime-test | Terminal capability detection accuracy |
| reimpl-OQ5 | needs-spike | Distributed-lock for multi-host OAuth refresh |
| reimpl-OQ6 | needs-maintainer-decision | Should port preserve streamProxy (remote-agent) feature? |
| reimpl-OQ7 | needs-spike | Browser chat component framework choice |

These are the **terminal phase's open questions** — items that the
pipeline cannot close because they require a prototype, runtime
evidence, or product/architecture decision per-port.

## Carry-Forward Routed

Post-pipeline (target_phase ∈ {spike, delta, amendment} per template
guidance):

| ID | Target | Description |
|---|---|---|
| reimpl-PostCF1 | spike | Performance prototype of flush-on-every-event vs deferred-flush |
| reimpl-PostCF2 | delta | If framework revision adds `correct-prior-phase` kind, apply sem-CR1/2 |
| reimpl-PostCF3 | spike | Distributed-lock prototype |

## Framework Feedback (optional)

The full-with-deep-audit pipeline (all 7 phases) was a heavy run on
~100K LOC. Some observations for a future framework revision:

1. **Subagent timeouts** — large-scope contract / pass extraction
   agents timed out twice; splitting into smaller per-topic agents
   recovered. The framework should consider an explicit guidance for
   when to split (perhaps "≥ 1 of: 50 source files OR ≥ 25 LOC ÷ 1024
   words extraction estimate").
2. **Cross-phase corrections** — sem-CR1 / sem-CR2 surfaced via
   carry-forward IDs but with `kind: defer-to-phase`. A dedicated
   `kind: correct-prior-phase` (with a `correcting: <phase-id>` field)
   would make the audit trail cleaner. Already noted in protocols /
   reimpl-spec phases' Framework Feedback sections.
3. **Severity-cap policy** — the user-locked policy worked well; the
   convention C3 ("severity cap policy gets documented in the output
   header") should land in the framework template guidance. Defect
   reports with no policy declaration are ambiguous between
   "cap-suppressed" and "absent" findings.
4. **Defect cross-reference convention** — convention C8 ("cite
   defect IDs in hazard rows") and C13 ("spec annotates divergences
   from defects") both proved load-bearing for round-trip
   auditability. Should land in framework template guidance.
5. **Phase carry-forward inheritance** — the workflow/status.yaml
   schema for carry_forward worked well, but **inheritance** isn't
   explicit (e.g., when arch-CF8 was eventually closed by sem 5.2,
   the original arch carry_forward entry didn't auto-clear). A
   convention or schema field for "closes: <id>" with bidirectional
   sync would help.

## Next Session Pointer

**There is no next session within this pipeline.** The pipeline is
complete.

Post-pipeline work options:
- **Spike**: prototype reimpl-PostCF1 (flush-on-event latency) or
  reimpl-PostCF3 (distributed lock) before committing to a target
  language.
- **Delta**: apply spec deltas as feedback / new evidence emerges
  (skill at `.codecarto/skills/spec-delta-application/SKILL.md` is
  available for this).
- **Implementation**: pick a target language (Go, Rust, Python,
  TypeScript-on-Bun, etc.) and start the MVP scope tier from the spec.

The `reimplementation-spec.md` is the canonical contract; this thread
hands off the spec to whoever picks up the implementation work.
