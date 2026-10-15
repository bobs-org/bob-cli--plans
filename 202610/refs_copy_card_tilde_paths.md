---
tier: tale
title: Refs Copy card prefers ~ for home paths
goal:
  Every text path the ⌘Y Copy card copies or previews shortens the home directory to ~
  when it can.
size: small
decisions:
  pdf_option:
    ask: Now that P copies the ~ PDF path, should Option-P copy the absolute PDF path?
    choices:
      absolute:
        Option-P copies the absolute /Users/… PDF path, keeping one opt-out from ~
      none:
        Option-P copies the same ~ path; no card row copies an absolute home path as
        text
    default: absolute
    why: P copies ~ either way; keeping the absolute form behind Option costs nothing
    answer: absolute
proposed_by: bbugyi200.apollo.7x
decided_by: auto
create_time: 2026-10-15 17:46:48
status: wip
---

# Plan: Refs Copy card prefers `~` for home paths

## Goal

In Bob Mac Capture's Bob Refs Copy card (opened with ⌘Y), any **text** that is a
filesystem path should replace the user's home directory with `~` whenever it can. Today
`P` copies `/Users/bryan/bob/lib/…pdf` (only Option-P gives `~/bob/lib/…`), and Option-N
copies the absolute note path `/Users/bryan/bob/ref/…md`. After this change both copy
`~/bob/…`.

"When it can" means the path is the home directory itself or sits under `<home>/`. A
path outside home (for example a vault on `/Volumes/…`), or a card opened with no home
directory, keeps its absolute path. The card must never show a duplicate or misleading
`~` variant in that case.

## Scope

All changes are in the linked repo **bob-mac-capture**. Open it with your `/sase_repo`
skill (`sase repo open bob-mac-capture -r "<reason>"`) and work only in the path it
prints. Read its `AGENTS.md` first.

What changes (the copy card's catalog lives in `Sources/RefsCore/RefsCopy.swift`;
`RefsCopyCatalog.init` builds every row):

| Key            | Today                                       | After                                                                                                        |
| -------------- | ------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `P` (primary)  | absolute PDF path                           | `~` PDF path, or absolute when it can't be shortened                                                         |
| Option-P       | `~` PDF path (`PDF path · ~`)               | see **pdf_option** below                                                                                     |
| `N` (primary)  | vault-relative note path                    | unchanged (`bob ref show` needs it, as the shared-marker help says)                                          |
| Option-N       | absolute note path (`Note path · absolute`) | `~` note path (`Note path · ~`), or absolute with the old label when it can't be shortened                   |
| `F` / Option-F | file objects                                | payload unchanged (pasteboard file URLs must be absolute); the Option-F narration tooltip shows the `~` path |
| `O`            | `obsidian://open?path=<absolute>`           | unchanged: Obsidian's `path` parameter needs a real absolute path                                            |

The confirmation HUD (`RefsPanel.copySucceeded`) and the VoiceOver label already show
`row.preview`, so they follow the new previews automatically. Leave the inspector, the
⌘K actions menu, and Copy Diagnostic alone: they are outside copy mode.

## Implementation

### 1. `Sources/RefsCore/RefsCopy.swift`

1. In `RefsCopyFormat`, add a small helper next to `abbreviatedPath`, for example
   `homeAbbreviatedPath(_ path: String, homeDirectoryPath: String?) -> String`. It
   returns `abbreviatedPath(path, homeDirectoryPath:)` when a home path is given and
   `path` otherwise. Keep `abbreviatedPath` as is: it already handles a trailing-slash
   home, an empty home, `path == home` → `~`, and sibling prefixes like `/Users/test2`.
2. **Note path row (`.notePath`).** Keep the primary value `item.id`. Build the
   alternate from `absoluteNotePath` through the helper:
   - When shortened (result differs from the absolute path): label `Note path · ~`,
     payload and preview the `~` path, confirmation `Copied Note Path · ~`.
   - Otherwise: keep today's alternate exactly (`Note path · absolute`, absolute
     payload, `Copied Absolute Note Path`).
   - Keep today's behavior of building an alternate whenever `absoluteNotePath` is
     non-nil.
3. **PDF path row (`.pdfPath`).** Make the primary `value` the shortened path, keeping
   label `PDF path`, the `.missing` tag rule, and confirmation `Copied PDF Path`.

   > [!decision] pdf_option = absolute Only when the shortened path differs from
   > `absolutePDFPath`, add an alternate with label `PDF path · absolute`, payload and
   > preview `absolutePDFPath`, and confirmation `Copied Absolute PDF Path`. When the
   > path can't be shortened, leave the alternate `nil`, so Option-P falls back to the
   > primary like other rows without an alternate.

   > [!decision] pdf_option = none Give the `.pdfPath` row no alternate at all. Option-P
   > copies the primary `~` path.

   Either way, delete the old `PDF path · ~` / `Copied PDF Path · ~` alternate. It
   duplicated the primary row when the path wasn't under home, which was a latent bug.

4. **PDF file row (`.pdfFile`).** Leave the payload as
   `.fileURL(URL(fileURLWithPath: absolutePDFPath))` and the narration alternate's
   payload as is. For the tooltip, add an optional `help: String?` to
   `RefsCopyAlternate`. Make it the last init parameter, default `nil`, so
   `RefsCopyFormat.quoteAlternate` and other callers still compile. In
   `RefsCopyRow.resolved(alternate:)`, prefer `alternate.help` when it is set and
   otherwise keep the current derivation (text value / `url.path`). Pass the narration
   alternate
   `help: RefsCopyFormat.homeAbbreviatedPath($0.path, homeDirectoryPath: homeDirectoryPath)`.
