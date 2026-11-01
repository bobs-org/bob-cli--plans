---
tier: epic
title: Bob Mac Capture directive menus and clipboard previews
goal: Complete bob-cli-ey.3 with unselected directive and type menus, a grammar-2
  compatibility gate, real clipboard previews, and no retired capture paths.
phases:
- id: wire-fixtures
  title: Decode the directive contract and generate current wire fixtures
  size: medium
  depends_on: []
  description: 'wire-fixtures: add grammar, initial_selection, directive spans and
    candidates, and clip status decoding; migrate template.directive and route-only
    global destinations; generate truthful directive and clipboard fixtures from the
    current Bob binary.'
- id: retire-syntax
  title: Remove retired Mac capture paths and migrate their scenarios
  size: medium
  depends_on:
  - wire-fixtures
  description: 'retire-syntax: delete template-start rewrites, the template route
    row, global parent routing, and retired span mappings; migrate affected fake-bob
    scenarios, generated fixtures, tests, and documentation to percent directives.'
- id: menus-gate
  title: Implement unselected menus, directive rows, and the grammar gate
  size: medium
  depends_on:
  - retire-syntax
  description: 'menus-gate: honor initial_selection across completion contexts, render
    Bob-authored directive rows and bare-hash type menus, style directive spans, and
    require grammar at least 2 for capture without losing the draft.'
- id: clip-preview
  title: Read clips in live preview and render the Clip card
  size: medium
  depends_on:
  - menus-gate
  description: 'clip-preview: authorize live clipboard reads only from the current
    Bob parse, refresh on activation and pasteboard counter changes, render all clip
    modes and not_read, and complete the Mac preview documentation and regression
    coverage.'
proposed_by: bbugyi200.apollo.bob-cli-ey.3
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-ey.3
create_time: 2026-10-31 23:32:57
status: done
bead_id: bob-cli-ey.3.1
---

- **PROMPT:** [prompts/202610/mac_directives.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/mac_directives.md)
- **PARENT:** [202610/capture_drawers_and_directives.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/capture_drawers_and_directives.md)
- **BEAD:** [bob-cli-ey.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ey/bob-cli-ey.3.1.md)

# Bob Mac Capture directives: implementation child epic for bob-cli-ey.3

## Objective and scope

Complete the `mac-directives` phase of `plan:202610/capture_drawers_and_directives.md`,
assigned as `bob-cli-ey.3`. The resulting app opens `%` and bare-`#` menus with no
selected row, renders Bob's directive explanations, previews explicitly requested
clipboard content, and refuses to capture with a Bob grammar revision below 2. Retired
grammar has no compatibility paths. `schema_version` remains 1.

This is a child epic because wire migration, selection across the panel's many prompts,
and asynchronous clipboard presentation each need separate regression work. All four
phases are sequential: they touch shared models, the panel, and fake-bob. Each is
bounded implementation work for one coding agent. No parallel agents are needed within a
phase.

At planning time, `bob-cli-ey.3` is in progress with no earlier implementation notes.
The CLI prerequisite `bob-cli-ey.1` is closed. Bob CLI commit `b290d833` exposes grammar
2 and directive completion; preceding commits implemented `%c`, `%n`, `%l`, `%t`, and
the bare-hash refusal. The inspected Mac head, `91a5d2b`, still contains
`CaptureTemplateStart`, required global-destination `mode`, required clip `mode`,
first-row-only selection, and the literal-clipboard caption. This work has not landed.
Recheck the assigned bead and repository history before implementing a phase.

## Authority and boundaries

- Open `bob-mac-capture` through `/sase_repo`; read its `AGENTS.md` before editing.
  Paths below are relative to that repository unless explicitly marked bob-cli. Read the
  parent plan using `sase artifact read`, never a direct sidecar file read.
- Bob owns syntax, completion order and replacement ranges, directive copy and refusals,
  previews, and writes. Swift renders Bob's data and orchestrates processes. Do not
  infer `%c` with a regex, parse templates, or read/write vault notes in Swift.
- The parent's DECISIONS remain final. This child needs **no memory edits**; the
  authorized glossary/decision changes belong to the parent CLI phases. Do not edit
  other memory, `CHANGELOG.md`, Cargo versions, or historical plan/research artifacts.
