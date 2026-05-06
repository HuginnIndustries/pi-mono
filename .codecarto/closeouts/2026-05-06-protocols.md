# Closeout — 2026-05-06 — protocols

## Summary

Documented 11 protocol boundaries, 6 event-catalog entries, 6 state machines
(stream FSM, deferred-flush FSM, TUI render FSM, streamProxy connection FSM,
OAuth refresh FSM, extension reload FSM), 7 persistent-schema sub-sections,
and a 27-row compatibility-hazards table. Resolved 7 inbound carry-forwards
(arch-CF3, arch-CF4, arch-CF5, con-CF1, con-CF2, con-CF3, con-CF4) plus
dsm-RT6. Surfaced 5 new open questions (all needing runtime evidence) and 2
carry-forwards routed to porting / reimplementation-spec.

Critical finding (resolves dsm-RT6): session JSONL `_persist` buffers in
memory until the **first assistant message**, then flushes — a crash
before the first assistant arrives loses the user prompt entirely.
Documented as state machine SM2 + hazard H15. Worth flagging for the
porting/reimpl phase as a contract decision: either preserve the
optimization with a documented expectation, or fix it.

Important corrections to prior phases:
- 22 ExtensionEvent union members / 29 subscribable names (architecture
  said ~25; contracts said 22 union but the 29 from on-overload
  expansion is more accurate user-facing count).
- Session schema lives in `core/session-manager.ts` (no separate
  `session-jsonl.ts` file as task brief assumed).
- streamProxy uses Bearer auth + custom `data: <json>\n` framing (not
  canonical SSE).
- Kitty keyboard push escape is `\x1b[>7u` (not `>1u`).

Next session: defect-scan-semantic (passes 3, 4, 5).

## Files Touched

- **Added:**
  - `findings/protocols/protocols-and-state.md` (primary)
  - `scratch/proto-{events,sessions,tui,proxy,extension-events}.md` (gitignored)
- **Modified:**
  - `findings/public-surfaces/public-surfaces.md` (append: wire-format catalogs)
  - `findings/runtime-lifecycle/runtime-lifecycle.md` (append: deferred-flush + reload + streamProxy + TUI render lifecycles)
  - `findings/state-and-storage/state-and-storage.md` (append: 10-type record catalog, identifiers, branching modes, encoded-cwd, lock semantics family)
  - `findings/config-model/config-model.md` (append: config-crossing-boundaries, hot-reload boundary, migrations as crossings)
  - `workflow/status.yaml` (protocols → complete; current_phase → defect-scan-semantic)
  - `THREAD_LOG.md` (index entry)
- **Deleted:** none

## Tests / Gates

| Gate | Result | Notes |
|---|---|---|
| Validation block PASS | PASS | All 5 criteria PASS; no PARTIAL or FAIL. |
| Inbound carry-forwards resolved | PASS | 7 + dsm-RT6 RESOLVED; 5 new open questions are runtime/maintainer-only. |
| Secondary appends written | PASS | All 4 declared appends written with dated 2026-05-06 protocols-phase sections. |
| Critical hazard route check | PASS | dsm-RT6 deferred-flush, defect 2.1 non-atomic JSONL, defect 1.22 streamProxy out-of-order all surface as protocol-level hazards H14/H15/H12. |

## Decisions Beyond Prompt

- **D007** | Modeled the deferred-flush as state machine SM2 with explicit
  "in-memory buffer" pre-flush state | Originally considered listing it
  only as a hazard row, but the buffer-vs-flushed distinction is
  load-bearing for the reimpl-spec (which must decide whether to preserve
  the optimization). Promoting it to a state machine makes the data-loss
  surface impossible to miss in downstream phases.
- **D008** | Listed 27 compatibility hazards (vs the architecture phase's
  6) | The protocols phase rubric is broader than architecture's
  portability-priorities table — it encompasses every wire-format,
  filesystem, and async-ordering invariant. The 27 rows aggregate
  protocol-level hazards from all prior phases plus new ones surfaced
  during this phase's targeted reads (e.g., `data: <json>\n` framing,
  `\x1b[>7u` Kitty push specifics, encoded-cwd lossy collisions).
