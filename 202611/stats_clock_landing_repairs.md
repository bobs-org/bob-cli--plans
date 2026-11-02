---
tier: tale
title: Repair Stats clock diagnostics and finish the landing
goal: Correct drift and overnight overlap diagnostics, then close bob-cli-fl.3.1 and
  parent phase bob-cli-fl.3.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-fl.3.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-fl.3.1
status: done
---

- **PARENT:**
  [202611/stats_clock.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/stats_clock.md)
- **BEAD:**
  [bob-cli-fl.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fl/bob-cli-fl.3.1.md)

# Repair Stats clock diagnostics and finish the landing

## Outcome

Finish the remaining work in `bob-cli-fl.3.1` (Complete Stats clock selection and
reporting), then close that epic and its parent **phase** `bob-cli-fl.3` in this same
coding turn. Its approved plan is `plan:202611/stats_clock.md`; the outer contract is
`plan:202611/off_clock_pomodoros.md`. This is one bounded Rust diagnostics repair,
regression coverage, documentation correction, and closeout. It does not require another
epic or a separate land agent.

The original DECISIONS remain final. Keep recorded clock keys authoritative, Worked as
the default, formulas isolated from the request lens, schema version 1, and deprecated
aliases. No memory changes are needed: the authorized Stats sentence already exists in
`glossary:off-clock-pomodoro`.

## Landing evidence

The lander read every note on `bob-cli-fl.3.1` and children `.1`, `.2`, `.3`, both
linked plans, and parent phase `bob-cli-fl.3`. The three child phases are closed. Their
implementation commits on master are `4d950c58`, `4d2133b5`, and `daf785b6`; `a6c10106`
is an alternate history copy of the second phase, not additional unintegrated work. At
audit time HEAD was `daf785b6`.

Source review confirmed request/pinned clocks, canonical formula contexts, adoption
gating, exact daily/grouped/part sub-shares, clock totals independent of metric filters,
session filtering, CLI aliases/completion, human/CSV/JSON contracts, and the authorized
glossary sentence. Remaining defects are in `src/native/stats/homes.rs`:

1. `session_drifted` returns false for an empty lineup instead of consulting
   `event_task::clock_verdict(&[])`, whose answer is Worked/Unlinked. A keyed off-clock
   row with no counted links therefore never reports disagreement.
2. `index_task_lines` scans every raw Markdown line with `is_checkbox_line` and
   `trailing_block_id_in_line`, without excluding frontmatter or fences. It can resolve
   a fenced example as a real task, or let a fenced duplicate suppress a real task. This
   also diverges from the writer's parsed-task lookup in
   `capture_pomodoro_start::resolve_clock_fact`.
3. `detect_overlaps` normalizes overnight end minutes but partitions comparisons by note
   date. A 23:30–00:30 session on Oct 9 and a 00:15–00:45 session on Oct 10 produce no
   overlap, although both belong to the requested Oct 9–10 window and share 15 minutes.
   It also allocates every pair into `details`, sorts them all, and only then truncates
   to 50. The serialized list is bounded but intermediate time and memory are not.

These findings are source-confirmed; reproduce them through the current engine before
fixing. The checkout-local `target/debug/bob` was an older pre-clock binary and rejected
`-k`, so it was not used to claim a current-HEAD runtime reproduction. Use Cargo's
resolved target directory/current test binary for verification.

Integration review covered `71a65dad` (counted-link clock writer) and `0ebd0714`
(capture lists), which landed after the first Stats commit. It also checked the
creation-window commits `f171e3cc` (event settling), `62a82104` (ID suggestions), and
`2d5ebfc9` (event buffer closeout). Writers and settling already emit keys and counted
links Stats consumes. The writer's empty-lineup and parsed-task semantics make the drift
repair an integration requirement. No additional CLI or capture product change is
needed. This is a direct master checkout, not a PR.

## Follow-up triage already completed

Do not file these again. `/sase_new_task` searches across all statuses, a last-week task
sweep, and review of active epic scopes produced these outcomes, recorded on
`bob-cli-fl.3.1`:

