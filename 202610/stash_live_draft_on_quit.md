---
tier: tale
title: Stash the live Mac Capture draft when the app quits
goal:
  An unsent Bob Mac Capture draft survives the app restart that just install-all
  performs, and also menu Restart and Quit, by landing in the canceled-draft stash.
size: medium
decisions:
  restore:
    ask: Should the relaunched app put the stashed draft straight back in the editor?
    choices:
      stash: Draft waits at the top of the stash; Control-S plus one key restores it
      editor:
        Next hotkey shows the draft already in the editor; adds a stash flag and launch
        restore
    default: stash
    why:
      It is the requested fix, reuses the Control-S restore, and adds no launch-time
      state
    answer: stash
proposed_by: bbugyi200.athena.155
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-28 09:36:48
status: wip
---

# Plan: Stash the live Mac Capture draft when the app quits

## Problem

`just install-all` (bob-cli `scripts/install_all` → `mac_capture_step`) runs
bob-mac-capture's `just install`. After the new bundle verifies, `Scripts/install.sh`
runs `BobMacCaptureInstallHelper restart`. `InstallRelauncher.restart` calls
`NSRunningApplication.terminate()` on the running copy, waits up to 10 s for it to exit,
then opens the new bundle. `terminate()` sends a normal quit Apple event, so the old
process runs `AppDelegate.applicationWillTerminate`. That method never touches the
editor draft. The draft lives only in memory (`CapturePanelModel.attributedDraft`), so
any unsent text is lost. This holds whether the panel was open mid-typing or hidden with
an Escape-retained draft.

The same loss happens on **Quit Bob Mac Capture** (Cmd-Q or the menu) and on **Restart
Bob Mac Capture** (`AppRelauncher` → `NSApp.terminate`). The docs promise it today: "an
unsent draft is discarded" (`docs/install.md`, `docs/capture-panel.md`).

Background: the plan that added Restart (plan `202608/bob_mac_capture_restart.md`)
listed "Do not persist the capture draft to disk" as a non-goal, on the grounds that
"drafts are deliberately never persisted". Plan `202608/persist_canceled_draft_stash.md`
then made the canceled-draft stash a write-through file in Application Support. That
already put captured text on disk under a reviewed privacy contract. This plan reverses
the old non-goal at Bryan's request. The live draft goes into that same stash, so no new
store is needed.

## Approach

The terminating process pushes the live draft onto the existing canceled-draft stash
from `applicationWillTerminate`. It uses the same rules as Control-C. The stash is
write-through (a synchronous atomic write, about 1 ms), and the install helper waits for
the old PID to exit before it opens the replacement. So the new process always loads a
stash that already holds the draft.

This applies to **every graceful quit**: install restart, Restart, Quit, Cmd-Q, and
logout. It does not apply to install restarts only, for two reasons:

- All of these paths arrive as the same `NSApplication` terminate. The app cannot
  reliably tell the install helper's quit event from the user's.
- Closing the panel is already never destructive. **Discard** is the only permanent
  discard (see the doc comment on `CapturePanelModel.prepareForRetainedClose`).

Rules, mirroring `stashDraftAndClose`:

- Only a semantically nonempty draft is stashed (`hasDraft`: whitespace-only text is
  dropped).
- A Task Go-To (`/`) draft is not stashed, matching Control-C's documented behavior.
- Stashing off (capacity 0) stores nothing. `CanceledDraftStash.push` already returns
  nil, and the user has opted out of drafts on disk.
- The exact text is stored. There is no dedupe; repeated identical stashes are already
  separate entries by contract.
- A draft whose capture is still in flight (`isSubmitting`) is stashed anyway. A
  possible duplicate entry in the stash is better than losing text, and restore is
  always explicit.
- No UI work happens at termination: no status text, no dismissal, no clearing the
  editor, no analysis. The process is exiting.

Out of scope: force quit, crash, `SIGKILL`/`SIGTERM`, and power loss still lose an
unsent draft. Covering those would need continuous draft autosave, which is a separate
feature. Text typed into inline prompts (block ID, Pomodoro name, relation label) and
picker filters is not draft text and is not stashed.

## Implementation

Open bob-mac-capture with the `/sase_repo` skill (`sase repo open bob-mac-capture`) and
read its `AGENTS.md` first. The doc change in step 6 is in this bob-cli checkout.

### 1. `CapturePanelModel` (bob-mac-capture `Sources/BobMacCapture/CapturePanelModel.swift`)

