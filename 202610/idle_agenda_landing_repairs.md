---
tier: tale
size: medium
title:
  "Idle agenda landing repairs: fix the verified agenda bugs in bob-mac-capture and
  bob-cli, then close epic bob-cli-66"
goal:
  The idle Pomodoro agenda shows the current snapshot and refresh state. The panel opens
  centred where the compact bar always opened. The bob --tasks contract matches the epic
  spec. Epic bob-cli-66 is then closed with its plan file marked done.
proposed_by: bbugyi200.athena.bob-cli-66.land
bead: bob-cli-66
create_time: 2026-10-09 22:12:58
status: wip
---

- **PARENT:**
  [202610/idle_capture_pomodoro_agenda.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/idle_capture_pomodoro_agenda.md)
- **BEAD:**
  [bob-cli-66](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-66/README.md)

# Idle agenda landing repairs (epic bob-cli-66)

## Why

Epic `bob-cli-66` (plan `plan:202610/idle_capture_pomodoro_agenda.md`, the "epic plan"
below) shipped `bob capture-pomodoros -t/--tasks` in bob-cli and the idle agenda in
**bob-mac-capture**. All 7 phases are closed, bob-cli `just check` is green, and
bob-mac-capture CI was green at `f8c9c28`. The land agent's source audit found
epic-caused bugs that CI did not catch. The model tests call `refreshAgendaPlan(today:)`
by hand, and no test checks the panel's horizontal position. This tale fixes those bugs
and then closes the epic. **There is no other agent after you: you finish the landing
(final section).**

Read the epic plan's "Design specification" §1–§10 for the intended behavior; it is the
spec for everything below. Read `sase bead read bob-cli-66 -r "<why>"` for the epic's
notes (the two `LAND VERIFICATION` / `LAND TRIAGE` notes summarize this audit).

## Ground rules

- **bob-cli** changes happen in this project's workspace. They are committed for you
  when your turn ends; do not commit bob-cli yourself.
- **bob-mac-capture** is the linked repo. Open it with
  `sase repo open bob-mac-capture -r "<reason>"` and work only in the path it prints.
  Read its `AGENTS.md` if present. This plan explicitly authorizes you to commit to
  bob-mac-capture `master` with `/sase_git_commit` from that checkout, using
  conventional headers (`fix(agenda): …`, `test(agenda): …`). Run `git pull --rebase`
  first. `f51cdc1` (bob-cli-5y.12, File-under picker) landed after the epic and edits
  `CapturePanelModel.swift` and `CapturePanelView.swift`.
- **Thin client** (`decisions:mac-capture-is-a-thin-client`): the app never reads the
  vault. Every fact comes from bob.
- **No macOS toolchain here.** GitHub Actions (`macos-26`) is the only compiler for
  AppKit targets. Write conservatively: Swift 5 mode, `@MainActor` on AppKit types,
  long- stable APIs, swift-format default style (4-space indent, ≤ 100 columns), no long
  `+` / `??` chains. For CaptureCore changes, run Linux
  `swift test --filter CaptureCoreTests` on a scratch copy of the tree under `/tmp`
  first.
- **CI loop** (bob-mac-capture): after each push, find the run with
  `gh run list -R bobs-org/bob-mac-capture --workflow CI --commit <sha> --json databaseId`.
  Watch it in the foreground with
  `gh run watch <id> -R bobs-org/bob-mac-capture --exit-status`, using an explicit
  timeout of at least 60 minutes. On failure, read
  `gh run view <id> -R bobs-org/bob-mac-capture --log-failed` and grep for ` error:`.
  Fix forward until green.
  - Three known pre-existing timing flakes are tracked elsewhere:
    `CapturePanelModelTests.testStartPendingListPreviewsTrimmedDraftWithStartDisabled`
    (bob-cli-4k), `RefsPanelModelTests.testRefreshReordersFromNewDataKeepingSelection`
    (bob-cli-61), and `RefsLibraryTests.testTriggersDuringRefreshRunExactlyOneFollowUp`
    (bob-cli-67).
  - If those are the only failures and every agenda suite passes, rerun the failed job
    (`gh run rerun <id> --failed -R bobs-org/bob-mac-capture`, up to twice). Do not edit
    those tests.
  - Record the green run URL and SHA in your close note.
