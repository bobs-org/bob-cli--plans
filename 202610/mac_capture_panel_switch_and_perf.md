---
tier: epic
title: Bob Mac Capture Ctrl+Tab panel switch and show-path performance
goal: 'Ctrl+Tab (and Ctrl+Shift+Tab) in either Bob Mac Capture panel switches to the
  other panel. Capture keeps its draft and caret, and Refs keeps its remembered view.
  Showing, switching, and typing in both panels do less redundant main-thread work
  and spawn fewer bob processes, with no change to what bob decides.

  '
decisions:
  open_surfaces:
    ask: Should Ctrl+Tab switch even while a picker, prompt, stash, shelf, or Refs
      card is open?
    choices:
      switch: Switch at once; the open surface closes as on a hotkey switch, draft
        and view kept
      block: Consume Ctrl+Tab until that surface is finished or cancelled, like Control-\
    default: switch
    why: Matches what the global hotkeys already do from those states; the gesture
      always works
    answer: switch
  arrival_reset:
    ask: Should a Ctrl+Tab return to Refs keep the remembered view even if a new reference
      arrived?
    choices:
      keep: Restore the remembered view; the new reference joins the list without
        moving selection
      reset: Act like the Refs hotkey; reset to defaults and select the new reference
    default: keep
    why: The switch promises that each panel comes back exactly as it was left
    answer: keep
  show_refresh_window:
    ask: Should re-shows within 5 s skip re-spawning Refs Today, capture-targets,
      and template refreshes?
    choices:
      coalesce: Skip them inside a 5 s window; watchers, captures, and mutations still
        refresh at once
      always: Keep spawning all three on every show, as today
    default: coalesce
    why: Rapid switching otherwise spawns about three bob processes per switch and
      re-renders on return
    answer: coalesce
phases:
- id: panel_switch
  title: Ctrl+Tab panel switch with preserved state
  depends_on: []
  size: medium
  description: 'panel_switch: add the coordinator switch, Ctrl+Tab routing in both
    panels with a deferred hop, the Refs panelSwitch presentation source, round-trip
    state tests, switch signposts, and the keyboard/refs/capture docs.'
- id: capture_perf
  title: Capture show, keystroke, and refresh optimizations
  depends_on:
  - panel_switch
  size: medium
  description: 'capture_perf: remove no-op republishes, the per-render executable
    stat, and repeated view-tree walks; add the fuzzy-matcher fast path and the coalesced
    show refreshes for targets and templates; add measuring signposts.'
- id: refs_perf
  title: Refs show, keystroke, ranking, and refresh optimizations
  depends_on:
  - panel_switch
  size: medium
  description: 'refs_perf: drop the unread published row rect, cache the filter field,
    coalesce library-publish passes, skip unchanged inspector and Today republishes,
    precompute ranking sort keys, fix the stale prepared-item cache, and coalesce
    the panel-open Today refresh.'
proposed_by: bbugyi200.apollo.9q
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-20 15:55:11
status: wip
bead_id: bob-cli-a8
---

- **PROMPT:** [prompts/202610/mac_capture_panel_switch_and_perf.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/mac_capture_panel_switch_and_perf.md)
- **BEAD:** [bob-cli-a8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-a8/README.md)

# Plan: Ctrl+Tab panel switch and performance for Bob Mac Capture

All code changes land in the linked `bob-mac-capture` repository. Open it with
`sase repo open bob-mac-capture -r "<reason>"` and read its `AGENTS.md` first. Paths
below are relative to that repository. Nothing changes in bob-cli; the thin-client rule
(`decisions:mac-capture-is-a-thin-client`) is untouched because every change is
presentation, orchestration, or caching.

## Background (verified while planning)

- `BobPanelCoordinator` (`Sources/BobMacCapture/BobPanelCoordinator.swift`) already owns
  the one-panel rule. `showRefs(source:)` retains a visible Capture draft through
  `CapturePanelModel.closeRetainingDraft()` and records `returnToCaptureAfterCopy`.
  `showCapture()` hides Refs, then presents Capture. `AppDelegate.showCapturePanel()`
  wraps `showCapture()` and also calls `refreshTargetsWhenPossible()`.
