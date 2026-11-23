---
tier: epic
title: Finish the core property editor's landing repairs and real-app acceptance
goal: The =+ property editor survives in-flight writes, keeps the requested row, shows
  confirmed writes on every surface, and passes a reviewed real-Obsidian run covering
  every bob-cli-pf.4.1 acceptance bullet.
parent_bead: bob-cli-pf.4.1
phases:
- id: write-safety
  title: In-flight write safety, Esc, and YAML escape
  size: medium
  depends_on: []
  description: 'write-safety: make cancel/close/undo during a pending write safe,
    register modal sessions for generation and rename events, narrow old-bob detection,
    route Esc through the close ladder, and close the modal on y/Ctrl+E from the originating
    leaf.'
- id: row-fidelity
  title: Row identity, overlays, diagnostics, and value fidelity
  size: medium
  depends_on:
  - write-safety
  description: 'row-fidelity: keep requested keys across reconciliation, publish confirmed
    overlays to cards and drawers, show recovery diagnostics, preserve datetime/tag/null
    spelling, and fix add-flow completion and draft loss.'
- id: acceptance-writes
  title: Real-Obsidian acceptance for write paths
  size: medium
  depends_on:
  - row-fidelity
  description: 'acceptance-writes: extend the property-editor real-app harness to
    every write-path and recovery bullet with disk byte assertions, run it on athena,
    and fix what it finds.'
- id: acceptance-surfaces
  title: Real-Obsidian acceptance for panes, entry points, and visuals
  size: medium
  depends_on:
  - acceptance-writes
  description: 'acceptance-surfaces: cover panes, history reopen, YAML Done, close
    keys, unload, every entry point, and reviewed light/dark narrow/wide screenshots
    on athena; document the run in validation.md.'
proposed_by: bbugyi200.apollo.bob-cli-pf.4.1.land
proposed_with: claude/opus
create_time: 2026-11-23 03:09:52
status: done
bead_id: bob-cli-pf.4.1.7
---

- **PROMPT:** [prompts/202611/property_editor_landing_repairs.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/property_editor_landing_repairs.md)
- **PARENT:** [202611/property_editor_core.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/property_editor_core.md)
- **BEAD:** [bob-cli-pf.4.1.7](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-pf/bob-cli-pf.4.1.7.md)

# Finish the core property editor's landing repairs and real-app acceptance

The epic `bob-cli-pf.4.1` (plan `plan:202611/property_editor_core.md`) shipped the
generic `=+` Property Editor in the linked `bob-plugins` repository across six closed
phases (bob-plugins commits `8ac47900`, `a5c22699`, `a7c91350`, `15e1fa6b`, `54550765`,
`c60b2d3e`). Its land agent read every phase and the source, and found that the work is
not yet complete:

- Several defects make the modal hang or lose history in normal use, and none are
  covered by the unit tests.
- The real-Obsidian acceptance never ran: the phase host lacked Obsidian. The harness
  `scripts/obsidian-property-editor-e2e.cjs` exercises only a fraction of the required
  bullets.
- `docs/develop/validation.md` has no instructions for that harness.

This child epic finishes that work. Its `parent_bead` is `bob-cli-pf.4.1`. After this
epic lands, its land agent resumes `bob-cli-pf.4.1`'s interrupted landing: it closes
that epic, then the phase `bob-cli-pf.4`. No phase of this plan closes either of them.

All product work happens in the linked `bob-plugins` repository. Open it with
`/sase_repo` (`sase repo open bob-plugins -r "<reason>"`), use the path it prints, and
read its `AGENTS.md`. Paths below are relative to that checkout. The source lives in
`plugins/bob-ledger-tools/src/` fragments, so never edit `main.js`. Register any new
fragment in `src/fragments.json`, then regenerate with `npm run build`. Every
hand-edited fragment must stay at or below 1000 lines. `298f` is about 773 lines and
`298i` about 708, so put substantial additions in new `298n…` fragments that patch the
prototype the same way `298i` and `298l` already do.

