---
tier: epic
title: Blocked checkbox rendering and verification for bob-cli-ft.3
goal: Blocked task checkboxes show their proven reason across Live Preview and rendered
  source-note surfaces, with accessible tooltips, reversible toggling, a shared miniature
  API, and verified light/dark Obsidian behavior; bob-cli-ft.3 is completed.
phases:
- id: paint_api
  title: Shared blocked checkbox paint, miniature API, and contrast
  size: medium
  depends_on: []
  description: 'paint_api: implement the gated held/waiting/stale CSS, glyphs and
    pip, listener-free api.blockMarks.render, and computed contrast tests. Record
    measured tokens and geometry in the contract. No editor or reading-view decorations
    yet.'
- id: live_preview
  title: Live Preview decorations, session toggle, and refresh lifecycle
  size: medium
  depends_on:
  - paint_api
  description: 'live_preview: render block marks on visible Live Preview task rows;
    add lazy checkbox tooltips, the session toggle, popout gating, and coalesced invalidation
    from Tasks, ladder, midnight, and resume. Keep rendering free of disk reads and
    selection-only rebuilds; prove behavior with focused node tests.'
- id: rendered_views
  title: Reading view, embeds, hover previews, and Dataview task marks
  size: medium
  depends_on:
  - live_preview
  description: 'rendered_views: map rendered task rows to canonical source text or
    a conservative own-text fallback; classify and refresh the row and its own checkbox
    in place. Cover nested rows, query origins, cleanup, and both toggle directions
    with DOM tests.'
- id: obsidian_e2e
  title: Real-Obsidian harness, screenshots, and rendering integration
  size: medium
  depends_on:
  - rendered_views
  description: 'obsidian_e2e: add an isolated scratch-vault CDP harness and offline
    self-test, exercise BM source-note vectors in both themes and views, verify real
    hover/click/toggle behavior, inspect screenshots against the accepted mockup,
    and fix integration defects within this rendering scope.'
proposed_by: bbugyi200.athena.bob-cli-ft.3
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-ft.3
create_time: 2026-11-02 07:34:43
status: done
bead_id: bob-cli-ft.3.1
---

- **PROMPT:** [prompts/202611/block_marks_editor.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/block_marks_editor.md)
- **PARENT:** [202611/blocked_reason_marks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/blocked_reason_marks.md)
- **BEAD:** [bob-cli-ft.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ft/bob-cli-ft.3.1.md)

# Blocked checkbox rendering for bob-cli-ft.3

## Objective and handoff

Complete the `editor_marks` phase of `plan:202611/blocked_reason_marks.md`, tracked by
`bob-cli-ft.3`. Blocked task checkboxes will show why they are blocked in Live Preview,
reading view, embeds, hover previews, and Dataview TASK views. Ship the shared miniature
box renderer needed by later phases. Stored Markdown remains `[?]`.

This is a child epic of **bob-cli-ft.3**, not a replacement for `bob-cli-ft`. Its four
phases are sequential because they share generated `main.js`, styles, manifest, and
documentation inventory. Each phase is medium, direct implementation work for one coder.
The child epic's land agent owns combined verification and completion of `bob-cli-ft.3`;
child phase workers close only their own assigned beads. No worker or land agent in this
child epic may close `bob-cli-ft` or any higher ancestor.

## Verified starting point

- `bob-cli-ft.3` is in progress and has no earlier implementation notes or recorded
  remainder. No reopen or manual status transition is needed.
- Contract phase `bob-cli-ft.1` landed as bob-cli commit `16f3a28f`.
  `docs/blocked-marks.md` contains BM1–BM27 and the accepted surface contract.
- Model phase `bob-cli-ft.2` is closed; bob-plugins commit `d58b1fa` contains
  `142c-block-marks-model.js`, `272c-plugin-block-marks.js`, and
  `scripts/test-ledger-tools-block-marks.cjs`. `api.blockMarks` v1 exposes `model` only.
  `blockMarksEnabled` already defaults to true. The ledger manifest is 1.86.0 at
  inspection time. Recheck the current version before bumping it.