- Drawer menu implementation, task-section renames, attached-type removal, and the
  revision-3 minimum belong to sibling `bob-cli-ey.5` and the CLI drawer phases. Accept
  grammar 3 or later, but this child requires only grammar 2. Do not implement drawer
  semantics. The generic no-selection mechanism must be reusable by that phase.
- Keep valid `@@route`, saved-template shelf identities, separator toggles, aggregate
  one-call capture, task-type behavior outside the changed bare-`#` case, and existing
  task/Pomodoro preview cards working.
- Never create task beads from these phases. Record discovered work on the assigned
  phase with `sase bead note <assigned-id> 'PROPOSED FOLLOW-UP: ...'`.
- A child phase closes only its own assigned bead, after `sase bead epic-symbols` and
  resolving/re-keying its entries. The child epic's land agent owns complete-scope
  verification and closure of `bob-cli-ey.3`; ordinary child phase workers do not. Never
  close `bob-cli-ey` or any higher ancestor. Do not set bead status by hand.

## Wire contract to implement

Use the actual landed Rust serializers as the source of truth:
`src/native/capture_parse.rs`, `capture_complete/model.rs`,
`capture_complete/directives.rs`, and `capture_clip/model.rs` in bob-cli.

- Parse and completion have top-level integer `grammar`; decode a missing value as
  missing so the gate can distinguish it. Never silently substitute 2 on wire decode.
- Completion adds optional `initial_selection: "none"`. Omission preserves existing
  first/best/suggested-row behavior. The flag applies to all completion contexts.
- A `directive` span has optional `directive` (`clip`, `note`, `template`, `listen`) and
  `count`. Bare `%` has the span and `directive` need without a directive value.
- Directive candidates have `token`, `name`, `summary`, optional `detail`, optional
  `disabled_reason`, `replacement`, and `cursor_after`. Do not require task/route-only
  fields or assume that a directive candidate supplies `kind` or `label`.
- Clip read output has `status: "read"`, `mode`, `lines`, `attachments`, optional
  `snippet`, and `entries`. Modes are `inline`, `lines`, `attachments`, `snippet`,
  `history`. Unread output is **only** `status: "not_read"` and `count`; it has no
  `mode` or collections. History entries are clip objects. Remove `clip.header`.
- Template parse metadata uses `template.directive {range,line}`, replacing
  `template.header`. Preserve source ranges for the typed draft and template rows.
- A global destination contains route/range/needs as applicable, without `mode` or
  `block_id`. Remove its sub-bullet summary. Preserve the semantic `ref_listen` boolean
  and `pomodoro_note` mode/kind; only their old span spellings disappear.

## Phase wire-fixtures

Work primarily in `Sources/CaptureCore/CaptureModels.swift`, fixture generators, and
decoding tests.

1. Implement the above wire models, constructors, CodingKeys, and decoders. Reuse
   existing candidate fields (`name`, `disabledReason`, `detail`) where appropriate; add
   token/summary rather than synthesizing them from replacement text. Keep nil
   distinguishable for absent grammar. Update direct constructor call sites affected by
   removing global mode/block ID, clip header, and template-header metadata.
2. Remove global-parent scope rendering from `CaptureGlobalDestination.scopeSummary` and
   related summary/notification helpers when removing those fields. Delete obsolete
   positive assertions instead of retaining an alternate old-wire model.
3. Add reproducible `Scripts/generate-*-fixtures.sh` coverage for directives and clips.
   Require explicit `BOB_BIN` built from the current bob-cli checkout containing the CLI
   prerequisite, verify its grammar, and record the binary revision and inputs. Use
   isolated vault/config/cache/time; mock clipboard/history commands through
   `BOB_CLIPBOARD_CMD` and `BOB_CLIPBOARD_HISTORY_CMD`. Never read the real clipboard.
4. Generate parse and completion cases for bare/typed `%`, bare/typed `#`, enabled and
   disabled rows, URL-only `%l`, `%n` target detail, `%t`, escaped/literal percent,
   later batch items, and Unicode before replacement ranges. Generate clip previews for
   every mode, history, and `not_read`, plus current global-route and template metadata.
   Extend existing generators where they own these scenarios. Assert successful outputs
   and expected error envelopes; do not manufacture grammar-2 payloads by adding a field
   to an old-syntax fixture.
