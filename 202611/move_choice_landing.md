---
tier: tale
size: medium
title: Finish and land the `- ^^` move choice (bob-cli-h0)
goal:
  The `- ^^` Move card and `bob capture -m` promise exactly what the planner then does
  for every row state. The execution half of the contract is pinned by tests, the
  compatibility and stamping docs mention `- ^^`, and epic bob-cli-h0 is closed.
proposed_by: bbugyi200.apollo.bob-cli-h0.land
proposed_with: claude/opus
bead: bob-cli-h0
create_time: 2026-11-04 16:49:22
status: wip
---

- **PARENT:**
  [202611/move_choice.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/move_choice.md)
- **BEAD:**
  [bob-cli-h0](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-h0/README.md)

# Plan: Finish and land the `- ^^` move choice (bob-cli-h0)

Epic `bob-cli-h0` (`plan:202611/move_choice.md`) shipped all eight phases. The CLI
commits are d216f7eb, bfbf482f, abf427c2, cf2e3c9f, and 458b53b9. The Mac commits are
bob-mac-capture 0df42dc, 4906d94, and 0245a75. The land agent verified the work and
found three gaps:

1. Some `move_choice` rows promise something the planner then does not do (MQ17 is
   broken).
2. Answered `-m` runs have no tests.
3. Several doc edits from the contract phase were never made.

This tale fixes those gaps and then closes the epic. The published contract is
`docs/capture.md` → "Choosing parent tasks in" (`#choosing-parent-tasks-in`) and
`docs/reference/json-outputs.md` → "Move-choice results". Where this plan says
"contract", it means those two sections.

Do not touch Bob Mac Capture. It renders Bob's strings, so fixing Bob's copy fixes the
card. Do not edit any memory note; the epic decided `memory_decision_record = no`. Never
edit `Cargo.toml`'s version or `CHANGELOG.md`.

## 1. Make the card's prediction equal the planner's outcome

Builder: `src/native/capture/move_consequence.rs`, with `predict` and `describe`. Row
builder: `src/native/capture/move_choice.rs`, with `build_slot` and `guard_for_row`.
Outcome callers: `src/native/capture/project_note.rs`.

The land agent reproduced each case below. Use a temp vault with
`BOB_NOW=2026-09-20 14:31:07` and `TZ=UTC0`, plus a dry run with no `-m` (the
prediction) against `--dry-run -m <pick>` (the outcome).

| Row                                                                                                     | Predicted today                                                                  | Planner outcome (`moved_tasks[].detail`, links)                         |
| ------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `[?] … [scheduled:: 2026-09-25] ^b`, `^`                                                                | `Moves into cash_x.md · stays Blocked until 2026-09-25`                          | `stays Blocked until Sep 25`                                            |
| same row, `:`                                                                                           | `Moves in · links into today's plan`                                             | `pulls Sep 25 forward · becomes Next · links into today's plan`, 1 link |
| ID-less `[?] #task Idless` with no other blocker, `:`                                                   | `Moves in · links only if Bob can unblock it · adds ^idless`, `adds_link: false` | `links into today's plan · adds ^idless`, 1 link                        |
| `^l` already linked under an unnamed timed Pomodoro (`- [ ] (1400-1500) Work` + `\t- [[cash#^l]]`), `^` | `Moves into cash_x.md`                                                           | detail `" link follows"` (empty name, leading space)                    |
| same row, `:`                                                                                           | `Moves in · becomes Next · links into today's plan`, `adds_link: true`           | `keeps its  link` (double space), 0 links                               |

Fixes:

- **One date format.** Delete the hand-rolled `display_move_date` in `move_choice.rs`.
  Reuse `project_note.rs`'s `display_move_date` (`%b %-d`), at `pub(super)` visibility.
  Feed `own_scheduled` through it, so `predict` and `describe` both read `Sep 25`.