- No block-mark paint, editor extension, rendered-view adapter, toggle command, or
  block-mark e2e exists in the inspected tree. Recheck before implementing to avoid
  duplicating another landed change.
- `sase bead epic-symbols bob-cli-ft.3` currently reports none.
- The existing real-app harness is `scripts/obsidian-task-link-marks-e2e.cjs`, using
  `scripts/obsidian-startup-smoke-lib.cjs`. `xvfb-run` and `/snap/bin/obsidian` are
  present; the `obsidian` on PATH may be the CLI shim, so use the Electron executable.
- The lifecycle fragment is already 998 lines and the priority mixin 969. Keep new work
  in separate fragments; do not exceed the 1000-line fragment limit.

Read context through audited artifact commands:

```bash
sase bead read bob-cli-ft.3 -r "Need rendering scope and final epic decisions"
sase artifact read plan:202611/blocked_reason_marks.md "Need parent display contract"
sase artifact read file:explicit:246f98d1d5cd86fcb4a9ba34 "Need accepted visual research"
sase artifact read file:default:6f2a31a39e300befd5a88907 "Need checkbox comparison mockup"
```

The research document reference failed to resolve during planning; its durable
`file:explicit:` snapshot above was readable. The image reference resolves to the
accepted `blocked_reason_checkboxes__cld_mockup.png`. Use audited reads, then the
resolved image path for visual inspection. The mockup illustrates appearance; the
contract and final parent decisions govern behavior.

Open bob-plugins using `/sase_repo` and
`sase repo open bob-plugins -r "Implement the blocked checkbox rendering phase"`. Read
its `AGENTS.md`, use only the returned path, and keep bob-cli contract edits in the
primary checkout. Read the docs memory through `/sase_memory_read` before documentation
changes. No memory file may be edited.

## Fixed contract and boundaries

The parent decisions are final: **own_first**, **chevron**, **quiet**, and
**memory_decision_record=no**. Do not ask them again or add a memory decision.

Reuse the landed classifier and its exact tooltip copy. Eligibility is a canonical `[?]`
task with an exact `#task` tag outside fences/code, including quote-prefixed tasks. The
renderer supplies fence context that the line-local model cannot know. Open own
prerequisite wins over project hold, which wins over own future date. Within dates,
recurrence wins over ladder priority, then hand-scheduled date. Cold or unresolved
prerequisite input remains neutral. No live reason with known inputs is `stale`. The
date must be strictly later than local today.

| Key        | Family  | Checkbox                              |
| ---------- | ------- | ------------------------------------- |
| `dep`      | held    | solid red with white `&`              |
| `prj`      | held    | solid red with white chevron-up mask  |
| `p1`–`p4`  | waiting | hollow digit, ordered red-to-grey ink |
| `repeat`   | waiting | hollow existing recurrence mask       |
| `date`     | waiting | hollow `?`                            |
| `stale`    | stale   | muted dashed `?`                      |
| null model | neutral | existing plain `?`                    |

Each marked row receives `data-bob-block`, `data-bob-block-family`, and only when
applicable `data-bob-block-more` for the highest losing reason. Never inherit these
semantics into a nested child's checkbox. Every live losing reason remains in the
model's tooltip; do not turn `Also scheduled ...` into a promise of unblocking.

Scope stops at source-note and Dataview rendering plus the shared miniature API.
Priority-bar and project-tag folds belong to `bob-cli-ft.4`; Tasks query CSS/BM24,
dependency chip adoption, Task Card adoption, and whole-feature final visual review
remain with their existing parent phases. Do not implement those early. Keep a Tasks
query fixture available for later expansion, but this phase's JS must skip
`.plugin-tasks-list-item` rows. No CLI, Rust, reconciliation, scheduling, task storage,
or status registry changes. No mark writes the vault or intercepts clicks.

## Phase paint_api

