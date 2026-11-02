---
tier: tale
size: medium
title:
  Finish Off-Clock Pomodoros — restamp every running-lineup edit, tab-safe settling, and
  closeout
goal:
  Every capture gesture that changes the running session's counted Task Links restamps
  its [t::]/[off::] key, settled MEETs use the note's indentation, the off-clock docs
  match the writers, and epic bob-cli-fl is verified and closed.
proposed_by: bbugyi200.apollo.bob-cli-fl.land
proposed_with: claude/opus
bead: bob-cli-fl
create_time: 2026-11-02 13:22:41
status: wip
---

- **PARENT:**
  [202611/off_clock_pomodoros.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/off_clock_pomodoros.md)
- **BEAD:**
  [bob-cli-fl](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fl/README.md)

# Finish Off-Clock Pomodoros (bob-cli-fl landing repairs)

## Why

The bob-cli-fl land agent audited all seven phases plus the integration with everything
that landed since the epic started. Most of it holds:

- the merged off-clock code checked out against the plan;
- the shared vectors match bob-plugins byte for byte;
- the gcal description rewrite keeps the `#offclock` aliases working;
- bob-mac-capture CI builds, and every off-clock XCTest passes.

Four gaps remain. Each is caused by the epic itself, so each is epic work.

1. **Running-lineup edits that skip the restamp.** Plan phase `off-clock-rule` step 3
   says: "Capture gestures that add links to or drop links from the running session
   re-stamp its key." `pomodoro_link.rs` (links and `~K` drops) does this through
   `restamp_session_clock` (`src/native/capture/meet_event_run.rs`). Three capture
   writers change the running entry's counted Task Links without it:
   - **`=*` merge** (`src/native/capture/pomodoro_merge.rs`,
     `plan_pomodoro_merge_item`). It moves the absorbed entry's Task Links under the
     source, stages the result (around the `planner.stage(&day_file, …)` after
     `let entry_line = src_zero + 1;`), and never restamps. Its `_warnings` parameter is
     unused. Merge-by-default (50887216) makes bare `=*` hit the running session far
     more often. Reproduced on a scratch vault:
     - Running `- [ ] (**0905-0930** [t:: 25m]) — BOB` has no links. A planned `WALK`
       links only `[[bob#^walk]]`, which is `#offclock`. After `=*` the row is
       `[t:: 25m]) — BOB + WALK`; it should be `[off:: 25m]`.
     - Running `[off:: 25m]) — WALK` merges a planned on-clock `FIX`. The row stays
       `[off:: 25m]`, no `pomodoro_clock_mixed` warning appears, and
       `pomodoro_blocks[].off_clock` reports `true`.
   - **`=/` split** (`src/native/capture/pomodoro_split.rs`, around the
     `session_clock(...)` / `set_ledger_clock(renamed_text, kind)` block near line 500).
     It only re-applies the row's existing key. Reproduced: splitting `FIX` out of a
     running `[t:: 25m]) — WALK + FIX` leaves `WALK`, which now links only the
     `#offclock` walk task, at `[t:: 25m]`.
   - **Capture blocked-link consumption** (`src/native/capture/project_deps.rs`, the
     `task_status_hooks::apply_blocked_consumption_with_options` call near line 381;
     this covers the `&`, `*`, and link-guard Consume branches). It removes Task Links
     from today's open Pomodoros, which can include the running one, with no restamp.
     This one was found by source inspection only, so reproduce it before you fix it.

   Closes self-heal: `x` recomputes K. So the wrong key lives only on the running row,
   the Now card, and the capture JSON. That is exactly what the restamp rule exists to
   prevent.

2. **Settled MEETs hard-code two-space child indentation**
   (`src/native/task_status_hooks/settle.rs`, the
   `insert.push(format!("  - 🍅 ~~[[#^{}]]~~", …))` line near 471). Bryan's vault
   indents Pomodoro children with tabs. gcal's closed-MEET writer derives the unit with
   `dominant_indent_unit` (falling back to `"\t"`; see `src/native/gcal/plan.rs` around
   line 1434), and the plan said settling would reuse gcal's insertion.
