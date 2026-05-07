# Thread Log — Index

This file is an **index** of per-session closeouts. Each session writes a full closeout to
`closeouts/<YYYY-MM-DD>-<phase-or-module>.md` using `templates/closeout-template.md`, and
appends one line here pointing to it.

The body of each session lives in the closeout file, not in this index. This pattern scales
forever: per-session files are individually small and read-budget-cheap, and avoid the
heredoc-vs-edit sync risks that bite append-to-large-file workflows once the file grows past
~50 KB.

## Format

```
- YYYY-MM-DD — <phase-or-module> — <one-line-summary> — [closeout](closeouts/YYYY-MM-DD-phase-or-module.md)
```

## De-dup discipline

Before appending, scan the bottom 5 entries. If you see a line with the same date AND same
phase-or-module AND same summary, do not append — the prior session already wrote it. The
framework has no programmatic dedup gate; this is human-discipline. (See
`Apply 20 spec deltas to Thaumaturge.txt` for the incident that established this rule.)

A one-liner to surface duplicates from the shell:

```bash
grep -E '^- [0-9]{4}-[0-9]{2}-[0-9]{2}' .codecarto/THREAD_LOG.md | sort | uniq -d
```

## Project Mode

This project runs in **degraded no-orchestrator mode** (per GUIDE.md §First-Time Project Setup).
A single implementing session executes all 7 phases of the full-with-deep-audit pipeline
end-to-end. `CONVENTIONS.md` and `DECISIONS.md` are not maintained; per-phase closeouts capture
proposed conventions and decisions in their respective sections only.

## Entries

<!--
  Append one line per session below this marker.
  Example:
  - 2026-05-02 — framework-feedback-pass — applied 6 spec-blockers + 5 clarifications from FEEDBACK_INDEX.md — [closeout](closeouts/2026-05-02-framework-feedback-pass.md)
-->

- 2026-05-06 — architecture — mapped 5 packages (pi-ai, pi-agent-core, pi-tui, pi-web-ui, pi-coding-agent), 6 portability hazards, 9 carry-forward items routed — [closeout](closeouts/2026-05-06-architecture.md)
- 2026-05-06 — defect-scan-mechanical — 39 high/critical findings (incl. 2 critical data-loss); resolved arch-CF6 + arch-CF7; 11 items routed to semantic phase — [closeout](closeouts/2026-05-06-defect-scan-mechanical.md)
- 2026-05-06 — contracts — documented 12 surfaces, 18 acceptance scenarios; resolved arch-CF1/CF2 + dsm-RT5/RT7; surfaced 6 architecture-phase corrections — [closeout](closeouts/2026-05-06-contracts.md)
- 2026-05-06 — protocols — 11 boundaries, 6 event catalogs, 6 state machines (incl. deferred-flush + reload + OAuth refresh), 27 hazards; resolved arch-CF3/4/5 + con-CF1/2/3/4 + dsm-RT6 — [closeout](closeouts/2026-05-06-protocols.md)
- 2026-05-06 — defect-scan-semantic — 37 high/critical (3 critical incl. 5-channel project-RCE breach); all 11 inbound carry-forwards closed; 2 cross-phase corrections routed to porting — [closeout](closeouts/2026-05-06-defect-scan-semantic.md)
- 2026-05-06 — porting — 5-layer concept map; 20 features sorted by priority; 76 high/critical defects bucketed by recommendation (~58 fix / ~14 port differently / ~4 leave); 2 cross-phase corrections applied — [closeout](closeouts/2026-05-06-porting.md)
- 2026-05-06 — reimplementation-spec — language-agnostic spec; 10 modules (M1-M10), 14 required behaviors, 6 FSMs (SM1'-SM6'), 4 schemas, 20 acceptance scenarios; all 5 port-CF carry-forwards resolved with explicit policy positions; pipeline COMPLETE — [closeout](closeouts/2026-05-06-reimplementation-spec.md)
- 2026-05-06 — post-pipeline-oq-triage — 11 of 30 open questions resolved or partially resolved (6 from fresh source reads, 4 already-answered surfaced, plus 1 partial); 3 new mechanism-level findings (NF1 listener live-Set iteration, NF2 ExtensionEvent union drift, NF3 streamProxy CRLF handling) — [closeout](closeouts/2026-05-06-post-pipeline-oq-triage.md)