1. Add a small element-builder fragment, registered in `src/fragments.json`, and extend
   `blockMarksApi()` in `272c-plugin-block-marks.js` with synchronous, never-throwing
   `render(host, model, { decorative })`. Use the host's ownerDocument for popouts.
   Append one listener-free `span.bob-block-mark` with the same data attributes as rows;
   return it, or null for invalid input, a disabled gate, or unavailable DOM. Meaningful
   marks get `role="img"` and the model tooltip as `aria-label`; decorative marks get
   `aria-hidden="true"`. Preserve model(), the namespace's v1, and the top-level API
   version. Accept the canonical synthetic digit model shape needed by the later Task
   Card phase; never require a source task.
2. Add all paint under `body.bob-block-marks` in `styles.css`. Cover miniature spans,
   Live Preview `.HyperMD-task-line[data-task="?"][data-bob-block]` inputs, and reading
   `li.task-list-item[data-task="?"][data-bob-block] > .task-list-item-checkbox`. Beat
   the vault snippet by specificity, never `!important`. Scope custom properties to each
   owned checkbox/miniature so a marked parent cannot change an unmarked child's glyph
   or ink. Draw the entire marked box even when the snippet is absent, and keep hit area
   and row height unchanged.
3. Use text via `--task-status-symbol` for `&`, digits, and `?`, with tabular digits.
   Use a 16×16 chevron mask with the approved `M4 10.25 8 6.25l4 4` geometry and
   approximately 2.1 stroke; reuse `--bob-date-glyph-repeat` for recurrence. Record
   final geometry. Draw the 6px pip with a 1.5px background-colored ring, no layout
   footprint, visible overflow, and no pointer handling. The pip ink follows `more`.
4. Preserve 13% held row tint and 2px rail; waiting uses 5.5% tint, 45% rail, 1.5px
   border and 12% ink wash; stale has dashed 1.5px border, no wash/tint/rail. Start from
   the contract's red-to-grey ramp (72% anchor; 68/38/12% level mixes), then calibrate
   against measured default-theme tokens. Keep any correction local to marked boxes: do
   not redefine the global blocked color or alter unmarked rows. If white on the default
   solid red fails 4.5:1, use the smallest local darkening mix needed to meet the
   requirement and document it explicitly. Keep the held shape, white glyph, red hue,
   and quiet waiting decision. Transitions are at most 120ms, with none under reduced
   motion.
5. Add `scripts/test-ledger-tools-block-marks-contrast.cjs`, registered in npm test.
   Follow the sRGB/luminance helpers in `test-ledger-tools-task-type-identity.cjs`. Read
   shipped CSS values rather than maintaining a second independent palette. Composite
   the row tint onto the background, then the box wash onto that row; enforce 4.5:1 for
   every waiting glyph (P1–P4, hand date, recurrence) and white on held fill in default
   light and dark themes. Add element/API tests for both aria modes, null/bad input,
   pips, a synthetic digit model, and a foreign ownerDocument.
6. Update bob-plugins `docs/develop/plugin-apis.md`; record measured color ratios, final
   tokens, and glyph geometry in bob-cli `docs/blocked-marks.md`. Build and validate
   using the common phase checks below. This phase creates shared paint; editor and
   reading rows are attached by subsequent phases.

Done when the miniature API and contrast tests pass and the CSS can paint every family
without changing existing unmarked task rows.

## Phase live_preview

1. Add separate runtime/editor fragments and register them in `fragments.json` and
   `310-install-methods.js`. Hook setup through `284-plugin-task-link-marks.js`'s
   existing mark setup fan-out. Use plugin-registered cleanup instead of growing the
   nearly full lifecycle fragment. Add the block-mark setup, refresh effect, coalesced
   scheduler, and cleanup before installing the extension.
2. Implement `Prec.highest(ViewPlugin.fromClass(...))` with `Decoration.line` attributes
   on visible eligible lines. Mirror priority extension field and file checks but
   deliberately omit its `selectionSet` rebuild. Rebuild on document or viewport
   changes, Live Preview changes, path changes, and the dedicated effect. Deduplicate
   lines across visible ranges; exclude fenced/code content using the existing
   syntax/fence helpers and preserve quote-aware matching. Use current editor text so
   unsaved field/status edits immediately change or remove marks.