- Move the Task Go-To test in `stashDraftAndClose`
  (`picker?.source == .taskGoto || pickerChip?.source == .taskGoto || lastAnalysisNeedsTaskGoto`)
  into a private computed property, for example `draftIsTaskGoto`. `stashDraftAndClose`
  keeps its exact behavior and status strings.
- Add a termination entry point next to `stashDraftAndClose`, for example:

  ```swift
  /// Called once from applicationWillTerminate so Quit, Restart, and an install
  /// restart never drop unsent text. It only pushes onto the write-through stash: the
  /// process is exiting, so no status, dismissal, clearing, or analysis runs.
  @discardableResult
  func stashDraftForTermination() -> CanceledDraftEntry? {
      guard hasDraft, !draftIsTaskGoto else {
          return nil
      }
      return canceledDraftStash.push(plainDraft)
  }
  ```

  Match the file's 4-space, 100-column style, and format only the lines you touch.

### 2. `AppDelegate` (`Sources/BobMacCapture/AppDelegate.swift`)

- In `applicationWillTerminate`, call `panelModel?.stashDraftForTermination()`
  **first**, before the hotkey, watcher, Refs, and process-client teardown. That way a
  teardown problem cannot skip the stash.
- Rewrite the comment there. The stash is still write-through, so the only
  terminate-time work is pushing the live editor draft. Force quit, a crash, and
  `SIGKILL` still lose an unsent draft. Keep the warning against a terminate-only stash
  flush.
- Go through `panelModel` (which holds the shared stash). Never touch the delegate's
  lazy `canceledDraftStash` here. Under `swift test` that lazy property would build the
  real file-backed stash.
- Change `private var panelModel` to an internal `var`, like the neighboring
  `hotKeyRegistry` and `refsLibrary`, so a test can inject a model.

### 3. UI copy

- `Sources/BobMacCapture/CapturePanelView.swift` (Stash button `.help`, currently
  "Restore a draft canceled with Control-C (Control-S)."): say that the stash also holds
  a draft left unsent at quit or restart. Example: "Restore a draft canceled with
  Control-C or left unsent at quit (Control-S)."
- `Sources/BobMacCapture/SettingsView.swift` (Canceled Draft Stash footnote, "Canceled
  drafts are stored on disk and survive quitting or restarting. …"): add that quitting
  or restarting with an unsent draft stashes it.

### 4. Tests (bob-mac-capture)

In `Tests/BobMacCaptureTests/CapturePanelModelTests.swift`, next to the
`stashDraftAndClose` tests, use an in-memory `CanceledDraftStash(capacity: 10)`:

- A nonempty multi-line draft with surrounding whitespace is pushed with exact text. The
  model's `plainDraft`, `statusText`, and the `panelDismisser` count are unchanged, and
  the return value is the pushed entry.
- A whitespace-only draft and an empty draft push nothing and return nil.
- Capacity 0 pushes nothing.
- A draft that is already in the stash (an earlier Control-C) adds a second entry,
  newest first.
- A draft with `templateOrigin` set is pushed with its text.

In `Tests/BobMacCaptureTests/CaptureTaskGotoPanelTests.swift`, next to the existing
Control-C Go-To test:

- A Task Go-To draft is not pushed.

In `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`, next to the other
`AppDelegate()` tests:

- Construct `AppDelegate()` and inject a `CapturePanelModel` built with an in-memory
  stash. Set a draft, call
  `applicationWillTerminate(Notification(name: NSApplication.willTerminateNotification))`,
  and assert the stash holds the draft.
- With no model injected, the same call does not crash.

A round-trip through `FileCanceledDraftStashStore` already has coverage in
`Tests/CaptureCoreTests/CanceledDraftStashStoreTests.swift`. Add a store-backed test
only if you change CaptureCore.

> [!decision] restore = editor Add this on top of steps 1–4 when the reviewer picks
> `editor`.
>
> - CaptureCore `PersistedCanceledDraft` gains `restoreOnLaunch: Bool?`, and its public
>   init takes a defaulted parameter. Encode the field only when it is true, so ordinary
>   entries and old files stay byte-identical. Keep `schemaVersion` 1: synthesized
>   Codable decodes a missing optional key as nil, and older builds ignore the unknown
>   key. Add decode/encode tests in `CanceledDraftStashStoreTests.swift` that run under
>   `just portable-check`.
> - `CanceledDraftEntry` carries the flag, and `CanceledDraftStash.push` takes
>   `restoreOnLaunch: Bool = false`. `stashDraftForTermination` passes `true`. Add
>   `CanceledDraftStash.takeRestoreOnLaunchEntry()`. It returns the newest flagged
>   entry, removes that entry, clears the flag on any other entries, and persists once.
> - In `applicationDidFinishLaunching`, right after `panelModel = model`, call a new
>   `CapturePanelModel.restoreDraftStashedAtQuit()`. With an empty editor, it installs
>   the taken entry's text with the caret at the UTF-8 end. It leaves the model in the
>   state an Escape close leaves it (hidden panel, `statusText` "Draft retained"), and
>   it does not schedule analysis. The next hotkey then behaves like any retained-draft
>   show. With a nonempty editor (which cannot happen at launch), it leaves the stash
>   untouched.
> - The model and stash tests cover the flag round-trip, take-and-clear, and restore
>   into an empty editor. A Control-C entry must never auto-restore.
> - Docs in step 5 say the relaunched app puts the draft back in the editor as a
>   retained draft. They also say it stays in the stash if the new build never launches.

### 5. Mac Capture docs (bob-mac-capture `docs/`, mounted into Bob's site)

These replace every "unsent draft is discarded" promise:

- `docs/install.md`, "Updating, reinstalling, and rollback". Replace "an unsent draft is
  discarded" with: a nonempty unsent draft goes onto the canceled-draft stash (a Task
  Go-To draft and capacity 0 excepted), so restore it with Control-S after the restart.