- **Capture state today.** `closeRetainingDraft()` keeps `plainDraft`, `templateOrigin`,
  `priorityRollSeed`, and error state. The caret lives in `model.editorSelection`, which
  is bound to the `TextEditor`. Hiding closes the stash picker, the template shelf, open
  pickers (`closePickerForDismissal` restores the caret), inline prompts, the pull-in
  choice, and agenda chip expansions. `prepareForPresentation()` re-runs analysis for a
  retained draft on purpose, so close-preview timing never goes stale.
- **Refs state today.** `RefsPanelModel.prepareForPresentation(source: .library)`
  restores the remembered query, scope, Newest/Tabs filters, selection, and inspector
  place (commit `5935822`). It does not restore them when a qualifying new reference is
  pending. Then `resetForArrival()` resets to defaults and selects that reference
  (commit `7a41234`), and a 10 s `visibleArrivalDeadline` lets an arrival reset an
  untouched open panel. `panelDidHide()` closes the Copy card and the Today-link card,
  plus a Rename card that is neither in progress nor failed.
- **Ctrl+Tab is unbound today.**
  - In Capture, `CaptureKeyCommandRouter` returns `.consumeKey` for it inside isolated
    prompts and `nil` everywhere else.
  - In Refs, `RefsKeyRouter` returns `nil`. The local monitors are installed per
    controller with `NSEvent.addLocalMonitorForEvents`.
  - Both panels are non-activating `NSPanel`s, and Refs hides itself on `resignKey`.
- **Hot-path findings that the perf phases address** (file:line verified at planning
  time; re-check, since lines drift):
  - `SavedTemplateShelf.finishList` assigns `templates` unconditionally, and the Capture
    model's `$templates` sink calls `objectWillChange.send()`.
    `updateTargetCacheSnapshot` always assigns a snapshot that carries a fresh
    `refreshedAt`. `publishAgendaPlan` always assigns `agendaPresentation`, and the
    agenda store's unchanged path re-sets `status = .ready`. Each of these re-renders
    the whole Capture panel after every show, even when bob returned identical data.
    That contradicts the agenda decision's "Byte-identical output publishes nothing".
  - `CapturePanelModel.pullInCapabilityNote` is read in SwiftUI `body`. It reaches
    `BobProcessClient.guardFlagKnownUnsupported`, which `stat`s the bob executable
    (`identityToken`) on every render.
  - `CaptureEditorTextConfiguration.configureEditorTextViews` walks the whole view tree
    on every show. The picker and pull-in filter lookups walk it on every keystroke
    while a picker is open. In Refs, `findFilterField` walks the tree twice per
    keystroke (`repairFilterFocusIfNeeded` and `markedTextInFilterField`).
  - `FuzzyMatcher.match` re-folds already-folded tokens. It also allocates the full
    O(m×n) DP before knowing whether the token is even a subsequence.
  - `RefsPanelModel.selectedRowRect` is `@Published`, yet no view reads it; only the ⌘K
    menu does. `RefsListView` writes it on every geometry change of the selected row, so
    the whole panel re-renders while scrolling.
  - `library.$items`, `$signals`, and `$todayRefreshGeneration` each spawn their own
    `Task { libraryDidPublish() }`. Each pass is O(n) and reassigns `listing`.
    `applyInspectorSignals` publishes `signals` once per field. Inspector loads
    republish identical content. Today passes reassign `signals.today` and rewrite the
    snapshot file even when Today is unchanged.
  - `RefsRanking` sort comparators call `activityOrdinal` / `compareOpenedDesc`, and
    with them `Calendar` math, on every comparison. `RefsPreparedCache` keys on
    id/title/stem/headings but hands back the cached `RefItem`. A changed `state`,
    `added`, or author therefore ranks with stale fields until a title, stem, or heading
    changes. That is a latent correctness bug.
  - Each show spawns `capture-targets` (no throttle), `capture-templates`, and the Refs
    Today `bob plan` (single-flight only). The Refs pull already has a 5 s panel-open
    throttle (`RefsLibrary.pullThrottleInterval`). The idle agenda's per-show
    revalidation is required by `decisions:idle-capture-shows-ledger-agenda` and must
    stay.

## Verification rules (every phase)

Follow `bob-mac-capture/AGENTS.md` exactly. Never land Swift you have not compiled:

1. Run `just portable-check` on a host with Swift; athena has one if this host does not.
   It covers `CaptureCore` and `RefsCore`, including their tests.
