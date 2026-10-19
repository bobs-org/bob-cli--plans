---
tier: tale
title: Repair task-move landing gaps and finish nested closeout
goal:
  Make task moves preserve unambiguous identities and Markdown content, verify guarded
  execution, and close bob-cli-9y.1.1 plus parent phase bob-cli-9y.1.
size: medium
proposed_by: bbugyi200.athena.bob-cli-9y.1.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-9y.1.1
create_time: 2026-10-19 18:32:35
status: wip
---

- **PARENT:**
  [202610/task_move.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/task_move.md)
- **BEAD:**
  [bob-cli-9y.1.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-9y/bob-cli-9y.1.1.md)

# Finish the task-move landing and close bob-cli-9y.1.1

## Objective and authority

Repair the bounded correctness and acceptance gaps found while landing
**bob-cli-9y.1.1**, then finish its landing and close its parent phase **bob-cli-9y.1**
in this same coding turn. This tale has no later land agent. Do not wait for this tale's
own commit, SHA, push, or CI before closing: the host commits the completed turn
afterward.

Read the epic and its two children with `sase bead read`, including their landing notes,
and read `plan:202610/task_move.md` through `sase artifact read`. The authoritative
parent scope is Phase move-cli in `plan:202610/sase_hub_split.md`. Implement only the
remaining work below; keep the existing command, pure planner, guarded writer, and JSON
version 1 design. The root causes and affected functions are identified below, so one
coding agent can implement this as a medium tale without another architecture/planning
phase.

The final parent decisions remain `later_notes=hub`, `move_in_flight=yes`,
`installs_backlog=yes`, `personal_homes=routed`, and `widen_plans_prj=yes`. There are no
new reviewer decisions. Do not perform the live hub migration, install the CLI, edit
plugins or memory, or close the outer epic **bob-cli-9y**.

Use the memory-read skill for `cli_rules.md` and the governing dependency, sticky-lane,
freshness, and project-blocking decisions. Preserve task state and authored content;
reconciliation stays explicit, with no freshness stamps, dependency drops, prerequisite
promotion, or ledger-link consumption.

## Audit already completed

- Both children are closed: **bob-cli-9y.1.1.1** implemented the planner in `bfca3419`,
  and **bob-cli-9y.1.1.2** wired the command in `8b0f3782`. Their only notes were read
  completely. The temporary test-only module gate is gone, and production dispatch,
  completion, docs, and install-help smoke exist.
- Reviewed the actual planner, adapter, archive-helper changes, tests, routing, and
  docs. The scoped Git write flow, full Markdown read-set/membership guards,
  destination-first/source-last ordering, and explicit reconcile reporting exist.
- Fetched `origin` during landing. `master` and `origin/master` were `8b0f3782`;
  `git log bfca3419..origin/master` contained only the epic's second commit. There was
  no unrelated drift to integrate. Repeat this comparison at closeout and integrate new
  relevant changes if any appear.
- The command phase reports a passing `just check`. The lander did not claim a new gate
  pass. Its attempted fixture probe could not launch because the current Cargo target
  directory lacked a built `bob`; no fixture ran or vault changed. Findings below are
  from source inspection; reproduce them with regression tests against the pre-fix
  behavior before fixing them.
- `sase bead epic-symbols bob-cli-9y.1.1` found no exemptions. `just --summary` showed
  no `symvision` recipe. Recheck both during closeout.
- **Follow-up triage is complete and recorded on bob-cli-9y.1.1.** Neither child has any
  `PROPOSED FOLLOW-UP:` entry. There were zero independent out-of-scope proposals,
  duplicate corroborations, newly filed tasks, or declined proposals. All gaps below are
  this epic's own unfinished requirements. Include this outcome in the final close note.
  Do not file these repairs as unrelated tasks.

## 1. Repair selector and note resolution

In `src/native/task_move/command.rs`, `build_cli` declares the required selector
`ArgGroup` without enabling multiple members. Permit `-b` and `-m` together while still
requiring at least one selector. Add a real CLI regression with mixed, repeated,
overlapping, out-of-order selectors; assert deduplication and source order in JSON and
destination bytes. The existing mixed-selector test calls the pure planner and does not
exercise clap. Keep options and aliases unchanged.

In `src/native/task_move/plan.rs`, `resolve_note` uses the general
`vault_links::NoteIndex::resolve_detailed` directly. That helper prefers a root path and
falls back to basename even after an explicit path misses. Make command argument
resolution honor the mover's contract:

- A bare stem with both `source.md` and `folder/source.md` is ambiguous and lists both
  paths. An explicit root filename (`source.md`) or directory path can disambiguate.
- `missing/source` must not silently select `source.md`; missing explicit paths fail.
  Preserve the existing absolute/escaping-path refusals.
- Use consistent note identity for destination-project dependency refusal and reference
  repairs. Currently command selection uses the case-insensitive shared vault index,
  while `task_depends_on_dest_prj` and repair use the older case-sensitive archive
  index. A valid `[[DEST#^prj]]` link must not bypass the self-dependency refusal for
  `dest.md`, and `[[SOURCE#^x]]` must not become a dangling backlink after moving
  `source.md`'s `^x`.

