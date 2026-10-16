---
tier: tale
size: medium
title: Land bob-cli-84 — repair the Highlights Tabs filter and close the epic
goal:
  Bob Mac Capture builds again with the Tabs filter. Every BobMacCaptureTests target
  type-checks, and the defects the land agent verified are fixed. Epic bob-cli-84 is
  closed, with its plan file marked done.
proposed_by: bbugyi200.apollo.bob-cli-84.land
bead: bob-cli-84
create_time: 2026-10-16 08:48:54
status: wip
---

- **PARENT:**
  [202610/highlights_tabs_switcher.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/highlights_tabs_switcher.md)
- **BEAD:**
  [bob-cli-84](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-84/README.md)

# Plan: Land bob-cli-84 — repair the Highlights Tabs filter and close the epic

## Context

Epic **bob-cli-84** ("Bob Refs Tabs filter — a Highlights tab switcher") has six closed
phases, committed to the **`bob-mac-capture` linked repo** at:

- `1cd958e` signing;
- `94a4e58` tabs core;
- `a4009a4` tabs model;
- `1b7ead4` tabs views;
- `1e2a9bd` entry, settings, and the AX reader.

Read the epic and its plan first:

```bash
sase bead read bob-cli-84 -r "Need the epic scope, decisions, landing notes, and plan path"
```

The PLAN path that command prints is the epic's binding design (Shared contracts,
A1–A11).

The land agent verified these facts on master `1e2a9bd`:

- **The Mac app does not build.**
  `Sources/BobMacCapture/Refs/AXHighlightsTabsReader.swift:174-178` infers `tabButtons`
  as `[Any]`. CI run 38899044286 fails at **Build** with errors at lines 198 and 223.
  Phase 84.6 never compiled on the Mac.
- **BobMacCaptureTests has not compiled in CI since `e095347`.** That commit is the
  bob-cli-7x rename landing, which landed after this epic started.
  `Tests/BobMacCaptureTests/RefsInspectorTests.swift:483-485` passes `await` expressions
  inside XCTest autoclosures. As a result, none of this epic's Mac tests have ever run.
  This landing fixes it as integration work.
- **These two fixes are proven.** With both applied to a scratch copy on the Mac,
  `Scripts/bundle.sh` exits 0. With `-sdk` added to the scratch typecheck script, all
  four test targets type-check. Athena `just portable-check` passes (1,236 tests).
- **Reviewers found more defects, and the land agent confirmed each one.** They are
  listed below by file. Line numbers refer to `1e2a9bd`.

Open the repo first:

```bash
sase repo open bob-mac-capture -r "Land bob-cli-84 fixes"
```

Then read its `AGENTS.md`. Work and commit only in the path that command prints. Keep
the epic's rules:

- **AX containment.** Every `AXUIElement*` call stays in `AXHighlightsTabsReader.swift`,
  and `AXIsProcessTrusted*` stays only in `AccessibilityTrust.swift`.
- **Privacy.** Tab names and paths are never logged, signposted, or persisted, and never
  appear in fixtures copied from Bryan's live session.
- **House style.** Doc comments on every type and public member, and pure logic in
  `RefsCore`.
- **No bob-cli code change.**

## Fixes

### 1. Build and test-compile blockers

1. **`AXHighlightsTabsReader.swift:174-178`.** Replace the
   `tabGroup.flatMap { … } ?? []` closure with an explicitly typed branch. This exact
   form was verified to compile:

   ```swift
   let tabButtons: [AXUIElement]
   if let tabGroup {
       setMessagingTimeout(on: tabGroup)
       tabButtons =
           try optionalAttribute(tabGroup, "AXTabs" as CFString, deadline: deadline)
           as? [AXUIElement] ?? []
   } else {
       tabButtons = []
   }
   ```

2. **`RefsInspectorTests.swift:483-485`.** Hoist each awaited value into a local before
   its assertion. For example, `let firstPageCount = await first.pageCount`, then
   `XCTAssertEqual(firstPageCount, 7)`. Do the same for the two
   `loader.intrinsics(for:id:)` calls.

### 2. Tests that fail as soon as the target compiles

1. **`RefsPanelDesignTests.swift:1201`.** `renderTabsLoaded` unwraps
   `rowContent(for: "ref/chat/team_fit_review.md")`, but the fixture item whose PDF is
   `lib/chat/team_fit_review.pdf` has the id `ref/chat/today_report.md`.
   - Fix the id.
   - Then trace **every** Tabs render in `testRefsPanelStatesToPNG` by hand against the
     fixture library (around line 1455) and `makeTabsSnapshot`. Check the row ids,
     counts, selected ids, captions, and states. All of them run inside one test
     function, so the first throw hides every later one.
