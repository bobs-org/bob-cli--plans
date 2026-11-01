---
tier: tale
size: medium
title: Finish sealed project-offer refusal and incomplete editor contracts
goal:
  Route-less sealed offers refuse capture without writing, every recognized sealed offer
  keeps its project mode while requesting missing fields, and bob-cli-ex.2.1 and its
  parent phase complete their normal closeout.
proposed_by: bbugyi200.apollo.bob-cli-ex.2.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-ex.2.1
create_time: 2026-11-01 09:10:07
status: wip
---

- **BEAD:**
  [bob-cli-ex.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ex/bob-cli-ex.2.1.md)

# Remaining work for bob-cli-ex.2.1

## Context and limits

This is the remaining-work tale from the relaunch audit of child epic `bob-cli-ex.2.1`.
Its parent is phase `bob-cli-ex.2` of epic `bob-cli-ex`. The child was closed, but this
audit reopened it and its ancestors after reproducing an unmet contract. Its linked plan
is `plan:202610/project_note_offer_form.md`; the authoritative parent contract is
`plan:202610/project_note_seals.md`. Read those artifacts through `sase artifact read`,
and read the assigned epic and parent phase with `sase bead read`. The child plan now
has `status: in_progress`.

The offer engine and the earlier landing repairs already landed in `9a30ec4c`,
`e9b9cd1d`, `e8dd3b65`, and `42dd92bc`. Parent landing commit `86a63e84` also fixes
duplicate offers after selected rows receive IDs. Preserve that work. The audit read
every child note and reviewed the actual renderer, planner, shared colon placement,
selection/net-pass plumbing, grammar, rewrites, output, tests, and docs. Do not redo
those phases or the Mac implementation.

Inherited decisions remain `initial_marks=none` and `memory_glossary=yes` for the two
already-updated successor glossary strands. This repair requires no memory edits, no new
CLI options, no package version change, and no CHANGELOG change.

## Reproduced remaining defects

At bob-cli `81b3b5a4` (equal to fetched `origin/master`), a freshly built binary in an
isolated vault with an existing `mac_inbox.md` reproduces:

1. `bob capture --no-clip -f json -- 'Finish packet @:pkt@'` exits 0 with `kind: task`
   and writes the unfinished marker into the inbox. With a child `- Draft memo`, that
   child becomes an inbox task child too. Dry run also reports success. Named
   `@:pkt@#admin` has the same execution failure. The caret form `@^pkt@` correctly
   returns the route error.
2. `capture-parse` recognizes `@:pkt@` as `pomodoro_project_note` with `needs: [route]`,
   but `@:pkt@#admin` and `@:pkt@#"Lit Name"` return `mode: incomplete` with the same
   route need. `@cash:@` and `@cash:pkt@#` also lose the project mode when requesting an
   ID or name. The approved child plan explicitly requires `pomodoro_project_note` for
   every Offer form and project modifiers on marker/placeholder spans; a Pomodoro-name
   span remains independent.

Execution root cause: `capture_language/tokens.rs::is_pomodoro_marker_candidate`
requires an alphabetic route before the colon. The terminal dispatch therefore does not
call `parse_pomodoro_route_token` for a route-less sealed offer and falls through to
prose. `capture_language/tests/grammar.rs::execution_parses_project_note_offer_markers`
explicitly asserts that incorrect fallback. The unnamed editor branch compensates for
the generic incomplete mode only when its ID is nonempty and route is empty; the other
sealed-offer branches do not maintain the approved mode contract.

## Implement the bounded repair

1. Recognize a route-less sealed colon project marker in the execution destination path
   and return the existing Pomodoro route-required error before any writes. Scope the
   guard to the sealed project family; preserve existing unsealed `@:` / `@:id`
   behavior, literal mid-prose handling, forced-route semantics, and teaching
   diagnostics for retired `@:id+` forms. Exercise both leading and trailing destination
   positions, named and quoted names, and authored children. Inspect
   `parse_terminal_route_token`, `validate_special_terminal_markers_line`, and
   `retired_route_less_project_note_error` so the two destination paths agree. Do not
   change the offer writer or make incomplete drafts into tasks.
2. In `capture_language/editor_classify.rs`, consistently keep `pomodoro_project_note`
   on recognized sealed Offer markers while retaining the precise route/ID/name needs
   that prevent submission. Audit unnamed, named, quoted-name, missing-route,
   missing-ID, and unfinished-name branches together. Keep byte-correct, nonoverlapping
   spans and `project_note: offer` on the marker and its route/ID placeholders; name
   spans/placeholders keep their independent name behavior. Preserve caret Create
   behavior and normal unsealed colon forms.