2. Changes under `Sources/BobMacCapture*` or `Tests/BobMacCapture*` also need the Mac:
   rsync a scratch copy (without `.build`/`.git`), run `Scripts/bundle.sh` and
   `just typecheck-tests`, then delete the copy. Never use Bryan's checkout or
   `just install` there.
3. If the Mac is offline, say in the final report that the macOS targets are unverified.
4. Async tests must never await an untimed continuation or gate.
5. Run `just format-lint` before committing. A parse error in lint skips every later CI
   step.

Docs edits run `just docs-check`.

## Phase panel_switch: Ctrl+Tab panel switch with preserved state

### Behavior

- **What the keys do.** In either panel, Ctrl+Tab and Ctrl+Shift+Tab switch to the other
  panel. With two panels both directions reach the same place. Caps Lock is ignored when
  matching modifiers. Any other added modifier (Command, Option) leaves the key to
  AppKit.
- **Switching from Capture to Refs.**
  - The draft is retained exactly as on the Refs hotkey (`closeRetainingDraft`).
  - Refs then shows with the new `RefsPresentationSource.panelSwitch`.
  - `returnToCaptureAfterCopy` is set as it is for any Refs-over-Capture show, so a
    successful Refs copy still brings Capture back with the draft.
- **Switching from Refs to Capture.** Refs hides, Capture shows through the same path as
  the Capture hotkey: `prepareForPresentation`, focus on the editor, and a
  `refreshTargetsWhenPossible()`, subject to phase capture_perf's window.
- **Key repeats.** A held Ctrl+Tab (`event.isARepeat`) is consumed without switching, so
  holding the key never ping-pongs.
- **IME composition.** While IME composition is active (marked text in the focused text
  view), Ctrl+Tab passes through to the text system.
- **Task-goto dispatch.** In Capture, while a task-goto dispatch is in flight
  (`pickerDispatchInFlight`), Ctrl+Tab is consumed, as every other key is.
- **Open surfaces.**

> [!decision] open_surfaces = switch Ctrl+Tab is matched before any modal branch:
>
> - Capture: before the template-shelf, stash, prompt, pull-in, and picker branches in
>   `CaptureKeyCommandRouter.command(for:context:)`.
> - Refs: before the Today-link, Rename, and Copy card routing in
>   `RefsPanelController.processKeyEvent`.
>
> The surface then closes through the existing hide paths, exactly as the global hotkeys
> close it today. A failed Rename card still survives, and a rename or Today-link
> already running still finishes.

> [!decision] open_surfaces = block While any Capture modal surface is open (template
> shelf, stash, task-ID or Pomodoro prompt, relation-label prompt, pull-in choice,
> picker), Ctrl+Tab returns `.consumeKey`. The same applies in Refs while the Copy,
> Rename, or Today-link card is open. Everywhere else it switches.

### Implementation

1. **Coordinator.** Add a `BobPanel` enum (`capture`, `refs`) and
   `@discardableResult func switchPanels(from origin: BobPanel) -> BobPanel?` to
   `BobPanelCoordinator`:
   - From `.capture` while Capture is visible: run `showRefs(source: .panelSwitch)` and
     return `.refs`.
   - From `.refs` while Refs is visible: run `showCapture()` and return `.capture`.
   - When the origin panel is no longer visible, do nothing and return `nil`. A stale or
     duplicate deferred request therefore cannot bounce back.
2. **Refs source.** Add `case panelSwitch` to `RefsPresentationSource`, with a doc
   comment. In `RefsPanelModel.prepareForPresentation(source:)`:

> [!decision] arrival_reset = keep `.panelSwitch` always runs
> `restoreLastViewForPresentation()`, even when `arrivals.hasQualifying(...)`. It then
> runs `arrivals.clearPending()`, because the fresh listing already contains the new
> reference at its ranked place. It sets `visibleArrivalDeadline = nil`, so an arrival
> that lands while the switched-to panel is open joins through `libraryDidPublish`'s
> existing join path instead of resetting. A failed Rename card is replayed exactly as
> in `.library`. `refreshForOpen()` still runs on show.

> [!decision] arrival_reset = reset `.panelSwitch` shares the `.library` branch
> unchanged, so the arrival reset applies. Keep the enum case anyway: signposts and
> tests can tell switches apart.