5. Add focused portable decoding tests: all new rows decode without unrelated fields;
   `not_read` decodes without `mode`; missing/lower/higher grammar remains observable;
   directive/count and UTF-8 ranges survive; template.directive and route-only globals
   match real Bob. Update existing assertions impacted by model changes. Cover
   nested/history clips and normalized batch capture objects.

Run the common compile/check requirements below. The exit condition is a working
current-wire model and reproducible fixtures, with no old-field decoding dependency.
This phase does not activate the compatibility gate or implement the menus/cards.

## Phase retire-syntax

This phase owns the broad migration of existing scenarios so the interaction phases
start from a consistent current grammar.

1. Delete `Sources/CaptureCore/CaptureTemplateStart.swift`, its portable and hosted
   tests, and template-start keystroke/retry/provenance branches in
   `CapturePanelModel.swift` and controller hooks. Keep unrelated separator rewrite
   recovery and its undo/caret tests. Typing `@` must not be intercepted for a template.
2. Remove the `@@` template candidate, hint, hard-coded row, acceptance special cases,
   and save-template fallback message. Command-S presents Bob's returned error copy.
   Preserve shelf save, insertion, preview, and successful capture with `%t` drafts; use
   Bob's draft response rather than constructing template syntax in Swift.
3. Delete global-parent handling and retired `global_sub_bullet_*`, `template_header`,
   `clipboard`, `ref_listen`, and `pomodoro_note` span mappings wherever present. Remove
   retired span strings from completion trigger sets too. Introduce the shared directive
   semantic category and palette case now if needed to keep switches exhaustive; the
   menus phase completes its presentation.
4. Migrate `Tests/Fixtures/fake-bob`, positive fixtures, test drafts and expected
   ranges, and existing generators from trailing hash notes, old clip/listen syntax, and
   template headers to `%n`, `%c`, `%l`, and `%t`. Replace removed global-parent
   scenarios with valid `@@route` coverage or remove them when they test only deleted
   behavior. Regenerate, rather than hand-edit, Bob-authored JSON and source offsets.
   Recheck `Scripts/generate-task-type-fixtures.sh`: it currently teaches `jot idea #`
   as a Pomodoro note. That scenario must now describe an incomplete task type.
5. Update affected tests in template presentation/shelf, model, process-client,
   reference presentation, File-under, notification, and completion suites. Keep
   successful save/reload, one-step undo for supported gestures, mixed batches, literal
   escaped markers, and `%t` row source mapping covered.
6. Update current-syntax examples and explanations in `docs/capture-panel.md`,
   `capture-previews.md`, `keyboard.md`, `saved-templates.md`, `task-pickers.md`, and
   `protocol.md`, plus any affected current documentation discovered by the sweep. Do
   not add migration/deprecation sections or error text teaching retired spellings.

Run the common checks. The parent's retirement sweep must be clean apart from the
literal-clipboard caption, which belongs to the final clipboard phase. The final scope
has no positive old-syntax fixtures or behavior. Task-section/drawer migrations remain
owned by the sibling phase.

## Phase menus-gate

1. Represent no selected completion explicitly in `CapturePanelModel` and propagate it
   through prompt snapshots/restoration, completion caching, row views, task-type
   presentation, accessibility, and scrolling. Audit every consumer of
   `selectedCompletionIndex`, including arithmetic and array indexing; unset must never
   become row 0 through clamping or defaulting. Server `initial_selection` takes
   precedence over suggested-row defaults. Include responses routed into a dedicated
   picker, not only `applyNonActiveTaskCompletion`. An omitted flag keeps today's
   selected-row behavior.
2. Update keyboard routing in the controller/key-action types as well as model
   acceptance. With nothing selected, Return submits the unchanged draft through the
   normal capture path; Bob's incomplete-marker error remains visible and the draft
   stays. Tab takes row 0 as specified by the parent menu/footer, while Down/Ctrl-N
   select row 0 without inserting. Further navigation follows existing behavior; Escape
   closes the menu, and typing obtains a fresh ranked selection. Preserve explicit
   selected-row acceptance and open-after-capture behavior. A disabled row consumes
   acceptance without editing or submitting, including mouse acceptance.
