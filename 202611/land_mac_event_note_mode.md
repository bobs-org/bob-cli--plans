---
tier: tale
size: medium
title: Finish and land Bob Mac Capture Event note mode (bob-cli-ji.7.1)
goal:
  "Close the gaps the bob-cli-ji.7.1 landing review found: existing saved notes open
  without Bob highlights, render fixtures show placeholders instead of the editor and
  picker rows, the status banner leaks a raw state token, and the epic grew two
  over-budget reference pages. Then close epic bob-cli-ji.7.1 and its parent phase
  bob-cli-ji.7."
proposed_by: bbugyi200.apollo.bob-cli-ji.7.1.land
proposed_with: claude/opus
bead: bob-cli-ji.7.1
create_time: 2026-11-10 17:37:09
status: wip
---

- **PARENT:**
  [202611/mac_event_note_mode.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/mac_event_note_mode.md)
- **BEAD:**
  [bob-cli-ji.7.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ji/bob-cli-ji.7.1.md)

# Finish and land Bob Mac Capture Event note mode

## Context

Epic **bob-cli-ji.7.1** (plan `202611/mac_event_note_mode.md`, under phase
**bob-cli-ji.7** of epic **bob-cli-ji**) added Bob Mac Capture's Event note mode. All
six phases are closed. bob-cli commits: `8e2b93c7` (effective buffer reasons and full
fixtures) and `23e7f90d` (docs). bob-mac-capture commits: `8273c8f` contract, `7cb516f`
rail, `809b66a` session, `cdd7384`/`f32a575`/`90b6f9e` picker, `5bf8907`/`62a7b70`
editor, and `1498f34` docs.

The land agent verified the contract, fixtures (Mac `Tests/Fixtures/gcal-note/*.json`
match bob-cli `tests/fixtures/gcal_note/json/*.json` exactly), session, routing, menu,
Calendar-row action, and docs. It found the gaps below, which this epic caused and which
must be finished before it closes. Changes that landed since the epic started
(notebooks, stats/periods, capture `hash_fallback`, the CI-gate retarget `f2d7a45`) do
not touch event notes. No further integration is needed.

Read `decisions:mac-capture-is-a-thin-client` and
`decisions:private-event-notes-beat-google-per-field` with `sase memory read`. The app
renders what Bob returns: no client grammar, no parsing note text in Swift. This tale
edits **no memory files**.

All Swift and Mac docs work is in the linked **bob-mac-capture** repo. Open it with
`/sase_repo` (`sase repo open bob-mac-capture -r "..."`), read its `AGENTS.md`, and use
only the printed path. Paths below are relative to bob-mac-capture unless marked
bob-cli.

### Known pre-existing CI failures (not this tale's work)

Master's macOS Test step is already red on the epic's base commits, from failures
tracked elsewhere:

- `CaptureAliasTipPanelTests` `testFilteringKeepsTheShownTipAnchoredToTheSigil`,
  `testOptionTypedHashOpensTheHashTip`, and
  `testTypingQueryBeforeTheRouteListOpensKeepsTheTip`, which fail on every run (task
  **bob-cli-jx**). The same suite's intermittent `testEditBeforeTheReplyDropsTheTip` and
  `testChangedSeparatorReplyDoesNotDisableTheNextTipRequest` are also covered by
  bob-cli-jx and bob-cli-jk.
- `CaptureSeparatorToggleTests` 200 ms-sleep flakes (**bob-cli-jc**).
- `just typecheck-tests` errors in `CaptureRelationEqualsAliasTests` and
  `CaptureModifierAliasTests` (**bob-cli-jy**).
- docs-check word budgets for capture-panel, capture-previews, keyboard, and plugins
  bob-navigation-hotkeys (**bob-cli-jz**) and protocol.md (**bob-cli-h9**).

Do not fix or weaken those tests here, and do not file beads for them; they are already
filed. Every other failure is a regression you must fix.

## 1. Highlight a saved note as soon as it loads