- **Carry the guard's dates.** In `guard_for_row`, map `LinkGuardVerdict::PullForward`
  to `LinkGuard::PullForward { date }`. Today both branches map it to `Link`. The date
  is the row's own `scheduled` value in display form. For `DropDependencies`, set
  `pulled_forward` to that date when the row has a single future own `scheduled` value.
  That is the case where `capture_task_toggle::task_update_has_single_future_scheduled`
  holds and the link step pulls the date forward. The outcome side computes
  `pulled_forward` from the before/after line in `project_note.rs`; match it.
- **ID-less rows get a real guard verdict.** `guard_for_row` returns `StatusOnly` for a
  `[?]` row without a block ID, so the card under-promises. Assess these rows through
  the same `link_guard::assess_link_target` the outcome uses. Two ways to do it:
  - stage the row's `mint` into a scratch copy of the staged parent and assess that ID;
  - or factor the guard so it can assess a task by line.

  Either way, the predicted verdict must equal what the minted run reports. Keep
  `StatusOnly` only for the capture-complete picker, which never scans the vault (§3.5
  of the epic plan).

- **Unnamed Pomodoro links are links.** `build_slot` sets `today_link` from
  `pomodoro_chip.name`, so a chip without a name reads as "not linked". Treat any chip
  as an existing link: `:` must report `adds_link: false` and no `becomes Next`. In
  `move_consequence.rs`, word an empty or absent name the same way in `predict` and
  `describe`:
  - `^` reads `today's link follows` in both (no `its`);
  - `:` reads `keeps today's link`.

  Named Pomodoros keep the existing pinned wording. The detail stays `WORK link follows`
  (pinned by `tests/cli/capture/project_note.rs` around line 1790). The `^` consequence
  stays `its WORK link follows` (pinned by
  `tests/cli/capture/complete_project_task_move.rs`). The `:` copy stays
  `keeps its WORK link`.

- **Compute the guard once per row.** `guard_for_row` is called inside the
  `for sigil in ['^', ':']` loop, so it assesses twice per row. Hoist it above the loop
  (§3.11 of the epic plan: the graph is built at most once per report).
- **Explicit nulls in rows.** The contract's JSON example shows `mint`, `scheduled`,
  `pomodoro`, `locked`, `disabled_reason`, and `answer` as explicit `null`, and
  `block_id`/`section` as null for ID-less or unsectioned rows. `MoveChoiceRow` instead
  omits them with `skip_serializing_if`. Drop `skip_serializing_if` on those eight
  `Option` fields so every documented row field is always present. Keep
  `MoveChoicePomodoro.name` omitted when absent; the contract does not list it as
  nullable. This is additive for clients: bob-mac-capture decodes with
  `decodeIfPresent`.

The one documented asymmetry stays: for a named existing link, the `^` consequence adds
`its` (`its WORK link follows`) while the detail is `WORK link follows`. State it in
`docs/capture.md` next to the sentence "The suffix following the consequence head
matches `project_note.moved_tasks[].detail`". Every other suffix must be byte-identical.

## 2. Pin the execution half with CLI tests

