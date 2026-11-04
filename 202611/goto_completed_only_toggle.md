---
tier: tale
title: "Go To picker: Command-X switches between open and completed tasks"
goal:
  In Bob Mac Capture's Go To picker, Command-X flips between open tasks only and
  completed tasks only, with bob's capture-complete -C returning just the closed rows.
size: medium
proposed_by: bbugyi200.athena.186
proposed_with: claude/opus
create_time: 2026-11-04 14:23:04
status: wip
---

# Go To picker: Command-X switches between open and completed tasks

## Context

Epic `bob-cli-h3` (plan `plan:202611/goto_completed_toggle.md`, now closed) added
**Command-X** to Bob Mac Capture's Go To picker (`/`, or `\` on a blank draft). Today
the toggle switches between:

- **off:** open tasks only, from `bob capture-complete --all-tasks … -- '/'`;
- **on:** open tasks **plus** a trailing **Completed** section, from the same call with
  `-C`/`--closed-tasks`.

Bryan wants the on state to show **only** completed tasks. Command-X should flip between
two disjoint views: open tasks only, and completed tasks only.

### Design (decided here)

- **Bob owns the scope, not the Mac.** `capture-complete -C/--closed-tasks` changes from
  "append closed rows after the open rows" to "return the closed rows instead of the
  open rows". The Mac keeps sending the same flag and renders what comes back. This
  follows `decisions:mac-capture-is-a-thin-client` ("new capture behavior lands in
  bob-cli first"). It also removes wasted work: in closed mode bob no longer runs the
  open-task discovery. The rejected alternative was a Mac-only change that fetches
  open+closed and drops the open rows. That would leave bob's flag with a mode no client
  wants.
- **Keep the flag name.** "closed tasks" still reads correctly. The Mac's older-bob
  detection keys on `--closed-tasks`, and the flag has never shipped. The latest tag is
  `v0.4.0` (2026-10-27), and `git tag --contains f54a352c` is empty. So redefining it
  now breaks no released contract.
- **Unchanged from h3:** closed rows still cover the whole vault, including `done/`
  archives, ref notes, and daily notes, and still include canceled tasks (h3 decisions
  `completed_scope = vault`, `canceled_rows = closed`). They keep the same row shape and
  the same order: newest close date first, undated rows last, ties by `note_path` then
  `line`, and score first for a non-empty query. Each fresh Go To session still starts
  in the open view, and Escape plus the chip reopen keep the session's choice (h3
  `sticky_toggle = session`). The Mac scope state machine (`CaptureTaskGotoScopeState`,
  the 70c0839 race repair) does not change.
- **Mac copy for the new semantics:**
  - Scope capsule: `Go To` when off, `Go To · Completed` when on (was
    `Go To · + Completed`).
  - Footer hint: `⌘X Show Completed` when off, `⌘X Show Open` when on (was
    `Hide Completed`).
  - Hint accessibility label: "Command X to show completed tasks" when off, "Command X
    to show open tasks" when on.
  - Status after a toggle: `Showing completed tasks, N tasks` or
    `Showing open tasks, N tasks` (was `Hiding completed tasks, N tasks`).
  - Loading status: `Loading completed tasks…` or `Loading open tasks…`, chosen by
    `request.includesClosedTasks`. Today it always says "completed".
  - Unchanged: the older-bob hint `Update bob to show completed tasks`, the chip-refetch
    failure text, and the empty-state wording.
- **Keep the Swift identifiers** (`includesClosedTasks`, `includeClosedTasks:`,
  `appliedIncludesClosedTasks`, about 130 sites). Renaming them would churn the
  race-tested state machine for no behavior gain. Instead, update their doc comments to
  say `true` means the Go To list is the closed-only scope.
- **Keep the Mac's mixed-list presentation code** in `TaskLinkPickerPresentation.swift`:
  the trailing Completed section and open-before-completed ranking. With the new bob it
  receives a single-group list, so the grouped view is one **Completed** section in
  bob's order. With an interim bob built between f54a352c and this change it still
  renders sanely. Do not add a Mac-side group filter; bob is the authority.
- **Deploy order:** land and push the bob-cli commit before the Mac commit.

## Part 1 — bob-cli (this repo)

1. `src/native/capture_complete/candidates.rs::task_goto_candidates`:
   - When `closed_tasks` is true, return early from a new private helper (for example
     `closed_task_goto_candidates(bob_dir, query)`). Move the existing closed-row code
     into it: `capture_dependency_tasks::discover`, the `!task.open` filter,
     `match_score` over `dependency_search_fields`, the
     score-then-`closed_goto_order_key` sort, and the `TaskGotoCandidate` mapping. It
     returns only those rows plus the vault catalog's warnings.
   - Closed mode must not call `capture_link_tasks::discover`.
   - The open path, without the flag, stays byte-identical. Drop its now-dead
     warning-merge/extend tail.
   - Rewrite the doc comment. Behind `closed_tasks`, the context lists closed rows from
     the vault-wide catalog **instead of** the open rows, with the same filtering,
     ordering, and row shape described today.