- **Pixels.** After the final green run, download `render-fixtures`
  (`gh run download <id> -R bobs-org/bob-mac-capture -n render-fixtures -D <tmpdir>`).
  Open the `agenda-*` PNGs with the Read tool and check them against epic plan §6. In
  particular, check that a folded header with session notes shows its time or
  `= starts it` normally, with the notes chip as its own capsule, and that no `+0 lines`
  chip appears.
- **Fixtures stay synthetic.** Signposts carry names and counts only.

## Part A: bob-cli (`src/native/capture_pomodoros.rs`, `src/native/capture_pomodoros_agenda.rs`)

A1. **`starts_at` / `ends_at` are always present with `--tasks`.** Epic plan §1 types
them as `"YYYY-MM-DDTHH:MM"` or null. Today `CapturePomodorosOutputEntry`
(`capture_pomodoros.rs` ~:206-209) skips them when `None`, so untimed entries lack the
keys.

- With `--tasks`, emit both keys on every listed entry (value or `null`).
- Without `--tasks`, keep them absent so the default output stays byte-identical. For
  example, use `Option<Option<String>>` with the outer `None` meaning "no `--tasks`".
- Regenerate the goldens with `BOB_UPDATE_GOLDENS=1`.
- Update the `docs/capture.md` `--tasks` paragraph ("…timed entries, `null` otherwise").

A2. **Human header separator.** `human_success` (~:855-870) prints
`· Fri 2026-10-09 2 done (2h 0m)`. It must print `· Fri 2026-10-09 · 2 done (2h 0m)`
(epic plan §1): add the second `styler.separator()`.

- Unresolved rows print the raw link twice, because the warning already starts with it
  (~:1043-1050). Print the link once.
- Add a CLI test of plain (no-color) `--tasks` human output that asserts the header
  suffix, a role-badged header, an item row, a `!` warning row, and `✓ N done`.

A3. **Nested log markers keep their own kind.** `tag_log_subtrees`
(`capture_pomodoros_agenda.rs` ~:966-980) overwrites a marker line's `log` with the
enclosing log's kind before pushing it. A Schedule log under a nested task inside a Work
log is therefore tagged `work`. Save the marker's own parsed kind before inheriting,
then push that kind. Add a unit test with that shape (Work log → nested
`- [ ] #task inner` → `**Schedule log**` → entry), asserting the schedule marker and
entry are tagged `schedule`.

A4. **`completed_summary.minutes` reuses the duration helper.** Epic plan §1 says to
reuse the range-duration helper in `capture/pomodoro_adjust.rs` and widen its
visibility. `range_minutes` (~:699-706) recomputes end − start from the normalized
`HHMM-HHMM` and ignores a recorded `[t:: Nm]` or legacy ⏱ duration.

- Compute each completed entry's minutes the way the close planner does:
  `adjustment_duration_for_range` / `duration_from_range_text` (~:723-741, widen to
  `pub(crate)` as needed) on the entry's raw ledger range text, falling back to the
  normalized span.
- Add a test: `- [x] (**08:00 - 09:00** [t:: 25m])` counts 25.
- Update the docs sentence for `completed_summary`.

A5. **Coverage gaps.** Add a focused test with a non-empty `ledger_notes` on a resolved
item, and one with an open entry linking a DONE task. Do not change the entries inside
`agenda-current.json`, because the Mac presentation tests and fixtures depend on its row
shape; A1's key additions are the only golden changes.

A6. **Small cleanups.**

- Remove the `let _ = offset;` (~:520) and `let _ = strike_spans;` (~:771) no-ops and
  their dead bindings.
- Cache unreadable notes too, so each distinct note is read once (the cache's `Option`
  should hold `None` for unreadable).