3. Add session command `toggle-block-marks` / `Toggle blocked-reason marks`. Use
   `applyMarkGateClass` and `registerMarkGatePopoutSync`; show the exact Notice
   `Blocked-reason marks on` or `Blocked-reason marks off`. Both directions refresh all
   relevant editor views and rendered hosts, and trigger the existing Tasks rerender
   event. Add a rendered-refresh hook that the next phase implements; keep this phase
   independently runnable. Toggle off removes owned attributes and tooltip metadata, and
   model/render return null. Unload removes gates, timers, listeners, and owned
   decorations/attributes without disturbing other plugins' metadata.
4. Add `scheduleBlockMarksRefresh()` to `scheduleLiveWidgetRefresh()` after existing
   Tasks cache generation invalidation. Hook date-mark midnight rollover. Register
   visibilitychange through `registerDomEvent`, also for popouts, and refresh when
   returning to visible. Refresh ladder configuration outside per-line rendering on
   setup, the existing minute tick, and resume; compare semantic ladder content and
   invalidate only when it changes. Do not introduce a per-row timer.
5. Close the existing hot-path gap: `blockMarkModelForLine` and project tooltip lookup
   currently call `priorityLadderSnapshot()`, whose expired cache can stat or read a
   file. Feed them the already-refreshed in-memory ladder; null uses the classifier's
   default digit map. Keep the existing memo-local prerequisite index. Cache
   project-task lookup by note path per memo generation so visible project rows do not
   repeatedly scan the entire Tasks array. Do not fork classification or date wording.
   Prove warm render/model calls do no disk or subprocess work.
6. Install capture-phase `mouseover` on the editor for a marked row's own existing
   checkbox. Resolve its current line/model and set input `aria-label` plus
   `data-tooltip-position="top"` before Obsidian's tooltip handler. Preserve checkbox
   semantics; no wrapping, replacing, click handlers, preventDefault, or
   stopPropagation. Keep saved preexisting attribute values for removal; refresh
   previously labelled inputs when a model changes. If real-app verification later
   proves lazy attachment insufficient, use the parent's approved requestMeasure
   write-pass fallback.
7. Add focused editor/runtime tests for neutral and marked rows, fence and inline-code
   exclusions, unsaved edits, removal, viewport/path/Live Preview switches, refresh
   effects, no selection-only rebuild, stable attributes, cache warm-up, config/day/
   resume refresh, toggle/popout cleanup, and checkbox tooltip ownership. Register these
   tests and update the session-toggle docs.

Done when editor behavior and invalidation are tested, rendering is read-only and cheap,
and both toggle directions are reversible. Real hover/click proof follows in the e2e
phase, not by claiming node stubs demonstrate Electron behavior.

## Phase rendered_views

1. Add a separate reading adapter and post-processor. Find only each eligible row's own
   checkbox; skip Tasks query rows and code/pre ancestors. Use `ctx.getSectionInfo` text
   and the row's section-relative `data-line`, accounting for full-note versus
   section-only text. Verify exact status and source identity before applying a model.
   Borrow source-mapping helpers from `276-plugin-today-marks-reading.js` without its
   vault.cachedRead fallback.
2. Retain canonical row text and source path before other processors replace fields and
   tags. Choose processor order deliberately and test it with date, priority,
   recurrence, and task-tag marks enabled. For refreshed source sections, update
   retained text or invalidate it; never keep a stale source mapping on a recycled row.
3. If source mapping is unavailable, use `dependencyReadingOwnText` conservatively:
   exclude nested lists, preserve code exclusion, and require canonical raw fields and
   exact task eligibility. Its raw implementation flattens child element text, so do not
   treat a transformed date/priority pill or inline code as trustworthy source syntax.
   Reconstruct only the list checkbox prefix from an exact row status; leave uncertain
   rows neutral rather than manufacturing a `stale` result from lost field information.
   No disk reads or title matching.