2. `src/native/capture_complete/model.rs`: update the `TaskGotoCandidate` doc comment
   ("plus optional trailing closed rows … behind `--closed-tasks`"). Say that
   `--closed-tasks` returns closed rows only, and that each row is either an open row
   with `note_kind` or a closed row with `group: "completed"`.
3. `src/native/capture_complete/cli.rs`:
   - `closed_tasks_arg()` help: "List done and canceled tasks from the whole vault
     instead of open tasks (task_goto context only)".
   - Long-about sentence: replace "Pass -C/--closed-tasks to append done and canceled
     tasks from the whole vault as a trailing `completed` group, newest close date
     first." with one sentence saying -C/--closed-tasks lists only done and canceled
     tasks from the whole vault, instead of the open rows, as `completed` rows with the
     newest close date first.
   - Leave the example and the alphabetical arg order alone.
4. `tests/cli/capture/task_goto.rs` (uses `write_closed_example`: 8 open rows and 5
   closed rows):
   - Rename `complete_goto_closed_tasks_appends_newest_first` to something like
     `complete_goto_closed_tasks_lists_only_closed_newest_first`. For both `-C` and
     `--closed-tasks`, assert exactly 5 candidates, all `group: "completed"`, with no
     open row. The texts must be
     `["Shipped feature", "Ref done", "Canceled idea", "Old fix", "Dropped idea"]`. Keep
     the existing per-row field assertions (`closed`, `status_type`, `route` locator,
     `note_path`, absolute `target`, no `note_kind`, null `scheduled`/`pomodoro`, empty
     `replacement`).
   - Rename `complete_goto_closed_query_filters_with_open_first` (for example to
     `complete_goto_closed_query_filters_closed_rows_only`). With `/fix` and `-C`, every
     candidate is `completed`, `Old fix` is present, and the open `Fix deep bug` is
     absent.
   - `complete_goto_closed_human_lines_show_closed_date`: `13 candidates` becomes
     `5 candidates`. Also assert that an open row's text, such as `Fix deep bug`, is
     absent.
   - Keep `complete_goto_without_flag_omits_closed_rows` and
     `complete_task_link_ignores_closed_flag` as they are. They must still pass.
   - The `closed_goto_order_key` unit tests in `candidates.rs` need no change.
5. `docs/capture.md`, Go To section, the paragraph starting "Pass `-C`/`--closed-tasks`
   to append …":
   - Say the flag lists done and canceled tasks from the whole vault **instead of** the
     open rows. Drop "after the open rows" and "open matches always come first".
   - Say warnings come from the vault catalog.
   - Keep the catalog, order, and row-shape text.
   - Adjust the earlier field list where it says "optional `note_kind` (present on open
     rows, omitted on closed rows)" only if the wording now misleads.
6. Run `just docs-gen` to refresh `docs/reference/cli/capture-complete.md`.
   - Baseline: `just docs-check` on master already fails generated drift for
     `docs/develop/decisions.md`, a known, unrelated issue tracked on `bob-cli-b6`. If
     `docs-gen` also rewrites that page, revert it with
     `git checkout -- docs/develop/decisions.md` so this commit stays scoped.
   - Then run `just docs-check`. Confirm the only failure, if any, is that pre-existing
     page.
7. Gates:
   - `cargo clippy --all-targets`
   - `cargo test --test cli task_goto`
   - `cargo test capture_complete`
   - a full `cargo test` if time allows
   - Format only the files you touched (`rustfmt --edition 2024 <files>` or
     `cargo fmt -- <files>`). `cargo fmt --all -- --check` already fails on master in
     `src/native/capture_language/{draft,editor_pomodoro}.rs`, which belong to active
     epic `bob-cli-h2`. Do not reformat those files.
   - Smoke test against the real vault, read-only:
     `bob capture-complete -C -c 1 -f json -- '/'` (via `cargo run --release --`) must
     return only `group: "completed"` rows. Record its wall time next to the no-flag
     call.
8. Commit:
   - Subject:
     `feat(capture): list only closed tasks under capture-complete -C for task_goto`.
   - Re-check `git tag --contains f54a352c` at commit time. If it is non-empty (the
     release train shipped the old semantics meanwhile), use `feat(capture)!:` per
     `decisions:bob-cli-releases-ride-a-daily-train`.
   - Do not touch `CHANGELOG.md` or the `Cargo.toml` version.

## Part 2 — bob-mac-capture (linked repo)

Open it with `/sase_repo` (`sase repo open bob-mac-capture -r …`). Read its `AGENTS.md`
first: compile rules, `.swift-format` (format only the lines you change), the docs gate,
and the scratch-copy Mac typecheck.

1. `Sources/CaptureCore/CapturePickerPresentation.swift`:
   - `scopeCaption(includesClosedTasks:)` returns `Go To · Completed` for `.taskGoto`
     when the flag is true.
   - `coreKeyHintItems`, `.taskGoto` case: the toggle hint becomes
     `("⌘X", includesClosedTasks ? "Show Open" : "Show Completed")`.
   - `keyHintsAccessibilityLabel(includesClosedTasks:)`: "Command X to show open tasks"
     when on, "Command X to show completed tasks" when off.
   - Keep every other source's hints byte-identical.