3. **Stale off-clock docs.**
   - `docs/reference/json-outputs.md` still says "MEET starts add
     `pomodoro_start.off_clock`", but every off-clock start reports it now.
   - `docs/capture.md` around lines 2817 and 2834 shows a bare timed row
     `(**0920-0950**)` in the `!` completion example. No writer produces bare timed rows
     any more.
   - `docs/capture.md` "Which clock a session is on" does not list merge, split, or
     consumption among the running-lineup edits, and the Merging and Splitting sections
     never mention the clock key.
4. **Undocumented gcal directive change.** Since 35c0a39a (description parsing), a
   plain-text description list item `- #offclock` counts as an off-clock directive line,
   because the list marker is stripped first. HTML `<li>` already counted.
   `docs/gcal.md` (directive lines, around 533–546) should say so.

Follow-up triage is already done and recorded on bob-cli-fl (new tasks bob-cli-g3, g4,
and g5; a +1 on bob-cli-e4; a DISCOVERED ISSUE note on bob-cli-fs). Do not refile any of
it.

## Work

### 1. Restamp after every capture write that changes the running session's links

- **Merge.** In `plan_pomodoro_merge_item`, after the merged contents are staged, call
  `restamp_session_clock(planner, &request.bob_dir, &day_file, &day_rel, entry_line, now, warnings)`
  for the source entry.
  - Plumb `now` from the caller in `src/native/capture/plan.rs` (around line 866), the
    same way the start and link paths get it.
  - The function already does nothing for untimed `()` rows, so planned-entry merges are
    unchanged.
  - Re-read the staged contents after the restamp before building the summary/output, so
    `task_line`/`pomodoro_blocks` reflect the new key.
  - Rename `_warnings` to `warnings` so `pomodoro_clock_mixed` reaches the output.
- **Split.** Replace the keep-the-existing-key normalization with a restamp of every
  timed entry whose lineup the split changed: the remaining source and, when timed, the
  split-out entry. Use `restamp_session_clock`, or the shared verdict it wraps.
- **Consumption.** After capture consumption removes links from a still-present running
  timed entry, restamp that entry. An entry deleted because consumption emptied it needs
  nothing.
- **Audit.** Grep `src/native/capture*` for any other capture writer that adds, moves,
  or removes Task Link bullets under an open timed entry, for example task moves that
  carry Task Links. Each one must restamp too, or the report must state why it cannot
  change K.
- **Constraints.** Do not touch the `x` close path or plugin code. Keep `--dry-run`
  parity: the dry-run output must show the same key and warning as the real write.

### 2. Tests (Rust only; do not edit the shared vector files)

Add cases to `tests/cli/capture/pomodoro_clock.rs` as extra tests beside the existing
restart and link-start extras. Do not add them to
`tests/fixtures/pomodoro_clock/vectors.json`: bob-plugins mirrors that file byte for
byte and has no merge, split, or consumption harness for it.

- `=*` into running, both directions: worked→off-clock, and off-clock→mixed with `[t::]`
  plus `pomodoro_clock_mixed`.
- An explicit `=*#name` into running.
- `=*` merging two planned entries stays untimed.
- `=/` split leaving an all-`#offclock` remainder becomes `[off::]`.
- `=/` split leaving an on-clock remainder stays `[t::]`.
- Consumption that leaves only `#offclock` links on the running row becomes `[off::]`.
- For each, assert the dry-run vs real-write parity of the row and the warning.
- Before fixing anything, run each new test against unmodified code and confirm it
  fails. Keep that red-before/green-after evidence for the close note.

### 3. Settle indentation

In `settle.rs`, build each settled MEET's child bullets with the note's dominant indent
unit:

- Reuse `dominant_indent_unit` (or the gcal helper that wraps it) and fall back to
  `"\t"`.
- Keep `- 🍅 ~~[[#^id]]~~` otherwise unchanged.
- Add reconcile tests for a tab-indented note (children get `\t- 🍅 ~~…~~`) and a
  two-space note (children keep two spaces).
- Rerun the existing settle tests, e.g. `cargo test --test cli settle` and the
  task_status_hooks lib tests.

### 4. Docs

- `docs/reference/json-outputs.md`: `pomodoro_start.off_clock` is reported for any
  off-clock start, not only MEET starts.
