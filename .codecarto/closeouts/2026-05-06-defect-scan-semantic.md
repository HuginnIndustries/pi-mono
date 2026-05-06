# Closeout — 2026-05-06 — defect-scan-semantic

## Summary

Ran Pass 3 (concurrency), Pass 4 (security/trust), Pass 5 (API contract
violations). **37 high/critical findings**: 3 critical (4.1 5-channel
project-RCE breach, 5.7 deferred-flush data loss, 5.8 non-atomic rewrite)
and 34 high. All 11 inbound carry-forwards resolved. **Important
correction**: dsm-RT1 was framed around `<repo>/.pi/models.json`
`!`-prefix, but actual project-RCE perimeter is the 5-channel bundle in
4.1 (extensions + packages + shellCommandPrefix + shellPath + npmCommand
from project `settings.json` + extensions/). Routed forward as `sem-CR1`
to porting for cross-phase correction. 8 of 18 contract acceptance
scenarios FAIL on current source.

7 new open questions surfaced (all needing runtime evidence or
maintainer policy).

Next session: porting phase. Synthesize.

## Files Touched

- **Added:**
  - `findings/defect-scan-semantic/semantic-defects.md` (primary)
  - `scratch/sem-pass{3,4,5}.md` (gitignored)
- **Modified:**
  - `workflow/status.yaml` (defect-scan-semantic → complete; porting carry_forward populated incl. 2 cross-phase corrections; current_phase → porting)
  - `THREAD_LOG.md` (index entry)

## Tests / Gates

| Gate | Result | Notes |
|---|---|---|
| Validation block PASS | PASS | All 7 criteria PASS; no PARTIAL or FAIL. |
| Inbound carry-forwards resolved | PASS | All 11 closed (carry-forward-closure table in primary). |
| Severity-cap policy | PASS | Per-finding rows for high/critical only; medium/low aggregated to tallies. |
| Pass 5 spec-reference column | PASS | Every Pass 5 row cites at least one of F-CLI-N / F-AI-N / F-AGENT-N / F-TUI-N / F-WEB-N / F-EXT-N / EC1-6 / SM1-6 / PS1-7. |

## Decisions Beyond Prompt

- **D010** | Reframed mechanical defects 1.6, 1.12+2.5, 1.13, 1.15, 1.17,
  1.19, 1.20, 1.21, 1.22, 2.1, 2.2 as Pass 5 contract violations rather
  than just citing them | The Pass 5 lens (spec-vs-impl drift) is
  materially different from the mechanical lens, even when the underlying
  code is the same. A mechanical "throw on width-overflow" finding is a
  Pass 1 logic defect. The same code, viewed against F-TUI-1 + SM3, is
  a contract drift between "TUI is a robust output layer" and the
  observed transition into "process exit." Filing both lenses (with the
  Pass 5 lens citing the mechanical pass for source-level fix detail)
  gives the porting phase both anchors.
- **D011** | Surfaced sem-CR1 and sem-CR2 as carry-forward "corrections"
  rather than amending prior primary outputs | Per GUIDE.md rule "phase
  outputs can't correct prior phases" — corrections live where they are
  surfaced (with carry-forward routing into porting where the synthesis
  can apply them). This preserves the audit trail.
- **D012** | Treated dsm-RT4 (`getAttributionHeaders` privacy review) as
  a low-severity tally entry rather than a per-finding row | The audit
  found only opt-in version + platform + arch reporting; no PII /
  install-id / fingerprint. Severity-cap policy correctly suppresses it.

## Proposed Conventions

### C9 Reframe Mechanical Defects in Pass 5 When the Spec Lens Adds Value

**Why:** A defect that is a logic error mechanically (e.g., width-overflow
throw kills process) is also a contract violation when viewed against
the protocols/contracts spec. Filing both lenses (with cross-references)
gives the porting phase a single per-defect entry with multiple anchor
points to fix.