2. `Sources/BobMacCapture/CapturePanelModel.swift`:
   - `startGotoClosedTasksFetch`: announce `Loading completed tasks…` or
     `Loading open tasks…` from `request.includesClosedTasks`.
   - `applyGotoClosedTasksSnapshot`: announce `Showing completed tasks, N tasks` or
     `Showing open tasks, N tasks` from `state.appliedIncludesClosedTasks`.
   - Selection logic stays: keep the row if its key still exists, otherwise take the
     first row. Because the two views are now disjoint, a toggle normally lands on the
     first row.
3. Doc comments only: in `Sources/CaptureCore/CaptureTaskGotoScopeState.swift`,
   `Sources/BobMacCapture/CapturePickerState.swift`, and
   `BobProcessClient.captureComplete(_:cursor:includeClosedTasks:)`, say that `true`
   requests or describes bob's closed-only Go To list. No identifier renames.
4. Fixture `Tests/Fixtures/task-goto-complete-closed.json`: drop the 8 open rows so it
   matches the new bob output. The candidates are just the 3 `completed` rows
   (`done/old.md`, the canceled `mac_inbox.md` row, and `References/old.md`); keep the
   envelope fields as they are. `fake-bob` needs no change, since it already serves this
   file for `--closed-tasks`.
5. Tests: update expectations for the closed-only fixture.
   - `Tests/BobMacCaptureTests/CaptureTaskGotoPanelTests.swift`:
     - Counts of `11` become `3`.
     - `Hiding completed tasks, 8 tasks` becomes `Showing open tasks, 8 tasks`.
     - `Showing completed tasks, 11 tasks` becomes `Showing completed tasks, 3 tasks`.
   - Rework `testCommandXShowsAndHidesCompletedTasksWhileKeepingFilterAndSelection` into
     a test that checks all of the following:
     - The `fix` filter text is kept across both toggles.
     - The on view has `done/old.md|12:deadbeef` and no open rows.
     - Selection falls back to the first row of each new view.
     - Toggling off restores the open rows without the completed row.
     - Both argv lines are recorded.
   - `testLatestOnFailureFromCompletedSnapshotPreservesCompletedRows` and the other race
     tests keep their logic; only their counts and strings change.
   - Grep the test targets for `Hide Completed`, `+ Completed`,
     `Hiding completed tasks`, and `candidates.count == 11`, and update every hit,
     including `CapturePickerDesignTests.testKeyHintsDocumentPickerKeyboardContract` and
     any caption assertions.
   - `TaskGotoPickerPresentationTests` use hand-built mixed lists that test generic
     index behavior. Keep them, and add one test: a completed-only candidate list groups
     into a single `Completed` section in bob's order, and filters to completed matches
     only.
   - Async tests must never await an untimed continuation.
6. Docs:
   - `docs/task-pickers.md`, Go To section:
     - Rewrite the Command-X paragraph. Command-X switches between open tasks and
       completed tasks only. Completed tasks come from the whole vault catalog (done
       archives, reference notes, daily notes; done and canceled), shown as one
       **Completed** section in bob's newest-close-date order with undated rows last.
     - A filter ranks by bob's match score. Each row shows its status glyph, locator,
       and `Closed Oct 3` date capsule.
     - The capsule reads **Go To · Completed**. Session default, Escape/chip reopen, and
       older-bob behavior stay as written.
     - Change the key line to "Command-X Show Completed / Show Open".
     - Also delete the duplicated `## Go To Picker` heading at the top of that section.
   - `docs/keyboard.md`, "Capture picker keys": "Command-X toggles completed tasks"
     becomes "Command-X switches between open and completed tasks".
   - `docs/protocol.md`:
     - In the required-commands bullet, say "closed rows only, instead of open rows,
       when Go To requests them".
     - In the degradation-table row, "Command-X adds closed rows" becomes "Command-X
       switches to closed rows".
   - Run `just docs-check`.
7. Gates:
   - Run `just portable-check`. If `swift` is missing locally, run it on athena.
   - Best effort on the Mac: rsync a scratch copy without `.build`/`.git`, run
     `Scripts/bundle.sh` and `just typecheck-tests`, then delete the copy. Never use
     Bryan's checkout or `just install`.
   - If the Mac is offline, say in the final report that the macOS targets are
     unverified.
8. Commit after the bob-cli commit, with subject
   `feat(capture): make Go To Command-X switch to completed tasks only`.

## Acceptance

- `bob capture-complete -C -c 1 -f json -- '/'` returns only `group: "completed"` rows,
  newest close date first. Without `-C`, the output is byte-identical to before.
- In Mac Capture, `/` shows open tasks. Command-X shows only completed tasks under
  `Go To · Completed` with the `⌘X Show Open` hint, and Command-X again returns to open
  tasks only.
- Filter text survives both toggles. Fresh sessions start open. Escape and chip reopen
  keep the session's choice. An older bob still shows the update hint.
- No memory note needs changing: none describe Go To or `--closed-tasks`.