3. **Capture routing.**
   - Add `CaptureKeyCommand.switchPanel`.
   - Add `editorHasMarkedText: Bool` to `CaptureKeyRoutingContext`. Fill it cheaply from
     `(panel.firstResponder as? NSTextView)?.hasMarkedText() ?? false`, which needs no
     tree walk.
   - At the top of `command(for:context:)`, match Tab (keyCode 48) with modifiers equal
     to `.control` or `[.control, .shift]` after removing `.capsLock`:
     - marked text → `nil`;
     - `pickerDispatchInFlight` or a repeat → `.consumeKey`;
     - otherwise → `.switchPanel` (subject to `open_surfaces`).
   - Plain Tab and Shift-Tab behavior is untouched.
   - `CapturePanelController` gets `var panelSwitcher: () -> Void = {}`. Its
     `perform(.switchPanel)` schedules `panelSwitcher` on the next main-queue turn
     (`DispatchQueue.main.async`) and returns `true`. Deferring keeps the other panel's
     local monitor from seeing the same key event after it becomes key.
4. **Refs routing.**
   - Add a pure `RefsKeyRouter.isPanelSwitch(keyCode:modifiers:) -> Bool` with the same
     modifier rule.
   - In `RefsPanelController.processKeyEvent`, after the `suspendHideOnResign` and
     `flagsChanged` early returns and before card routing (see `open_surfaces`): when
     `isPanelSwitch`, pass the event through while the filter, rename, or Today-link
     field holds marked text. Otherwise consume it, and unless it is a repeat, schedule
     `panelSwitcher` on the next main-queue turn.
   - Add `var panelSwitcher: () -> Void = {}` to `RefsPanelController`.
5. **AppDelegate wiring.** Wire both `panelSwitcher`s to a new
   `AppDelegate.switchPanels(from:)`. It calls `panelCoordinator?.switchPanels(from:)`
   and, when the result is `.capture`, `refreshTargetsWhenPossible()`, just like
   `showCapturePanel()`. Wrap the switch in a `panel-switch` signpost interval and emit
   `panel-switch-key` events.
6. **Docs.**
   - `docs/keyboard.md`:
     - Add a short "Switching panels" section near the top.
     - Add a Ctrl-Tab row to the Bob Refs table.
     - Add a Ctrl-Tab / Ctrl-Shift-Tab row to the editor table, with per-column text
       that follows `open_surfaces`.
   - `docs/refs.md`:
     - Under "Opening Bob Refs" and "Updates", state that a switch restores the
       remembered view. Under `arrival_reset = keep`, it never resets for an arrival.
     - In the one-panel bullet, mention Ctrl+Tab.
   - `docs/capture-panel.md`: one sentence saying a switch retains the draft and caret,
     like Escape.
   - `docs/diagnostics.md`: add the new signposts to its list.

### Tests

- **`RefsKeyRouterTests`.** `isPanelSwitch` for Ctrl-Tab, Ctrl-Shift-Tab, and with Caps
  Lock. It must be false for plain Tab, Shift-Tab, Command-Ctrl-Tab, and
  Option-Ctrl-Tab.
- **Capture router tests** (alongside the existing `CaptureKeyCommandRouter()` tests in
  `Tests/BobMacCaptureTests`):
  - `.switchPanel` in the editor, with completion visible, and with each modal surface
    visible (or `.consumeKey` under `block`);
  - `.consumeKey` for repeat and for `pickerDispatchInFlight`;
  - `nil` with marked text;
  - plain Tab and Shift-Tab unchanged.
- **Coordinator tests** (`RefsEntryPointsTests`, reusing `makeCoordinator`/`EventLog`):
  - From Capture: logs `close-retain` then show-refs with `.panelSwitch`, and keeps the
    draft.
  - From Refs: logs `hide-refs` then `show-capture`.
  - A stale origin is a no-op.
  - After a switch from Capture, `dismissRefsAfterCopy` restores Capture.
- **`RefsPanelModelTests`:**
  - A `.panelSwitch` show after hiding restores query, scope, Newest, Tabs, selection,
    and inspector place.
  - Under `keep`, a qualifying pending arrival does not reset; the new id is in the
    listing, the selection is unchanged, pending is cleared, and a later `.library` show
    does not reset.
  - `.library` behavior is unchanged (the existing arrival tests stay green).
