---
tier: tale
title: Finish section-completion round trips and land capture lists
goal:
  Every offered section selector reaches its advertised heading, and verified epics
  bob-cli-fn.5 and bob-cli-fn close normally.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-fn.5.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-fn.5
create_time: 2026-11-02 10:33:46
status: wip
---

- **PARENT:**
  [202611/capture_list_section_names.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/capture_list_section_names.md)
- **BEAD:**
  [bob-cli-fn.5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fn/bob-cli-fn.5.md)

# Finish section-completion round trips and land capture lists

Repair the remaining selector defect in `bob-cli-fn.5`, verify it, and finish the
landing of that epic and its parent plan bead `bob-cli-fn` in this same coding turn.
This is one bounded Rust repair, not another phased epic. Do not wait for this turn's
commit SHA, push, or CI: the host commits only after the turn ends.

## Context and constraints

Read these through their audited commands:

- `sase bead read bob-cli-fn.5 -r "Need landing audit and remaining scope"`
- `sase bead read bob-cli-fn -r "Need inherited decisions and previous landing notes"`
- `sase artifact read plan:202611/capture_list_section_names.md "Need approved selector contract"`
- `sase artifact read plan:202611/capture_route_lists.md "Need parent closeout acceptance"`

The approved choices remain final: a column-zero list header starts an item;
project-note bullets share list ID grammar; the authorized Bulk Capture glossary update
is already done; no decisions record is written. Make no memory edits. Keep grammar
revision 6 and the Mac minimum revision 3. Do not touch the block-ID suggester, its
accepted calibration thresholds, Cargo versions, or CHANGELOG.md. Bob remains the
grammar authority. This repair needs no Swift changes.

The land audit reviewed both fn.5 children and every note, implementation commits
`b483c93b` (Rust) and `f4ed5bb` (Mac), and concurrent changes through Rust `a7931a64` /
Mac `f4ed5bb`. Both fetched origin/master refs matched at audit. Relation-successor
ordering/identity, meeting-start warnings, gcal ordering, shared text escaping, and
cancel-log indentation use compatible paths; no separate integration repair was
identified. Check any newer drift before closeout.

The Mac source already applies native ID splices and consumes Bob's section replacement.
The lander independently read CI run `41466582525` at snapshot
`f6cab4f270a19e186bea9450b8e48c5eda2287ab`: Build and Lint passed, all 26
`CaptureListPanelTests` passed (including all seven native Undo tests), and the 21
failures are exactly the existing `bob-cli-f3` / `bob-cli-fu` set. Bundle and install
gates were skipped. These failures are not remaining fn.5 work.

Follow-up triage is finished and recorded on fn.5: neither fn.5.1 nor fn.5.2 has a
`PROPOSED FOLLOW-UP:` note. Their notes describe addressed implementation/test work and
the already-tracked baseline CI failures; no new task was warranted. Parent fn notes
#2/#3/#8 record the older proposals' individual outcomes, including the deliberately
declined decisions-record and plugin-example proposals. Carry these outcomes into the
close notes. New unrelated findings still require `/sase_new_task`; epic-caused defects
stay epic work.

## 1. Reproduce the actual emitted-selector failure

`src/native/capture/sections.rs::section_selector_replacement` checks
`unquoted_selector_spelling(title)` only on the raw title. Its slug branch only checks
nonempty plus `resolved_bullet_heading_line(contents, slug, false)`. `selector_slug`
preserves punctuation, including a leading straight or curly quote. Consequently a
heading whose title includes the quotes, such as `## "Future Work"`, can emit
`"future-work"`. The resolver-only check treats those bytes as an unquoted selector;
`capture_language/tokens.rs::parse_route_token` instead decodes an exact name
`future-work`. It can silently target a different heading `## future-work`, or fall into
the zeroth section. Curly opening quotes have the same mismatch. This is
source-confirmed; first pin it with an executable regression before changing production
code.

The final quoted fallback is also returned without checking its target. Two identical
headings, case-insensitive duplicates, or an H1 shadowed by a non-H1 heading can thus
yield an offered row that cannot address its advertised heading. The approved contract
requires each offered replacement to reach the picked heading by identity, not merely to
be nonempty or look quoted.

Extend the existing section tests in `src/native/capture/tests/placement.rs`,
`src/native/capture_complete/tests/routes_tasks.rs`, and `tests/cli/capture/lists.rs` as
appropriate. Favor a small table-driven completion-apply-capture regression over testing
private implementation details:

- Straight and curly quote-leading titles, with and without spaces, and a decoy heading
  named after the decoded slug.
- A quote/backslash-bearing title exercising escaped exact fallback.
- Existing raw single-word, spaced slug, and slug-collision cases.
- Duplicate exact titles, case variants, and H1/non-H1 preference.

Request `capture-complete`, apply its actual byte replacement range, then parse and
execute a list with a unique note under that header in a temporary vault. Assert the
note lies between the intended heading and the next heading, and that the preamble and
decoy are untouched. Include an ordinary section capture to pin the shared resolver. Do
not touch the live vault.