**Bug:** `Sources/CaptureCore/GcalNoteSession.swift` `select(identity:)` keeps the
fallback `selectedEvent` and calls `loadShow()`, which sets phase `.resolving`. The
editor therefore appears at once (`showsEditor` is true while resolving), and
`GcalNoteEditorView.onAppear` calls `session.requestPreview()` only when `editorText` is
non-empty, which it still is not. When the show response arrives, `applyShow` loads the
scope text with `isEditorDirty == false`, sets `.editing`, and never requests a preview.
`GcalNotePresentation.livePreview`'s non-dirty branch returns `spans: []`, so an
existing note shows no route, clock, buffer, reason, or prose highlights until Bryan
types. The same happens when `setScope` → `applyRestoredDraft`, or `applyShow`, restores
a non-dirty, non-empty draft. (The CI render looked fine only because `ImageRenderer`
rebuilt the view after the text had loaded.)

**Fix, in the session rather than the view:**

- When `applyShow` (without `thenRetryWrite`) or `applyRestoredDraft` leaves a non-empty
  editor text, schedule a validation preview (`schedulePreview()`, which runs
  `gcal note … -d`) whether or not the text is dirty. Keep blank text unpreviewed: a new
  blank editor still shows the show response's effective preview, and no empty
  preview/save is sent. Preview is a dry run, never a write; `save()` still needs a
  dirty or non-blank editor exactly as today.
- Remove the `onAppear` preview request from `GcalNoteEditorView`, or keep it harmless.
  The session must not depend on view lifecycle.
- Check `livePreview` for a non-dirty editor with a matching valid preview. The
  `matchingValid` branch is already first, so spans should paint. Make sure
  `statusLabel` reads "Updating preview…" only while the phase is `.previewPending`.
- Keep stale-response rules: the preview token still includes generation, identity,
  scope, and text.

**Tests:**

- `Tests/CaptureCoreTests/GcalNoteSessionTests.swift`: opening an event whose show note
  is non-empty records a `gcal note … -d … -- <loaded text>` argv
  (`FAKE_BOB_GCAL_NOTE_ARGV_PATH`) without any `setEditorText`. Switching to a recurring
  scope whose stashed draft is non-empty previews that text. Opening a blank scope sends
  no preview. Use finite waits.
- `Tests/BobMacCaptureTests/GcalNoteEditorTests.swift`: host the panel/editor, open an
  event, and assert the event-note `NSTextView`'s text storage carries highlight
  attributes without typing. The static `preview.json` input text
  (`@family -30 Drive there +15 Drive home`) differs from `show.json`'s note, so use a
  fixture map whose show and preview texts agree. If no Bob-generated pair agrees,
  regenerate one in bob-cli per `Tests/Fixtures/gcal-note/README.md`
  (`BOB_UPDATE_GCAL_NOTE_FIXTURES=1 cargo test --test gcal_note_cli`, with a new
  synthetic fake-gog case), copy it, and keep both fixture directories identical. Never
  hand-edit spans.
- Update existing tests that assumed no preview runs on open, such as preview-count or
  phase assertions, without weakening what they check.

## 2. Render the real picker and editor in the design fixtures

**Gap:** `Tests/BobMacCaptureTests/GcalNoteEditorDesignTests.swift` renders through
`RenderFixtureWriter`, which uses `ImageRenderer`. `ImageRenderer` cannot draw
AppKit-backed views. In CI run 42906549245's `render-fixtures` artifact, every
`event-note-*` PNG shows a yellow "prohibited" placeholder where the `NSTextView` editor
and the filter `TextField` belong. The picker's `ScrollView` rows render blank, so
`event-note-picker-*` and `event-note-saved-waiting-*` show no events. The epic's render
review (picker, editor, invalid, saving, saved/waiting, unsupported, at light/dark ×
full/minimum width) was therefore not possible.

**Fix:** render these surfaces with a hosted `NSHostingView` in an offscreen borderless
`NSWindow`, then `cacheDisplay`. Follow the existing patterns in
`CaptureMergeInDesignTests.renderHosted` and
`CapturePanelAgendaLayoutTests.exportPanelImage`. Use an explicit content height large
enough for the picker list (the picker uses `maxHeight: .infinity`, so a fitting-size
window would collapse its `ScrollView`). Keep the
`event-note-<state>-<720|480>-<light|dark>.png` names, and skip without
`BOB_MAC_CAPTURE_RENDER_DIR` as today.

