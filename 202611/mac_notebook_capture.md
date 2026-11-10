---
tier: tale
title: Present notebooks in Bob Mac Capture
goal:
  Complete bob-cli-j8.8 with compatible notebook decoding, pickers, previews, success
  messages, and macOS verification.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-j8.8
proposed_with: codex/gpt-6-astra
bead: bob-cli-j8.8
create_time: 2026-11-10 02:40:02
status: wip
---

- **PARENT:**
  [202611/bob_notebooks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bob_notebooks.md)
- **BEAD:**
  [bob-cli-j8.8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-j8/bob-cli-j8.8.md)

# Present notebooks in Bob Mac Capture

Complete **bob-cli-j8.8**, the `mac-notebook-capture` phase of
`plan:202611/bob_notebooks.md`. The approved epic is authoritative; the relevant
research is `research:202611/bob_notebooks/bob_notebooks.md`, Capture and V8–V9. The
result is a Mac capture panel that clearly presents notebook destinations, note bullets,
notebook creation, and successful captures using Bob's JSON.

## Scope and evidence

This is one bounded implementation task in the linked **bob-mac-capture** repo. Open it
with `/sase_repo` and use the returned checkout; read its `AGENTS.md`. All Swift paths
below are relative to that repository. Read the assigned bead again before starting,
preserving any work that has landed since this proposal.

At planning time bob-cli-j8.8 is in progress, has no earlier work notes, and has no
`--epic-symbol` entries. Its CLI dependency bob-cli-j8.5 is closed. The primary checkout
includes notebook note and creation planning plus completion assistance (including
commit `93cf930c`). No notebook handling exists in the inspected Swift sources. The
alias phase bob-cli-j6.2 is closed; the linked client includes `b1a90faf` and subsequent
alias lifecycle changes. Preserve those changes.

The concrete gaps are:

- `CaptureModels.swift` decodes an unknown `CaptureBlockIDIntent` as `.link`;
  `BlockIDDecodingTests.swift` explicitly pins that behavior.
- `BlockIDPickerContext.isNewIDMode` recognizes only `.new` and `.projectNote`.
- `CaptureTarget` drops notebook title, summary, owner, and breadcrumb metadata;
  completion candidates already have optional `title` and `summary` fields.
- `CapturePanelModel.fileUnderCompletion` excludes only `kind == "record"`, so notebooks
  would be offered as reference parents.
- `ParentTaskPickerPresentation.baseRow` always emits a task glyph, and its
  accessibility label always starts with a task status.
- Existing preview-file pairing accepts only record kinds. `CaptureNoteFile` already
  supports missing `record_type` and contains Bob's exact Markdown.

The parent decisions are final: `collapse_assist = yes`. The rewrite already lives in
Bob and requires no new Swift grammar. No memory change belongs to this phase; leave the
epic's approved glossary and decision edits to its wrap-up phase. Do not change the
grammar minimum revision, release version, or changelog, migrate the vault, or add
notebook semantics to task-only backend scanners.

## Wire contract and compatibility

Read `decisions:mac-capture-is-a-thin-client`, the notebook glossary, and `docs.md`
through `/sase_memory_read`. Bob remains the only grammar, preview, and mutation
authority. Use additive optional decoding and Bob-provided replacement ranges. Older Bob
responses must still decode and retain their current behavior.

Use current bob-cli code and isolated-vault JSON as the contract:

- `src/native/capture/notebook_note.rs` defines per-item `notebook`: `route`, `title`,
  `breadcrumb`, nullable `section`, `section_action`, `block_id`, and `parent_block_id`.
  The item kinds are `notebook_note`, `notebook_sub_note`, and `notebook_create`.
- Creation also emits top-level `note_files[]` with `path`, `action`, `properties`,
  `body`, and exact `markdown`; `record_type` is omitted. Pair by the item's
  `relative_target`, respecting draft order and the existing shared-file policy.
- Notebook routes carry `kind: "notebook"`, `title`, `summary`, `owner`, and
  `breadcrumb` in targets/completion. Preserve Bob's route identity and insertion text
  while presenting its breadcrumb and summary.
