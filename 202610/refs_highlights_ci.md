---
tier: tale
title: Repair Highlights session focus and RefsPanel CI expectations
goal:
  Fix the two RefsPanel state propagation defects and align the six failing tests with
  the intended Highlights session contract, with compilation evidence for the lander.
size: medium
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.2
proposed_with: codex/gpt-6-astra
bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.2
create_time: 2026-10-26 13:14:41
status: wip
---

- **PARENT:**
  [202610/backlog_batch14_ledger_integrity_mac_ci_and_cleanup.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch14_ledger_integrity_mac_ci_and_cleanup.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.2.md)

# Repair Highlights session focus and the RefsPanel CI expectations

Complete phase `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.2` from
`plan:202610/backlog_batch14_ledger_integrity_mac_ci_and_cleanup.md`. Its target task is
`bob-cli-ci`; leave that task and all ancestor epics open for the batch lander. This is
one bounded coding agent's work: two state propagation fixes in one model, corrections
to two test files, and compilation/verification. The phase had no previous-work or
unfinished-remainder notes when read on 2026-10-26. The per-test evidence below is
already recorded on the phase before source edits.

## Scope and repository

Use `/sase_repo` to open configured linked repository `bob-mac-capture` and read its
`AGENTS.md`. All source paths below are relative to that repository. The planning
checkout was at `d8a828f`; fetched `origin/master` advanced to `c7a5f37` while
investigating. Inspect/rebase onto current master before editing and retain concurrent
Copy card work. No changes to product semantics, CLI contracts, memory, releases, live
vault files, or installed applications are required. The parent decisions
`theme_field_strip = writers` and `legacy_key = pin` are final and outside this phase.

Consult these audited artifacts again if needed:

- `bead:bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.2`
- `bead:bob-cli-ci`
- `plan:202610/backlog_batch14_ledger_integrity_mac_ci_and_cleanup.md`
- `bead:bob-cli-cc`

The cc.7–cc.9 Mac phases were closed at planning time; cc's lander is still active. A
coordination note on `bob-cli-cc` explains the overlap and the invalid Copy restore
fixture. Do not independently rewrite
`testHighlightsCopyShowStashesAndLibraryRestoresTheFilter` or its code picker paths. If
a shared model fix resolves it too, record that. Otherwise cc owns its correction.

## Contract and failure classification

Commit `1311536` intentionally made a Highlights-key show an ephemeral session: start
from default filters, select the focused PDF, and reveal its current-page notes. The
next ordinary library show or panel switch restores the preceding view, including
selection and scroll positions. Ordinary Tabs use still remembers its state and consumes
a tab switch's preselection once. The commit's `docs/refs.md` and
`docs/refs-inspector.md` changes, `RefsTabsRanker`, and presentation source branches
establish these expectations; do not revert them to satisfy historical tests.