## 2. Make every offered replacement lex and resolve correctly

Keep the preference order raw title, hyphen slug, escaped quoted literal. Require
unquoted spelling safety for the slug too; do not return a leading-quote slug under an
unquoted resolver check. Check the exact fallback's decoded title against the intended
heading line as well. Reuse existing capture grammar and exact-name escaping; do not
invent another grammar or space spelling.

If none of those spellings reaches a heading (for example the second of two identical
titles), omit that unreachable candidate. Retain the heading that the existing resolver
actually reaches; keep its document-order/H1 rules unchanged. Do not add ordinal
selectors or change execution's established fallback/tier semantics. Make the helper
return an optional result if that is the clearest way to keep candidate construction
honest.

Ensure a partial quoted query still covers the whole literal and the user can accept the
safe replacement on a header with existing bullets. Confirm the shell completion adapter
preserves a quoted replacement under its existing quoting rules; add a focused
regression if current coverage does not establish that. The shell protocol carries raw
values and its adapters handle shell escaping, so do not blindly double-quote values in
`capture_complete/shell.rs`.

## 3. Verify the completed behavior and current integration

Run the focused regression(s) on the pre-fix tree to establish the failure, then verify
the fix. Run `just check` (use `sase tool run check` and `/sase_monitor` if it
escalates) and `just docs-check`. Never run `just check-full`. A check-full failure
beside a passing check would be test infrastructure, not epic work.

Recheck the original Future Work screenshot scenario, partial/incomplete section
literals, header route/section completion with bullets, and the existing
`capture_off_clock_list_matches_equivalent_batch` regression. Preserve prior grammar,
ID, and undo behavior. No Swift code is planned; retain the independently verified Mac
CI evidence above. If Swift changes become necessary, follow the linked repo's
compile/CI instructions rather than landing uncompiled Swift.

Fetch/read current base history through the normal workspace/repo workflow and review
drift since the audit SHAs. Resolve any conflict caused by this epic before closing.
Record verification commands/results and the follow-up outcomes on fn.5.

## 4. Finish this landing and recheck the directly parented plan

This is the final implementation step, performed before this turn ends. There is no
later land agent for this tale, and none of these steps depends on its own future
commit, SHA, push, or CI.

1. Read fn.5 and every descendant with `sase bead read`, including all notes and
   linked-plan readiness. Verify the approved fn.5 scope is now complete. Run
   `sase bead epic-symbols bob-cli-fn.5`. Resolve every listed symbol, or re-key an
   exemption only to a still-open later bead that actually needs it. The audit found no
   entries, but recheck the final tree.
2. Close normally with
   `sase bead close bob-cli-fn.5 --note "<specific scope, source/commit, integration, regression, check and Mac-CI evidence; all follow-up outcomes>"`.
   If leftover symbols reject closure, resolve them and retry. If named phases are
   incomplete, finish/reopen them; never force successful nested landing. `--force` is
   only for a deliberately canceled/superseded outcome with a real reason, never merely
   to bypass the readiness check.
3. Run `just symvision` if the recipe exists; otherwise record its absence (it was
   absent at audit). Open the plans repository through `/sase_repo`, resolve
   `plan:202611/capture_list_section_names.md`, and set its frontmatter `status: done`.
   Use the printed repository/path, not a hard-coded workspace.
4. Run `sase bead read bob-cli-fn.5 -r "Need the parent link"`. Its parent is the plan
   bead **bob-cli-fn**, not a phase. Resume that landing now: reread fn's previous
   landing notes and every descendant and note, including fn.1, fn.2, fn.3, fn.4, fn.5,
   fn.5.1 and fn.5.2 and anything added since this plan. Read
   `plan:202611/capture_route_lists.md` through `sase artifact read`, recheck its
   accepted decisions and completion criteria, rerun descendant and linked-plan
   readiness checks, and review drift since fn's LAND AUDIT 2 plus this audit. Its
   previous blockers were exactly fn.5's scope; do not assume closure merely from the
   child's closed status.
5. If fn is fully complete, run `sase bead epic-symbols bob-cli-fn`, resolve or
   legitimately re-key all entries, and close normally:
   `sase bead close bob-cli-fn --note "<what was rechecked, all descendant notes/follow-up outcomes, final integration and verification>"`.
   Run `just symvision` if available, then set `status: done` in the resolved
   `plan:202611/capture_route_lists.md`. Never force. If fn is incomplete or ambiguous,
   leave it open and append a concrete blocker note instead.
6. Fn had no parent at audit. Recheck; if it now has one, follow the user's ancestor
   rule: close only a directly parented completed phase and stop for its existing
   lander, or fully re-audit and normally close directly parented plan ancestors,
   retiring their symbols and updating each linked plan. Stop at the first
   incomplete/ambiguous ancestor and record/report its blocker.
7. Submit `/sase_final` for every repository changed, with the correct completed bead
   state, and report the actual closed beads and verification limitations.