2. **`RefsFooterTests.swift:16-18`.** `RefsFooter.hintsText` now adds `⌘T Tabs` to every
   library open hint. Update the second assertion to match. Re-check the `hintVariants`
   assertions against `RefsFooter.swift`.
3. **`AXHighlightsTabsReaderTests`.** It expects `.notRunning` for an unknown bundle id.
   The reader checks trust first, so on an untrusted CI runner it returns `.notTrusted`
   instead. Fix 3.1 below repairs this.

### 3. AX reader (`Sources/BobMacCapture/Refs/AXHighlightsTabsReader.swift`)

1. **Check order.** In `readOnQueue` and `selectOnQueue`, check the running process
   (`.notRunning`) **before** trust (`.notTrusted`). That is the plan's order, and it
   makes the CI test deterministic.
2. **Windows without a tab bar.** Such a window becomes one selected tab named only from
   its `AXDocument`. Drop the window-title fallback (lines 182-183, plus
   `fileName(fromWindowTitle:)` if it becomes unused). Otherwise Highlights' Settings or
   panel windows show up as phantom "Not in your library" rows.
3. **Selecting a document in a window without a tab bar.** Today `select` returns
   `.notFound` because `tabButtons` is empty (lines 85-90). When the located window has
   no tab buttons and the located tab is its single document tab:
   - skip `AXPress`;
   - activate Highlights and `AXRaise` that window;
   - verify the window's `AXDocument` file name still equals the target;
   - return `.selected`, or `.failed("Highlights didn't switch tabs")`.
4. **Verify the pressed window, not a re-located one.** After `AXPress`, lines 110-118
   re-locate the target with the stale `windowOrdinal`, and `AXRaise` can reorder
   `AXWindows`. Instead, re-read the tabs of `window.element` itself: find its tab group
   and read `AXTabs`. Require the single button titled `target.fileName` to have
   `AXValue == 1`. This matches the plan's "re-read that window's tabs".
5. **Error mapping.** `optionalAttribute` (lines 301-311) must throw a name-free
   `ReadFailure` for `.cannotComplete` ("Highlights didn't respond to the tab read") and
   `.apiDisabled`. Keep returning nil for unsupported or missing attributes and invalid
   elements.
6. **Unverified switch event.** When the verify step fails, log it by event name only
   with `CaptureSignpost.event("refs-tabs-select-unverified")`. It is a `StaticString`
   and carries no names.

### 4. RefsCore (`Sources/RefsCore/RefsHighlightsTabs.swift`, `RefsTabsRanking.swift`)

1. **Dedupe synthetic rows.** `HighlightsTabMatcher.match` must keep one collision or
   unmatched row per file name: the current occurrence, else the first in window order.
   This matches the plan contract "one per unmatched file name". Keep the `#N` suffix
   code as a defensive fallback. Update `RefsHighlightsTabsTests` around lines 130-149,
   which currently expect `cv.pdf#2`. Add a test showing the current occurrence wins.
2. **Page captions.** Set `pageCaption` only on a window's selected tab
   (`tab.isSelected ? window.pageCaption : nil`), per A11: no page numbers for
   background tabs. Add an assertion for it.
3. **Pre-selection.** In browse pre-selection (`RefsTabsRanking.swift:77`), pick index 1
   only when `highlightsIsFrontmost`, at least 2 rows exist, **and**
   `currentIDs.contains(ids[0])`. Otherwise pick index 0. If the current tab is scoped
   out, row 1 is already the previous tab, and the goal is "previous tab pre-selected".
   Add a golden test: frontmost, current tab outside the scope → row 1.
4. **Rename helper.** Add a small public helper, for example
   `RefsOpenTab.renamingItem(from oldID: String, to newID: String) -> RefsOpenTab`, with
   a golden test. It rewrites `itemID`/`rowID` when they equal `oldID`, and rewrites
   `candidateIDs`, keeping them sorted. Fix 5.4 uses it.

### 5. Panel model (`Sources/BobMacCapture/Refs/RefsPanelModel.swift`)