- In the `editor` and `invalid` states, set editor text that matches the preview
  fixture's `input.text`, so the PNG shows Bob's highlights and the red error underline.
  `invalid.json`'s input is `@famly`.
- Add one hosted capture-panel render, or an assertion, at a small screen height,
  following `CapturePanelAgendaLayoutTests`' viewport helpers. It must show the
  event-note surface fits the panel with the footer and Save/Clear/Back visible.

## 3. Human status banner after save/clear

`GcalNotePresentation.statusBannerText` prints Bob's raw state token, for example
`waiting_note · lands in 2026/20261213.md when that day syncs`. Show a human banner
built from what Bob returns, without inventing semantics:

- Prefix with the write action: `saved` → "Saved", `cleared` → "Cleared", `unchanged` →
  "Unchanged".
- Then Bob's `detail`. If the detail already starts with the action word
  (case-insensitive, e.g. `saved_elsewhere`'s `saved · …`), show the detail alone,
  capitalized.
- If the detail is blank, fall back to the action label, then to the raw state.
- Never show `waiting_note`, `sync_failed`, or other snake_case tokens when a detail
  exists.

The session keeps `statusBanner` as a `GcalNoteStatus`. If the action is needed, pass
the write response's `action` alongside it. Cover all six write fixtures (`saved`,
`applied`, `cleared`, `unchanged`, `saved_elsewhere`, `saved_sync_failed`) in
`Tests/CaptureCoreTests/GcalNotePresentationTests.swift`. `sync_failed` must still read
as saved and show Bob's recovery detail.

## 4. Undo the epic's growth of over-budget reference pages

Before the epic, bob-mac-capture `62a7b70` had `docs/keyboard.md` at 6065 words and
`docs/protocol.md` at 6149, already over the 6000 cap. Commit `1498f34` raised them to
6097 and 6217, although the plan said not to expand over-budget pages. Read docs memory
(`sase memory read docs.md`). Tighten the event-note additions so each page is no larger
than its pre-epic count:

- keyboard.md: at most one short sentence linking `event-notes.md`. The full key table
  already lives there.
- protocol.md: the command bullet plus one compact compatibility row.

Trim redundant nearby wording if needed, but drop no contract facts and keep the
`event-notes.md` links. Measure with bob-cli's counter, for example from bob-cli
`tools/docs`: `uv run python -c 'from bob_docs.markdown import count_words; ...'`.
Getting these pages under 6000 belongs to bob-cli-jz and bob-cli-h9, not this tale. If
the banner copy in §3 changes anything `docs/event-notes.md` describes, update that
page.

## 5. Small cleanup