- Keep one warning-bounding helper: either reuse `bounded_warning` in the agenda module
  instead of its duplicate `bounded`, or make `bounded_warning` private again.
- Do not do perf refactors; release timings are 4–9 ms.

Run `just check` (not `just check-full`). Then run `bob capture-pomodoros -t` and
`bob capture-pomodoros -t -f json` against the live vault (read-only) to eyeball the
header and the null keys.

## Part B: bob-mac-capture

Fix in this priority order. B1–B3 are the must-fix bugs.

B1. **The agenda renders the previous snapshot, and status changes never re-plan.** In
`CapturePanelModel` (the store binding near `agendaSnapshotCancellable`, ~:950-971), the
`$snapshot` sink ignores its value and calls `refreshAgendaPlan()`. That function reads
`agendaStore?.snapshot` and `agendaStore?.status` (~:476-511), but `@Published` emits in
`willSet`, so it sees the old values.

- After launch, the first snapshot is planned against nil ("Loading today…"), and the
  show-time `.unchanged` refresh never corrects it.
- Each later snapshot renders the one before it.
- `status` has no subscriber at all, so the stale marker never appears in production
  (`finishFailure` only sets `status = .stale`). Recovery never clears it.
- Old bob ends on "Loading today…" instead of hiding the agenda: `snapshot = nil` fires
  the sink while `status` is still `.loading`.

Fix:

- Subscribe to `$snapshot.combineLatest($status)` (or have the store publish one atomic
  state value).
- Plan from the **emitted** values: cache them on the model (for example
  `agendaSourceSnapshot` / `agendaSourceStatus`), and have `refreshAgendaPlan` read the
  cache instead of the store. Budget, width, expansion, and countdown re-plans then use
  the same source.
- Reset expansions, the measurer's snapshot note, and the pinned planning day only when
  the emitted snapshot differs from the last one planned.
- In `CaptureAgendaStore.finishClientError`'s `.unsupportedOption` branch (~:258-272),
  check that the generation is still current **before** mutating `capability`,
  `snapshot`, or `status`. A late result from an old executable after Recheck Bob or
  reset must be discarded.
- **Tests** must not call `refreshAgendaPlan` by hand. Drive the real store with fake
  bob: switch `FAKE_BOB_AGENDA_FIXTURE` between refreshes, or use an existing store test
  seam, then assert:
  - the model's presentation follows the latest snapshot;
  - a failure after a good snapshot shows the stale marker, and the next success clears
    it;
  - unsupported leaves `agendaPlan == nil` (compact bar).
- Update the existing `CaptureAgendaModelTests` that relied on manual re-plans.

B2. **The panel opens at the screen's left edge, below where the compact bar used to
open.** `CapturePanelController.placePanelAtEyeLine` (~:420-468) replaced the show-time
`panel.center()` but sets only y and keeps `panel.frame.origin.x`. That x starts at 0
because `makePanel()` creates the panel at origin `.zero`, and nothing else sets it.
`CapturePanelPlacement.eyeLineTop` (CaptureCore) also uses the exact vertical centre.
AppKit's `center()` puts windows "somewhat above center", so the bar sits lower than
before. That contradicts epic plan §5: the compact bar must sit exactly where it does
today.

Fix by deriving the eye line from AppKit itself:

- When the per-visible-frame cache misses, size the hidden panel to the compact content
  height, call `panel.center()`, and read the resulting top (`frame.maxY`). Cache that
  top per visible frame in `CapturePanelPlacement`; adapt its API so it stores a
  provided top instead of computing one, and update its CaptureCore tests.
- On every show, set the target content size and the origin. Use the horizontally
  centred x (from `center()`, or `visibleFrame.midX − frame.width / 2`) and y = top −
  frame height.
- Confirm this runs before the panel is ordered front, so nothing flickers.
- Keep the existing sizer clamp, so typed previews still grow downward and slide up only
  at the bottom edge.
- Add controller tests:
  - after show, the panel's `frame.midX` equals the visible frame's `midX` (± 1 pt);
  - the compact top equals what `center()` gives the compact panel;
  - the top is identical across shows with and without the agenda (keep the existing
    eye-line tests).
