---
tier: tale
size: small
title:
  "Close out bob-cli-2y: remove the In Progress rollback leftovers, fix two stale doc
  lines, and land the epic"
goal:
  The dead hooks plumbing the sticky-lanes change orphaned is gone, docs/capture.md and
  docs/plan.md describe what actually ships, and epic bob-cli-2y is closed with its plan
  file marked done.
proposed_by: bbugyi200.apollo.bob-cli-2y.land
bead: bob-cli-2y
create_time: 2026-09-30 19:13:24
status: wip
---

- **PARENT:**
  [202609/retire_now_sticky_lanes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/retire_now_sticky_lanes.md)
- **BEAD:**
  [bob-cli-2y](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2y/README.md)

# Close out epic `bob-cli-2y` (retire `#now`, sticky lanes, ledger-derived Today)

## Context

The land agent for epic `bob-cli-2y` verified all twelve phases (`bob-cli-2y.1` …
`bob-cli-2y.12`) against their notes, the epic plan
(`plan:202609/retire_now_sticky_lanes.md`, the PLAN path printed by
`sase bead read bob-cli-2y`), the bob-cli commits (33d5622, 63305f0, 85f7901, d06102c,
e57d33d, 473cca3, 297ecb4), bob-plugins (b9d9828, 3297b25, 053a076), and Bob Mac Capture
(fe5d1d5, ec4ad58). The work is complete and deployed except for three small leftovers
the epic itself caused. This tale fixes them and then closes the epic. It is the whole
remaining landing: nothing else resumes it.

There were no unrelated commits on bob-cli, bob-plugins, or bob-mac-capture since the
epic started, so there is no integration work. Every phase `PROPOSED FOLLOW-UP:` has
already been triaged and recorded on the epic bead (see its "LAND TRIAGE" note); do not
re-triage them or create task beads for them.

## Step 1 — Remove the plumbing only the In Progress rollback used

Phase `hooks-sticky` (commit 33d5622) deleted `clear_stale_in_progress` and
`Transition::ClearInProgress`, but the epic plan also said to remove "any plumbing only
it used". Two pieces were left behind:

1. **`reachable_identities`** in `src/native/task_status_hooks/references.rs` (about
   line 409). Its only production caller was the removed
   `let recent_activity = reachable_identities(&recent_activity_roots, &dependency_edges);`
   in `sync.rs`. `cargo clippy --lib --bins --all-features` now warns
   ``function `reachable_identities` is never used``. Delete the function and its only
   test, `rolling_reachability_is_cycle_safe_and_includes_dependencies` in
   `src/native/task_status_hooks/tests/sync.rs` (plus its import there). Keep
   `recent_activity_roots`, `desired_statuses`, and the recovery-rank graph walk: they
   still feed `directly_recent` and the Blocked recovery rank. Remove any import that
   becomes unused (for example `VecDeque`, only if nothing else in the file uses it).
2. **`FileScan.note_kind`** in `src/native/task_status_hooks/model.rs` (about line 157).
   It is now write-only: set in `sync.rs` when files are scanned (about lines 282
   and 288) and after daily normalization (about line 400), and never read. Its only
   reader was `clear_stale_in_progress`. Remove the field, those three writes, and the
   `note_kind: NoteKind::Other` initializers in `tests/sync.rs` (about lines 234, 241,
   345). **Keep** the `note_kind()` function, `NoteKind`, `is_area_or_project`, and the
   `note_kind_uses_shared_area_and_project_frontmatter_predicates` test: `compose.rs`
   still calls `note_kind(contents).is_area_or_project()`.

Behavior must not change. `docs/task-status-hooks.md` already says note kind plays no
role in lane decisions, so it needs no edit.

## Step 2 — Fix two stale doc lines

1. `docs/capture.md`, in the named-start section (about line 1082): the sentence ends
   "with the same `tasks[].index`/`tasks[].now` numbering." Phase `capture-now-removal`
   removed the `now` field from start task rows (`PomodoroStartTaskJson` has no `now`).
   Change it to "with the same `tasks[].index` numbering."