- **Capture round trip** (`CapturePanelModelTests`): build a draft with a non-end caret,
  a template origin, and an error. `closeRetainingDraft()` then
  `prepareForPresentation()` keeps `plainDraft`, `editorSelection`, `templateOrigin`,
  and the error. Mirror an existing hosted controller test and also assert the backing
  `NSTextView`'s `selectedRange()` after hide and show.
- **Controller tests:**
  - `RefsPanelController.processKeyEvent` with a synthesized Ctrl-Tab returns `nil` and
    fires `panelSwitcher` only after the main queue drains, using a bounded wait.
  - A repeat event never fires it.
  - The same holds for `CapturePanelController.perform(.switchPanel)`.

## Phase capture_perf: Capture show, keystroke, and refresh optimizations

Every item must be behavior-preserving except `show_refresh_window`. Measure before and
after with the new signposts on the Mac when it is reachable, and report numbers. If the
Mac is unreachable, say so.

1. **No-op republish guards.**
   - **`SavedTemplateShelf.finishList` and its failure paths.** Assign each `@Published`
     only when it changed (`templates != response.templates`, and likewise for the
     rest).
   - **`CapturePanelModel.updateTargetCacheSnapshot`.** Skip the assignment when
     `targets`, `stale`, and `errorDescription` are equal; keep the status-text update
     for errors. First grep that nothing reads `refreshedAt` from the model's snapshot.
   - **`publishAgendaPlan`.** Assign `agendaPresentation` and `agendaPlan` only when
     `!=`. Keep the re-plan itself, since it picks up the current `now`.
   - **`CaptureAgendaStore`'s unchanged path.** Set `status = .ready` only when the
     status differs. `lastRefreshedAt` still updates; it feeds only the Settings
     diagnostic. Verify that no Capture panel view observes the store directly.
   - **Tests.** Add tests that an identical refresh leaves the model's
     `objectWillChange` count unchanged. Existing agenda and template tests stay green.
2. **No `stat` in `body`.** In `BobProcessClient.guardFlagKnownUnsupported`
   (CaptureCore), return `false` under `stateQueue` when `unsupportedGuardFlags` is
   empty, before computing `identityToken`. The result is identical: an identity
   mismatch only empties the set. Add a portable test with a counting seam, or assert
   the result is unchanged across an executable replacement.
3. **View lookups.**
   - Configure each editor `NSTextView` once: keep a weak reference plus its
     `ObjectIdentifier`. On show, reconfigure only when the cached view is gone or no
     longer in the panel (`textView.window !== panel`); in that case fall back to the
     walk.
   - For the picker and pull-in filter fields, try the first responder first: a field
     editor whose delegate is the `CapturePickerFilterNSTextField` with the wanted
     identifier. Fall back to the walk only when that misses.
   - Keep the existing `*-focus-repaired` signposts.
4. **FuzzyMatcher fast path** (CaptureCore).
   - Before allocating the DP, run an O(n) greedy subsequence check and return `nil`
     when it fails. This is exact: the DP returns `nil` iff no in-order alignment
     exists.
   - Avoid re-folding tokens that `FuzzyQuery` already folded, through an internal
     pre-folded overload; keep the public signature.
   - **Tests.** Folding is idempotent. A seeded randomized equivalence test compares the
     new `match` against the old implementation, kept as a test-only oracle, on scores
     and positions.
5. **Show-refresh window.**

> [!decision] show_refresh_window = coalesce
>
> - **Targets.** `AppDelegate.refreshTargetsWhenPossible` takes a reason. Show-triggered
>   calls (`showCapturePanel` and the switch) are skipped when a targets refresh started
>   less than 5 s ago. Watcher, Recheck Bob, launch, and failure-recovery calls always
>   run.
> - **Templates.** `SavedTemplateShelf.refresh(reason: .show)` is skipped when the last
>   refresh started less than 5 s ago. `.launch`, `.submit`, `.mutation`, and `.recheck`
>   always run.
> - **Shared clock.** Both use an injectable `now` and one shared constant named after
>   `RefsLibrary.pullThrottleInterval`.
> - **Agenda.** The idle agenda's per-show refresh is untouched.
> - **Tests.** Cover inside-window skips and outside-window runs, and that non-show
>   reasons are never skipped.
> - **Docs.** Note the window in `docs/capture-panel.md` and `docs/saved-templates.md`
>   where they describe show refreshes.