Remove the redundant `try` at
`Tests/BobMacCaptureTests/EventTimeRailDesignTests.swift:32` (Swift "no calls to
throwing functions occur within 'try'" warning, proposed by bob-cli-ji.7.1.3 note #2).

## 6. Verify

1. `just portable-check` in bob-mac-capture. If `swift` is missing locally, read the
   tailnet memory and run it on a scratch copy on athena.
2. Best effort: on a scratch Mac copy (excluding `.git`/`.build`), run
   `Scripts/bundle.sh` and then `just typecheck-tests`. The only allowed errors are
   bob-cli-jy's two files. If the Mac is offline, say so, and rely on CI for the macOS
   targets.
3. In bob-mac-capture, run `just docs-check`. In bob-cli, run `just docs-check` with
   `BOB_DOCS_BOB_MAC_CAPTURE_DIR` pointing at the opened Mac checkout. Only the
   pre-existing bob-cli-jz/bob-cli-h9 budget errors may remain, and keyboard and
   protocol must be at or below 6065 and 6149.
4. If bob-cli fixtures changed: `cargo fmt --check`, the clippy gate,
   `cargo test --test gcal_note_cli`, then `just check`. Never edit the Cargo version or
   CHANGELOG.
5. Run `Scripts/macos-ci-check` from bob-mac-capture on the final dirty tree through
   `/sase_monitor`, and wait for the staging command to exit. Required:
   - Lint Swift formatting and Build pass.
   - Every `GcalNote*`, `EventTimeRail*`, and `CapturePanelAgendaLayout*` test passes.
   - Test-step failures are limited to the pre-existing bob-cli-jx/bob-cli-jc/bob-cli-jk
     tests listed above.

   Download that run's `render-fixtures` artifact into a fresh empty directory and
   inspect the `event-note-*` PNGs. The picker must show grouped rows with chips, the
   editor highlighted note text, invalid the red underline and messages, and saving,
   saved-waiting, and unsupported their states. Check light, dark, 720, and 480, plus
   the small-height panel render. Record the snapshot SHA, run URL, and step verdicts.
   Any later Swift change invalidates this run, so rerun it.

## 7. Closeout (final step, same turn as the final code)

Do not wait on or order this after this work's own commit, push, or post-push CI. The
`Scripts/macos-ci-check` snapshot run above is the required evidence.

1. Run `sase bead epic-symbols bob-cli-ji.7.1`. For each listed `--epic-symbol` entry,
   resolve it: wire it up, privatize it, add a non-test pragma, or delete it, per the
   Symvision epic-whitelist policy. Re-key a Justfile line only to a still-open bead
   that genuinely needs it. (At landing review there were none.)
2. Close the epic: `sase bead close bob-cli-ji.7.1 --note "<verification>"`. Summarize:
   - the source commits above, plus this tale's commits;
   - the fixture identity check;
   - the session/highlight, render, banner, and docs fixes;
   - the macos-ci-check SHA, URL, and step verdicts, with the pre-existing failures
     named by bead;
   - the render PNG review and docs-check results;
   - the follow-up triage already recorded on bob-cli-ji.7.1: bob-cli-jx, bob-cli-jy,
     and bob-cli-jz filed; +1 on bob-cli-h9 and bob-cli-jc; the redundant `try` fixed
     here.

   Never use `--force` merely to make the close succeed.

3. Run `just symvision` in bob-cli and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of bob-cli plan file
   `202611/mac_event_note_mode.md`, the PLAN path shown by
   `sase bead read bob-cli-ji.7.1 -r "..."`, in the plans sidecar.
5. Handle the parent: `bob-cli-ji.7` is a **phase** bead ("Bob Mac Capture Event note
   mode"). Verify that this child epic delivered that phase's description: Command-E,
   status menu, and Calendar-row entry; picker; highlighted editor; hatched rail; Lands
   as preview; save/clear/undo through `bob gcal note -f json`; docs; and tests. For
   "green exact-SHA macOS CI", record that every event-note test and gate passed, and
   that the only red Test-step failures are the pre-existing bob-cli-jx/jc/jk tests,
   which also fail on base `6b6052c`.
   - Run `sase bead epic-symbols bob-cli-ji.7` and resolve any entries.
   - Then run `sase bead close bob-cli-ji.7 --note "<what you verified>"`.
   - Do **not** close epic `bob-cli-ji` or any other ancestor; its own land agent owns
     that.
6. Finish through `/sase_final`, committing both bob-cli and bob-mac-capture changes.

## Acceptance

- Opening an event with a saved note paints Bob's highlights without typing. Blank
  scopes send no preview. Session and hosted tests prove both.
- Event-note render PNGs show the real picker rows, highlighted editor, invalid
  underline, saving, saved-waiting, and unsupported states, in light/dark at 720/480,
  plus a small-height fit.
- The status banner never shows a raw snake_case state when Bob gives a detail.
- keyboard.md is at most 6065 words and protocol.md at most 6149 (bob-docs counter).
- The redundant `try` is gone.
- `just portable-check` is green, and the exact-SHA macOS run shows only the listed
  pre-existing failures.
- bob-cli-ji.7.1 and bob-cli-ji.7 are closed with evidence, the plan file is
  `status: done`, `just symvision` is clean, and bob-cli-ji stays open.