The parent plan's DECISIONS and scope stay final. Rename, reorder, selection, clipboard,
filtering, kind conversion, and the YAML value stage belong to `bob-cli-pf.5`. Native
Properties takeover belongs to `bob-cli-pf.6`. Do not ship controls for any of those. Do
not change the bob `props` protocol, the Cargo version, the changelog, or any SASE
memory.

## Already verified (preserve)

The landing review confirmed the following against source and focused tests. Keep it
working.

- **Wire contract:** props argv and envelope decoding, typed error codes, and the
  null-fingerprint write refusal.
- **Old-bob detection:** exit 2 counts as old bob only with "props" plus
  unrecognized-subcommand text.
- **History:** keyed by normalized path, bounded to 20/20, and `framing` round-trips.
  Redo clears on a new edit. No-op, failed, and stale operations consume nothing. Rename
  migrates history, delete releases it, and unload clears it.
- **Overlay store semantics:** null stays distinct from absence, a 5 s lease, and
  retirement on a matching or superseding change.
- **Role gates and wire kinds:** gates cover flip, delete, and stages. Only the 7 wire
  kinds are sent.
- **Preview and Enter:** a 250 ms debounce, and Enter waits for a matching preview. IME
  composition, edits, and selection changes cancel a pending Enter. Panes are saved
  before apply. Date tokens are rejected across DST.
- **Prose surface:** S16 is registered.
- **Add flow:** source-key anchors with one `prop.set`, no placeholder keys, and cancel
  writes nothing. The first property uses the empty-mapping fingerprint.
- **The `+` arm:** inherits every guard and accepts Shift. `==` and `=-` are unchanged.
- **Entry points:** the palette command, the Command Mode row, and the drawer buttons
  that appear only while the hook is set.
- **API and hand-offs:** API failure codes; record, intentions, and album hand-offs pass
  the field.
- **YAML excursion:** `kind: "props"` leaves album and record excursions intact and
  never writes.

## Phase write-safety

Fix the defects that hang the modal or corrupt history around a pending write. Line
numbers are from the landing review; recheck them.

1. **Cancel or close during a save.** `propertyEditorCancelStage` (`298i`) calls
   `session.setDraft([])` while `propertyEditorApplyStage` awaits `session.apply()`. The
   revision bump turns bob's completed write into `superseded`, which drops its history
   entry and overlay. `propertyEditorApplyStage` then returns before clearing
   `pendingApply`, so every later open, flip, delete, add, and `openPropertyEditor`
   ("Wait for the property save to finish") refuses forever.
   - While a write is in flight, cancel, close, row clicks, and selection changes must
     not invalidate it. Either defer the draft clear until the outcome arrives, or
     refuse the gesture with a visible "Saving…" notice.
   - Always clear `pendingApply` in a `finally`.
   - A completed write must record its history entry and overlay even if the modal or
     stage closed meanwhile, because the session or service owns history, not the DOM.
   - Treat an unknown outcome as unknown. Do not retry it.
   - Apply the same rules to flip, delete, and add applies (`298f`, `298l`).
2. **Undo and redo while busy.** `propertyEditorHistory` (`298f`) and `replayHistory`
   (`298c`) have no in-flight guard. Two quick `u` presses both come back `superseded`
   and both re-push. Every later undo is then stale, and redo is lost. Serialize history
   replays, and refuse `u`/Ctrl+R while any write for that path is pending. An entry is
   popped only once its replay's outcome is known, and it is restored exactly once on a
   refusal.
3. **Session generation and rename.** The modal creates its session (`298f`, around
   line 165) without registering it in `plugin.propertySessions`, so
   `propertyBumpGeneration` (`298d`) never notifies it. A vault `modify` between preview
   and Enter therefore leaves "Checking value…" forever, because `canApply` is false and
   no new preview is requested.
   - Register and unregister the modal session.
   - When the generation changes while a draft or pending Enter exists, re-request a
     preview, or re-arm the pending Enter so it applies after a fresh matching preview.
   - Retarget or close an open modal on rename, and close it on delete with a notice.
   - Consider bumping only for the target path and the panes that show it, rather than
     for every vault `modify`. Preserve body-only-change revalidation.