3. Replace the grammar assertion that route-less sealed offers become literal task text.
   Add focused unit/editor tests and public CLI regressions in the existing grammar,
   editor modes/spans, and CLI parse/project-note suites. At minimum cover `@:pkt@`,
   `@:pkt@#admin`, `@:pkt@#"Lit Name"`, leading/trailing placement, missing ID/name
   editing states, and non-ASCII text before markers. Check editor needs and modifiers,
   not only mode. For execution, use isolated vault snapshots to prove both dry-run and
   real submission refuse without changing any file; include a mixed draft with a valid
   earlier item to prove atomic refusal. Retain successful routed offers and
   retired-marker regressions.
4. Read `docs.md` through the memory-read skill before documentation edits. Keep the
   existing capture grammar/reference description accurate about incomplete sealed
   project markers; change only copy made incorrect or missing by this repair. Do not
   regenerate unrelated content. If CLI help changes, use the documented generator
   rather than editing generated help pages.

This is a single-agent medium tale: two shared grammar paths plus focused protocol
regressions and closeout. No new architecture or phase delegation is needed.

## Verification and drift

Run focused regression tests for the changed grammar and public capture paths, then
`just check` as the canonical file-change gate. Do not run `just check-full`. Run
`just docs-check` if documentation/help changed. Follow `/sase_monitor` for commands
that need a handoff; a named `sase tool run check` starts inline and its escalation, if
any, must be joined instead of rerun. Baseline ToolRun
`d6f4f2dfc0694788f1f60175641541e2` (`just check`) passed with exit 0 in 637522 ms,
including 4265 library tests and 2612 CLI tests. That green baseline misses the
reproduced contracts above; rerun the canonical check after the repair changes the tree.

The audit through `81b3b5a4` reviewed all commits since first child commit `9a30ec4c`.
Later directive migrations (`%c`, `%n`, `%l`, `%t`), drawer handling, and drawer JSON
vocabulary are integrated with the valid sealed forms; Keep changes need no offer
integration. Recheck any newer drift before closing.

## Follow-up disposition already completed by the lander

The sole `PROPOSED FOLLOW-UP` is `bob-cli-ex.2.1.3` note 1: special-case trailing
`#hide` on project task bullets. The lander used `/sase_new_task`, searched bug and all
task types, swept all tasks from the last week, and reviewed the 12 active epic scopes
including `bob-cli-ey` and `bob-cli-ey.7`. There is no duplicate or causal active epic;
`bob-cli-bu` is an unrelated relation glossary task.

Declined: a free edge hash marker is deliberately a task type. Fresh current-binary
smokes return `type_marker_not_applicable` for trailing `#hide`, while escaped `\#hide`
and mid-text `#hide` parse and capture correctly. Published docs teach those literal
forms. No task, corroboration, or memory change is warranted. This outcome and the audit
evidence are recorded on `bob-cli-ex.2.1`. Carry the outcome into the close note; do not
reopen that grammar choice.

## Final step: complete the landing in this same coding turn

The coder owns this entire closeout; there is no later lander for a tale. Do not wait
for your own commit SHA, push, or CI result. Host finalization commits the turn only
after you finish, so close against the verified working tree.

1. Re-read `bob-cli-ex.2.1` and its three closed phases `bob-cli-ex.2.1.1`, `.2`, and
   `.3` for intervening notes. Confirm the original offer acceptance and both remaining
   contracts above are satisfied. Record the current tests, drift review, and the
   declined trailing-hide proposal in the close note.
2. Run `sase bead epic-symbols bob-cli-ex.2.1`. Resolve every listed entry (wire,
   privatize, add a justified non-test pragma, or delete), or re-key only an exemption
   still needed by a genuinely open later bead. The audit found none. Close normally
   with
   `sase bead close bob-cli-ex.2.1 --note "<verification, integration, test evidence, and follow-up disposition>"`.
   Never use `--force` to advance a successful landing. Resolve any reported unfinished
   descendants or symbols before retrying.
3. Run `just symvision` if that recipe exists (absent at the audit baseline). Use
   `/sase_repo` to open `plans` before editing the sidecar. Set `status: done` in
   `202610/project_note_offer_form.md`, the plan linked by
   `plan:202610/project_note_offer_form.md`. Use the opened repository path; do not
   assume a numbered workspace path.
4. Read `bob-cli-ex.2` and verify this completed child satisfies its `offer_form` scope
   and inherited decisions. Audit `sase bead epic-symbols bob-cli-ex.2`, retire/re-key
   any entries, and close only that parent phase normally with
   `sase bead close bob-cli-ex.2 --note "<child completion and verification>"`. Leave
   containing epic `bob-cli-ex` to its land agent; do not force or cascade its close.
   Record a concrete blocker there if newly discovered work prevents completing the
   parent phase.
5. Finish with `/sase_final`, declaring every edited repository, including the plans
   sidecar. The source commit should describe the sealed-offer refusal and editor fix.
   Report what was verified and the completed child/phase closeout.