- `docs/capture-panel.md`:
  - In the Panel bullet, replace "Restart discards an unsent draft" with the same rule
    for Restart and Quit.
  - In the drafts paragraph near "Escape retains a nonempty draft", add that a graceful
    quit stashes the live draft exactly like Control-C, without clearing anything on
    screen.
  - In "Canceled drafts" and "Stash", count a quit-stashed draft as a stash entry. A
    force quit or crash still loses an unsent draft.
- `docs/settings.md`, Canceled Draft Stash note ("Canceled drafts live on disk and
  survive quit and restart"): add the quit-stash rule.
- `docs/permissions.md`: change "Canceled drafts retained with Control-C are stored in"
  to cover drafts stashed with Control-C or by quitting with an unsent draft. The
  storage, mode, and never-in-`UserDefaults` rules are unchanged.
- `docs/keyboard.md` needs no change. Its Control-C row stays accurate.

### 6. bob-cli guide (`docs/guides/recover-mac-capture.md`, step 1)

Step 1 says only a stashed draft can be recovered. Add one sentence: Quit, Restart, and
the restart that `just install` / `just install-all` performs stash a nonempty unsent
draft, so it is waiting in the footer **Stash** (Control-S). Following the docs routing
memory, leave `docs/guides/install-everything.md` unchanged; none of its steps become
wrong.

No `scripts/install_all`, `Scripts/install.sh`, or install-helper change is needed. The
helper's normal terminate already reaches `applicationWillTerminate`. Do not edit
`CHANGELOG.md` or any version field.

## Verification

1. bob-mac-capture on Linux: `just portable-check` (lint parse for every Swift file;
   build and test CaptureCore/RefsCore).
2. AppKit targets: run `Scripts/macos-ci-check` from the bob-mac-capture checkout
   through `/sase_monitor`, because it can take many minutes. It must pass every
   required macOS step (Lint, Build, Test, Bundle, plist/signature, Launch smoke,
   Install and reinstall). Never commit Swift that has not compiled. Also do the
   best-effort Mac scratch compile in bob-mac-capture's `AGENTS.md` (`bundle.sh` and
   `just typecheck-tests` on an rsynced copy; never Bryan's checkout or `just install`).
   If the Mac is offline, say so.
3. Docs: `just docs-check` in bob-mac-capture (mounted pages through bob-cli's gate) and
   `just docs-check` in bob-cli.
4. Commit both repositories.
5. In the final report, list these manual checks for Bryan on the Mac:
   - Type a multi-line draft, leave the panel open, and run `just install-all`. After
     the restart, Control-S shows the draft first. Restoring it brings back the exact
     text.
   - Repeat with the draft retained by Escape (panel hidden), then with menu **Restart**
     and **Quit** → relaunch.
   - A Task Go-To (`/`) draft and capacity 0 leave the stash unchanged.