Keep command-argument disambiguation separate from established contextual link
resolution rules. Use a mover-specific adapter or narrowly scoped helper support; do not
change other commands' resolution or archive semantics as a side effect. Test explicit
nested paths, case variants, root/nested duplicate stems, and ambiguous/unresolved links
without guessing an identity.

## 2. Make intake placement and fragment editing respect Markdown context

`tasks_intake_insert_byte` finds `## Tasks` with a fence mask, but then scans its body's
task examples and headings without that mask. For a destination body:

````markdown
## Tasks

```markdown
- [ ] #task Example ^example
```
````

its current `last_task_end` selects the line before the closing fence, so the incoming
real task can become code. It also inserts before trailing intake prose because it
chooses the last task rather than the end of the intake.

Compute the insertion boundary from real section/group headings outside fences. Append
after all existing intake content and before the first real `###` group or next section,
respecting the badge strip. Keep area section creation and project without-Tasks
refusal. Do not broadly reformat unrelated text.

Audit the mover's identity and fragment scanners at the same time:
`fragment_occurrences`, `validate_fragment_identities`, and the reused raw
`block_ids_in_markdown` count trailing `^tokens` in fenced examples; the mover's
`repair_fragment_markdown_links` does not mask fences or inline code. Keep code examples
and frontmatter out of live anchor/identity enumeration and leave code bytes untouched.
Use the existing Markdown/code-span helpers and retain correct handling of actual child
anchors and nested tasks. Keep any archive-specific behavior unchanged by making the
mover's context explicit where necessary.

Use full source/destination postimage assertions for fenced tasks/headings, trailing
prose, badge strips, real child anchors alongside fenced examples, inline-code and
fenced Markdown links, CRLF, and no-final-newline boundaries. Preserve original endings
except necessary insertion separators. Retain the existing collision/pathless-link tests
and add exact assertions for co-move links, destination-local links, and unchanged
review fields.

## 3. Validate the complete identity map before any rewrite

The current planner verifies source-local anchor counts and only the first `[id::]`
field, then uses `dependency_id(...).unwrap_or_default()` while creating
`MovedBlockTargets`. The shared metadata repair rewrites matching tokens across every
note, without any mover check for aliases in the path encoding.

Add a validation pass over the snapshot and proposed old/final identities:

- `a/b.md` and `a__b.md` both encode `^x` as `a__b__x`. Moving one must not silently
  change the other's `[id::]` or ambiguous dependency tokens. Reject ambiguity relevant
  to either an old or a final mapped identity before producing postimages. Also cover a
  final destination identity colliding with another note's encoding. Unrelated malformed
  identities outside the mapping are not a reason to reject every move.
- Validate every existing live task id field in each moved fragment, including nested
  tasks. A canonical first field followed by a stale second field must not survive as a
  supposedly valid moved identity. Do not interpret code examples as live fields.
- Propagate dependency-id construction failures; never synthesize empty tokens. For
  example, moving `source.md`'s anchored task with `[id:: source__x]` into
  `dest space.md` must not produce `[id:: ]`. Refuse an unrepresentable required
  identity with a clear error. Keep anchorless and absent-id behavior from the approved
  contract; do not mint fields or invent an alternate encoding.

Keep the map complete before transforming pieces and apply it once. Preserve field
spelling/spacing, aliases, embeds, and the collision-chain reservations. Test zero-write
failure through the CLI for representative invalid identities, plus planner postimages
for valid nested paths, child anchors, co-moves and multi-token dependency fields.

## 4. Complete retry/error handling and the missing integration evidence

Keep this work in the mover's adapter; do not refactor reroll or the generic writer into
a new framework.

`scan_vault` converts `CaptureError` to a string before `plan_and_apply` sees it, so an
initial unstable capture is terminal even with retry budget. Preserve enough typed
failure information to retry transient concurrent/unstable reads before any write,
including a fresh scan, within the configured budget. Do not retry deterministic
selection, permission, encoding, or unsupported-file errors. Zero budget still makes
exactly one attempt. Add deterministic test injection at the scan boundary rather than
timing a race.

`Report::fail` ignores all later failures once partial apply has set an error. If
committing the applied subset then fails, that Git failure is currently lost. Retain the
primary partial-write/recovery error and expose secondary commit/sync failure details
(an additive warning is sufficient). Keep nonzero exit and never automatically replay
selectors after any applied file. Do not imply a partial move completed successfully, or
tell the user to rerun it blindly.

Finish the acceptance tests promised by the approved command phase:

- Normal local-bare-remote flow with a pending local edit and a peer edit pulled before
  planning. Make the peer edit affect the selected task or a backlink so the test proves
  the planner uses the post-sync snapshot. Assert a separate pre-sync commit, the exact
  scoped move commit, successful post-sync push, and JSON stage records.