5. Leave `hiddenReason`, the key map, `RefsCopyCard` state handling, and row order
   unchanged. `RefsCopyCard` already forwards `homeDirectoryPath` into every catalog
   rebuild. `RefsPanelModel.openCopyCard` already passes
   `FileManager.default.homeDirectoryForCurrentUser.path`, so it needs no change.

Match the file's existing style (trailing-closure `map`s, `valueRow` for text rows,
swift-format layout).

### 2. Tests

`Tests/RefsCoreTests/RefsCopyAlternatesTests.swift` (portable; `makeCard()` uses home
`/Users/test` with vault `/Users/test/bob`):

- In `testAlternatesKeepEveryRowInPlaceAndPreviewExactPayloads`, update the expected
  alternates. `.notePath` becomes `.text("~/bob/" + item.id)`. `.pdfPath` becomes
  `.text("/Users/test/bob/lib/a.pdf")` under `absolute`; under `none`, drop that entry
  and assert instead that the `.pdfPath` row has no alternate and resolves to itself.
- Add `testHomePathsCopyWithTilde`:
  - `P` primary payload, preview, and `handle(.key(.pdfPath))` write are
    `~/bob/lib/a.pdf`, and the label and confirmation are as above.
  - Option-N's label is `Note path · ~` and its confirmation is `Copied Note Path · ~`.
  - The `.pdfFile` primary payload is still `.fileURL` of the absolute
    `/Users/test/bob/lib/a.pdf`.
  - The narration alternate's resolved `help` is `~/bob/lib/a.m4a`.
  - The Obsidian link `path` query item is still absolute.
  - Under `absolute`, Option-P's label and confirmation are `PDF path · absolute` and
    `Copied Absolute PDF Path`.
- Add `testHomePathsStayAbsoluteWhenTheyCannotBeShortened`. Cover three cases: a vault
  outside home (`/Volumes/vault/…` with home `/Users/test`), a sibling-prefix home
  (`/Users/test2/…` with home `/Users/test`), and `homeDirectoryPath: nil`. In each
  case:
  - `P` copies the absolute path, and the `.pdfPath` row's `alternate` is `nil`.
  - Option-N's payload is the absolute path, with label `Note path · absolute` and
    confirmation `Copied Absolute Note Path`.
- Keep `testTitledLinksAndHomeAbbreviationBoundaries` passing. Add a line or two for the
  new helper's `nil`-home passthrough.

`Tests/RefsCoreTests/RefsCopyTests.swift`: these catalogs use `/vault/…` paths with no
home, so they should pass unchanged. Confirm they do; don't weaken them.

`Tests/BobMacCaptureTests/RefsPanelModelTests.swift`, in
`testCardFreezesResolvedNoteAndExistingNarrationPaths`: the harness vault lives under
`FileManager.default.temporaryDirectory`, which may or may not be under home. Compute
the expected Option-N payload as
`RefsCopyFormat.abbreviatedPath(<absolute note path>, homeDirectoryPath: FileManager.default.homeDirectoryForCurrentUser.path)`
instead of the raw absolute path. Add the same kind of assertion for the `.pdfPath`
primary payload.

`Tests/BobMacCaptureTests/RefsCopyCardDesignTests.swift`: add one more fixture,
`try write(card: alternatesCard(), name: "refs-copy-card-home-paths")` with alternates
off. The primary `~` previews then get a rendered light/dark review alongside the
existing `refs-copy-card-alternates` fixture.

### 3. README

Update the `### Copy card` section of `README.md`:

- Add one sentence after the fixed-key list: paths under your home folder copy with `~`
  in place of it (for example `~/bob/lib/…`). Only the Obsidian link and the PDF and
  narration file copies stay absolute, because they need a real path.
- Rewrite the Option sentence. Option-N now copies the note's full `~` path. Under
  `absolute`, Option-P copies the absolute PDF path; under `none`, remove Option-P from
  that sentence. Keep "Each row previews the exact value it copies." true.

## Verification

Per bob-mac-capture's `AGENTS.md`, never land Swift you have not compiled. This host has
no `swift`.

1. **Portable gate (required):** rsync a scratch copy of the opened repo to `athena`,
   excluding `.build` and `.git`, under a fresh temp directory. Then run
   `just portable-check` there: swift-format lint, `swift build --build-tests`, and
   `swift test` for CaptureCore/RefsCore. Fix everything until it is green, then delete
   the scratch copy. `athena` has Swift 6 and `just`.
2. **macOS targets (best effort):** the `BobMacCaptureTests` edits can only be
   type-checked on the Mac. rsync a scratch copy (no `.build`/`.git`) to `mac` (SSH user
   `bbugyi`) and run `./Scripts/bundle.sh` and `just typecheck-tests` there, then delete
   the copy. Never use Bryan's checkout or `just install` on the Mac. If the Mac is
   offline, say in the final report that the macOS targets are unverified.
3. **Behavior check:** on a card for a ref in the default `~/bob` vault:
   - `P` previews and copies `~/bob/lib/…pdf`.
   - Option-N previews and copies `~/bob/ref/…md`.
   - Under `absolute`, Option-P previews and copies `/Users/…/bob/lib/…pdf`.
   - `O`, `F`, and Option-F are unchanged.

   The RefsCore tests above cover this. Manual app testing is optional.

## Out of scope

- Normalizing symlinked or non-canonical vault roots, or a literal `~` typed into the
  Settings Bob directory: paths are compared exactly as `RefsLibrary` builds them.
- Any change to bob-cli or the `bob ref` JSON contract. Copy formatting stays app-local.
- Inspector, actions menu, and diagnostic text.
