---
tier: tale
size: medium
title: Repair Bob Refs tab closing and finish bob-cli-8l landing
goal:
  Close Highlights tabs with Ctrl-X while Ctrl-D always scrolls, repair close-worker and
  save-sheet handling, integrate write guards, and close epic bob-cli-8l after
  verification.
proposed_by: bbugyi200.apollo.bob-cli-8l.land
bead: bob-cli-8l
status: done
---

- **PARENT:**
  [202610/highlights_tab_close.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/highlights_tab_close.md)
- **BEAD:**
  [bob-cli-8l](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-8l/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-8l.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-8l.land.md)
- **COMMITS:**
  - [175153c](https://github.com/bobs-org/bob-mac-capture/commit/175153c8e7f5913a53f03d710c6f3b0c01f5ce74)
    — feat(refs): close Highlights tabs with Ctrl-X, repair worker/sheet/guards

# Finish the Bob Refs tab-close landing

This is only the remaining work from the landing audit of `bob-cli-8l`. Implement it
directly, verify it, and finish that epic's closeout in the same turn. There is no
separate land agent for this tale. Do not wait for this tale's commit SHA, push, or CI
result: its code is committed by the host after its turn ends. Do not introduce a
dependency on that future commit.

## Context and fixed decisions

Read `sase bead read bob-cli-8l -r "Need the current landing scope and notes"` and the
original plan through
`sase artifact read plan:202610/highlights_tab_close.md "Need the accepted close contracts"`.
The plan's resolved path is the original epic plan file whose frontmatter must be marked
`status: done` at closeout; resolve it again in your own workspace.

The detailed landing evidence is `file:explicit:639045a6614971c0076beb44`; read it
through `sase artifact read` if more source-audit detail is needed. The pre-handoff epic
note records the completed follow-up triage: `sase bead +1 bob-cli-8c` succeeded with
this evidence reference, attributed to proposing bead `bob-cli-8l.2` note #1.

All code lives in the linked `bob-mac-capture` repository. Open it with `/sase_repo`
(`sase repo open bob-mac-capture -r "Implement bob-cli-8l landing repairs"`) and read
its AGENTS.md. Use only the printed checkout path. No bob-cli feature code or canonical
memory change is planned.

The original four phases are closed. The audit reviewed these actual epic commits:

- `7fd75c8` (`bob-cli-8l.1`): RefsCore contracts, session close/quit, reopen stack,
  selection, copy, and portable tests. The phase note's earlier SHA was superseded by
  this SASE_BEAD-labelled final commit.
- `211c699` (`bob-cli-8l.2`): Accessibility close ladder and guarded terminate.
- `7954c28` (`bob-cli-8l.3`): close queue/model/router/actions/toast integration.
- `7ee2499` (`bob-cli-8l.4`): row control, motion, footer/inspector, design fixtures,
  and README. This was the audited linked-repo master and origin/master.

Keep the final DECISIONS: `library_rows=tabs_only` and `reopen=reopen`. Ctrl-X only
closes in Tabs; library rows gain no background AX read or open-tab glyph. Keep
Shift-Command-T reopening from the capped in-memory stack without recording an open. The
epic's user note #1 overrides its old Ctrl-D wording: Ctrl-X closes, Ctrl-D continues
the existing preview half-page scroll/reveal behavior everywhere.

The post-start Mac changes are `f996588` (Capture start/close-card layout, independent)
and `936f232` (Link to Today hardening, relevant to step 4). The bob-cli drift through
`07fef5f0` consists of article formatting/adoption, note-snapshot rendering/create, and
duplicate Pomodoro destination tracking (`e646cbd8`, `16c4640a`, `eeeeab0f`, `93a76ed3`,
`07fef5f0`). Those paths require no tab-close integration: the feature does not invoke
bob or mutate the vault/open log. Recheck any newer drift before closing, preserving
existing changes rather than resetting files to the cited SHAs.

The audit read implementation and tests but did not execute new Swift verification.
Phase notes report passing portable-check plus Mac bundle/test typecheck. The Mac probe
recorded only AX shapes: a tab has an AXButton child with nil subrole and a window
close-button attribute. Keep that supported ladder shape.

## 1. Move closing to Ctrl-X across every entry point

Change `Sources/BobMacCapture/Refs/RefsKeyRouter.swift` to use X (key code 7) for
`.closeSelectedTab`, only when `selectionIsOpenTab` is true. D (key code 2) always
returns `.scrollInspector(.down)` for the existing Control-D context. IME composition
passes through. Preserve Ctrl-U and Shift-Command-T/Command-T behavior. Outside a
closable Tabs row, Ctrl-X passes through; a stale `.closeSelectedTab` dispatch is a
consumed no-op and never becomes a preview scroll or another tab close.

Update the `.closeSelectedTab` doc comment and fallback in `RefsPanelModel.swift`,
`.closeTab` key equivalent in `RefsActionsMenu.swift`, `RefsRowView.closeTabHelp`,
`RefsFooter` Tabs variants, and `RefsInspectorView.tabFallbackHint`. Audit README's
panel key table, Tabs subsection, actions menu, troubleshooting, and Bob Refs shortcuts
table. Change only closing references to Ctrl-X; keep all Ctrl-D scrolling instructions.
Preserve plain X typing and Command-X text editing.

Update router/model/menu/footer/views assertions and fixtures. Tests must show: Ctrl-X
closes a selected Tabs row; it does not close in the library or without a closable row;
Ctrl-D scrolls with and without a selected Tabs row; IME and unrelated modifier variants
do not close; a stale close command does not scroll. Search for `⌃D`, `Ctrl-D`,
`Control-D`, and closing key-equivalent `d` to ensure no UI still advertises closing
with D. Do not blindly rewrite existing inspector scroll docs.

## 2. Stop the close worker when the FIFO is empty

`RefsPanelModel.startCloseWorkerIfNeeded` currently checks only `closeWorker == nil`.
Its task drains `closeQueue`, clears its handle, and calls itself unconditionally. That
schedules an endless chain of empty MainActor tasks after the first close.

Require nonempty queued work before starting a worker, and leave the handle nil when it
has drained. Retain FIFO, at most one reader close at a time, and queue survival across
hide/show. Keep state settled for refreshes superseded by a close, without incorrectly
treating a still-running fresh read as finished.

Add a meaningful regression to the existing fake-reader model harness that observes the
queue reaching idle (a narrow internal test seam is fine), then enqueues a second batch
and proves it runs once in order. Preserve rapid-close neighbor selection,
hidden-outcome behavior, and closing-row filtering against in-flight refreshes. Do not
rely only on reader call counts: the old endless empty tasks make no additional reader
calls.

## 3. Detect save sheets before the close deadline is exhausted

In `AXHighlightsTabsReader.swift`, `verifyClosed` returns nil only when its deadline has
elapsed. `closeOnQueue` then calls `needsAttentionOrFailed` with the same deadline; the
first `readSnapshot -> requiredAttribute -> checkDeadline` throws before it can inspect
any AXSheet. A save sheet therefore receives the generic failed outcome.

Make the bounded verification path inspect for the target's save sheet while it still
has time. Prefer checking the fresh snapshot/window records during each poll and
returning `.needsAttention` immediately after the existing activate/raise handoff. An
explicitly reserved part of the same total budget is also valid, but do not restart a
full deadline on each poll or add an unbounded read. Retain verified title-count
decrease, exited-app handling, ~50ms polls, approximately 2-second total close budget,
and name-free signposts. Do not press close a second time on timeout.

Factor only the minimum orchestration needed to inject scripted reads, time, and
sheet/activation effects for deterministic tests; retain all real `AXUIElement*` calls
in this reader file. Exercise the actual production verification path with fake state:
close succeeds after polls; target remains with a sheet and yields needsAttention with
one activate/raise; no sheet reaches a bounded failure; the app exits and yields an
empty closed snapshot. A pure helper test disconnected from the reader's control flow is
insufficient. Never use live Highlights actions to test it.

Also repair `Tests/BobMacCaptureTests/AXHighlightsTabsReaderTests.swift`: the unknown
bundle test asserts `.notRunning` unconditionally even though trust is checked first.
Assert the documented trusted/untrusted outcomes, close matching read rather than the
protocol default, `quitIfNoDocuments == true`, and no process launch. Inject trust if
useful, or constrain the environment-dependent result to exactly `.notRunning` /
`.notTrusted`; do not weaken it to accept arbitrary failures. This test was expanded by
the epic and must work on ungranted CI hosts.

## 4. Integrate closing with Rename and Link to Today writes

Commit `936f232` hardened Link to Today and its card/write coordination. Existing
`perform(.showActions)`, `.select`, and `.activate` guard `isRenaming` and
`isLinkingToday`; the new close commands and `closeTab(id:)` do not. The row button and
VoiceOver close action bypass the controller's card-key routing.

Apply the same in-progress-write exclusion at the model dispatch/mutation boundary so
keyboard, row button, VoiceOver, and actions-menu dispatch cannot close a tab, change
selection, or hide the panel during either write. Reflect unavailability in close
eligibility/control presentation where appropriate. Before an allowed close, clean up
non-running cards consistently with the surrounding row/action behavior; do not cancel
an in-flight vault write. Preserve the existing Link to Today listing, preview
cancellation, overlay generations, refresh invalidation, and notifications.

Extend model tests using the existing pending Rename/Today-link fixtures: direct
`.closeTab(id:)` and `.closeSelectedTab` produce no reader call or optimistic removal
while each write is running, then closing works after settlement. Keep synthetic Tabs
rows closable and keep the library decision unchanged.

## 5. Verify the integrated result

- Follow bob-mac-capture AGENTS.md: lint first and run `just portable-check`. On this
  Linux host, use the documented Swift-equipped tailnet host if Swift is absent. Read
  the `tailnet.md` reference through `/sase_memory_read` before remote access.
- Any change to `Sources/BobMacCapture*` or `Tests/BobMacCapture*` must be compiled in a
  temporary Mac scratch copy, best effort: rsync excluding `.build` and `.git`, run
  `Scripts/bundle.sh` and `just typecheck-tests`, then remove that copy. Never use
  Bryan's checkout, run `just install`, or alter the live Highlights session. The Mac
  Command Line Tools lack XCTest: typechecking is not test execution. If the Mac is
  offline, report the macOS targets as unverified, per AGENTS.md.
- On an available Xcode test host, run the focused router, model, reader, footer, and
  views XCTest suites against this tree. Do not gate this tale's closeout on its own
  future commit or CI result. Keep the portable tests meaningful if polling
  orchestration is moved into Foundation-only code.
- Run `just check` in bob-cli for the final landing verification. Do not run
  `just check-full`. A check-full-only failure is test infrastructure, not epic work.
  Use `/sase_monitor` for commands requiring a handoff, with instructions to resume this
  tale's remaining verification and closeout; never end with a live command.
- Review the final diff, updated shortcut copy, and newer commits since the audit.
  Record actual commands/outcomes, distinguishing executable tests from typechecks.
  Bryan's GUI smoke test is still manual; use Ctrl-X in its close steps and verify
  Ctrl-D scrolling in both Tabs and library. Do not claim that smoke test ran.

## 6. Finish epic bob-cli-8l closeout in this tale

The landing agent has already triaged every proposed follow-up. There is exactly one,
`bob-cli-8l.2` note #1: pending Accessibility policy task `bob-cli-8c` must include
close-button AXPress, the bounded select-close-restore fallback, and fresh-zero-doc
guarded terminate. It is corroborated on that existing task with source evidence; the
outcome is recorded on `bob-cli-8l`. Do not create a duplicate or edit canonical memory
as part of this tale. Include that triage outcome in the epic close note.

Re-read all four phase beads and their notes (`bob-cli-8l.1` through `.4`), the epic
notes, and its linked plan; reconcile any newer note or descendant. Confirm the user
shortcut correction and all defects above are addressed and the fixed DECISIONS remain
satisfied. If this tale is registered as an open descendant, finish and close that
completed tale bead normally before closing the epic (or close both atomically with
`sase bead close <tale-id> bob-cli-8l --note "<verified work>"`). Never close unfinished
descendants or wait for the host's later code commit.

Run `sase bead epic-symbols bob-cli-8l`. For every returned Justfile entry, resolve the
symbol by wiring it, making it private, adding an appropriate non-test pragma, or
deleting it; re-key an exemption only to a still-open later bead that genuinely needs
it. Repeat until none remain. The initial audit found none.

Close normally with:

```sh
sase bead close bob-cli-8l --note "Verified all four phase scopes and notes against 7fd75c8, 211c699, 7954c28, and 7ee2499 and the repaired source. Preserved tabs_only and reopen decisions. Addressed Bryan's Ctrl-X correction, idle-worker restart, save-sheet deadline, trust-dependent test, and Rename/Today write-guard integration. Reviewed non-epic f996588/936f232 and subsequent drift. Follow-up from bob-cli-8l.2 #1 was corroborated on existing bob-cli-8c; no duplicate or memory edit. [Replace this bracket with actual verification commands/results and any Mac limitation.] No epic-symbol exemptions remain."
```

Replace the bracket with actual evidence. If close is rejected for exemptions, finish
their cleanup and retry normally. If named phases/descendants are unfinished, complete
or reopen the relevant work. Never use `--force` merely to succeed or to advance
successful nested landing. Only a deliberate canceled/superseded outcome may use force
with an explicit reason; that is not expected here.

After closing, run `just symvision` wherever that recipe is available. The audited
bob-cli and bob-mac-capture justfiles have no such recipe, so check current availability
and record that fact if still absent. Set `status: done` in the frontmatter of
`plan:202610/highlights_tab_close.md`, using its resolved PLAN path.

Finally run `sase bead read bob-cli-8l -r "Need the parent link"`. The audit found **no
parent bead**, so no parent closure is currently required. If a parent has since been
added, follow the original landing rules: verify and normally close only a parent phase;
for a parent plan recheck its prior landing note, all descendants and notes, linked
plan, and post-child drift before retiring exemptions, closing, running available
symvision, and marking its plan done. Repeat only through fully complete plan ancestors;
stop and note any incomplete or ambiguous parent.

Use `/sase_final` as the last normal-turn action to declare the linked-repo code and any
other changed repository for host commit. Report the epic closure and actual
verification limits. The host commit happens after this closeout, by design.