3. Add `directive` needs/spans to `shouldRequestCompletion`. Bare `#` opens the existing
   task-type menu immediately through Bob's need, with no selected row. Do not treat
   either incomplete sigil as a quiet picker that hides Bob's refusal.
4. Render the directive context through `CompletionRowContent.swift` and the panel:
   token, name, summary, detail, disabled reason in Bob's order and wording. The
   selected-row footer uses Bob's summary; with no selection it shows the parent's
   static key hint. Directive spans use one distinct tint family in both appearances.
   Swift does not recreate eligibility, ordering, target descriptions, or `%l` logic.
5. Gate capture on `capture-parse` grammar >= 2. A missing/lower value displays exactly
   `Update bob to use drawers and % directives`, disables Capture, and preserves the
   draft. Gate keyboard, button, open-after, and any direct/resumed submission entry
   points; no confirm-twice or translation fallback. A pending/failed parse cannot
   bypass the check. Bind capability evidence to the process client identity and current
   analysis; invalidate it on Recheck Bob/client replacement. Older responses must not
   overwrite a newer compatible state. Keep schema checks intact.
6. Regenerate any remaining normal parse/completion fixtures and update intentional
   in-memory test doubles to state their actual grammar. Retain explicit missing/1
   compatibility cases. Add hosted tests for Return/Tab/Down/Ctrl-N/Escape, filtering,
   disabled acceptance, snapshot restoration, generic contexts, Unicode offsets, and
   stale response generations. Test missing/1/2/3 grammar, draft preservation, blocked
   subprocess submission, and recovery after a compatible Recheck Bob.
7. Finish menu/key docs and the `docs/protocol.md` compatibility table for minimum
   grammar 2. Extend design/presentation fixtures for unselected and disabled menus.

Run the common checks. This phase exits with functional menus and the compatibility
gate; clipboard dry runs still use `--no-clip` until the final phase.

## Phase clip-preview

1. Thread evidence from the current `capture-parse` result into `startLivePreview` and
   `BobProcessClient.captureLivePreview`. Omit `--no-clip` **only** when that same
   draft's accepted parse reports a `directive` span with `directive == "clip"`. Scan
   the Bob-provided batch spans as appropriate. A bare `%`, escaped/literal `%c`, or a
   previous draft's clip span never authorizes a read. Keep `--dry-run` and JSON
   arguments unconditional. Update `preconditionLivePreviewArguments` to enforce this
   evidence-sensitive invariant.
2. Preserve debounce, process-lane cancellation, priority-roll seed, and analysis
   generation/client-identity checks. Removing a directive or closing/rechecking the
   panel cancels or invalidates older reads. A delayed clip result must not restore
   stale content. Reuse the same Bob clipboard path as submission; Swift never reads
   pasteboard content or writes attachments during preview.
3. On panel activation, refresh the preview. While the panel is key and its current
   compatible parse has a clip directive, observe only
   `NSPasteboard.general.changeCount` at no more than 1 Hz. A change triggers the
   debounced preview; stop polling on hide, focus loss, draft removal, and teardown. Use
   an injectable counter/clock or bounded test seam for lifecycle tests. Preserve
   current-draft and generation checks when the clipboard changes without an edit.
4. Add a portable clip presentation model and a hosted Clip card, integrated per capture
   item in batch previews. Render kind/count headers, Bob's exact child lines, snippet
   destination, attachment summaries, readable image thumbnails from
   `attachments[].source`, and entry dividers for history. `not_read` shows a quiet
   placeholder. Missing/unreadable image sources remain useful attachment rows. Do not
   depend on a dry-run destination file existing. Preserve task-block preview
   deduplication so clip lines are not accidentally repeated inside the same block.
5. Delete `livePreviewUsesLiteralClipboard` and the literal-clipboard caption, tests,
   and stale comments claiming every live preview skips clipboard reads. Complete the
   preview/permission/protocol documentation for explicit reads and counter-only
   polling, without claiming that merely opening `%` reads the clipboard.