4. Dataview TASK rows may arrive after the initial markdown post-processing and have a
   different origin than the query note. Reuse the task-theme query-origin helpers and
   existing rendered-host refresh lifecycle. Recover a source path from explicit row
   metadata/backlink when available; never read a result index as a source line or use
   the query note stem for a project reason. Handle delayed Dataview rendering through
   its existing refresh events or a bounded host-local observer, with teardown and
   self-mutation guards. Unprovable origins remain neutral. Do not add Tasks JS.
5. Apply the common row attributes and input aria tooltip, with no new click handlers.
   Retain original values of attributes the plugin owns and restore them when a mark
   becomes neutral, a checkbox changes lane, the toggle turns off, or the plugin
   unloads. Track connected hosts independently of whether they currently have a mark so
   cold-cache rows and toggle-off views can gain marks later. Reclassify in place
   through the live-preview phase's shared scheduler; cover embeds, hover previews, main
   window, and popouts. Dispose detached hosts; do not rerender the whole vault.
6. Add DOM tests for all families and pips, exact source mapping, section offsets,
   quote/list nesting, cold-cache-to-ready, removed fields, re-used rows, mismatched
   status/origin, transformed-field fallback rejection, Dataview delayed rendering,
   Tasks exclusion, tooltip restoration, toggle off/on, and detached-host cleanup.
   Update bob-plugins `docs/bob-ledger-tools.md` under marks and rendered blocks.

Done when all covered rendered surfaces reuse the classifier, nested children stay
independent, and refresh/toggle removal and restoration work without reading files.

## Phase obsidian_e2e

1. Add `scripts/obsidian-block-marks-e2e.cjs`, based on the existing task-link harness
   and shared smoke library. Retain isolated vault/profile creation, explicit Electron
   executable selection, CDP target ownership checks, watchdogs, structured failure
   reports, and cleanup limited to the owned process/files. Do not open the live vault.
   Keep generated fixture vaults/profiles under the active workspace. Add
   `scripts/test-obsidian-block-marks-e2e.cjs` invoking `--self-test`, and register that
   offline wrapper in npm test. Self-test must be clearly distinct from a real Obsidian
   pass and must exercise fixture/assertion/report logic.
2. Build an easy-to-read BM source note with open x/y and closed d prerequisite
   fixtures, project context, all keys including p3/p4, multi-reason pips, neutral
   examples, unchanged lane/closed/cancelled examples, fences, inline code, nested
   children, and a quoted task. Keep relative day offsets equivalent to the contract
   vectors so the fixture remains useful after November 2026. BM6/22/23 are separate
   cold-cache/config/day scenarios, not misleading extra static rows. BM24 and the
   BM1/BM7 fold assertions belong to later parent phases and are excluded here.
3. Copy the locally built plugin into the scratch vault. Include the existing vault's
   task-statuses.css snippet enabled via appearance.json, and installed Tasks and
   Dataview plugins when available, using the fixture-copy pattern from the existing
   harness. Write deterministic fixture plugin settings and the `#task` filter. If
   repository access is needed for live-vault assets, use `/sase_repo` first and read
   from its returned path; never edit deployed plugin sources. Missing external
   prerequisites are reported as explicit unrun coverage, not passing assertions.
4. Assert models and DOM attributes on actual visible rows, paging/scrolling the fixture
   so every applicable vector is exercised in Live Preview and reading view, in light
   and dark. Exercise real embeds, hover preview, and Dataview TASK results. Test the
   toggle via the registered command: removal of row/tooltip attributes, restoration of
   the old computed appearance, then reactivation without reopening the note. Compare an
   identical checkbox click with the gate on and off to prove the same stored status
   change. Drive a real hover and assert Obsidian's tooltip text; attribute presence
   alone does not prove the tooltip works.
5. Probe computed glyph/mask, family paint, pip geometry, checkbox hit area, and row
   height. Check nested children do not inherit parent glyphs, theme changes repaint,
   and marks work with the snippet disabled. Exercise natural cache/document refresh and
   bounded config/day scenarios; clearly distinguish controlled-clock tests from an
   actual midnight observation. Fix integration defects in these four phases' code,
   without adding the later folds, Tasks CSS, or card/chip adoption.