4. **Old-bob classification.** `apiPropertyShow` (`298d`, around lines 205-212) sets the
   plugin-wide `propertyCapability` from any decoded report. An error envelope without
   `capabilities` (for example a missing note or an I/O failure) therefore marks bob as
   old for every note. Set it to `false` only for the typed `needs-bob` outcome. Leave
   the capability unchanged for error reports, and keep them actionable and read-only.
5. **Esc through the close ladder.** `PropertyEditorModal` must override `onEscapeKey`
   to call its back-out ladder, as `242p-plugin-record-card.js` (`onEscapeKey` →
   `recordBackOut`) and the album card do. Obsidian's scope runs Escape before contentEl
   listeners (see the note near `241y`, around lines 756-765). Esc must peel the menu,
   then the stage or add draft, then the modal, exactly like `q` and Ctrl+[.
6. **YAML escape closes the modal and targets the right leaf.**
   - `y`/Ctrl+E (`298f`, the `yaml` action) leaves the modal open over the Source view.
     Close the modal first, as the record card does (`242p`, around line 587), while
     preserving the selected key for Done.
   - `propertyEditorEntryLeafFor` (`298m`) takes the first leaf that shows the note.
     Prefer the leaf the editor was opened from (recorded at open), then the active
     leaf, then any leaf.
   - If the excursion turns stale (rename, delete, or leaf closed), remove the Done
     button, restore the mode where possible, and show a notice instead of leaving a
     dead button.
7. **Lock wait.** Surface the shared controller's lock-waiting state in the status line,
   and let Esc cancel the wait (`stopWaiting`) without retrying an uncertain write.
8. **Stale developer-facing strings.** Remove the phase-era strings in `298f`:
   - "Add and filter arrive in later phases"
   - "opens a typed stage in the next phase"
   - "the typed stage for it arrives in a later phase"

   Browse `=` on a loose row of a species note opens that note's owning card editor, as
   `docs/property-editor.md` promises. On an ordinary note it explains that no card
   editor exists.

Add failing-first unit tests for each item in the existing suites
(`scripts/test-ledger-tools-property-update.cjs`, `…-property-editor-shell.cjs`,
`…-property-editor-stages.cjs`, `…-property-editor-entrypoints.cjs`) or in a new suite
registered in `package.json`'s test list.

## Phase row-fidelity

1. **Keys survive reconciliation.**
   - The constructor (`298f`, around lines 104-110) builds the selected id from
     `cacheRows` indexes. Report rows use `key::line`, so `propertyEditorReconcile`
     (`298f`, around lines 301-306) falls back to the first row.
   - Reconcile the selection by key, then by duplicate ordinal among same-key rows, then
     by nearest surviving source position.
   - Do the same after edits, undo, or redo that shift lines.
   - Done-reopen in `298m` (around line 500) must use the same nearest-row rule instead
     of `rows[0]`.
   - Cover `openPropertyEditor({key, mode: "edit"})` opening that key's stage (not the
     first row's) once the report arrives.
2. **Confirmed overlays reach cards and drawers.** `propertyPublishOverlay` (`298d`) is
   a no-op, and nothing subscribes to `propertyOverlayStore()`.
   - Subscribe the property-card drawer and foot (`240o`/`240p`/`240r`,
     `242kc-plugin-record-sheet-foot.js`) and the editor itself, so a confirmed
     post-image paints until source or cache catches up.
   - Keep the store's null, deletion, and ordering semantics. Do not use the record
     overlay's null-as-unset conversion.
   - Rendering must not spawn bob or write.
3. **Drawer refresh on hook change.** `propertyEditorRefreshDrawers` (`298m`) only
   schedules refreshes. The album, bank, intention, and record surface signatures hash
   neither the hook nor its presence, so open drawers never gain or lose
   `＋ Add property` / `✎ Edit all =+`. Include the hook's presence in those signatures,
   or force a repaint, and test register and unregister on an open drawer.
4. **Recovery diagnostics.** Read bob's `frontmatter.error` (message and line), each
   row's `diagnostic`, and `ownership_diagnostics` (see bob-cli `docs/props.md`).
   - Show bob's message verbatim, and target `y` at the parse-error line instead of the
     first property's line (`298e`, around lines 169-184).
   - For duplicate keys, target the duplicate line.
5. **Datetime spelling.** `298h` (around lines 212-230) always appends `±HH:MM` and
   seconds. bob keeps a supplied offset when the existing value has none (bob-cli
   `src/native/props/emit.rs` `restyle_offset`), so editing `2026-11-22T09:41` writes
   `…T09:41:00-05:00`.
   - When the existing value has no offset, and for new datetimes, send local wall time
     without an offset, unless the user typed an explicit offset (whose instant is then
     retained).
   - Match the existing seconds precision.
   - Add tests for no offset, `Z`, `+0000`, `-0500`, `-05:00`, and an explicit typed
     offset.
6. **Tags fidelity.** `items.map(propertyEditorNormalizeTag)` (`298h`, around line 313)
   coerces `2026` to `"2026"` and a null item to `""`, so an untouched stage counts as
   changed. Normalize only the items the user edited. An untouched stage is a no-op.
7. **Null stays null.** Opening a null text row and pressing Enter without typing
   currently writes `""` (`298h`, around lines 28 and 294). Per the parent contract, an
   untouched stage on a null value is a no-op ("No changes."). Update the stages test
   that asserts the old behaviour.
8. **Add-flow precision.**
   - In `298k` (around lines 173-176), `picked || exact` lets a highlighted substring
     match beat an exact name. An exact typed name must win on Enter or Tab, unless the
     user explicitly moved the highlight.
   - `o`, `O`, and Mod+; (`298l`, around line 139) quietly cancel an open value stage
     whose input lost focus. Refuse with the same "finish the open edit first" notice
     that `=` uses.

Add failing-first unit tests for every item.

## Phase acceptance-writes

Read `docs/develop/validation.md` (the record-card and card-foot sections) and the
existing harnesses (`scripts/obsidian-record-card-e2e.cjs`,
`scripts/obsidian-card-foot-e2e.cjs`). Real acceptance runs on **athena**, which has
`/snap/obsidian/current/app/obsidian` and `xvfb-run`. apollo has no Obsidian.

Reach athena with `ssh athena` (Tailscale). Copy the current bob-plugins tree and a
clean copy of the current bob-cli source into a fresh `mktemp -d` directory on athena.
Build `bob` there, resolving the Cargo target directory rather than assuming it. Then
run:

```bash
BOB_CLI_DIR=<copy> BOB_BIN=<built bob> xvfb-run -a node scripts/obsidian-property-editor-e2e.cjs \
  -e /snap/obsidian/current/app/obsidian -r <report.json> -o <shots-dir>
```

Check the harness's actual flags first. Run long steps through `/sase_monitor` with
continuation instructions. Never open Bryan's live vault.

Extend `scripts/obsidian-property-editor-e2e.cjs` (split helpers into `…-e2e-lib*.cjs`
modules) so that real keyboard input in real Obsidian covers each of the following, with
disk assertions:

- **Project and day notes:** edit, add, delete, undo, and redo. After every write, undo,
  and redo, assert the exact note bytes. Key order, comments, and body bytes must be
  unchanged outside the edited span.
- **Plain note with no frontmatter:** first add, complete undo (framing removed), then
  redo.
- **Record field hand-off:** Enter or `=` on a card row opens the record card on that
  field.
- **Derived and identity rows:** Enter, delete, and the checkbox key are refused, and
  `y` opens YAML at that key.
- **Old bob (shim):** read-only rows and the update hint.
- **Recovery:** invalid, unclosed, and duplicate-key YAML each show bob's diagnostic,
  and `y` targets the error line.
- **Value kinds:** checkbox flip; datetime with no offset, with `-05:00`, and from a
  relative token; tags (no leading `#`); prose text with the `[[`/`((` band if the
  navigation plugin is loaded; ordered list; null versus empty-string preservation.
- **Name completion:** type a known vault name, accept it with Tab, and confirm its
  type. Shift+Enter keeps a literal name.

Replace the >1000-byte screenshot check with recorded assertions. Keep
`scripts/test-obsidian-property-editor-e2e.cjs` (the offline self-test in `test:unit`)
and `scripts/fixtures/fake-bob.cjs` in step with the harness. Fake-bob success is not
acceptance. Fix every product defect the real run finds, with a unit test for each.

Record the bob-cli source SHA and binary provenance, as the record-card harness does.
Copy the report and screenshots back. Register them with `/sase_artifact`, and cite the
refs in the phase note.

## Phase acceptance-surfaces

Extend the same harness (and its offline self-test) to cover:

- **Panes and drafts:** two panes showing one note with unsaved editor text; apply saves
  both panes first; an external modify during a draft produces a stale refusal that
  keeps the draft and requires a fresh preview.
- **History and YAML return:** close and reopen keeps history; `y` then Done restores
  the original leaf mode and reopens on the same key (or the nearest row after an
  external delete).
- **Close keys:** Esc, `q`, and Ctrl+[ peel menu, then stage, then modal, with focus
  inside and outside text fields. `q` types in a text field; Ctrl+[ never does.
- **Unload:** disabling the plugin while the modal is open, and during a pending
  preview, leaves no listeners, timers, or late reopen.
- **Entry points:** `=+` in Live Preview Vim normal mode (including Shift+`=`), the
  palette **Edit note properties**, the Command Mode row, a drawer key-label
  click/Enter, drawer `＋ Add property` and `✎ Edit all =+`, and the sibling API
  `openPropertyEditor({filePath, key, mode})`. Each must reach the same working editor,
  including on an ordinary note and on an invalid-YAML note.
- **Visuals:** light and dark at narrow and wide widths. Open and **review every PNG**
  for clipping, focus ring, selected-row and locked-row legibility, and record the
  review in the report.

Run on athena as in acceptance-writes, fix what it finds, and register the artifacts.
Then document the property-editor harness in `docs/develop/validation.md`, next to the
record-card and card-foot sections. Cover the athena prerequisites, the exact command,
provenance, self-test versus real acceptance, and screenshot review. Update
`docs/property-editor.md` if any shipped behaviour changed. Read the `docs.md` reference
memory before editing docs, and run `npm run docs:check`.

## Every phase

- Run the focused suites, `npm run build`, `npm test`, `npm run validate`, then
  `bob plugins sync`.
  - Full-suite timing flakes already tracked as `bob-cli-km` (event-link-times),
    `bob-cli-io` (frees-marks), and `bob-cli-pg` (taskLinks p95) do not block a phase.
    Confirm each passes in isolation and cite the tracker.
  - If sync refuses an overwrite, follow `AGENTS.md`'s procedure. Never pass `-F` on
    your own.
- Before closing your phase, run `sase bead epic-symbols <phase-id>` and resolve or
  re-key every entry.
- Record discoveries as `PROPOSED FOLLOW-UP:` notes on your phase. Create no beads.
- Close only your own phase. Never close `bob-cli-pf.4.1`, `bob-cli-pf.4`, or
  `bob-cli-pf`.

## Land

This epic's land agent verifies every phase note and the real-app evidence. It closes
this epic, then resumes `bob-cli-pf.4.1`'s landing through the `parent_bead` link:

- re-verify;
- record what this epic fixed;
- close `bob-cli-pf.4.1` and mark `plan:202611/property_editor_core.md` done;
- close phase `bob-cli-pf.4` with the pf.5 and pf.6 boundary recorded.
