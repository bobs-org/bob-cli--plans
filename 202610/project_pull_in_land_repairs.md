---
tier: tale
size: medium
title: Repair bob-cli-74's project pull-in defects in both engines, then close the
  epic
goal: Rust and the task-status-cycler agree with each other and with docs/task-dependencies.md
  §12.10 / §12.11 on every case listed below. Each fix is pinned by a test, the fixed
  bob and plugins are installed on this host, and epic bob-cli-74 is closed with its
  plan file marked done.
proposed_by: bbugyi200.apollo.bob-cli-74.land
bead: bob-cli-74
status: done
---

- **PARENT:**
  [202610/project_pull_in_line_model.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/project_pull_in_line_model.md)
- **BEAD:**
  [bob-cli-74](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-74/README.md)

# Plan: finish landing bob-cli-74 (project pull-in repairs + closeout)

## Context

Epic **bob-cli-74** (plan `plan:202610/project_pull_in_line_model.md`) ships the "pull
an emptied project's `^prj` in as a successor link" feature. All six phases are closed:

- bob-cli commits `eb03584` (contract docs), `5fc27f0` (cli-model) and `309e98a`
  (cli-capture);
- bob-plugins commits `00f3e8f` (nav 2.28.0) and `9e05379` (cycler 1.35.0);
- bob-cli-6x is closed;
- no `--epic-symbol` entries exist and the epic has no parent bead.

The landing audit (notes on bob-cli-74) found defects that this epic caused. They are
listed below with their current locations. Line numbers are approximate; confirm each
one by reading the code. Each defect was reproduced on throwaway vaults during the
audit.

This tale fixes all of them and then closes the epic. Read §12.10, §12.11, §12.5 and
§12.6 of `docs/task-dependencies.md` first. They are the contract. If a fix changes a
contract statement, update the doc in the same change.

Follow-up triage is already done and recorded on bob-cli-74, so do not redo it:

- duplicates got a +1 on bob-cli-6m, -6v and -6w;
- new tasks bob-cli-75 (decision record), -76 (Mac copy) and -77 (ledger glyph).

## Rules

- **Known failures.**
  - bob-cli `just check` may fail only the two bob-cli-6m percent-listen tests.
  - bob-plugins `npm test` may fail only the four bob-cli-6v unblocked-glyph cases, the
    bob-cli-6w decision-card Keep case, and the bob-cli-3w stage-ranker timing flake.
- **Commands.** Do not run `just check-full`.
- **bob-cli scope.** No new CLI options or subcommands.
- **Safety.**
  - Never edit `~/bob`.
  - Against `~/bob`, run `bob task reconcile`, `bob projects sync` and `bob capture`
    only as dry runs.
  - Experiments use `BOB_DIR=<temporary copy>`.
  - Do not touch the MacBook. It was unreachable, and Bryan's pilot steps are already
    recorded on bob-cli-74.6.
- **bob-plugins.**
  - Open it with `sase repo open bob-plugins -r "<why>"` and read its `AGENTS.md`.
  - Edit only `src/` fragments, keeping each ≤ 1000 lines (`085-successor-plan.js` is at
    954).
  - Then run `npm run build`, `npm test` and `npm run validate`.
- **Memory.** Do not edit memory.

## Part A — bob-cli (Rust)