- **D009** | The 22 vs 29 event-name divergence in extensions reflects
  union-vs-subscribable distinction | `ExtensionEvent` union has 22
  members, but the typed `on()` overloads expose 29 because
  `SessionEvent` is itself an 8-member union flattened in the overloads.
  Both numbers are correct in their respective contexts; primary uses
  29 (user-facing) and 22 (internal union).

## Proposed Conventions

### C7 State Machines as First-Class Protocol Artifacts

**Why:** The session JSONL deferred-flush behavior could easily have
been documented only as a hazard row (#15) and lost in the synthesis.
Promoting it to state machine SM2 with explicit transitions makes the
data-loss surface unmissable for porting and reimpl-spec authors.

**How to apply:** When a protocol-phase finding has materially different
behavior in different states (in-memory buffer vs flushed; locked vs
unlocked; valid vs needs-refresh), promote it from a hazard row to a
state machine entry. Hazards are presence/absence; state machines are
conditional behavior.

### C8 Cite Defect IDs in Hazard Rows

**Why:** The hazards table cites mechanical defect IDs (e.g., 1.13, 2.1,
P6.4) and dsm-RT* / arch-CF* IDs in the Notes column. This makes the
defects ↔ protocols round-trip auditable: a porting-phase reader looking
at hazard H15 can immediately find the underlying defect and open
question.

**How to apply:** Every hazards table row should cite at least one
defect ID, carry-forward ID, or open-question ID in its Notes. Hazards
that are pure protocol-level should still link to their source citation
(file:line).

## Open Questions Left Behind

| ID | Kind | Description |
|---|---|---|
| pro-OQ1 | needs-fixture-capture | streamProxy SSE heartbeat/keepalive (continues arch-OQ3) |
| pro-OQ2 | needs-runtime-test | Concurrent-reader behavior on session JSONL mid-rewrite |
| pro-OQ3 | needs-spec-ruling | Deferred-flush durability — intentional optimization or contract gap? |
| pro-OQ4 | needs-runtime-test | `\x1b[?2026` synchronized output fallback when terminal lacks support |
| pro-OQ5 | needs-runtime-test | EPIPE on stdout.write mid-paint behavior |

## Carry-Forward Routed

| ID | Target Phase | Description |
|---|---|---|
| pro-CF1 | porting | Integrate H1-H27 hazards table with mechanical + semantic defect catalogs into a single Defect Synthesis table |
| pro-CF2 | reimplementation-spec | Re-express SM1-SM6 state machines as language-agnostic FSMs in the reimpl spec |

## Framework Feedback (optional)

The protocols-phase template's "Persistent Schema Notes" section is
loosely defined — the rubric lists topics (append-only vs mutable,
branching, compaction) but not a structure. I split it into 7 numbered
sub-sections for navigability, but a more opinionated template with
explicit sub-headings (Identifiers, Append vs Rewrite, Branching, Lock
Semantics, etc.) would standardize across projects. Worth a thought
upstream.

## Next Session Pointer

Run **defect-scan-semantic** (passes 3, 4, 5). Read
`findings/defect-scan-semantic/SKILL.md`,
`findings/defect-scan/passes/{03-concurrency-and-resources,04-security-and-trust,05-api-contract-violations}.md`.
Required reads: architecture-map, behavioral-contracts,
protocols-and-state, mechanical-defects.

11 inbound carry-forwards (5 to Pass 4 security, 3 to Pass 3 concurrency,
2+ to Pass 5 API contract violations + arch-CF8 OpenAI-compat URL matrix
+ con-CF6 streamSimple sync-throw drift). Apply the same high/critical
severity cap as mechanical (medium/low get tally rows only).

Pass 5 findings MUST cite the contract or protocol they violate (using
the section anchors in `behavioral-contracts.md` and
`protocols-and-state.md`).