- Fix the README wording if it describes the eye line differently.

B3. **The close-comma count stays nil after a transient failure.** `finishFailure` nils
`currentTaskLinkCount` but keeps `lastBytes`. The next byte-identical refresh takes the
`.unchanged` branch (~:220-225), which never restores the count. In `.unchanged`, when
the last good snapshot is current, re-publish its count (for example through the
idempotent `publish(_:)`). Add a store test: fail, then an unchanged refresh brings the
count back.

B4. **The Settings diagnostic lags one change behind.** In `AppDelegate` (~:105-124),
the `$status` and `$lastRefreshedAt` sinks call `diagnosticLine()`, which reads the old
values because the sinks fire in `willSet`. Make the diagnostic a pure function of
`(status, lastRefreshedAt)` (a static on the store is fine) and feed it the emitted
values through `combineLatest`. Unit-test the function: unsupported, stale or failed,
and last refreshed.

B5. **A bob settings change never resets the store.** `observeBobSettings`
(`AppDelegate` ~:687-699) `combineLatest`s two `dropFirst()` publishers, so it fires
only after both settings have changed. Even then, the store keeps its old client and
vault path.

- Merge the two publishers (mapped to `Void`) and debounce them.
- Then run the same reconfiguration path as `recheckBob()`: configure the process
  client, hand it to the panel model and the agenda store, restart the vault watcher if
  recheck does, and reset the store. Check first whether some other path already
  reconfigures the client on these settings, and do not duplicate it.

B6. **Relevance filter directory rule.** `CaptureAgendaRefreshFilter.isRelevant`
(~:75-80) applies "a directory was renamed or removed" to the batch-wide OR of flags. So
`.git/rebase-merge/` going away during a vault git sync, or any `.git` directory event
in the same debounce window, spawns a refresh.

- Apply the directory rule only when the batch contains at least one path under the
  vault root with no dot-prefixed component. Per-path flags in `VaultChangeBatch` are
  also fine if cleaner.
- Add table cases: a `.git/...` directory removal is irrelevant; a visible folder rename
  is relevant.

B7. **Two visual bugs.**

- A task with nothing hidden gets a `+0 lines` chip (`CaptureAgendaPresentation`
  ~:778-785; `agenda-current`'s second task hits it). When the hidden count is 0, show
  no chip, no chip accessibility label, and nothing to expand.
- `CaptureAgendaFitPlanner.headerWithNotesChip` (~:306-327) appends the notes chip to
  the header's trailing accessory text, and `headerAccessoryView` (`CaptureAgendaView`
  ~:715-726) then draws the whole string as one chip. That puts the time, the countdown,
  overdue orange, and the coloured `= starts it` inside a `· 2 notes` capsule.
  - Give `CaptureAgendaRow` a separate chip field. The header keeps its normal styled
    trailing accessory, and the notes chip renders as its own capsule button.
  - Include the chip field in the row key.
  - Update the presentation, planner, and height-consistency tests.

B8. **Spec-compliance nits** (do all; each is small):

- Now-card wash: 0.12 under Increase Contrast
  (`@Environment(\.colorSchemeContrast) == .increased`), 0.06 otherwise
  (`CaptureAgendaView` ~:322; epic plan §6).
- Warning rows use `lineLimit` 1, not 2 (`CaptureAgendaPresentation` ~:673; epic plan §4
  "always one line").
- Clearing the draft to blank always triggers one background revalidation, not only
  while a dim-hold was active (`noteAgendaDraftCleared`, `CapturePanelModel` ~:417; epic
  plan §7).
- An item whose `index` is nil gets no invented positional number in its accessibility
  label (`CaptureAgendaPresentation` ~:646). Use "Task, <status>, <title>".
- In-place updates use an opacity cross-fade only; height is never animated (epic plan
  §7). Replace `.animation(agendaUpdateAnimation, value: plan.rows)`
  (`CaptureAgendaView` ~:142) with an opacity-only transition that leaves layout
  unanimated, and keep the Reduce Motion behavior.