- Staging failure before replacements using `ApplySession.fail_staging`: no note
  changes, exact deferred paths, retained recovery evidence, nonzero outcome.
- Failure immediately after destination replacement as well as the existing later
  failure before source replacement. Assert order, exact applied/deferred paths,
  existing recovery directory, preservation of source and concurrent text, no unsafe
  retry, and JSON failure/hint. Exercise orchestration exit/reporting, not only the
  inner plan-and-apply tuple. Cover secondary commit failure after partial apply and an
  ordinary commit failure without post-sync.
- Post-sync conflict via a deterministic fixture hook, adapting
  `tests/randomize.rs::randomize_post_sync_conflict_keeps_local_commit_and_warns`.
  Assert remote-wins behavior, conflict details, and a failed move report.
- End-to-end co-move with collision and a daily-note backlink, followed by archive or
  reconcile in the fixture to show canonical identities remain usable. Assert
  freshness/keeps/unblocked bytes separately immediately after the move.
- Dry-run still changes no notes/Git/state and works under a held maintenance lock.
  Include Git-state assertions, since existing dry-run coverage is non-Git.

Reuse `tests/cli/task_move.rs` and its isolated environment/local-remote fixtures, and
the adapter's existing session-factory test seam. Split test/helper modules if needed
for readability. Do not access Bryan's vault or installed CLI. Keep single-object stdout
JSON and progress on stderr. Update docs only for corrected placement, refusal, or
secondary-error details; no new CLI options are needed.

## 5. Verify and integrate

Run the focused mover planner/adapter/CLI tests, archive regressions, reroll, help and
completion tests, then **`just check`**. Do not run `just check-full`. Use
`/sase_monitor` for commands that exceed the inline limit and give its continuation the
remaining closeout steps explicitly. A passing final gate must return to this closeout;
a generic prepared completion that merely commits is insufficient while the epic remains
open.

Demonstrate real `bob task move --dry-run -f json` and `--offline` paths in temporary
fixtures; the integration tests may supply that evidence. Record actual results, not
only previous phase claims. Do not repeat or broaden passing checks without a new
change, failure, or specific unresolved concern.

Re-read epic/child notes and compare new commits on the current branch and
`origin/master` since `bfca3419`, excluding this epic and this repair's work. Integrate
any new relevant callers/conflicts before close. If genuinely unrelated follow-ups are
newly discovered, follow `/sase_new_task`, identify the proposing bead, and append every
triage outcome to bob-cli-9y.1.1 before its close.

## 6. Finish the nested landing in this turn

After all repairs and checks pass, perform this exact closeout. It must not depend on
this work's own commit, push, or CI result.

1. Run `sase bead epic-symbols bob-cli-9y.1.1`. Resolve every listed entry by wiring,
   privatizing, adding an appropriate non-test pragma, or deleting it; re-key only when
   a genuinely still-open later bead needs the exemption. Check readiness of both closed
   children and any new descendants. Fix any outstanding epic work before proceeding.
2. Close normally with `sase bead close bob-cli-9y.1.1 --note "<verification>"`. The
   concrete note must summarize the original two commits, these remaining-work repairs,
   all notes/decisions reviewed, post-start integration findings, actual test and
   fixture results, and the recorded zero-proposal follow-up triage plus any later
   outcomes. If the close names leftover exemptions, clean them and retry. If it names
   incomplete work, finish it or deliberately handle a genuinely canceled/superseded
   scope. Never force to make a successful landing advance.
3. Run `just symvision` if that recipe is available. If it remains absent (as at the
   original landing review), explicitly record that in the closeout evidence; do not
   invent a recipe. If a gate finds stale exemptions, fix and verify them.
4. Use `/sase_repo` to open `plans` before modifying its files and use only the printed
   checkout path. The epic's PLAN is **202610/task_move.md** in that checkout
   (`plan:202610/task_move.md`). After the normal close, set only its frontmatter
   `status: done`, preserving its other fields. Sidecar artifact context reads must
   still use `sase artifact read`. Include the plans checkout in the final declaration's
   commit obligations when changed.
5. Run `sase bead read bob-cli-9y.1.1 -r "Need the parent link"` and
   `sase bead read bob-cli-9y.1 -r "Verify the parent move-cli phase is complete"`. The
   known parent is phase **bob-cli-9y.1**. Confirm the entire move-cli scope from
   `plan:202610/sase_hub_split.md` is satisfied by the now-complete child plan. Run
   `sase bead epic-symbols bob-cli-9y.1`, resolve or legitimately re-key any entries,
   then close only this phase normally:
   `sase bead close bob-cli-9y.1 --note "<move-cli implementation and checks verified>"`.
   Re-run `just symvision` if available after this close. Leave the containing epic
   **bob-cli-9y**, its other phases, and its plan status to its waiting land agent. Do
   not use `--force` to advance this successful nested landing.
6. Finish with `/sase_final` and a concise report of the two closures, checks, and any
   remaining outer-epic work. If any required scope remains incomplete, keep the
   affected bead open and record the specific blocker instead of falsely declaring
   completion.