- `block_id.destination_kind` is optionally `"notebook"`; the intent for new notebook
  IDs remains `"new"`.
- `+` candidates carry `item_kind: "note"`. ID-less candidates use the existing
  `task_ref`, `requires_block_id`, and `block_id_suggestions` assignment protocol.

## Implementation

### 1. Decode notebook metadata and make unknown intents safe

In `Sources/CaptureCore/CaptureModels.swift`, add `.unsupported` for an explicitly
unknown block-ID intent. Keep absent/null intent's legacy `.link` default so old
responses retain their meaning. Update the misleading enum comment and the pinned
unknown-intent test. Keep recognized `link`, `new`, and `project_note` unchanged.

Decode optional `destinationKind` on `CaptureBlockIDField`, `itemKind` on
`CaptureCompletionCandidate`, notebook target metadata, and the per-item notebook
summary on `CaptureCommandSuccess`. Follow the existing tolerant optional decoding
conventions. Update initializers, coding keys, encode paths where present, and
candidate/target reconstruction sites so cached completions retain the metadata. Put
notebook-specific wire/presentation types in a focused CaptureCore file instead of
adding more unrelated presentation code to the large model file.

Update `CapturePickerPresentation.swift`, `BlockIDPickerIndex.swift`, and panel routing
so unsupported intents open plain new-ID typing and never browse, insert, or trigger
existing-task links. Use Bob's ID rules and replacement spans; disable actions when
required data is absent. Suppress task/Pomodoro/project follow-up teaching for
unsupported intents and notebook destinations. Preserve normal ID suggestion selection
and type-through for recognized notebook `new` intents.

### 2. Present notebook routes and note-bullet choices

Add `book.closed` for notebook kinds to every existing kind presentation map:
`CapturePickerView.noteKindIcon`, `noteKindHeaderIcon`, route completion rows,
`ParentTaskPickerPresentation.noteKindSubtitle`, Similar band presentation, and
`CapturePullInChoice`. Audit other kind maps, including TaskLink presentation, for the
same display fallback. This is presentation support, not permission to offer notebooks
in task-only pickers.

Show notebook route summaries as detail and use the returned breadcrumb/title for human
labels, with existing labels as the fallback. Cover both fresh server rows and cached
route rows; do not replace route/replacement strings with labels.

Change File-under to an explicit allowlist of `inbox`, `area`, and `project`, including
its last-used-parent handling. Ordinary capture route completion still offers notebooks.
Records and unknown future kinds stay out of File-under.

In `ParentTaskPickerPresentation` and its picker view, branch on `itemKind == "note"`
for the `+` rows: show a plain note glyph or no glyph, omit checkbox/status language
from the row, details, and accessibility text, and retain Bob's order, hierarchy,
disabled reasons, stable identity, and selection behavior. Do not filter out note rows
because their status is empty. Preserve the existing pending-ID flow: an ID-less note
calls `bob capture-task-id`, then inserts only the returned stable ID; cancellation or
assignment failure leaves the draft intact. Do not mint IDs or edit vault files in
Swift.

### 3. Render notebook previews and success messages

Add a portable notebook presenter used by `CapturePanelView`, `CapturePanelModel`, and
`NotificationService` so preview, success, batch summaries, and accessibility copy
agree. Drive it from `kind` plus decoded metadata, never by parsing draft text.

- A notebook note reads `📓 Note → <title>`, with `› <section>` when supplied and a
  distinct `new section` badge for `section_action == "create"`. Display a supplied
  block ID without implying that a task was created.
- A sub-note reads `📓 Note → <title> › ^<parent_block_id>`.
- Creation reads `📓 New notebook <path> · in <parent context>` using Bob's returned
  metadata. Show the supplied summary/body and exact Markdown from the matching
  `note_files[]` entry. Reuse the record preview's View Markdown disclosure without a
  record-specific heading or fabricated properties. Use the decoded breadcrumb for
  parent context; never derive ancestry by splitting a filename.
- Render note content and nested detail using existing rich-text primitives without task
  controls. Keep mixed drafts and create-then-use drafts correctly associated with their
  result items; never zip filtered items and files by array position.