- `bob-cli-fl.3.1.3` note #1: `editor_parse.rs:4286` calls `detect_list_header(&line)`
  with an already borrowed line. This is caused by capture-list commit `0ebd0714`, and
  was recorded as a DISCOVERED ISSUE on active epic `bob-cli-fn`, with `.1` identified
  as its owner. No separate task.
- The same proposing bead, note #2: `complete_drawers.rs:118,431` and
  `complete_relationship_owner.rs:317` still expect grammar 5 after that same commit
  raised `GRAMMAR_REVISION` to 6. Recorded on `bob-cli-fn` with the first issue; no
  separate task. The proposing worker reported clean-base reproductions of all three
  failing assertions.
- The same proposing bead, note #3: human `journal_gap` warnings print twice.
  `warning_lines` is unchanged since `9be32d97` and appends metric warnings already
  present in the report warnings. Independently reproduced on the pre-clock binary and
  confirmed in current source and the exact-output test. Filed the precisely scoped
  **small bug `bob-cli-fr`**, marked ready. Preserve default output in this repair; that
  separate task owns deduplication.
- No proposals were declined. Parent phase `bob-cli-fl.3` already has a note about its
  missing research artifact, for the outer epic lander to triage. This repair uses the
  approved plans and does not depend on that unavailable artifact.

The lander's `just check` passed formatting, then exited 101 during Clippy before tests
ran, with one denied compilation error in each lib target; phase 3 had already
identified the unrelated capture-list borrow above. Do not describe this as a green test
run. No `just check-full` was run or is authorized.

## Implementation

1. **Make drift use the same task and clock facts as writers.** Remove the empty lineup
   shortcut and apply the shared verdict to every fully resolved keyed lineup, including
   the empty one. An empty keyed off-clock row contributes one drift; an empty Worked
   row does not. Bare rows keep their unstamped diagnostic, conflicts keep their
   conflict diagnostic, and unresolved/ambiguous links do not invent drift. Replace the
   raw-line task map with lookup facts produced by the existing Markdown/task parsing
   rules over the already indexed buffers. Reuse parsed tasks/settings and in-memory
   lookups where possible; do not add a second vault scan or one file read per link.
   Match writer rules for ordinary and embedded links, same-note events, archives,
   duplicate block IDs, frontmatter, and fenced examples. Do not broaden task metric
   eligibility as a side effect. Continue using `line_has_off_clock_tag` on the actual
   task line.

2. **Sweep absolute intervals and bound detail collection.** Keep the current set of
   eligible closed timed sessions whose ledger dates fall in the displayed window,
   independent of lens, metric selection, and duration overrides. Convert each interval
   to a date-aware minute coordinate before comparing it, so the next day's early
   session can overlap an overnight predecessor. Compute the exact union covered at
   least twice with a sweep; touching endpoints contribute zero and triple overlaps
   count once. Never subtract this diagnostic from clock totals or change date
   attribution of metric minutes. Collect at most 50 deterministic pair details during
   traversal, rather than allocating all pairs first. Avoid quadratic scans over
   nonoverlapping sessions, including many rows on one day; keep aggregate computation
   exact after the detail cap is reached. Preserve existing same-note detail fields. For
   a cross-note pair, add an optional `paths` array aligned with `lines`, ordered by
   source date/line, so each row remains locatable; use the earlier source note date for
   `date` and document that convention. Omit `paths` for same-note pairs. This is
   additive schema-1 metadata, not a version bump.

3. **Prove the repaired edges and existing contracts.** Add focused engine/CLI
   regressions for empty lineups (including only-deferred links), fenced and frontmatter
   pseudo-tasks, a fenced duplicate beside a valid task, ordinary vs embedded
   eligibility, same-note and archived tasks, both drift directions, aliases, and
   unresolved/ambiguous/duplicate targets. Exercise an overnight cross-note pair in a
   window containing both notes (15 minutes for the example above), same-note overnight
   spans, adjacent endpoints, triple overlaps, exclusions outside the ledger-date
   window, and more than 50 overlapping pairs. Assert clock totals and recorded minutes
   are unchanged, all three lenses have identical diagnostics, details identify both
   source rows, and vault/config bytes are unchanged. Include a writer-to-Stats test
   using the existing counted link writer or settling fixture, so the integration is
   exercised rather than only hand-authored rows. Keep all existing formula, adoption,
   split-sum, session-filter, CSV, completion, and clean default-output tests passing.