1. **R1: `#hide` detection must use sync's tag boundaries**
   (`src/native/task_complete/`).
   - **Defect.** `project_pull_in.rs::pull_in_hide_removed` (≈296) and the pull-in
     branches in `successors.rs` use `capture_dependency_tasks::has_hide_tag`. That
     function matches only a whitespace-delimited `#hide`. Sync's scan (`tag_spans`) and
     `normalize_task_hide_tag` also accept punctuation boundaries (`#hide,`, `#hide.`,
     `(#hide)`).
   - **Failure (unlinked).** R is `- [ ] #task #prj Ship it #hide, ^prj` and N's last
     task closes unlinked. bob writes nothing. `bob projects sync --dry-run` then
     removes `#hide`, which breaks PP23.
   - **Failure (linked).** R's tag is removed, but the row says `hide_removed: false`
     and the row text still shows `#hide,`.
   - **Fix.**
     - For pull-ins, decide "R carries `#hide`" and `hide_removed` as
       `normalize_task_hide_tag(raw, false, true) != raw`.
     - Make the display text drop `#hide` with the same boundaries.
     - Do not change the pre-existing non-pull-in `Recover("hidden")` path at
       `successors.rs` ≈704. It is dependency-successor behaviour outside this epic.
   - **Tests.**
     - Add a planner test: a punctuation-bounded `#hide` in both the linked and the
       not-planned-today cases. Cite PP22/PP23.
     - Add a CLI test in `tests/cli/capture/project_pull_in.rs`: a live pull-in on a
       temp vault, then `bob projects sync --dry-run` names no event for N.
2. **R2: build the dependents snapshot when the cascade needs it.**
   - **Defect.** `inspect_pull_ins` (`project_pull_in.rs` ≈78–118) sets
     `needs_dependents` only when a C task has `[id::]` or R has `[dependsOn::]`. When R
     is `[?]` (a stale Blocked with no Depends-On), the §12.2 PROJECT RECOVERY cascade
     (`recover_project_members`) checks residents' own prerequisites only against C's
     notes.
   - **Failure.** A resident `[?] #prj_is_blocked #hide [dependsOn:: x] ^r` whose `x` is
     still open in another note is wrongly recovered to `[ ]`.
   - **Fix.**
     - `needs_dependents` is also true when a candidate R has status `?` and some open
       resident of N carries a non-empty `[dependsOn::]`.
     - Add this bullet to §12.10 **Gate**.
     - Mirror the same condition in the cycler's `projectPullInNeedsDependents` (Part
       B).
   - **Tests.** A planner/gate test, and a CLI `!` test where the resident stays `[?]`
     and tagged.
3. **R3: `reason: "project"` still-blocked rows carry `scheduled`.**
   - **Defect.** §12.5 says a `project` row carries `scheduled` when the project date
     blocks. The `StillBlockedProject` arm (now in `project_pull_in.rs` `emit_rows`
     ≈497–517) always sends `scheduled: None`, so nav's `· project until <date>` never
     shows for Rust rows. The cycler already sends the date.
   - **Fix.**
     - Thread the blocking project's `ProjectBlockReasons::future_scheduled`
       (`src/native/project_dependencies/blocking.rs`) through `project_still_blocks` or
       `project_states`.
     - Emit it as `YYYY-MM-DD` on the row. Keep `None` when only dependencies block.
   - **Tests.** Assert the JSON key in a CLI or unit test.