2. `docs/plan.md`, the "Obsidian Notices" row of the `## Surfaces` table (about line
   293). Its first example, `Linked · Next · NEXT 13/15`, does not match what ships:
   block-id-prompt's Ctrl+Shift+Enter Notices append the plan meter, not lane counts
   (tests assert `Linked · Next · plan 1/3 · 2/10` and
   `Unlinked · stays Next · plan 4/3 · 11/10 🔴`), while Alt+N's lane Notices carry the
   lane counts (`→ Ready · 2 tasks · unlinked 1 from today · NEXT 11/15 · PENDING 7/10`,
   `🔴 · prune at the weekly review` when over). Rewrite the row's text to state both,
   for example:

   > Ctrl+Shift+Enter link and unlink Notices append the plan meter, for example
   > `Linked · Next · plan 1/3 · 2/10` (🔴 when over a plan cap); Alt+N lane Notices
   > report the lanes, for example
   > `→ Ready · 2 tasks · unlinked 1 from today · NEXT 11/15 · PENDING 7/10`, with 🔴
   > plus a prune hint when over a lane cap.

   Keep the table's existing formatting style.

After both steps, this sweep must show no new hits (only the intentional retired-`#now`
tests, compat notes, and Pomodoro-name selector tests that are already there):
`rg -n 'tasks\[\]\.now|NEXT 13/15' docs README.md`.

## Step 3 — Verify

This repo's Justfile has no `check` recipe. Run:

- `just fmt` (cargo fmt --check) — must pass.
- `cargo test` — must pass (the land agent saw 1341 lib + 655 CLI tests green on
  297ecb4).
- `cargo clippy --lib --bins --all-features --message-format short 2>&1 | grep reachable_identities`
  — must print nothing.
- `cargo clippy --all-targets --all-features` still fails, but **only** on the
  pre-existing deny at `tests/cli/capture/pomodoro_name.rs:808` (`|| true`, owned by
  epic `bob-cli-28`). Confirm that is the only `error:`; do not fix it here. The other
  pre-existing warnings are tracked by task `bob-cli-v`; do not fix them here.

Do not run `just check-full`.

## Step 4 — Close epic `bob-cli-2y`

1. Run `sase bead epic-symbols bob-cli-2y`. When the land agent ran it, it printed "No
   --epic-symbol entries for bob-cli-2y." If any entry now appears, resolve it (wire it
   up, privatize it, add a non-test pragma, or delete it per the Symvision
   epic-whitelist policy), or re-key the Justfile line to a still-open bead that needs
   it.
2. Close the epic (it has no `parent_bead`, so nothing above it needs handling):

   ```bash
   sase bead close bob-cli-2y --note "<verification>"
   ```

   Use this verification text, appending one sentence for what Steps 1–3 changed and the
   test results you saw:

   > Land verification: read the epic and all 12 phase beads and notes, the epic plan,
   > and every epic commit (bob-cli 33d5622 63305f0 85f7901 d06102c e57d33d 473cca3
   > 297ecb4; bob-plugins b9d9828 3297b25 053a076; bob-mac-capture fe5d1d5 ec4ad58).
   > Hooks keep Next/In Progress outside daily notes (cleared_in_progress always []);
   > `@route+id!` is a link-presence toggle that never lowers a lane; the Today engine,
   > NEXT/PENDING lanes and `bob plan` schema 2 ship with T1–T9 vectors; `#now` is gone
   > from the capture grammar, pickers and rows (rg leaves only intentional retired-tag
   > tests and compat notes in bob-cli, bob-plugins and the Mac app). bob-plugins npm
   > test 873/873 and validate 6/6; deployed ledger-tools 1.7.0, block-id-prompt 1.15.0
   > and nav-hotkeys 1.42.0 match the repo byte-for-byte. Mac Capture CI green on
   > ec4ad58. Vault dash.md has TODAY/PENDING/NEXT/READY sections and chips, gtd_daily
   > NOW chores are cancelled and the morning-review/weekly-prune chores added,
   > hotkeys.json has no toggle-now-tag. chezmoi config has max_next 15 / max_pending
   > 10; installed `bob plan -f json` is schema 2. The Mac runs the sticky build (dry
   > run clears 0 / cleared_in_progress 0) with its original hooks cron line. No
   > unrelated commits landed on bob-cli, bob-plugins or bob-mac-capture after the epic
   > started, so no integration was needed. Follow-ups triaged in the LAND TRIAGE note
   > (bob-cli-28 note, bob-cli-v +1, new bob-cli-30; the rest resolved by rollout).

   Never use `--force` merely to make the close succeed. If the close is refused because
   `--epic-symbol` entries remain, finish that cleanup and close again.

3. Run `just symvision` if the recipe exists (this Justfile currently has none; if it is
   still missing, say so in your final response).
4. Set `status: done` in the frontmatter of the epic's plan file, the PLAN path printed
   by `sase bead read bob-cli-2y -r "Need the plan path to mark it done"`
   (`202609/retire_now_sticky_lanes.md` in the plans repo; it currently says
   `status: wip`). Change only that one frontmatter value.
