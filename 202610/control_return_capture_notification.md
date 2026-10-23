---
tier: tale
title: Always post the capture-success notification on Control-Return
goal:
  Every successful Bob Mac Capture submit made with Control-Return posts the same macOS
  capture-success notification as Return, then opens the notes in Obsidian.
size: small
proposed_by: bbugyi200.apollo.a2
proposed_with: claude/opus
create_time: 2026-10-23 10:14:56
status: wip
---

# Plan: Always post the capture-success notification on Control-Return

## Goal

When Bryan submits a capture from Bob Mac Capture with Control-Return (capture, open
Bob's reported notes in Obsidian, close the panel), the app must post the same macOS
capture-success notification that a plain Return submit posts. The notification must be
sent on every successful Control-Return capture, whether or not Obsidian opens.

All work happens in the linked `bob-mac-capture` repository. Open it with the
`/sase_repo` skill (`sase repo open bob-mac-capture -r "<reason>"`) and use only the
path it prints. Read that repo's `AGENTS.md` before editing. bob-cli itself needs no
changes.

## Root cause

`Sources/BobMacCapture/CapturePanelModel.swift`,
`completeSubmit(requestID:response:openAfterCapture:draft:choiceSeed:processClientIdentity:)`,
success branch:

```swift
if openAfterCapture,
   let url = ObsidianOpenURL.openNotesURL(for: openPlan)
{
    noteOpener(url) { [weak self] error in
        guard let self, let error else { return }
        self.postCaptureOpenFailure(message: error.localizedDescription)
    }
} else {
    postCaptureSuccess(
        captures: captures,
        globalDestination: success.globalDestination,
        openPlan: openPlan
    )
}
panelDismisser()
```

`postCaptureSuccess` runs only on the `else` path, so a Control-Return submit that
resolves an Obsidian URL never posts the success notification. It posts only the
"Captured, but Obsidian didn't open" notification, and only when the open fails. Commit
`57406f8` ("feat(capture): add Control-Return open pipeline") introduced this. Before
that commit the success notification was posted on every submit, and the open happened
after it. `docs/settings.md` currently documents the skip ("A Control-Return open
success skips the capture-success banner").

Every Control-Return entry point goes through this one branch: the key router's
`.submitAndOpen`, the footer **⌃↩ Open** chip,
`acceptSelectedCompletionThenSubmitAndOpen`, picker and Add-block-ID `.submitAndOpen`
follow-ups, and the successor (pull-in) choice card's
`confirmPullInChoice(openAfterCapture: true)`. Each reaches `submit` → `dispatchCapture`
→ `completeSubmit`, so fixing this branch covers all of them.

## Changes

### 1. Post the success notification unconditionally (`CapturePanelModel.swift`)

In the success branch of `completeSubmit`, replace the `if … else` above so that:

1. `postCaptureSuccess(captures:globalDestination:openPlan:)` is always called first,
   with the same arguments as today. The content is identical to the Return path,
   including Bob's open-plan payload, so clicking the banner or its open action still
   sends the `bob-open-notes` URL.
2. Then, only when `openAfterCapture` is true and
   `ObsidianOpenURL.openNotesURL(for: openPlan)` returns a URL, call `noteOpener(url)`
   as today. If it reports an error, still call
   `postCaptureOpenFailure(message: error.localizedDescription)`, which now arrives as a
   follow-up to the success notification. Its existing title, "Captured, but Obsidian
   didn't open", stays accurate.
3. `panelDismisser()` still runs exactly once afterwards.

Expected shape:

```swift
postCaptureSuccess(
    captures: captures,
    globalDestination: success.globalDestination,
    openPlan: openPlan
)
if openAfterCapture,
   let url = ObsidianOpenURL.openNotesURL(for: openPlan)
{
    noteOpener(url) { [weak self] error in
        guard let self, let error else { return }
        self.postCaptureOpenFailure(message: error.localizedDescription)
    }
}
panelDismisser()
```

Do not change `NotificationService` content builders, categories, routing, or the
failure-path notifications. Out of scope: read-only navigation opens that capture
nothing, such as Task Go-To (`/`, `taskGotoOpener`) and the similar-tasks row open. Refs
notifications are also out of scope.

### 2. Update the model tests (`Tests/BobMacCaptureTests/CapturePanelModelTests.swift`)

- Rename `testSuccessfulOpenSuppressesTheCaptureSuccessNotification` to
  `testSuccessfulOpenStillPostsTheCaptureSuccessNotification`. Assert:
  - exactly one `captureSuccessNotifier` call;
  - its `CaptureOpenPlan` is non-empty;
  - `noteOpener` ran once;
  - no `captureOpenFailureNotifier` call.

  To pin the ordering, append `"notify"` and `"open"` events to a shared array from the
  two seams and assert `["notify", "open"]`.

- Rename `testFailedOpenPostsCaptureOpenFailureInsteadOfSuccess` to
  `testFailedOpenPostsCaptureSuccessAndOpenFailure`. Assert one success notification and
  `openFailure == "sandbox denied"`.
- Leave `testEmptyBobOpenPlanDoesNotOpenObsidianAndKeepsSuccessNotification` and
  `testSuccessfulSubmitAndOpenDismissesPanelExactlyOnceAndOpensTarget` unchanged. Both
  must still pass.

### 3. Cover the successor-choice Control-Return route (`Tests/BobMacCaptureTests/CapturePullInChoiceModelTests.swift`)

In `testMarkedSubsetDispatchesOpaqueKeysOnceAndKeepsOpenAfterIntent`, which commits
through `controller.perform(.confirmPullInChoiceAndOpen)`, install a
`model.captureSuccessNotifier` counter before the commit and assert it fired exactly
once alongside the existing `opened.count == 1` assertion. This file uses 2-space
indentation; match it.

### 4. Correct the docs (`docs/settings.md`, "Notification copy" section)

Replace the sentence "A Control-Return open success skips the capture-success banner; an
open failure posts "Captured, but Obsidian didn't open" with the error." with wording
that states the new behavior: Control-Return posts the same capture-success notification
as Return and then opens the notes in Obsidian; if the open fails, a second
notification, "Captured, but Obsidian didn't open", carries the error. This is the only
docs sentence that describes the skip; recheck with
`grep -rn -i "capture-success banner\|didn't open" docs README.md`. bob-cli's
`docs/reference/mac-capture/` copy is a gitignored mount of these pages, so do not edit
it there.

## Verification

Follow bob-mac-capture's `AGENTS.md`; never land Swift that has not been compiled.

1. `just portable-check` in bob-mac-capture. It runs swift-format lint over `Sources`
   and `Tests`, then `swift build --build-tests` and `swift test` for the portable
   targets. If the local host has no `swift`, run it on athena; see the `tailnet`
   reference memory for access.
2. `portable-check` cannot type-check `Sources/BobMacCapture*` or
   `Tests/BobMacCapture*`, and both are changed here. Compile a scratch copy on the mac
   (best effort):
   1. rsync the repo there, excluding `.build` and `.git`;
   2. run `Scripts/bundle.sh` and `just typecheck-tests` in the copy;
   3. delete the copy.

   Never use Bryan's checkout or `just install`. If the mac is offline, say in the final
   report that the macOS targets are unverified. CI runs the XCTest suite.

3. `just docs-check` in bob-mac-capture, because `docs/` changed.

## Commit

Commit the bob-mac-capture changes in that repository, for example
`fix(capture): always post the capture notification on Control-Return`. No bob-cli,
bob-plugins, or memory changes are needed.