- Add notebook handling to single and batch success notifications and panel status.
  `friendlyKindLabel` returns `Notebook note` for both note kinds and `Notebook` for
  creation. Preserve a usable generic fallback if optional metadata is missing.

Keep notebook items out of the Similar-tasks band using Bob's returned results and the
existing draft-generation invalidation; ensure an old task band's rows do not linger
after the draft becomes a notebook note. Do not add client-side similarity search or
vault reads.

### 4. Prove compatibility and document the visible behavior

Add focused fixtures from the current Bob against an isolated `BOB_DIR` with a fixed
`BOB_NOW`, using the existing fixture conventions. Include a notebook under an area, a
child notebook, an ID'd note, an ID-less note, existing/new sections, notebook creation,
and a mixed create-then-use draft. Use these in portable presenter/decoder tests and
macOS panel tests; no real vault mutations are needed.

Extend the existing BlockID decoding/index, ParentTask picker, File-under panel, record
preview, and notification test suites, plus focused notebook tests. Prove:

1. Known/missing intents retain their behavior; an unknown intent cannot select or
   insert an existing-task link. Notebook ID composition follows Bob's suggestions.
2. Optional notebook metadata survives decode and cached-row reconstruction; omitted or
   malformed optional metadata cannot break legacy task/record responses.
3. Route summaries and notebook icons display correctly; File-under excludes notebooks
   even when one was the remembered route. Ordinary route completion continues to
   include them.
4. Note `+` rows have no task checkbox or status narration. Both existing-ID and ID-less
   selections work, including assignment cancellation/failure.
5. All three notebook kinds render the specified preview and success copy, including
   new-section badges, nested parent context, exact creation Markdown, mixed drafts, and
   missing optional metadata. Existing record-file pairing still passes.
6. Existing alias escapes and Bob-provided `@@route` collapse edits continue to use the
   normal rewrite/undo path; notebook UI introduces no new grammar gate.

Update the linked app's `docs/capture-panel.md` and `docs/keyboard.md` where they
describe destination kinds, `^`/`+`, `%n` creation, File-under, or the existing `@@`
collapse. If the preview contract lives in `docs/capture-previews.md`, update that
source too. Do not hand-edit mounted copies in bob-cli. Keep additions compact and
follow the documentation memory's ownership rules.

## Verification and completion

Run the linked repo's formatting and portable checks with its available toolchain
(`just portable-check` when Swift is available). Run the appropriate focused tests while
developing, then the full macOS gate: from the linked checkout, use
`Scripts/macos-ci-check`, which snapshots the working tree to a temporary branch and
checks the exact SHA. Run this long command through `/sase_monitor`; wait for the
monitor-start command itself to exit and finish through `/sase_final` when it reports
Staged. Resume from its result and fix any new failures before closure. If using a
normal push instead, monitor that exact SHA's macOS run to green. Linux checks alone
cannot verify the SwiftUI/AppKit targets. The macOS gate must include build, tests,
bundle/signature, launch smoke, and install checks.

Run `just docs-check` from bob-cli against the current mounted app docs and complete any
required repository checks. bob-cli-j6.2 recorded existing mounted-page word budget
failures tracked by bob-cli-h9; do not assume they still reproduce. For any failure,
compare against the clean base. A failure that reproduces identically on the clean base
is recorded as a `PROPOSED FOLLOW-UP:` note on bob-cli-j8.8 with evidence and the
existing tracker, and does not keep this phase open. Any new failure introduced here
must be fixed. Do not create task beads.

Record the changed behavior, test results, exact-SHA macOS evidence, and any baseline
failure evidence on bob-cli-j8.8. Before closing, rerun
`sase bead epic-symbols bob-cli-j8.8`; resolve any remaining entries or re-key them to a
still-open bead. Close **only bob-cli-j8.8** with
`sase bead close bob-cli-j8.8 --note "<what was verified>"` when its scope is complete.
Never close bob-cli-j8 or any ancestor plan. Use `/sase_final` as the final action and
declare every repository changed during implementation.