Add tests in `tests/cli/capture/project_note.rs` beside the existing
`capture_project_note_choice_*` tests. Use their temp-vault pattern (`write_file`,
`run_with_stdin`, `BOB_NOW=2026-09-20 14:31:07`, `TZ=UTC0`, `BOB_DAY_FILE`). Add a small
helper that runs `capture -f json [--dry-run]` with repeated `-m` values. Every one of
these behaviours was confirmed by hand before this plan; no test pins any of them today.
Name the tests after behaviours and cite the contract vector IDs in `docs/capture.md`
(its MQ table, not the epic plan's numbering).

1. **As if typed.** One vault answers `- New` / `- ^^` / `- Other` with `-m 4 -m :1`. A
   twin vault authors `- New` / `- :a` / `- ^d` / `- Other` instead. Assert:
   - the parent, the new project note, and the day note are byte-identical across the
     two vaults;
   - `moved_tasks` agree apart from `via` (`"choice"` vs `"bullet"`) and `draft_range`
     (the `^^` range);
   - `-m 3,1` equals `-m 1,3`;
   - numbers and keys (`cash.md#^a`) resolve to the same picks.
2. **Parent order.** Picks given as rows 4 then 1 land in `## Tasks` as New, row 1, row
   4, Other.
3. **Minting.** Picking an ID-less row writes `^<mint>` to the moved task, and `detail`
   ends `adds ^<mint>`. A mint that collides with an authored ID in the new note takes
   the next free suffix. A descendant-ID collision refuses `move_task_id_collision` and
   writes nothing. Dry-run JSON equals real-run JSON apart from `dry_run`.
4. **Offers.** `@cash:x@` + `- ^^` + `- New`:
   - with `-m 1`, row 1 is never in `pull_in` offers and `New` is;
   - with `-m 1 -p all`, row 1 stays unlinked and `New` links;
   - with `-m :1 -p all`, both link.
5. **Dated project.** With `s:3`, a `[?]` row answered `-m :<n>` moves in unlinked and
   its detail is `stays unlinked until the project starts Sep 23`. A Ready row is
   refused `move_choice_refused` (the refusal half is already tested).
6. **`#name` satisfied.** `@cash^x@#work` + `- ^^` with `-m :1` succeeds and links under
   WORK.
7. **Prediction equals outcome (MQ17 through `^^`).** One table-driven test runs every
   row state in the contract's consequence table, both sigils:
   - Ready, Next, and In Progress unlinked;
   - linked today under a named Pomodoro, and under an unnamed one;
   - Blocked only by its own open prerequisites;
   - Blocked only by its own future date;
   - Blocked for another reason (guard consumes; e.g. a prerequisite in another note
     plus a second `scheduled` field, or an inherited project block);
   - `[?]` into an `s:N` project;
   - ID-less, both Ready and `[?]`.

   Assert three things:
   - the dry-run row's consequence minus its head (`Moves into <dest>` / `Moves in`)
     equals the answered run's `detail`, with an empty suffix equal to a `null` detail
     and the one `its` asymmetry allowed;
   - `adds_link` is true exactly when the run's `project_note.task_links[]` gained an
     entry for that block;
   - `move_choice.slots[0]` flips to `answered: true` with the right `moved_count`,
     `linked_count`, and per-row `answer`.

8. **Rollback.** A capture with `- ^^` answered by `-m` leaves every file byte-identical
   when the source replace fails. Write it as a unit test in
   `src/native/capture/tests/commit.rs` that drives a full capture request (the request
   struct already has a `move_in` field). Use the existing `RENAME_FAIL_TARGET` hook the
   way `move_destination_is_rolled_back_when_source_replace_fails` and
   `transfer_merge_preserves_engine_order_and_rolls_back_failed_source` do. Also assert
   destination-before-source order where the existing harness exposes it.

Also add a `move_consequence.rs` unit test for the unnamed-link wording and the
`PullForward`/`DropPrerequisites` date plumbing.

## 3. Finish the contract docs

Read `sase memory read docs.md -r "<why>"` (via `/sase_memory_read`) before editing
`docs/`.

- **`docs/reference/compatibility.md`, "Capture-language revision":** append a
  grammar-11 paragraph in the style of the grammar 8–10 paragraphs. Cover:
  - a whole first-level `- ^^` bullet in a sealed project-note item is a move-choice
    slot, not a literal task named `^^`;
  - `capture-parse` reports `project_task_move_choice_marker` and
    `sub_bullet_move_choices`, keeps the parse complete, and leaves the
    `sub_bullet_moves` entry null so older clients never see an unfinished `- ^`;
  - `capture-complete` adds `action_continuation_keys: ["^"]` on a lone `- ^`;
  - `bob capture` gains `-m/--move-in`, and every capture JSON carries
    `move_choice.supported: true` for capability detection;
  - a four-row Mac × bob table from the epic plan §3.10: new×new full flow; old Mac ×
    new bob refuses `move_choice_required` write-free; new Mac × old bob warns
    `This bob predates - ^^; it would capture a task named ^^ · update bob` on the first
    ⌘↩ and sends `--move-in` only after a preview advertised support; old×old is
    unchanged.

  Do not describe grammar 12 or change the lead sentence's `grammar: 10` number; that
  belongs to epic bob-cli-h2 (noted on that bead).

- **`docs/task-move.md`, after the "Capture project-note move bullets" paragraph:** one
  sentence. `- ^^` picks (answered with `-m`) expand server-side into the same move
  bullets at the `^^` slot in parent order, and ID-less picks gain their minted ID in
  the same guarded batch.
- **`docs/freshness.md` §5 "Who stamps", `bob capture` row:** the move-bullet clause
  becomes "each open task a project-note capture move bullet or `- ^^` pick moves (same
  stamp as Ctrl+Shift+M)".