1. **Nothing persists between shows (A3).** `prepareForPresentation` keeps
   `matchedTabs`, `tabsHighlightsIsFrontmost`, and `tabsListing` from the previous show,
   so a re-show paints old rows as live. ↵ is then never held, and it switches to a
   stale pre-selection. After the `isRenaming` early return and before
   `rerankSelectingFirst()`:
   - clear those three;
   - bump `tabsRefreshGeneration`, so an in-flight read from a previous show can't land.

   A library show followed by ⌘T then also starts empty.

2. **Clear the hold on hide.** `panelDidHide` (lines 508-519) must clear `heldTabsOpen`.
   Otherwise a ↵ held before dismissing still switches tabs once the snapshot lands.
3. **Vanished matched rows.** In `rowContent`, the branch for a matched row whose item
   vanished (around lines 1004-1016) must pass `openTab: nil`. Such a row then renders
   as the existing unavailable row ("No longer in your library"), not as a Tabs fallback
   row with a false "↵ switches to this tab" hint.
4. **Rename in Tabs mode.** This interacts with the rename card from
   `9ac536c`/`e095347`.
   - In `runRename`, on success and before `freshListing(selecting: destination)`, remap
     `matchedTabs` with the helper from fix 4.4, `notePath` → `destination`.
   - Also remap the tab map in `tabsListing` if it is used before the rebuild.
   - Today the renamed row disappears, because matched tabs keep the old `itemID`.

### 6. Views

1. **Scope token.** `RefsSearchBar.swift:142` sends `.deleteBackwardOnEmpty`, which now
   clears Tabs first. Send `.setScope(.all)` instead, so clicking "Papers ×" clears the
   scope.
2. **VoiceOver labels.** `RefsRowView.swift:350-352`: for rows without an item, append
   "open in Highlights" only when `content.openTab != nil`, and append the current-tab
   phrase when `openTab.isCurrent`, as for matched rows. Unavailable library rows go
   back to their pre-epic label; see
   `git show 1b7ead4^:Sources/BobMacCapture/Refs/RefsRowView.swift`. Factor the label
   into a static helper if that makes it unit-testable, like the helpers
   `RefsTabsViewsTests` already covers.
3. **Permission card.** `RefsTabsViews.swift:62-65` joins strings, so the backticks
   around `tccutil reset Accessibility org.bobs.bob-mac-capture` render literally. Show
   the command as its own monospaced, selectable `Text` instead.

### 7. Signing scripts and Settings

1. **Missing identity value.** In `Scripts/install.sh`, `--identity` given as the last
   argument currently dies silently in `shift 2`. Print "missing identity" to stderr and
   exit 64. Keep `--identity ""` (which the justfile passes) meaning "resolve".
2. **Identity lookup.** `find_stored_signing_identity` pipes `security …` into `grep -q`
   under `pipefail`. A SIGPIPE can turn a found identity into a silent ad-hoc fallback,
   which resets the Accessibility grant. Capture the `security` output into a variable
   first (`|| true`), then grep the variable. Run `bash -n`, plus `shellcheck` if
   available.
3. **Settings refresh.** `SettingsView.swift:243-245` runs `codesign` synchronously on
   every app activation while Settings is open. Compute `isAdHocBuild` once, on appear.
   Keep refreshing only the trust status on activation.

### 8. Tests to add or adjust

The Mac test targets only type-check locally, so trace each test's async flow against
the source by hand.

- **`RefsPanelModelTests`** (use the existing `FakeTabsReader` and `makeHarness`). Add
  or adjust tests that show:
  1. a second Tabs show starts with no rows, holds ↵, and switches to the **new**
     snapshot's pre-selected row (fix 5.1);
  2. a held ↵ is dropped by `panelDidHide` (fix 5.2);
  3. a held ↵ is dropped on a failed read **without** a query edit
     (`testHeldEnterIsDroppedAfterQueryEditAndFailedRead` edits the query first);
  4. a `.notTrusted` read result maps to `.needsPermission(adHoc:)`;
  5. a vanished matched row has `openTab == nil` (fix 5.3);
  6. a rename in Tabs mode keeps the renamed row listed and selected (fix 5.4). Drive it
     the way the existing rename tests do.
- **`RefsEntryPointsTests`** and **`HotKeyRegistryTests`**:
  - a non-Off Tabs key stays unregistered while another app is frontmost;
  - Highlights termination unregisters the Tabs key too;
  - the ⇧⌘O and ⌘E presets collide with neither Refs ⌃⇧⌘O nor the ⌘O/⌃O takeover keys.
- **`RefsKeyRouterTests`, `RefsSearchBar`**: cover fix 6.1 if there is a testable seam;
  otherwise trace it by hand.