Inspected CI runs:
[40225855522](https://github.com/bobs-org/bob-mac-capture/actions/runs/40225855522) at
`1311536`, and
[40358201800](https://github.com/bobs-org/bob-mac-capture/actions/runs/40358201800) at
`d8a828f`. The latter has these exact failures:

| Test                                                        | Classification and evidence                                                                                                                                                                                                                          | Required correction                                                                                                                                                                                   |
| ----------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `RefsPanelDesignTests.testRefsPanelStatesToPNG`             | Outdated fixture assertions: `renderTabsLoaded` expects previous tab `morning_notes`, while the current tab is `today_report`; `renderTabsCloseHoverRow` calls that same selected current row unselected.                                            | Assert focused `today_report` for loaded state; use an actually unselected open row for the hover fixture. Keep the exact identity, Current pill, page caption, row count, and unselected-row checks. |
| `testHighlightsTabsShowResetsAndLibraryRestoresTheFilter`   | Invalid setup: query `keep` and scope `.papers` exclude `ref/chat/small_reading.md`; `.select` returns false before the session starts. Consequently selection and the 90-point inspector offset were never saved.                                   | Use a matching nonempty query, nondefault scope, and item; retain every reset/restore assertion, including offsets 90 and 200.                                                                        |
| `testHighlightsKeyFocusPublishesAndReveals`                 | Product defect: the loader's content publisher calls `tryFireHighlightsKeyFocus`, which reads the old stored content.                                                                                                                                | Evaluate focus from the emitted content snapshot and verify the event and revealed page.                                                                                                              |
| `testHighlightsKeyFocusReselectionRearmsAndCtrlDCancels`    | Same product defect blocks initial focus.                                                                                                                                                                                                            | Preserve and strengthen rearming/cancellation coverage after fixing publication.                                                                                                                      |
| `testTabSwitchUsesTheSwitcherPreselectionOnceOnTheNextShow` | Outdated setup: it enters with `.highlightsTabs`, then expects `.library` to keep Tabs instead of restoring the pre-session library. The waits for `.loaded` also contradict the restored `.off` state.                                              | Exercise ordinary Tabs with `.toggleTabs`/`.setTabsFilter(true)`, and separately verify ephemeral-session restoration.                                                                                |
| `testHighlightsKeySelectsCurrentTab`                        | Product defect triggered by publication order, even though the latest inspected run passed: the first failing run's only failure was `currentID == nil` at line 3125. `libraryDidPublish` rebuilds `RefsTabsListing` without forwarding `currentID`. | Retain current-tab identity through content refresh, with deterministic regression coverage.                                                                                                          |

The two model defects remain in fetched `c7a5f37`. Apple documents that
[`@Published` notifies during `willSet`](https://developer.apple.com/documentation/combine/published?changes=_5):
the callback receives the new value before stored-property reads expose it. This
explains the focus failure; a longer timeout cannot fix a missing notification.

## Implementation

1. In `Sources/BobMacCapture/Refs/RefsPanelModel.swift`, pass the content map received
   by `inspectorLoader.$content.sink` into focus evaluation. Other callers may use the
   loader's committed content. Use one coherent snapshot per evaluation: an explicitly
   supplied empty map or missing selected ID must not fall back to stale loader data.
   Keep the existing session, armed, loaded-tabs, selected-current-tab, page-number, and
   focus-range guards. Clamp/reveal from the same incoming content, publish once, and
   disarm. Avoid arbitrary sleeps, preview-only fixups, or asynchronous retries that
   could revive a canceled focus.

2. In `libraryDidPublish`, carry `currentID` when rebuilding `RefsTabsListing`. Preserve
   its documented invariant that it identifies a listed, in-scope row; clear it if the
   refreshed listing no longer contains that row. Keep existing frozen ordering,
   selection, preselection, and unavailable-row behavior intact. The ranker's two other
   construction sites already supply `currentID`.

3. Correct the invalid restore setup in
   `Tests/BobMacCaptureTests/RefsPanelModelTests.swift`. For example use `small`,
   `.chats`, and `ref/chat/small_reading.md` with `.newest`, or an equivalent explicit
   fixture. Assert the item remains in the filtered listing and selection succeeds
   before recording offsets. Assert exact restored query, scope, ordering, item,
   inspector offset 90, and list offset 200. Keep repeated Highlights shows,
   `.panelSwitch`, representer, and scan-result reset coverage. No production selection
   guards should change to accommodate this test.

4. Rewrite the ordinary tab-switch test to enter persistent Tabs through a normal filter
   gesture, switch to another PDF, and show the library again with an updated snapshot.
   Assert one-time current-tab preselection and subsequent preservation of a
   deliberately different manual selection. Add or adapt a separate ephemeral case that
   starts with a specific library view, switches a PDF inside the Highlights session,
   and asserts the original library view returns with Tabs off. Preserve
   open-count/recency assertions; use waits for the state each path actually promises.
   Name tests to describe the resulting contracts.

5. In `Tests/BobMacCaptureTests/RefsPanelDesignTests.swift`, update the loaded Tabs
   selection and fixture comments to the current PDF. Use `morning_notes` or another
   known noncurrent row for the unselected hover image, keeping the inequality assertion
   and closeability checks. Keep rendering all fixture states.

## Regression evidence

Strengthen existing tests rather than adding a broad new harness:

- Subscribe to `inspectorFocusRequests` before installing/loading the 13-note fixture.
  Assert exactly one focus for the selected ID at index 6 for page 7, with the matching
  notes-page generation and 10 visible notes. A late subscriber must not receive a
  replay. A content-only publication must not duplicate a consumed event.
- Exercise both arrival orders: content after the tab snapshot and cached content before
  it. Keep the unknown-page, nonnumeric-label, and outside-session negative checks. Make
  at least one event-driven regression fail on the old model without needing a second
  loader publication.
- Verify selecting another row clears focus, returning to the current row rearms it, and
  Ctrl+D cancels an armed focus even if content arrives afterward. Keep bounded test
  waits; do not increase timeout budgets or add untimed continuations.
- Force a library content/signal publication after a loaded Tabs snapshot, wait for
  evidence that the coalesced model refresh was applied, and assert selected ID,
  preselected ID, and current ID retain their exact expected values. Merely waiting for
  an already-true predicate before the refresh runs is insufficient. Cover the
  missing-current-row boundary without changing frozen-listing policy.

If a test exposes an additional directly related defect, record the evidence before
changing product behavior. Out-of-scope work goes only into a `PROPOSED FOLLOW-UP:` note
on the assigned phase; never create task beads.

## Verification and completion

1. Run `just portable-check` in the linked repository on athena. It parses all Swift and
   builds/tests CaptureCore and RefsCore, but does not compile AppKit targets.
2. Following the linked repository's `AGENTS.md`, read tailnet memory and use SSH alias
   `mac` for a best-effort isolated scratch build. Copy this opened checkout with `.git`
   and `.build` excluded, run `./Scripts/bundle.sh --identity -` and
   `just typecheck-tests`, capture results, then delete the scratch copy. Never use
   Bryan's checkout or install the app. The Mac's Command Line Tools lack XCTest; the
   typecheck stub verifies compilation, not test behavior. If offline, explicitly record
   the unverified macOS targets and leave CI verification to the lander.
3. Run the parent-required `just check` in bob-cli, never `just check-full`. For
   failures unrelated to this change, reproduce on the clean base when necessary and
   record exact baseline evidence and any existing owner bead in a `PROPOSED FOLLOW-UP:`
   note. Identical baseline failures do not keep the phase open. Use `/sase_monitor` for
   long commands; wait for its handoff command to exit.
4. Record the modified paths, per-test classification, actual check results, and
   compilation limitations on the phase. Real macOS XCTest results come from CI after
   landing; the batch lander must inspect the run covering the landed SHA, confirm all
   six targeted tests and the overall pipeline, and coordinate any remaining cc-owned
   Copy failure before closing `bob-cli-ci`. Do not report a portable or typecheck pass
   as a green XCTest run.
5. Run `sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.2` before closing;
   the planning run found none. Resolve any entries or re-key them to a still-open
   appropriate owner. Close only this assigned phase with `sase bead close` and a
   verification note once its implementation and available checks are complete. Leave
   `bob-cli-ci`, `bob-cli-cc`, and all ancestors to their land agents.
6. Use `/sase_final` as the last action before a normal completion response and declare
   every changed repository so the host commits the work. Do not hand-edit bead status
   or make manual git commits.