- **`docs/capture.md`:**
  - In "Offer a new project's tasks", add one sentence: tasks moved in through a `- ^^`
    answer are never offered as successors, whatever their sigil, while authored `- ^a`
    rows and the note's new tasks still are.
  - In the consequence table, change the `^` cell of "Blocked only by its own future
    date" to `Moves into cash_goog_exit.md · stays Blocked until Nov 9`, matching the
    planner. Add the unnamed-link wording from step 1, and add the `its` asymmetry
    sentence.
- **`docs/reference/json-outputs.md`:** add one sentence saying every listed row field
  is always present, with `null` where it does not apply. That matches the explicit-null
  fix.

If `bob capture --help` text changes (it should not), run `just docs-gen` and update
`tests/fixtures/help/*`.

## 4. Verify

- `cargo fmt --all -- --check` must pass.
- `cargo clippy --workspace --all-targets --all-features -- -D warnings` must report
  nothing in files this tale touches. On master `029b4266` it already fails with 8 deny
  errors in `src/native/capture_language/editor_pomodoro.rs` and `past.rs`. Those belong
  to active epic `bob-cli-h2` (noted there). So `just check` stops at clippy until h2
  fixes them; run its test stage directly.
- `cargo test --workspace --no-fail-fast`: every new and existing capture test must
  pass. On master the only known failures are the two `highlights::overwrite_local`
  tests (`bob-cli-h6`, owned by epic `bob-cli-h8`). If those or the h2 clippy errors are
  still present, report them and do not fix them here. A `just check` pass with a
  `just check-full` failure is a test-infrastructure bug, not remaining epic work; do
  not run `just check-full`.
- `just docs-check` must pass.
- `just symvision`, if available.

## 5. Close out the epic (final step)

The epic's land agent already recorded the follow-up triage on `bob-cli-h0` (lander
note). It also already:

- regenerated `tests/fixtures/note_pull/np-move/{preview,picked}.json` and closed
  `bob-cli-h5`;
- recorded the clippy and compatibility-doc issues on `bob-cli-h2`.

Do not redo those. After the code above is done and verified, in this same turn:

1. Run `sase bead epic-symbols bob-cli-h0`. At planning time it listed none. If any
   `--epic-symbol` entry appears, resolve it per the Symvision epic-whitelist policy:
   wire it up, privatize it, add a non-test pragma, or delete it. Re-key a Justfile line
   to another bead only if a still-open later bead needs the exemption.
2. Close the epic: `sase bead close bob-cli-h0 --note "<verification>"`. The note should
   say:
   - all 8 phases verified against the source;
   - the MQ17 prediction fixes (date format, pull-forward and drop dates, ID-less guard,
     unnamed links, single guard per row, explicit nulls);
   - the execution tests added;
   - the docs completed;
   - the commands run and their results, including the known non-h0 failures (h2 clippy,
     h6 highlights);
   - that the Mac walkthroughs are left to Bryan per the h0.8 note.

   Never use `--force` to make the close succeed. If the close is refused for leftover
   `--epic-symbol` entries, finish step 1 and close again.

3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN path
   `sase bead read bob-cli-h0 -r "Need the plan path"` prints: `202611/move_choice.md`
   under `sase repo path plans`. It currently reads `status: wip`.
5. `bob-cli-h0` has no `parent_bead`, so there is no parent to close.