**How to apply:** When Pass 5 surfaces a contract violation that is
already filed as a mechanical defect, file it again in Pass 5 with
"(Reframes mech X.Y as Pass 5 contract violation.)" in the defect cell
and cite the relevant Spec Reference. The mechanical row owns the
source-level fix detail; the Pass 5 row owns the contract anchor.

### C10 Cross-Phase Corrections Routed via Carry-Forward

**Why:** When Pass 4 found that the actual project-RCE perimeter is
different from what architecture/contracts/protocols had documented
(`models.json` vs the 5-channel bundle), amending the prior primary
outputs would violate the GUIDE.md "phase outputs can't correct prior
phases" rule. Routing as a porting-phase carry-forward (`sem-CR1`)
preserves audit trail and applies the correction during synthesis.

**How to apply:** Use `sem-CR<n>` (or `<phase>-CR<n>`) IDs for cross-phase
corrections. Cite the original phase's hazard/contract/protocol ID
explicitly. The downstream phase (porting / reimpl-spec) consumes them
during synthesis and reflects the corrected view in the bundle.

## Open Questions Left Behind

| ID | Kind | Description |
|---|---|---|
| sem-OQ1 | needs-runtime-test | Lock-stealing under proper-lockfile stale:30000 + slow OAuth |
| sem-OQ2 | needs-runtime-test | Microtask-interleaving timing for deferred-flush check-then-act |
| sem-OQ3 | needs-maintainer-decision | Built-in tools: per-call user confirmation? |
| sem-OQ4 | needs-maintainer-decision | `<cwd>/.pi/` default-trust posture; --trust-project flag? |
| sem-OQ5 | needs-spec-ruling | Listener-fan-out unguarded behavior — contract or defect? (continues con-OQ4) |
| sem-OQ6 | needs-runtime-test | mini-lit MarkdownBlock HTML-sanitization audit |
| sem-OQ7 | needs-runtime-test | Slow-OAuth-endpoint timing exploitability bounds |

## Carry-Forward Routed

| ID | Target Phase | Description |
|---|---|---|
| pro-CF1 | porting | Defect Synthesis consolidation (76 high/critical findings) |
| sem-CR1 | porting | Cross-phase correction: actual project-RCE perimeter is 5-channel bundle, not models.json |
| sem-CR2 | porting | Cross-phase correction: PS1 type discriminator is `session` not `session_info` |

## Framework Feedback (optional)

The "phase outputs can't correct prior phases" GUIDE.md rule worked
nicely here — surfacing sem-CR1 as a porting-phase carry-forward
preserved both the original phase's findings (so the audit trail is
clean) and routed the correction to the right place (synthesis).
Worth promoting in a future framework revision: an explicit `correct`
kind for carry-forward entries (`kind: correct-prior-phase`,
`target_phase: porting`, `correcting: <phase>` field).

## Next Session Pointer

Run **porting** phase. Read `findings/porting/SKILL.md` and
`templates/reverse-engineering-bundle.md`. Required reads (all 5 prior
primary outputs):
- `findings/architecture/architecture-map.md`
- `findings/contracts/behavioral-contracts.md`
- `findings/protocols/protocols-and-state.md`
- `findings/defect-scan-mechanical/mechanical-defects.md`
- `findings/defect-scan-semantic/semantic-defects.md`

Plus all secondary outputs and scratch files.

3 carry-forwards to resolve:
- **pro-CF1**: Consolidated Defect Synthesis table — combine 39
  mechanical + 37 semantic per-finding entries (76 total high/critical)
  into a single porting-oriented table with three recommendations per
  row (fix before porting / port differently / leave behind).
- **sem-CR1**: Apply the project-RCE perimeter correction (5-channel
  bundle, NOT models.json) into the synthesis. Update the
  Portability-Hazards section and Defect Synthesis to reflect the
  corrected view.
- **sem-CR2**: Apply the PS1 record-type catalog correction
  (`type: session` not `session_info`) in the bundle's Protocol notes.

After porting: Strategic Alignment Hook. The user already locked
**language-agnostic** for the reimpl-spec — confirm this is still the
choice (default template, no opinionated stack lock).