> [!decision] show_refresh_window = always Leave every show-triggered refresh as it is.

6. **Signposts.** Add intervals `capture-prepare` (around `prepareForPresentation`),
   `capture-dismiss`, and `templates-refresh`, and list them in `docs/diagnostics.md`.

## Phase refs_perf: Refs show, keystroke, ranking, and refresh optimizations

Every item must be behavior-preserving except `show_refresh_window`. Benchmark the
ranking items with `swift run refs-rank` on a large synthetic snapshot before and after
(the existing ranking perf guard test ranks 10k items), and report the numbers.

1. **`selectedRowRect`.** Make it a plain stored property, not `@Published`, after
   grepping that no view reads it or `$selectedRowRect`. The ⌘K anchor and the existing
   reset test keep working.
2. **Filter field cache.** Hold a weak reference to the Refs filter field, validated by
   `field.window === panel` and the accessibility identifier. Use it in
   `repairFilterFocusIfNeeded`, `markedTextInFilterField`, and `focusFilterField`, and
   fall back to `findFilterField` on a miss. Rename and Today-link lookups run only
   while their card is open; leave them as walks.
3. **Coalesce `libraryDidPublish`.**
   - Replace the three per-publisher `Task { await self?.libraryDidPublish() }` spawns
     with one scheduler: a `libraryPublishPending` flag, plus one `Task` that clears the
     flag and runs a single pass.
   - The pass reads current library state, so merging bursts is exact.
   - Adjust tests that count yields to await the coalesced pass (bounded waits only).
4. **Inspector and Today republishes.**
   - **`RefsInspectorLoader`.** Skip `content[id] =` and `applyInspectorSignals` when
     the built content equals what is cached; do not re-decode identical image data.
   - **`RefsLibrary.applyInspectorSignals`.** Mutate a local copy of `signals` and
     assign it once.
   - **Today pass.** Skip the `signals.today =` assignment and the snapshot save when
     Today is unchanged. Still bump `todayRefreshGeneration`, because optimistic
     Today-link overlays depend on it.
5. **Ranking.**
   - In `RefsRanking`'s section sorts, compute each item's activity, opened, and other
     sort keys once (decorate-sort-undecorate). Compute the today ordinal once per
     listing. Ties keep the `id` tie-breaker, so the order is identical.
   - Key `RefsPreparedCache` on direct equality: the `items` array (`RefItem` is
     `Equatable`) plus the outline headings. Drop the joined string key, and return
     prepared items that carry the current `RefItem`.
   - Add a regression test: two snapshots with the same id/title/stem but a different
     `state` or `added` rank by the new values.
   - The existing ranking tests and the cache build-count test stay green; update the
     latter only if its counting semantics legitimately change.
6. **No duplicate snapshot refresh.** `refreshIfStale` does nothing while a snapshot
   pass is already running.
7. **Panel-open Today window.**

> [!decision] show_refresh_window = coalesce
>
> - Add `RefsLibrary.requestTodayForPanelOpen()`, mirroring `requestPullForPanelOpen`.
>   It skips the panel-open Today refresh when the last panel-open Today refresh started
>   less than 5 s ago.
> - `refreshForOpen` uses it. Capture success, the 10-minute timer, wake, watcher
>   batches, and ⌘R refresh Today unthrottled, as today.
> - **Tests.** Cover the window with the injectable `now`.
> - **Docs.** Update the "refreshes Today … on every panel open" sentence in
>   `docs/refs.md`.

> [!decision] show_refresh_window = always `refreshForOpen` keeps calling
> `refreshToday(reason: .panelOpen)` on every show.

8. **Signposts.** Add a `refs-library-publish` interval and a `refs-today-skipped`
   event, and list them in `docs/diagnostics.md`. Phase capture_perf also edits that
   page: keep each edit to its own phrase so a rebase merges cleanly.

## Out of scope

- **No `@Observable` migration** for the 57-property `CapturePanelModel`. It is high
  impact but high effort and risk, so file it as a follow-up task bead if the measured
  numbers still show render cost.
- **No change to the Refs fade-in or its focus retry.** Neither is on the typing path.
- **No listing memoization across shows.** Ranking depends on `signals.now` and the
  "Just scanned" window, so a memo is a correctness risk.
- **No stale-preview display on Capture re-show.** It would touch the submit and pull-in
  guard flow.