6. Add argv tests for read/no-read, every clip mode and `not_read` presentation, `%c3`
   entry boundaries, readable/unreadable image fixtures, batch placement, activation
   refresh, <=1 Hz polling, and teardown. Cover changed clipboard while the draft is
   unchanged, removal during an in-flight preview, and delayed old parse and preview
   responses. Hosted async tests must have bounded waits.

Run the common checks and complete-scope acceptance below. Leave closure of
`bob-cli-ey.3` to the child epic land agent.

## Compile and check requirements for every phase

- Read the linked repository's current `AGENTS.md`. Format only changed Swift lines. Run
  `just portable-check` on a host with Swift; if unavailable locally, use an isolated
  scratch copy on tailnet `athena`. Never declare Swift verified merely because a source
  search or parse succeeded.
- For changes to `Sources/BobMacCapture*` or `Tests/BobMacCapture*`, attempt the
  required scratch-copy Mac compile: rsync without `.git`/`.build`, run
  `Scripts/bundle.sh` and `just typecheck-tests`, and remove the scratch copy. Use
  `/sase_memory_read` for tailnet details. Do not use Bryan's checkout or install the
  app. Mac Command Line Tools lack XCTest: hosted test execution belongs to macOS CI,
  while the scratch typecheck covers those targets with the repository's stub.
- If the Mac is unreachable after a bounded attempt, report exactly **macOS targets
  unverified**, retain the portable results, and carry the missing evidence to the land
  report. Do not silently skip the compile attempt or claim hosted tests ran.
- Run `just docs-check` for Mac documentation changes and `just check-scripts` for
  changed shell/fixture tooling. Use `/sase_monitor` for long commands and wait for the
  monitor handoff command itself to exit. Each phase records commands/results.
- If a failure reproduces identically on a clean base, record it as a
  `PROPOSED FOLLOW-UP:` note with reproduction evidence and an existing task ID if
  known, then close the completed assigned phase; do not leave it in progress for
  unrelated checks. Fix regressions caused by the phase before closing it.

## Child epic land acceptance

The land agent gathers passing phase evidence, inspects the integrated diff, and re-runs
checks when integration or changed code warrants it. Verify:

- With a current Bob, `jot idea #` and `jot idea %` open unselected menus. Return
  preserves the input and shows Bob's refusal; Tab takes the first row, typing filters,
  Escape closes, disabled rows do nothing, and normal contexts retain their selection.
- `%n`, `%t` with rows, URL `%l`, `%c`, and `%c3` use real regenerated Bob fixtures.
  Saved-template identities and one aggregate capture call remain intact. `%c` live
  previews display actual content without creating files; bare `%` never reads it.
- Grammar missing or 1 blocks every capture route with the prescribed message; grammar 2
  and higher work. Stale parses/previews and client replacement cannot bypass the gate
  or revive old clip content.
- Portable checks, linked docs/scripts checks, and required Mac build/typecheck evidence
  are recorded with their actual limits. Do not install or deploy the app.
- Run the parent's retirement sweep in bob-mac-capture:
  `git grep -n -e '@@template' -e TemplateStart -e start_template -e global_sub_bullet -e 'stay literal'`.
  It must find nothing outside immutable changelog/history. Also inspect removed span
  mappings, clip header, old positive clip/listen/note syntax, template.header, and
  global-parent summaries. Keep semantic `ref_listen`/`pomodoro_note` fields and valid
  `@@route`; do not delete unrelated prose just because it contains a sigil.
- Before closing any owned bead, run `sase bead epic-symbols <id>` and resolve its
  entries or re-key unfinished work to an appropriate still-open bead. At planning time
  `bob-cli-ey.3` has no entries. Recheck it before its final close.

After all child phases and the child epic are complete, the child epic's **land agent**
is authorized to close only the delegated original phase `bob-cli-ey.3` with
`sase bead close bob-cli-ey.3 --note "<integrated behavior and verification evidence>"`.
Do not close `bob-cli-ey` or any ancestor above that delegated phase. If the host
already closed a completed bead, record additional evidence as a note rather than
reopening it. Use `/sase_final` for the final declaration and all changed repositories.