Update the README's Tabs section only where it would now misdescribe behavior.

## Verification

1. **Athena (RefsCore and lint).**
   - `rsync -a --exclude .build --exclude .git` the opened repo into a fresh `mktemp -d`
     on `athena`.
   - Run `just portable-check` there. It must pass swift-format lint, the build, and all
     tests.
   - Delete the copy.
2. **Mac (app and test type-check), required for this landing.**
   - Rsync a fresh scratch copy to `mac` the same way.
   - Run `Scripts/bundle.sh`: exit 0 and no `error:` lines.
   - bob-cli-7o is still open, so patch **only the scratch copy's**
     `Scripts/typecheck-tests.sh`:
     `sed -i '' "s|-target \"\${target_triple}\"|-sdk \"$(/usr/bin/xcrun --sdk macosx --show-sdk-path)\" -target \"\${target_triple}\"|g" Scripts/typecheck-tests.sh`.
     Then run `./Scripts/typecheck-tests.sh` (prefix PATH with `/opt/homebrew/bin`). All
     four targets must print PASS.
   - Delete the copy.
   - Also delete the land agent's leftover `/tmp/bmc-land84.dBHw5N` and
     `/tmp/bmc-land84-*.log` on the Mac if they still exist.
   - Never use Bryan's checkout on the Mac, and never run `just install`.
3. Run `git diff --check` in the opened repo.
4. **If `ssh mac` is unreachable.** Retry over at least 30 minutes, using
   `/sase_monitor` on a `sleep` between attempts.
   - If it is still offline, **do not close the epic**, because an unverified Mac build
     is exactly the defect being landed.
   - Instead, add a note with `sase bead note bob-cli-84 "…"` saying that the fixes are
     committed but the Mac build and type-check are unverified, so the landing is still
     pending.
   - End the turn through `/sase_final` without closing.

Commit the bob-mac-capture changes with a message naming bead bob-cli-84. The final
declaration covers the opened repo and the plans repo.

## Closeout of epic bob-cli-84 (final step)

Run these only after verification steps 1–3 pass.

1. Run `sase bead epic-symbols bob-cli-84`. It was empty at triage. If entries appear,
   resolve each one (wire up, privatize, pragma, or delete), or re-key it only to a
   still-open bead that needs it.
2. Close the epic:

   ```bash
   sase bead close bob-cli-84 --note "<verification>"
   ```

   The note must state:
   - **Phase review.** The land agent reviewed all six phases and their notes against
     the plan's contracts and the commits `1cd958e`, `94a4e58`, `a4009a4`, `1b7ead4`,
     and `1e2a9bd`.
   - **What was broken.**
     - Master `1e2a9bd` failed to build on the Mac, at
       `AXHighlightsTabsReader.swift:174`.
     - BobMacCaptureTests had not compiled in CI since `e095347`
       (`RefsInspectorTests:483`), so the epic's Mac tests never ran.
   - **The fixes**, grouped as above.
   - **Verification results.**
     - Athena portable-check: the test count.
     - Mac: `bundle.sh` exit 0, and four targets PASS the type-check.
   - **Triage outcomes** (already recorded in a bob-cli-84 note):
     - The decisions record was filed as memory task bob-cli-8c.
     - The typecheck `-sdk` proposals got a +1 on bob-cli-7o.
     - The "XCTest stub lags" proposal was declined: it was a real compile error, now
       fixed.
   - **Declined review items.**
     - A show during a running rename ignores its source: the running rename card owns
       the panel by design.
     - `findTabGroup` depth-first ordering: the 84.3 live probe found the native tab
       group at depth 1.
     - The model's optional `tabs:` init parameter: behaves identically.
   - **CI caveat.** The landing commit's CI run is the first in which BobMacCaptureTests
     compile since `e095347`. Known pre-existing CI failures are tracked in bob-cli-7h,
     bob-cli-7p, bob-cli-61, and bob-cli-67.

   Never use `--force` merely to make the close succeed. If the close is rejected for
   leftover `--epic-symbol` entries, clean them up and close again.

3. Run `just symvision` in the bob-cli workspace if the recipe exists. Today bob-cli's
   justfile has no such recipe, so record that it is unavailable.
4. Set `status: done` in the frontmatter of the epic's plan file, the PLAN path printed
   by `sase bead read bob-cli-84` (`202610/highlights_tabs_switcher.md` in the plans
   repo).
5. bob-cli-84 has no `parent_bead`, so the landing ends here.