4. **R4: docs match the shipped Rust output.**
   - In `docs/task-dependencies.md` §12.6 (≈1241), the Rust human still-blocked suffix
     for `reason: project` is `· project <path>` (as `capture/output.rs` ≈2283 prints),
     not `· blocked by project`. The nav card keeps `· blocked by project` and
     `· project until <Mon D>`.
   - Fix `docs/capture.md` ≈3527 to list the `· project <path>` suffix.
   - Fix the §12.6 example `surfaced  Ship the outcome!` to the single space the code
     prints.
   - Document in §12.5 that pull-in still-blocked rows carry `project` (N's path) for
     every reason. Rust already does this; Part B makes the cycler match.
5. **Cleanup.**
   - `pull_in_needs_dependents` (`project_pull_in.rs` ≈64, re-exported in
     `task_complete/mod.rs` ≈15) is used only by tests and warns in non-test builds.
     Delete it and have the tests call `inspect_pull_ins(..).needs_dependents`.
   - Remove the private `raw_task_line` copy in `project_pull_in.rs` (≈696) and use
     `successors::raw_task_line`.
   - Narrow any `pub(crate)` widening from `5fc27f0` that has no out-of-module caller
     (`earliest_anchor`, `SlotAnchor` / `DayEntry` fields).
   - Update the stale comment at `capture/task_complete.rs` ≈130–141. It still says the
     `[id::]` gate skips successor planning; describe the §12.10 gate as shipped.
6. Run `cargo fmt`, then `just check`. Only the bob-cli-6m failures are allowed.

## Part B — bob-plugins task-status-cycler (and nav fallback)

Files are in `plugins/task-status-cycler/src/` unless noted. Mirror Rust
(`src/native/projects/surface.rs`, `scan.rs::parse_project`, `frontmatter_is_project`,
`model.rs::ProjectStatus::parse`, and `task_complete/project_pull_in.rs`).

1. **P1: path-qualified skip keys.**
   - **Defect.** `projectClosedSkipKeys` (`089-successor-project.js` ≈316) and its use
     in the REMAINING loop (≈238) key on `^blockId` / `id:` with no note path. A task in
     N that shares a block id with a closed task in another note is skipped.
   - **Failure.** A false pull-in: R is linked and `#hide` is stripped while Rust says
     REMAINING = 1.
   - **Fix.** Key on path + block id. Pure tests that pass pre-close text must still
     skip only N's own closed tasks.
   - **Test.** Two notes with the same `^review` id.
2. **P2: Rule step order.**
   - **Defect.** `planProjectPullIns` (≈635–654) checks cancelled/disabled before "no
     anchor". The Rule puts step 6 (`not_planned_today`) before steps 7–8, and Rust
     `classify_pull_in` agrees.
   - **Fix.** Check no-anchor first.
   - **Test.** `scripts/test-task-status-cycler-project-pull-in.cjs` ≈339 currently
     asserts `disabled` on an empty day. Make PP11 use a day where A is planned, and add
     a vector "no anchor + disabled/cancelled → `not_planned_today`".
3. **P3: cascade without `[id::]`.**
   - **Defect.** In `085-successor-plan.js`, the `closedTaskIds.size === 0` branch
     (≈661–683) returns `applyContext(...)` without `pullIns.cascadePaths`. Pulling in a
     `[?]` R therefore never recovers N's residents.
   - **Fix.** Run the same project-recovery wiring as the id branch (≈697–713), using a
     small shared helper (in 089 if 085 lacks room) so both branches call one path.
   - **Test.** Ctrl+Enter or a pure fixture with a stale `[?]` R and a
     `[?] #prj_is_blocked #hide` resident, where A has no `[id::]`. The resident
     recovers like PP25.
4. **P4: frontmatter `scheduled` validity.**
   - **Defect.** An empty `scheduled:` or a value Rust's `parse_project_schedule`
     rejects (for example `2026-10-20 # note`) is a scan issue in Rust, so there is no
     pull-in. The cycler treats it as no schedule or a valid date (≈122–133).
   - **Fix.** Mirror Rust's acceptance rules exactly.
   - **Tests.** Add vectors for the empty and the commented value.
5. **P5: recognition mirrors Rust, not 080.**
   - **Defect.** `projectSurfaceState` reuses 080's `projectDependencyInfo`, which
     disagrees with sync's recognition.
   - **Fix.** Give 089 its own recognition that mirrors Rust:
     - a missing or empty `status:` is wip;
     - `type:` matches the scalar, flow-list (`["[[project]]"]`) and block-list forms;
     - `^prj` line validity follows `is_valid_prj_task_line`.
   - **Constraint.** Leave 080 and the 6u resident-recovery callers unchanged.
   - **Tests.** One vector per difference.
6. **P6: SUBOPEN parsing fidelity.**
   - `projectMarkerHasOpenChild` must read the marker the way Rust's sub-block does:
     - stop at a blank line;
     - require the bullet content to start with `🧩 **Sub-projects:**` (not `includes`);
     - dedupe children by link name.
   - Keep PP6 green and add a blank-line vector.
7. **P7: copy and row parity.**
   - **Cycler copy.** In `087-successor-place.js` `successorNoticeText` (≈353), use
     `🔓 Surfaced: …` only for a row with an unchanged status and `hide_removed`, which
     is Rust's and nav's "surfaced" rule. Any other unlinked pull-in row (`[?]`→`[ ]`,
     already-planned →`[*]`) uses the existing `🔓 Unblocked: …` / lane copy plus the
     `· project has no open tasks` suffix.
   - **Nav fallback.** Make nav's `unblockedFallbackText`
     (`plugins/bob-navigation-hotkeys/src/285-unblocked-notice.js`) mirror the cycler
     exactly, including the all-project multi-linked case. Pick the cycler's §12.6 copy
     and update whichever side differs.
   - **Nav cleanup.** Remove nav's unused `projectEmpty` argument (≈399).
   - **Row shape.** Pull-in `still_blocked` rows include `project` (N's path), matching
     Rust (Part A step 4).
8. **Mirror R2.** `projectPullInNeedsDependents` also returns true when a candidate R is
   `?` and an open resident has `[dependsOn::]`.
9. **Cleanup.**
   - Drop the unused `after` computed for pull-in marks (089 ≈800–803), the unused
     `dailyAfter` option, and the surface's unused `today` field.
   - Rename `PROJECT_OPEN_MARKS`, which holds the closed marks, to
     `PROJECT_CLOSED_MARKS`.
   - Remove the redundant `[id::]` re-check and the always-true condition in
     `137-plugin-successor-project.js` (≈67, ≈93).
   - Compute `projectSurfaceState` once per note per close (memoize inside the pass).
   - Check the cheap frontmatter type before the whole-note scan-issue pass.
10. **Finish plugins.**
    - Minor-bump task-status-cycler to 1.36.0. If nav changed, also bump
      bob-navigation-hotkeys to 2.29.0 (re-check master first). Update their root
      `README.md` rows.
    - Run `npm run build`, `npm test` and `npm run validate`.
    - Deploy on this host:
      `bob plugins sync -n -r "<opened bob-plugins path>" -p task-status-cycler`, plus
      `-p bob-navigation-hotkeys` if bumped.
    - Commit bob-plugins.

## Part C — install and verify on this host

1. From the fixed bob-cli tree, run `just install` (never `--force`).
2. Run the dry runs `BOB_DIR=~/bob bob projects sync --dry-run` and
   `bob task reconcile --dry-run -f json`. Use `BOB_NOW` if today's daily note is
   missing. They must exit 0 with only explainable changes.
3. On a temporary copy of `~/bob`:
   - Repeat 74.6's live pull-in check with an R whose `#hide` is punctuation-bounded
     (edit only the copy).
   - Confirm that `bob projects sync --dry-run` there names no event for N.
   - Delete the copy.

## Part D — close out epic bob-cli-74 (final step)

1. Add a note to bob-cli-74 listing the repair commits and verification. Use
   `sase bead note bob-cli-74 "..."`, and include the `just check` / `npm test` failure
   sets.
2. Run `sase bead epic-symbols bob-cli-74`.
   - For every listed `--epic-symbol` entry, resolve the symbol: wire it up, privatize
     it, add a non-test pragma, or delete it.
   - Re-key an entry only if a still-open later bead needs the exemption.
   - The landing audit found none.
3. Close the epic:
   `sase bead close bob-cli-74 --note "<verified: phases 74.1–74.6 closed; landing repairs R1–R4/P1–P7 with tests; just check fails only bob-cli-6m; npm test only 6v/6w/3w; installed bob + cycler/nav on apollo; follow-ups bob-cli-75/76/77 filed and 6m/6v/6w +1'd; no integration drift>"`.
   - Never use `--force` merely to make the close succeed.
   - If the close is refused for leftover `--epic-symbol` entries, clean them up and
     close again.
4. Run `just symvision` and confirm the whitelist is clean.
5. Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN path
   shown by `sase bead read bob-cli-74 -r "<why>"`
   (`plan:202610/project_pull_in_line_model.md`, under the plans repo `202610/`
   directory).
6. bob-cli-74 has no `parent_bead`, so there is nothing further to close.