- The countdown `TimelineView` (`CapturePanelView` ~:490) mounts only while the panel is
  actually visible, not merely while `agendaVisible` is true.
- On day change or clock change, clear the model's pinned `agendaPlanningDay` and
  re-plan, so a hidden panel never paints yesterday's plan at the next show; it shows
  "Loading today…" until the refresh lands.
- Make the show-path spawn guard (epic plan §10) count every recorded bob invocation in
  `FAKE_BOB_RECORD_PATH`, not just `--tasks` lines.
- Strip budgeting changed in `c27359f`: the strip the view renders is budgeted once
  measured, with the full-strip upper bound as the unmeasured fallback.
  - Update the stale doc comments (`CaptureAgendaFitPlanner` ~:67-68,
    `CaptureAgendaPresentation` ~:207-208).
  - Rename `testStripBudgetsTheFullStripHeight` to match what it tests.
  - Fix any README sentence that still claims the upper-bound rule.
- Dead code: delete `CaptureAgendaRowMeasurer.clear()` / `cachedCount` and
  `CaptureAgendaLayoutMetrics.trailingAccessoryWidth` if nothing outside tests uses
  them; drop or adjust the tests that only exercise them.

B9. **Fixtures.** After A1 regenerates the bob-cli goldens, copy the six
`tests/fixtures/capture_pomodoros/agenda-*.json` goldens into bob-mac-capture
`Tests/Fixtures/`. Regenerate `agenda-yesterday.json` as `agenda-current.json` with
`date` moved back one day. The decoders already use `decodeIfPresent`, so explicit nulls
decode as nil. Confirm `CaptureAgendaModelsTests` still pass.

You may split Part B into a few commits. Each pushed commit needs its CI result read,
and the final `master` must be green.

## Final checks

- bob-cli `just check` passes.
- Linux `swift test --filter CaptureCoreTests` passes on a scratch copy of the final
  bob-mac-capture tree.
- bob-mac-capture CI is green on the final `master`, under the flake rule above, and the
  `agenda-*` PNGs have been reviewed.
- `bob capture-pomodoros -t` (human and JSON) looks right against the live vault.

## Closeout of epic bob-cli-66 (final step; do this in the same turn)

1. Run `sase bead epic-symbols bob-cli-66`. It was empty at audit time. For any entry
   now listed, resolve the symbol (wire it up, privatize it, add a non-test pragma, or
   delete it) per the Symvision epic-whitelist policy. Re-key a Justfile line only to a
   still-open later bead that needs it.
2. Close the epic: `sase bead close bob-cli-66 --note "<verification>"`. The note
   states:
   - the phases verified;
   - each fixed issue (A1–A6, B1–B9) with its bob-mac-capture SHA(s) and green CI run
     URL;
   - `just check` and Linux `swift test` results;
   - the PNG review;
   - that integration needed only a rebase onto `f51cdc1`;
   - that follow-up triage is recorded in the epic's `LAND TRIAGE` note.

   Never use `--force` to make the close succeed. If the close is rejected for leftover
   `--epic-symbol` entries, finish that cleanup and close again.

3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file, the PLAN path shown by
   `sase bead read bob-cli-66 -r "<why>"`
   (`plan:202610/idle_capture_pomodoro_agenda.md`, currently `status: wip`).
5. `bob-cli-66` has no `parent_bead`. Confirm with `sase bead read`; if that holds,
   there is nothing further to land.
6. In your final report, list what was and was not verified, and leave Bryan the epic
   plan's "Manual verification for Bryan" checklist. Add one item: the panel opens
   horizontally centred, with the input line exactly where the compact bar opened before
   the epic.

## Out of scope

- The v1.1 interactions, now tasks bob-cli-68, bob-cli-69, bob-cli-6a, and bob-cli-6b.
- The three CI flakes bob-cli-4k, bob-cli-61, and bob-cli-67.
- Any memory edits.
- Perf refactors of the bob-cli agenda builder.