- `docs/capture.md`:
  - Stamp the example rows near 2817 and 2834 (`(**0920-0950** [t:: 30m])`, keeping
    whatever the example's minutes are).
  - In "Which clock a session is on", add merge (`=*`), split (`=/`), and capture
    consumption to the running-lineup-edit row of the K table.
  - Add one sentence each in the Merging and Splitting sections: the running session's
    key is restamped from its new lineup, and a mixed result warns
    `pomodoro_clock_mixed`.
- `docs/gcal.md`, directive lines: a plain-text list item whose only content is
  directives (`- #offclock`) is a directive line, like an HTML `<li>`.
- Never write "untracked session" in new prose. Say off-clock session or Off-Clock
  Pomodoro.

### 5. Verify

Run `cargo fmt`, the new and existing focused tests, `just docs-check`, and
`just check`. Do not run `just check-full`.

`just check` may still fail `docs_reference::cli_reference_is_current` on
`docs/reference/cli/capture-complete.md`, `capture-parse.md`, and `relations.md`. That
drift comes from d8b4d4fb (bob-cli-fs.3) and is recorded on bob-cli-fs.

- If those three pages are the only failure, leave them alone and name the failure in
  the close note.
- If anything else fails, fix it when this work caused it.
- If this work did not cause it, record it with `/sase_new_task` before closing.

### 6. Close out epic bob-cli-fl (final step; do it in this same turn)

1. Run `sase bead epic-symbols bob-cli-fl`.
   - The land agent saw "No --epic-symbol entries for bob-cli-fl".
   - If any entry exists now, resolve it (wire it up, privatize it, add a non-test
     pragma, or delete it per the Symvision epic-whitelist policy). Re-key it only when
     a still-open later bead needs the exemption.
2. Close the epic with `sase bead close bob-cli-fl --note "<verification>"`. The note
   must summarize:
   - **This work:** the restamp fixes with red-before/green-after test names, the settle
     indentation fix, the docs, and the gate results, including any pre-existing failure
     by name.
   - **The land agent's verification:**
     - all 7 phases and the nested epics fl.3.1 and fl.6.1 are closed and audited
       against the plan and DECISIONS (settled_status=done; memory strands
       glossary:off-clock-pomodoro, glossary:pomodoro, glossary:event-task,
       decisions:off-clock-pomodoros-count-apart,
       decisions:ended-off-clock-events-settle present, untracked-sessions-omit-t-field
       superseded);
     - the shared `pomodoro_clock` / `meet_event_run` / `pomodoro_homes` /
       `event_task_close` vectors are byte-identical in bob-plugins;
     - gcal integration (35c0a39a, cba88d0d, 3d251ba2, db4e8a8a, fa1118c1, 1562baab) was
       checked with scratch-vault reproductions;
     - capture integration (a67cc79a, d32da8fa, 50887216, 2d45cd02, 0ebd0714) was
       checked, and the gap found is what this tale fixed;
     - Mac: master CI builds, and all 6 off-clock XCTests plus the
       `pomodoro_clock_mixed` presentation tests pass; the 43 unrelated CI failures are
       tracked by bob-cli-f3 and bob-cli-fu;
     - plugins: the focused off-clock suites pass and bob-ledger-tools 1.91.0 is live in
       the vault. The vault's task-status-cycler is a stale dirty bob-cli_21 deploy
       without fl.5's close code; that is tracked by bob-cli-e4 (+1 recorded) and was
       not forced;
     - follow-up triage (g3, g4, g5, the e4 +1, the bob-cli-fs note, and the declined
       proposals) is recorded in bob-cli-fl's FOLLOW-UP TRIAGE note.
   - Never use `--force` merely to make the close succeed.
3. Run `just symvision` if the recipe exists. This checkout's `justfile` has no such
   recipe; if it still has none, say so in the final report.
4. In the epic's plan file `plan:202611/off_clock_pomodoros.md` (plans sidecar
   `sase/repos/plans/202611/off_clock_pomodoros.md`), change the frontmatter
   `status: wip` to `status: done`.
5. bob-cli-fl has no `parent_bead`, so there is no parent to close.
