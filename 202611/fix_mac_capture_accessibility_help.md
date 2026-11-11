---
tier: tale
title: Fix bob-mac-capture build broken by invalid .accessibilityHelp modifier
goal:
  bob-mac-capture master compiles again, macOS CI is green, and just install works on
  the Mac.
size: small
decisions:
  fix_shape:
    ask: Should the invalid modifier line be deleted outright?
    choices:
      delete:
        Line 1598 is removed; the adjacent .help() already sets tooltip and
        accessibility hint.
      hint:
        Line 1598 becomes .accessibilityHint(...), repo convention, but the hint is then
        set twice.
    default: delete
    why:
      Apple docs say help(_:) configures the accessibility hint, so a hint modifier
      duplicates it.
    answer: delete
proposed_by: bbugyi200.apollo.dd
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-11 16:31:22
status: wip
---

# Fix bob-mac-capture build: invalid `.accessibilityHelp` SwiftUI modifier

## Problem

`just install` fails on Bryan's MacBook for the `bob-mac-capture` project. The Swift
build of `Sources/BobMacCapture/CapturePanelView.swift` dies with:

```
CapturePanelView.swift:1598:22: error: value of type 'ModifiedContent<ModifiedContent<some View,
CaptureFocusAdoption>, AccessibilityAttachmentModifier>' has no member 'accessibilityHelp'
```

Until this is repaired on `master`, every `just install` / `just install-all` run on the
Mac that pulls `bob-mac-capture` fails, and macOS CI is red for every subsequent push to
the repo.

## Root cause (confirmed)

Commit `4a69ed2` ("feat(capture): paint notebook section place menus from Bob chrome",
bead `bob-cli-ka.5`) added this chain to the capture draft `TextEditor` in
`CapturePanelView.swift` (lines 1597-1599):

```swift
.accessibilityLabel("Capture draft")
.accessibilityHelp(model.notebookSectionEditorHelp ?? "")
.help(model.notebookSectionEditorHelp ?? "")
```

`accessibilityHelp` is **not a SwiftUI View modifier** — it does not exist in any SDK.
It is the name of an AppKit `NSAccessibilityProtocol` method, which the authoring agent
evidently confabulated into a SwiftUI modifier. SwiftUI's real modifiers for this
purpose are `.help(_:)` (which per Apple's documentation "configures the view's
accessibility hint and its help tag, or tooltip, in macOS") and `.accessibilityHint(_:)`
(the convention used everywhere else in this repo). `git log -S accessibilityHelp`
confirms `4a69ed2` is the only commit that ever introduced the token, and it appears
exactly once in the repo — at the failing line.

The exact-SHA macOS CI run for `4a69ed2` **failed with the identical error** (run
43100764208, job "macOS 26 SwiftPM"), and it is the only compile error in the entire
failed log — everything else is pre-existing deprecation warnings. The phase worker
pushed to `master` without watching that run to green, violating the bob-mac-capture
`AGENTS.md` push gate ("After every push, watch the exact-SHA macOS run to green"),
which is how a non-compiling commit landed.

## Fix

All work happens in the `bob-mac-capture` linked repo. Open it with the `/sase_repo`
skill (`sase repo open bob-mac-capture -r "..."`) and use only the path it prints. Read
that repo's `AGENTS.md` before editing — the verification steps below come from it and
are mandatory.

1. In `Sources/BobMacCapture/CapturePanelView.swift`, apply the `fix_shape` decision to
   line 1598:
   - `delete` (default): remove the
     `.accessibilityHelp(model.notebookSectionEditorHelp ?? "")` line. The adjacent
     `.accessibilityLabel("Capture draft")` and
     `.help(model.notebookSectionEditorHelp ?? "")` lines stay exactly as they are.
   - `hint`: replace that line with
     `.accessibilityHint(model.notebookSectionEditorHelp ?? "")`.
2. Change nothing else. In particular, do not touch the deprecated `Text` `+`
   concatenation warnings (they predate this commit and are tracked separately) and do
   not act on the `PROPOSED FOLLOW-UP` note on bead `bob-cli-ka.5` (that belongs to the
   epic's land agent).

## Verification (per bob-mac-capture AGENTS.md — do not skip)

1. Read the tailnet reference memory first (`sase memory read tailnet.md -r "..."`) for
   SSH access conventions to the Mac.
2. Compile a scratch copy on the Mac, best effort: rsync the repo without
   `.build`/`.git` to a throwaway directory on the Mac, run `bundle.sh` and
   `just typecheck-tests` there, then delete the copy. Never use Bryan's checkout and
   never run `just install` on the Mac yourself. If the Mac is offline, say in the final
   report that the macOS targets are unverified locally and lean on CI.
3. Run `Scripts/macos-ci-check` from the bob-mac-capture repo before landing.
4. Land the one-line fix on `master` as a `fix(capture)` commit through the normal SASE
   commit flow for the opened linked repo.
5. After the push, watch the exact-SHA macOS CI run to green with `/sase_monitor`. The
   fix is not done until that run is green.

## Follow-through

- Read `sase/memory/sase_beads.md` first, then (following whatever note semantics it
  prescribes) add a note to bead `bob-cli-ka.5` recording that `4a69ed2` landed with red
  exact-SHA macOS CI (run 43100764208) and naming the hotfix commit that repaired
  `master`, so the epic's land agent knows why master moved under it.
- In the final report, tell Bryan that once CI is green he can rerun `just install` (or
  `just install-all`) on his MacBook to pick up the fix.

## Out of scope

- The macOS 26 `'+' was deprecated` warnings on `Text` concatenation
  (CapturePanelView.swift ~3030 and ~3850): introduced by earlier commits, warnings
  only, tracked as a separate task bead.
- The `place_menu` empty-notebook-section follow-up noted on bead `bob-cli-ka.5`: owned
  by the epic.