6. Run `xvfb-run -a node scripts/obsidian-block-marks-e2e.cjs` with explicit output and
   report paths. Use `/sase_monitor` for a long real-app run and wait for the monitor
   handoff command to exit. Capture at least light/dark × Live Preview/reading
   screenshots and evidence for the secondary surfaces. **Open and inspect every
   screenshot** against the audited research mockup: glyph alignment, baseline, stroke
   weight, pip clipping, held/waiting balance, and nested-row isolation. Iterate to a
   passing real-app report and acceptable visuals.
7. Store the report and reviewed screenshots as audited artifacts associated with the
   child phase, and note their canonical refs for the land agent. Update the contract
   with any measured mechanism changes and mark only checks actually verified. Leave
   iOS, third-party checkbox packs, real-reconcile recovery, and human scrolling checks
   for the parent visual-review phase.

Done when the harness, its offline wrapper, focused suites, and real-app checks pass,
and the screenshots have been inspected rather than merely generated.

## Checks and shipping for each child phase

- Recheck current changes, edit source fragments only, and regenerate `main.js` with
  `npm run build`. Never hand-edit generated bundles. Keep each hand-edited fragment at
  or below 1000 lines.
- Bump the touched ledger plugin's minor version once per child phase, from its current
  value; update its description only as behavior ships. Run `npm run docs:gen` before
  the inventory gate, focused new/affected node tests, `npm test`, `npm run validate`,
  and `npm run docs:check` in bob-plugins.
- Run `just docs-check` in bob-cli whenever its docs change. Use `/sase_monitor` when
  checks require a long run; do not end a turn with an unmanaged command running.
- Run `bob plugins sync` after changes to bob-plugins, verify deployment, and include
  both repositories' changes in the normal `/sase_final` declaration. Do not manually
  commit or leave new source/tests untracked at finalization.
- `bob-cli-ft.2` recorded eight pre-existing full-suite failures: task-wikilink
  performance (`bob-cli-f8`), keep-inbox timezone (`bob-cli-f7`), note-ready/relation
  startup doubles (`bob-cli-eh`), freshness real-date (`bob-cli-et`), and the 2026-11-02
  Pending schedule fixture (`bob-cli-ff`). These are leads, not blanket waivers. A
  failure must reproduce identically on this run's clean base before it is excluded from
  phase completion. Record commands, base SHA, and matching output in a
  `PROPOSED FOLLOW-UP:` note on the assigned phase, citing the existing task.
- Child phase workers must not create task beads. Record discoveries with
  `sase bead note <assigned-id> 'PROPOSED FOLLOW-UP: <summary — evidence>'` for the land
  agent. Re-run `sase bead epic-symbols <assigned-id>` before closing; resolve entries
  or re-key them to a still-open owner. Close only the assigned phase with a
  verification note. Do not hand-edit status or close ancestors.

## Child epic land agent

Re-read the parent decisions, child phase notes, and report artifacts. Verify the
combined model/render API, all refresh paths, generated bundles, docs, deployment,
contrast measurements, and real-app evidence. Run the focused integrated suites, full
npm checks and bob-cli docs gate on the combined tree, observing the clean-base failure
rule above. Review actual screenshots. Confirm later parent phases can call
`api.blockMarks.render` and `blockMarkModelForLine` without any new dependency or change
to stored Markdown.

The rendering scope is complete only when all applicable BM source-note vectors and
negative cases work in both themes/views, secondary surfaces have real evidence, click
behavior is unchanged, and toggling restores the prior appearance. Do not substitute an
offline self-test for this evidence or mark skipped real-app coverage as passed.

Close this child epic through its normal land workflow after its children are done. Then
run `sase bead epic-symbols bob-cli-ft.3`, resolving or re-keying leftovers, and close
**only bob-cli-ft.3** with
`sase bead close bob-cli-ft.3 --note "<verified surfaces, commands, screenshots, deployment, and any clean-base exceptions>"`.
This is the explicit completion handoff for the original assigned phase. Leave
`bob-cli-ft`, `bob-cli-ft.4`, and all other parent phases to their existing owners.
