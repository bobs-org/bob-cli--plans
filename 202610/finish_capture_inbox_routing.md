---
tier: tale
title: Finish capture inbox routing and land bob-cli-bk
goal:
  Preserve atomic snapshot writes and exact routed-task identity, correct the approved
  glossary wording, verify integration, and close bob-cli-bk.
size: medium
decisions:
  memory_inbox_file:
    ask:
      Correct the Inbox File sentence to include closing captures, as the epic already
      approved?
    default: true
    memory:
      - glossary:inbox-file
    corrects:
      A non-closing capture that changes an existing inbox task's status asks the same
      question in Bob Mac Capture's route card
    answer: true
proposed_by: bbugyi200.apollo.bob-cli-bk.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-bk
decided_by: auto
status: done
---

- **PARENT:**
  [202610/capture_inbox_routing.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/capture_inbox_routing.md)
- **BEAD:**
  [bob-cli-bk](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-bk/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-bk.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-bk.land.md)
- **COMMITS:**
  - [3398dd4](https://github.com/bobs-org/bob-cli--plans/commit/3398dd471d81a6aadc05c6086f7c4c9dd774a0dc)
    — docs(plans): mark capture inbox routing and finish tale done

# Finish capture inbox routing and land bob-cli-bk

Complete the bounded defects found by the land review, verify integration, and close
`bob-cli-bk` in this coding turn. This tale has no land agent. Do not wait for this
turn's commit, SHA, push, or CI: the host commits after the turn ends, and closing the
epic with the verified final source still uncommitted is the intended sequence.

## Reviewed scope and decisions

Read `sase bead read bob-cli-bk -r "Need the landing notes and final decisions"` and the
audited artifact `plan:202610/capture_inbox_routing.md`. The four phase beads
`bob-cli-bk.1` through `.4` are closed. The lander read every child note and reviewed
actual code and commits: bob-cli `bb52dd0a`, `4f13e979`, `e70f9ea4`, and linked
bob-mac-capture `2968311`, `7bfa312`. Preserve the final decisions: closes route;
side-effect status changes route; the Inbox File glossary update is authorized; the
separate capture-routing decision record was declined and is follow-up work. Bob remains
the only grammar, destination, and vault-write authority. Preserve `bob task move` CLI
behavior and additive capture JSON compatibility.

The review baseline is bob-cli `3bc7e45e` and bob-mac-capture `7bfa312`. Refresh the
post-baseline drift review before implementation. Open linked repos only through
`/sase_repo` and read their AGENTS.md. Read required reference memory through
`/sase_memory_read`, and use `/sase_memory_write` for the glossary correction.

## 1. Preserve the move snapshot's actual preimages

In `src/native/capture/inbox_route.rs`, `destination_snapshot` reads every Markdown file
into a pure snapshot. `apply_inbox_routes` computes a move plan, then calls
`planner.ensure_loaded` on each postimage. For a previously unstaged destination or
link-repair file this second disk read can adopt an intervening edit as the preimage
while staging the replacement computed from older bytes. The final preimage check then
succeeds and overwrites the edit. The existing
`ir_routed_batch_rolls_back_when_a_repaired_file_changes` regression edits after
planning and does not cover this window.

Bind newly staged files to the exact existence/content snapshot consumed by the move
planner. Reuse batch loading/staging primitives or add a narrow snapshot-aware primitive
in `capture/batch.rs`; do not add a maintenance lock or second write phase. Already
staged files retain their original disk preimages and current overlay. Treat any disk
drift from the consumed snapshot as the existing typed `NoteChanged` refusal. Keep the
change scoped to correct snapshot binding and the existing staged overlay.

Add a deterministic regression that changes a previously unstaged repaired file between
snapshot construction and staging (factor a pure staging helper or use a narrow test
hook, not timing-dependent sleeps). Assert no capture/move writes and preservation of
the concurrent edit. Cover a destination as well as a repaired inbound-link file, and
keep the existing post-planning rollback regression passing.

## 2. Preserve exact task identity through grouped moves and refusals

`locate_dest_line` currently searches for a unique description/status when a moved root
has no block ID. This rejects valid exact-line moves when the destination already
contains the same title/status or two same-title id-less roots move together.
`relocate_source_line` also falls back to text matching after earlier groups change the
source. `RoutedTaskMove.old_line` currently comes from each group's changed snapshot,
while `TaskBlockTracker` still holds coordinates from before all routing. Those
coordinates diverge after an earlier group removes lines.

Carry exact source-to-destination positions from the move planner, including roots
without IDs and nested tasks carried by them. A crate-internal destination-line field on
move records or an equivalent explicit line mapping is appropriate. Remap remaining
candidates and tracked tasks as groups are applied; preserve the original candidate keys
and numbering for responses. Do not mint IDs merely to make routing work, and do not
identify the moved task by a unique title/status search. Produce final destination
lines/IDs in `moved`, `task_blocks`, `pomodoro_blocks`, and relevant `open_notes`, with
freshness applied to the correct moved open root.

Add regressions for two identical id-less recovered inbox roots routed together, an
identical root already at the destination, and an id-less tracked candidate in a later
destination group after an earlier source removal. Include carried child content to
prove which root moved; assert final mirrors/landings and dry-run versus real-run
parity. Existing block-ID collision, completion-without-freshness, and
pull-in-plus-route tests must stay green.

Also fix `route_engine_refusal`: it always attributes a group error to `members[0]`,
even when `PlanError` identifies a later candidate. Map task-specific errors to that
candidate; use the first member only for genuinely destination-wide errors. Test one
valid member followed by a member that depends on the destination project task:
`refused.key`/`n` must identify the failing second member, with no writes.

## 3. Finish integration and the approved glossary correction

Post-start review already checked these interactions:

- `57d77612`/`6def3ac8`: shared relation preferences, cached label catalog, and capture
  commit invalidation. Routing must continue using the common batch commit.
- `895b9ba7`: one-off `||` labels and structured warnings; label-hide events run only
  after the capture notes commit. Add a combined bulk regression with a status-changing
  inbox item and a one-off relation item. A route guard/refusal writes neither notes nor
  hide events; a routed success repairs the relation links and appends the intended
  event once. Preserve warnings, cache invalidation, and dry-run behavior.
- `3bc7e45e` already updates the routing rollback test for the new
  `commit_capture_batch(&mut batch, &vault)` signature. Keep that integration.
- Theme journal extraction, label rename, theme completion repairs, annotation sections,
  stats, and listen changes require no replacement routing implementation.

> [!decision] memory_inbox_file

Correct `glossary:inbox-file`: remove only `non-closing` from the capture sentence,
keeping the distinct Obsidian closing exclusions. The epic's accepted
`memory_inbox_file=yes` already intended this wording; this tale's correction decision
carries that authorization explicitly. Run `sase memory init` and `just docs-gen` to
update generated material, then audited-read the strand again. The generated
`docs/reference/glossary.md` must agree. If this correction decision is declined, record
the unresolved decision mismatch on bk and do not close it.

## 4. Validate the final source

Run `just check` in bob-cli. Do not run `just check-full`. Run `just docs-check` for the
glossary/docs output. Execute the focused routing, task-move, pull-in, and combined
relation cases while developing; the canonical `just check` is the final file-change
gate. Classify unrelated failures through `/sase_new_task`; failures caused by this epic
must be fixed here. A `just check` pass with a separate `just check-full` failure is a
test-infrastructure issue, not remaining epic work.

The Mac phase's initial unverified-UI note has newer evidence: CI run
https://github.com/bobs-org/bob-mac-capture/actions/runs/40137474050 on `7bfa312` passed
Swift formatting and application Build. Test compilation failed solely on
`CaptureRelationPanelTests.swift:227`, `hidden` needing to precede `count`; this belongs
to active epic `bob-cli-bj` and has been corroborated there. Review newer linked commits
and CI before concluding validation. The Mac answered SSH during landing review. Use a
fresh scratch copy (rsync without `.build`/`.git`, never Bryan's checkout or
`just install`) for `Scripts/bundle.sh` and `just typecheck-tests`; delete the scratch
afterward. Run `just portable-check` on a Swift-capable host such as athena. These
commands are required if Swift is changed; verify existing route model/key/height suites
as tooling permits. Do not turn the independently tracked bj test bug into new bk
feature work, and report any remaining macOS test limitation honestly. Never claim
portable checks compile UI.

Use `/sase_monitor` before long commands, with a continuation that explicitly retains
this tale's remaining closeout steps. Do not submit a prepared completion that bypasses
closing bk and marking its plan done. Do not wait for this turn's own commit or CI
before closeout.

## 5. Close bob-cli-bk in this same work sequence

Re-read the epic and every descendant after the fixes and confirm all required work is
complete. The current epic has no `parent_bead`; confirm with
`sase bead read bob-cli-bk -r "Need the parent link"` again. Carry the following
completed triage outcomes from the lander's LAND TRIAGE note into the close note:

- bk.1 #1, bk.2 #1, bk.3 #1, bk.4 #1: one new ready small memory task `bob-cli-br`,
  related to `bob-cli-4t`; the latter is the distinct Obsidian contract.
- bk.2 #2: declined a new stale published-URL task because `57d77612` supplied
  `docs/reference/cli/relations.md` plus navigation and `4c46dadd` extended it; bk.4
  recorded a later passing docs-check.
- bk.2 #3 and #4 (also bk.1 #2's pre-existing reds): declined duplicate tasks;
  `09b26957` restores themes rename kinds/help and fixes polluted completion caches;
  bob-cli-bc close note #6 records the passing gate after these repairs.
- Mac hidden/count compilation issue: independent CI corroboration added to the
  responsible active epic bob-cli-bj, no new task.

Run `sase bead epic-symbols bob-cli-bk` immediately before closing. It was empty in the
review, but resolve every current entry or re-key it only to a still-open later bead
that truly needs it. Use the Symvision epic-whitelist policy. Never leave entries for a
bead being closed. If this tale is registered as an open descendant, finish and close
that completed tale normally before the containing epic; never force a successful nested
landing.

Close normally with:

```sh
sase bead close bob-cli-bk --note "<source/decision and drift review, exact fixes, verification results, all follow-up outcomes and any independent validation limits>"
```

If rejected for symbols, clean them and retry; if rejected for unfinished named phases,
finish or reopen the actual work. Never use `--force` just to make the close succeed.
Then run `just symvision` if available; if no recipe exists, record that. Open the plans
repo through `/sase_repo`, resolve `plan:202610/capture_inbox_routing.md` (its
repo-relative path is `202610/capture_inbox_routing.md`), and set only its frontmatter
`status: done`. Current parent inspection found no parent, so no ancestor close is
pending. If a parent is newly present, follow the land prompt's normal phase/plan
readiness rules before closing any ancestor. Finish with `/sase_final` after all
verification and mutations, declaring every changed repository.