4. **Document and verify.** Read `docs.md` through `/sase_memory_read` before editing
   documentation. Update `docs/stats.md` and, if it describes the affected detail shape,
   `docs/reference/json-outputs.md` for empty-lineup drift, genuine task resolution,
   cross-note overlap details, and the exact/capped distinction. Keep the authorized
   glossary sentence unchanged. Run `cargo fmt --all`, focused Stats lib/CLI tests,
   `just check`, and `just docs-check`. Do not run `just check-full`. If the known
   unrelated capture-list failures remain, verify their provenance on a clean base as
   the approved epic permits, record the exact failed checks, and keep all new/Stats
   tests green. Do not defer any failure caused by this repair or the Stats epic. Use
   `/sase_monitor` for long commands with continuation instructions that preserve this
   closeout duty.

## Final step: finish both required closes

The coder must finish the landing; no lander resumes after this tale. Closing is
performed in the same turn as the final code and must not depend on this turn's future
commit SHA, push, or CI result. The host commits after the turn ends.

1. Re-read `bob-cli-fl.3.1` and every current descendant with audited `sase bead read`
   commands, reviewing every note and linked-plan readiness. Read
   `plan:202611/stats_clock.md` and `plan:202611/off_clock_pomodoros.md` with
   `sase artifact read`; compare new commits since the audit HEAD `daf785b6` and resolve
   any interaction with these repairs. Confirm all original Stats scope and final
   DECISIONS still hold. If this tale has its own descendant bead, complete and close
   that bead normally after its checks before closing its ancestor; no commit is
   required first.
2. Run `sase bead epic-symbols bob-cli-fl.3.1`. Resolve every listed exemption by wiring
   it up, privatizing it, giving a policy-valid non-test pragma, or deleting it. Re-key
   a Justfile entry only when a still-open later bead really needs it. There were no
   entries at audit time; check again anyway.
3. Close the epic normally with
   `sase bead close bob-cli-fl.3.1 --note "<verified three-phase scope, diagnostic repairs, integration review, exact checks, and all follow-up outcomes above>"`.
   If rejected for leftover symbols or incomplete descendants, finish them and retry
   normally. Never force to make the command succeed or to advance this successful
   nested landing. A truly canceled/superseded branch requires an explicit reason and
   matching resolution, not a fabricated successful close.
4. Run `just symvision` if available (there was no recipe at audit time). Use
   `/sase_repo` to open the plans repository before editing it, and set `status: done`
   in the frontmatter of **`202611/stats_clock.md`**, the path returned for
   `plan:202611/stats_clock.md`. Do not mark the outer plan done.
5. Run `sase bead read bob-cli-fl.3.1 -r "Need the parent link"` and re-read
   **`bob-cli-fl.3`**. It is a phase of outer epic `bob-cli-fl`, not a plan bead. Verify
   that this child plan fulfills the parent's entire `stats-clock` section and that
   every descendant is closed. Run `sase bead epic-symbols bob-cli-fl.3` and retire or
   legitimately re-key any remaining exemptions. Then close only that parent phase with
   `sase bead close bob-cli-fl.3 --note "<complete Stats clock contract, nested diagnostic repairs, integration and check results reverified>"`.
   Leave **`bob-cli-fl`** and its outer plan to its waiting land agent. If the parent
   has become incomplete or ambiguous, record a concrete blocker note on it and report
   that instead of forcing closure.
6. Finish through `/sase_final`, declaring every changed repository (including the plans
   sidecar), with the correct completed/keep bead action. The close note must include
   the `bob-cli-fn` discovery routing and ready task `bob-cli-fr`, and must distinguish
   passing Stats tests from unrelated red repository gates.
