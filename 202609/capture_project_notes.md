---
tier: epic
title: Capture project notes with @route^id+ and @route:id+
goal: '`bob capture` can create a new sub-project note `<route>_<block_id>.md` — with
  a `parent` wikilink back to `<route>.md`, a seeded `^prj` lifecycle task, authored
  child tasks under `## Tasks`, and authored ALL-CAPS sections as `##` headers — from
  the new `@<route>^<block-id>+` and `@<route>:<block-id>+[#<pomodoro>]` markers,
  and Bob Mac Capture highlights and reports the new family correctly.

  '
phases:
- id: grammar
  title: Project-note marker grammar
  depends_on: []
  size: medium
  description: 'grammar: add the `+` project-note sigil to the `^` and `:` marker
    families in the shared capture grammar, with a new `CaptureKind`, editor modes,
    span kind, needs, and diagnostics.'
- id: render
  title: Project-note content renderer
  depends_on: []
  size: medium
  description: 'render: add a pure module that derives the project-note basename and
    renders its frontmatter, `^prj` task, `## Tasks` entries, and ALL-CAPS section
    headers from a parsed capture item.'
- id: execute
  title: Capture execution and JSON contract
  depends_on:
  - grammar
  - render
  size: medium
  description: 'execute: wire project-note planning into the capture batch planner
    — parent-note validation, collision rejection, Pomodoro linking on `^prj`, marker
    conflicts, JSON fields, and human output — plus end-to-end CLI tests.'
- id: docs
  title: Capture documentation and help text
  depends_on:
  - execute
  size: small
  description: 'docs: document the new marker family in the capture guide, `bob capture
    --help`, and the grammar tables, and cross-reference the projects guide.'
- id: mac
  title: Bob Mac Capture frontend support
  depends_on:
  - grammar
  - execute
  size: small
  description: 'mac: teach the macOS capture panel the new span kind and capture kind
    so the `+` sigil is highlighted and project-note results are labeled correctly.'
proposed_by: bbugyi200.apollo.1a
create_time: 2026-09-20 18:07:10
status: done
bead_id: bob-cli-25
---

- **PROMPT:** [prompts/202609/capture_project_notes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/capture_project_notes.md)
- **BEAD:** [bob-cli-25](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-25/README.md)

# Plan: Capture project notes with `@route^id+` and `@route:id+`

## Background

Today `bob capture '@cash^goog-exit' 'Finish the Google exit packet!'` writes an
ordinary task into `cash.md`. Bryan frequently wants the other outcome: a brand new
**sub-project note** whose `parent` frontmatter points back at `cash.md`, exactly like
the Obsidian Bob Navigation Hotkeys command **Create project note from task**
(`bob-navigation-hotkeys:create-project-note-from-task`, bound to
`<ctrl+shift+option+n>`). This plan brings that outcome to the CLI so it works from Bob
Mac Capture and from scripts, with no Obsidian window open.

### Reference behavior (the Obsidian hotkey)

The reference implementation lives in the `bob-plugins` repo at
`plugins/bob-navigation-hotkeys/main.js` (open it with the `/sase_repo` skill; it is a
linked repo, so never read it out of `~/bob/`). The behaviors this plan mirrors:

- `getProjectBasenameFromTaskBlockId(sourceBasename, blockId)` returns
  `` `${sourceBasename}_${blockId.replace(/-/g, "_")}` ``. So a `^goog-exit` task in
  `cash.md` promotes to `cash_goog_exit.md`.
- `createProjectNoteFile()` instantiates `_templates/new_project.md` through Templater
  and then calls `applyProjectCreationFrontmatter()`, which sets
  `frontmatter.parent = <wikilink to the source note>`, `type = "[[project]]"`,
  `status = "wip"`, and `scheduled` when the source task carried one.
- `getFrontmatterWikiLinkToFile()` renders `[[basename]]` for a vault-root note and
  `[[path/to/note|basename]]` otherwise. Every capture route is a vault-root note, so
  the CLI only ever needs the `[[basename]]` form.
- `buildProjectContentFromTask()` replaces the template's completion-criteria
  placeholder with the source task's description, inserts converted child tasks into
  `## Tasks` (`replaceProjectTasksPlaceholder`), and inserts converted sections
  (`insertProjectSectionNotes`).
- `buildProjectSeedFromChildBullets()` decides, per direct child bullet, whether it is a
  **section** (no checkbox, ALL-CAPS title, at least one nested list item of its own) or
  a **task**. Sections are title-cased (`FUTURE WORK` → `Future Work`, `NON-GOALS` →
  `Non-Goals`, `API DESIGN` → `Api Design`) and their descendants are copied verbatim;
  everything else becomes a `## Tasks` task line.
- `isAreaOrProjectNote()` gates the command: project notes can only be created from an
  area note or a project note.

`docs/projects.md` in this repo already documents that contract from the CLI side,
including the sub-bullet-to-section rule and the schedule transfer.

### What the CLI already has

- `src/native/capture_language.rs` owns the whole marker grammar, shared by
  `bob capture` and `bob capture-parse`. `CaptureKind` has `Task`, `TaskWithBlockId`,
  `Bullet`, `Pomodoro`, `SubBullet`, `PomodoroNote`, and `TaskToggle` variants;
  `parse_task_block_id_route_token` and `parse_pomodoro_route_token` parse the `^` and
  `:` families.
- `src/native/capture.rs` owns execution. `CaptureBatchPlanner` keeps an in-memory
  snapshot of every touched file (`currently_exists`, `read_existing`,
  `current_contents`, `stage`), so creating a new note is just `stage()` on a path that
  does not exist yet, and later batch items see it.
- `plan_capture_with_pomodoro_link()` already shows how to write a routed note and the
  daily ledger atomically, including the `[[route#^block-id]]` Task Link, named Pomodoro
  selection, and named future-Pomodoro creation.
- `src/native/projects.rs` exports `parse_frontmatter`, `frontmatter_value`,
  `frontmatter_is_area`, `frontmatter_is_project`, and `ProjectStatus`;
  `src/native/ capture_targets.rs` already uses exactly those four to decide whether a
  vault-root note is a routable area or non-terminal project.
- `src/native.rs` excludes `_templates` from vault scans, and no CLI code has ever read
  a Templater template. The CLI therefore renders project-note content directly rather
  than instantiating `_templates/new_project.md`.

## Grammar design

### The new marker shapes

| Marker                            | Meaning                                                                               |
| --------------------------------- | ------------------------------------------------------------------------------------- |
| `@<route>^<block-id>+`            | Create the project note `<route>_<block_id>.md`; do not touch the daily note          |
| `@<route>:<block-id>+`            | Same, and link the new note's `^prj` task from today's implicit current/next Pomodoro |
| `@<route>:<block-id>+#<pomodoro>` | Same, targeting the named open Pomodoro or creating that named future Pomodoro        |

The `+` sigil sits **immediately after the block ID**, before any `#` component. This
placement is deliberate and must not be changed to a trailing `+`:

- `@<route>:<id>#<name>` already accepts `+` inside the Pomodoro-name charset
  (`is_pomodoro_selector_component` is `is_selector_component` plus `+`), so
  `@sase:deep-fix#bugs+` is a **valid existing capture** naming the Pomodoro `BUGS+`. A
  trailing `+` sigil would silently change that input's meaning.
- Placing `+` right after the block ID takes nothing away, because `is_block_id` only
  accepts letters, digits, and `-`. `@cash^goog-exit+`, `@cash:goog-exit+`, and
  `@cash:goog-exit+#bugs` are all hard errors today, so the grammar slot is free.

The existing family-detection predicates already route these tokens correctly and must
not need changes:

- `is_task_block_id_marker_candidate` requires the caret to precede any of `#`, `:`,
  `+`, so `@cash^goog-exit+` stays in the `^` family.
- `is_pomodoro_marker_candidate` requires the colon to precede `#`, `^`, and `+`, so
  `@cash:goog-exit+` and `@cash:goog-exit+#bugs` stay in the `:` family.
- `is_sub_bullet_marker_candidate` requires `+` to precede `:`, `#`, `^`, so it does not
  steal either new shape.

Verify each of those three claims with a focused unit test rather than trusting this
paragraph.

### Rejected and unsupported spellings

- `@<route>^<block-id>+#<anything>` — the `^` family has no third component. Reject with
  a focused message naming `@<route>:<block-id>+#<pomodoro>` as the form that does take
  a Pomodoro name.
- `@<route>^<block-id>+!` and `@<route>:<block-id>+!` — `!` remains reserved for the
  explicit sub-bullet toggle. Keep the existing `!` diagnostics; do not extend
  `exact_explicit_toggle_prefix` to this family.
- `@@<route>^<id>+` and `@@<route>:<id>+` — a global declaration that creates a note
  would try to create the same note once per item. These already fail as
  `invalid_global_destination` through `classify_global_token`; add a regression test so
  the behavior is pinned rather than accidental.
- `@<route>:+`, `@<route>^+` — an empty block ID. Reject with the family's existing
  "requires a block ID" wording, not a project-note-specific message.

### Incomplete (interactive) forms

`bob capture-parse` must keep accepting the picker's in-progress spellings and simply
carry the project-note intent through them: `@^`, `@<route>^`, `@^<id>`, `@:`,
`@<route>:`, `@:<id>` are unchanged, and `@^<id>+` / `@:<id>+` report the project-note
mode with `needs: ["route"]`. A bare `@<route>^<id>+` / `@<route>:<id>+` is complete.

### `CaptureKind`, modes, spans, and needs

Add one variant:

```rust
CaptureKind::ProjectNote {
    block_id: String,
    pomodoro: Option<ProjectNotePomodoro>,
}
```

where `ProjectNotePomodoro { name: Option<String> }` distinguishes "no daily-note work"
(`^` form, `pomodoro: None`) from "link `^prj` under the implicit Pomodoro"
(`pomodoro: Some(ProjectNotePomodoro { name: None })`) and "link it under a named
Pomodoro" (`name: Some(..)`). Use whatever equivalent shape reads best in the existing
code, but keep the three states distinguishable without inspecting the raw token.

Editor surface:

- `EditorMode` gains `ProjectNote` (`"project_note"`) and `PomodoroProjectNote`
  (`"pomodoro_project_note"`), mirroring the existing `Task` / `PomodoroTask` split so a
  client can tell whether the daily note is involved.
- `SpanKind` gains exactly **one** new kind, `ProjectNoteMarker`
  (`"project_note_marker"`), covering the single `+` byte. The route and block-ID
  components keep their existing span kinds (`task_block_id_route` / `task_block_id` for
  the `^` form, `pomodoro_route` / `pomodoro_block_id` / `pomodoro_name` for the `:`
  form). This is what keeps Bob Mac Capture's route completion, block-ID completion, and
  picker flows working with no changes to its span-kind sets.
- `Need` needs no new variant: an incomplete project-note marker needs `route`,
  `block_id`, or `pomodoro_name` exactly as its base family does.
- `capture-parse` reports `block_id` as the authored block ID (the filename suffix
  source), and `section` as the Pomodoro name when one was typed — the same "whichever
  applies" reuse the `:` family already documents.

Diagnostic codes: reuse `invalid_task_block_id_route`, `invalid_task_block_id`,
`invalid_pomodoro_route`, `invalid_pomodoro_block_id`, and `invalid_pomodoro_name` for
their respective components, and add `invalid_project_note_marker` for the
project-note-specific shape errors listed above. Keep `bob capture-parse` non-failing:
these are diagnostics there and hard `CaptureError::usage` errors in `bob capture`, as
the module docs require.

## Project-note rendering design

### Filename

`<route>_<block-id with every '-' replaced by '_'>.md`, at the vault root, matching
`getProjectBasenameFromTaskBlockId`. The route is already lower-cased by the token
parser; the block ID keeps its authored case, because the Obsidian command preserves it
too. `@cash^goog-exit+` → `cash_goog_exit.md`; `@sase^TUI-Fix+` → `sase_TUI_Fix.md`.

### Content

Render the note directly — the CLI must not read or evaluate
`_templates/new_project.md`. The output must match what the Obsidian hotkey produces for
the same input:

```markdown
---
parent: "[[cash]]"
template: "[[new_project]]"
type: "[[project]]"
status: wip
created: 2026-09-20T14:31:07-0400
---

- [ ] #task #prj Finish the Google exit packet! #hide ^prj

## Tasks

- [ ] #task Call Morgan Stanley [created::2026-09-20]
```

Notes on each part:

- `parent` is `"[[<route>]]"`. Every capture route is a vault-root note, so the
  `[[path|basename]]` branch of `getFrontmatterWikiLinkToFile` never applies.
- `template: "[[new_project]]"` is kept so CLI-created notes are indistinguishable from
  hotkey-created ones; the Obsidian flow inherits it from the template and never strips
  it.
- `created` uses the same `YYYY-MM-DDTHH:mm:ss±ZZZZ` shape the template's
  `tp.file.creation_date` writes. **Hazard:** `bob_env::current_datetime()` returns a
  `chrono::NaiveDateTime` with no offset, so the renderer must attach the local offset
  (`chrono::Local`) to that naive value rather than calling `Local::now()` again —
  otherwise `BOB_NOW` stops controlling the field. Tests must pin `TZ` so the rendered
  offset is deterministic.
- `scheduled: YYYY-MM-DD` is added as a **frontmatter** line, immediately after
  `status`, when the item resolved a scheduled date. `docs/projects.md` is explicit that
  frontmatter `scheduled` is the sole project schedule and that `bob projects sync`
  strips inline `scheduled` fields from open `^prj` tasks, so an inline field would be
  deleted on the next sync.
- The `^prj` line is `- [ ] #task #prj <body> #hide ^prj`, where `<body>` is the capture
  item's normalized parent text. Do **not** truncate it: the Obsidian
  `truncateProjectTaskDescription` helper only shortens the notice text, not the note.
- `p:<N>` writes `[priority::<value>]` inline on the `^prj` line, before `#hide ^prj`.
  `docs/projects.md` states the `Ctrl+Shift+P` picker offers `priority` for `^prj`
  lifecycle tasks, so inline priority on `^prj` is the established spelling. A rolled
  `p:<N>` date still writes its `🗓️ **SCHEDULE LOG**` child under `^prj` — also the
  documented location — while the date itself lands in frontmatter as above.
- Checkbox status follows the existing capture contract: `[ ]` normally, `[?]` when a
  scheduled property was resolved, and `[*]` for the `:` form (a Pomodoro-linked capture
  is a Next task). A `:` form that also resolved a schedule starts `[?]`, which is
  exactly what `@<route>:<id>` does today. `bob task-status-hooks` reconciles derived
  Blocked state later, as it does for every other capture.

### Authored sub-bullets

`bob capture`'s authored grammar is bounded at two levels (column-zero children and
two-space grandchildren), so the mapping is a simplification of
`buildProjectSeedFromChildBullets`:

- A first-level authored bullet is a **section** when it has at least one nested
  authored bullet **and** its body matches the ALL-CAPS title shape
  (`^[A-Z0-9][A-Z0-9 \t&'(),./-]*$` with at least one `A-Z`). Render it as a
  `## Title Case` header appended at the end of the note, with its nested bullets copied
  in verbatim as `- <body>` lines. Title casing lowercases the whole body and then
  uppercases the first character of every alphanumeric run, so `FUTURE WORK` →
  `Future Work`, `NON-GOALS` → `Non-Goals`, `API DESIGN` → `Api Design`.
- Every other first-level authored bullet is a **task**:
  `- [ ] #task <body> [created::YYYY-MM-DD]` inside `## Tasks`, with its nested authored
  bullets rendered one indentation unit beneath it. A bare ALL-CAPS bullet with no
  nested bullets is a task, matching the Obsidian rule.
- Two section bullets whose title normalizes equally (trimmed, whitespace-collapsed,
  casefolded) merge into one section in source order, again matching the reference.
- When a rendered section title matches a `##` header the note already has, append to
  that section instead of adding a second header. In practice the note is freshly
  generated, so this only matters for `## Tasks` — an authored `TASKS` section bullet
  must merge into the generated `## Tasks` section rather than create a duplicate
  header. Pin that case with a test.
- When there are no authored task children, keep the template's placeholder task line
  `- [ ] #task (REPLACE WITH TASK DESCRIPTION) [created::YYYY-MM-DD]` under `## Tasks`,
  because `replaceProjectTasksPlaceholder` leaves it in place in exactly that case and
  the two creation paths should not diverge.
- **Out of scope:** the reference implementation also relocates managed
  `🗓️ **SCHEDULE LOG**` / `🛠️ **WORK LOG**` child bullets onto the new `^prj` task. That
  has no analogue here, because a capture draft authors a brand-new task rather than
  moving an existing one. An authored bullet that happens to look like a managed log is
  an ordinary task or section bullet, and the renderer must not special-case it.

Build this phase as a pure module — `src/native/capture_project_note.rs` — whose entry
point takes the resolved body, created date, optional scheduled date, optional priority
field, optional schedule-log lines, and the item's `Vec<AuthoredSubBullet>`, and returns
the rendered file contents plus a small summary (basename, section titles, task count).
No filesystem access, so it can be unit-tested exhaustively with `#[cfg(test)]` tests in
the module.

## Execution design

### Parent-note validation

`@<route>^<id>+` requires `<route>.md` to be an existing **area or non-terminal project
note**, mirroring `isAreaOrProjectNote()` and matching what `bob capture-targets`
already offers as a route. Reuse `parse_frontmatter`, `frontmatter_is_area`,
`frontmatter_is_project`, and `ProjectStatus::parse(...).is_terminal()` from
`src/native/projects.rs` — the same four calls `capture_targets.rs` makes — rather than
re-deriving the rule.

Read the parent through `CaptureBatchPlanner::current_contents`, **not** through
`fs::read_to_string`, so a project note created by an earlier item in the same batch can
be the parent of a later item. Errors:

- Missing parent note:
  `cannot create a project note under <route>.md: note does not exist (run 'bob capture-targets' to list routable notes)`.
- Parent is neither an area nor a project: `... is not an area or project note ...`.
- Parent is a done or canceled project: `... is a <status> project ...`.

Use the exact wording the implementer settles on consistently across the CLI error, the
JSON `error` string, and the tests; the three must agree.

### Collision

Reject when `<route>_<id>.md` already exists on disk **or** was already staged earlier
in the same batch: `project note already exists: <path>`. This mirrors the hotkey's
`Note "<basename>" already exists; rename it first` guard. Nothing is written.

### Writing

- `^` form: stage only the new note. `placement` is `"created"`.
- `:` form: stage the new note and the daily ledger together, following
  `plan_capture_with_pomodoro_link`'s structure — daily note resolved from
  `BOB_DAY_FILE` or `<bob-dir>/YYYY/YYYYMMDD.md`, the same implicit/named Pomodoro
  selection, the same named-future-Pomodoro creation, and the same "both notes validated
  before either is replaced" rule. The Task Link is `[[<route>_<id>#^prj]]`.
  `reject_duplicate_block_id` does not apply, because the destination is a brand-new
  file whose only block ID is `^prj`; do keep the existing "ledger already contains
  <link>" guard.
- The parent note `<route>.md` is **never modified**. `bob projects sync` owns the
  generated `- 🧩 **Sub-projects:** [[...]]` line, and `docs/projects.md` is explicit
  that the line is machine-owned. Say so in the human output hint (below) rather than
  writing it here.

### Marker conflicts

Reject, with focused messages in the style of the existing task-toggle rejections:

- `%...`, `--clip`: `project-note capture cannot be combined with % clipboard markers` /
  `... with --clip`. Rationale: authored children are re-routed into `## Tasks` and `##`
  sections, so "clipboard children under the captured parent" has no unambiguous home.
  This is a deliberate deferral, not an oversight — record it as such in the docs phase
  so a future change can lift it.
- Forced destination flags (`--route`, `--section`, `--task`, `--task-ref`,
  `--task-section`) already keep `@tokens` literal, so they cannot reach this kind.
  Confirm that with a test rather than adding a new rejection path.
- `s:<N>` and `p:<N>` are **allowed** and behave as described in the rendering section.

### JSON contract (schema version 1, additive)

`kind` is `"project_note"`. Additive and reused fields:

- `route` stays the **parent** route (`cash`), so route-based clients keep working.
- `relative_target` / `target` point at the new project note (`cash_goog_exit.md`).
- `route_label` is the written file name (`cash_goog_exit.md`). Document this as the one
  capture kind where `route_label` is not `<route>.md`; it is what Bob Mac Capture shows
  as the notification's destination, and the project note is the right thing to show.
- `task_line` is the rendered `^prj` line.
- `block_id` is `"prj"`.
- `placement` is `"created"`.
- New `project_note` object: `basename`, `parent_route`, `parent_link`, `tasks` (count
  of `## Tasks` lines written), and `sections` (ordered array of rendered section
  titles).
- `:` form additionally reports `day_file`, `block_link`, `pomodoro_link_placement`,
  `pomodoro_name`, and `creates_pomodoro`, matching `pomodoro_task` results.
- `scheduled`, `priority`, `priority_label`, and `schedule_log` keep their existing
  meanings. `sub_bullets` is **omitted**, because authored children are not rendered as
  child bullets of the captured line here; the `project_note` summary replaces it.

### Human output

Print the created note, its parent, the `^prj` line, the section titles and task count,
and — for the `:` form — the Pomodoro destination, following
`print_human_item_success`'s existing styling. Close with a short hint that
`bob projects sync` will add the parent's Sub-projects line and `bob task-status-hooks`
reconciles Blocked state, since neither is written here.

### Tests

Add `tests/cli.rs` cases covering, at minimum: the plain `^` creation; the `:` creation
with an implicit Pomodoro, a named open Pomodoro, and a created named future Pomodoro;
authored tasks and authored sections together; the `TASKS` section merging into
`## Tasks`; the no-authored-children placeholder; `s:<N>` and `p:<N>` writing
frontmatter `scheduled` plus the `^prj` Schedule Log; every rejection above; `--dry-run`
writing nothing; a two-item batch where item 2 parents onto the note item 1 created; and
a failing second item rolling the first item's new file back so no partial note
survives.

## Phase: Project-note marker grammar

Implement the grammar design above in `src/native/capture_language.rs` and
`src/native/capture_parse.rs`:

1. Extend `parse_task_block_id_route_token` and `parse_pomodoro_route_token` to accept
   the `+` sigil after the block ID and produce `CaptureKind::ProjectNote`.
2. Add the `CaptureKind::ProjectNote` variant, the two `EditorMode` values, the
   `SpanKind::ProjectNoteMarker` span, and the `invalid_project_note_marker` diagnostic.
3. Handle the incomplete interactive forms and the `@@` declaration rejection.
4. Add module tests asserting the three family-detection predicates still route
   `@cash^goog-exit+`, `@cash:goog-exit+`, `@cash:goog-exit+#bugs`,
   `@sase:deep-fix#bugs+` (unchanged: a Pomodoro named `BUGS+`), and `@cash+goog-exit`
   to the families this plan claims.
5. Add `bob capture-parse` tests for mode, `route`, `block_id`, `section`, `needs`,
   spans, and diagnostics on every new and rejected spelling.

`bob capture` will not compile against the new variant until the execute phase lands;
add the minimal `unreachable!`/`unimplemented` arm needed to keep the tree building and
leave a `TODO` naming the execute phase, or coordinate so the arms land together.

## Phase: Project-note content renderer

Create `src/native/capture_project_note.rs` implementing the rendering design above:
basename derivation, frontmatter, the `^prj` line, `## Tasks`, ALL-CAPS section
conversion with title casing and merging, and the placeholder fallback. Keep it free of
filesystem and clock access — take the created date, scheduled date, priority field, and
schedule-log lines as parameters — and cover it with module tests, including the
`API DESIGN` → `Api Design` acronym behavior, the `FUTURE WORK` merge of two equal
titles, a bare ALL-CAPS bullet with no children staying a task, and the `TASKS` merge.

Register the module wherever the other `capture_*` modules are declared.

## Phase: Capture execution and JSON contract

Implement the execution design above in `src/native/capture.rs`: the
`CaptureKind::ProjectNote` arm of `plan_capture_item`, a `plan_project_note_capture`
planner alongside `plan_capture_with_pomodoro_link`, the parent-note validation and
collision guards, the marker-conflict rejections, the `CaptureItemResult` fields, the
JSON `project_note` object, and the human output. Then add the `tests/cli.rs` coverage
listed above.

## Phase: Capture documentation and help text

Update `docs/capture.md`:

- Add all three marker rows to the "Grammar at a glance" table and to the `#`
  disambiguation table (`@cash:goog-exit+#bugs` names a Pomodoro, not a section).
- Add a "Project notes" section covering the filename rule, the rendered note, the
  `parent` link, the authored-children mapping, the schedule and priority placement, the
  parent-note requirement, the collision guard, the deliberate clipboard rejection, and
  the fact that `bob projects sync` — not capture — writes the parent's Sub-projects
  line.
- Extend the JSON-output section with `kind: "project_note"`, the `project_note` object,
  the `route_label` exception, and the omitted `sub_bullets`.
- Extend the `bob capture-parse` section with the two new modes, the
  `project_note_marker` span kind, and `invalid_project_note_marker`.

Update `src/native/capture.rs`'s `long_about` with a paragraph for the new family and
`after_help` with `bob capture '@cash^goog-exit+' 'Finish the Google exit packet!'` and
`bob capture '@cash:goog-exit+#bugs' 'Finish the Google exit packet!'` examples. Keep
`--help` output scannable and keep listed options alphabetical, per this project's CLI
rules.

Add a short cross-reference in `docs/projects.md` noting that `bob capture` can now
create a project note directly, next to the existing description of the
`<ctrl+shift+option+n>` hotkey.

## Phase: Bob Mac Capture frontend support

Open the `bob-mac-capture` repo with the `/sase_repo` skill and make the two small
changes the grammar design was shaped to keep small:

1. `Sources/CaptureCore/CompletionRowContent.swift`:
   `captureSemanticCategory(forSpanKind:)` must map `"project_note_marker"` to a
   category. Reuse `.explicitToggle` (which already covers the sigil-shaped
   `task_toggle_explicit_toggle`) unless the palette clearly wants its own case; the
   function's `default: .neutral` means an unmapped kind degrades to plain text rather
   than breaking.
2. `Sources/BobMacCapture/NotificationService.swift`: `friendlyKindLabel(_:)` should
   return `"Project"` for `"project_note"` / `"project-note"` instead of falling through
   to its generic title-caser.

Deliberately **not** required, and worth confirming with a quick read before concluding
the phase: the route and block-ID span kinds are unchanged, so `completionSpanKinds` and
`routeSpanKinds` in `Sources/BobMacCapture/CapturePanelModel.swift` already cover the
new markers, and the `@^` / `@:` interactive picker flows keep working untouched. If a
read shows otherwise, extend those sets and say so.

Add or extend tests under `Tests/CaptureCoreTests/` and `Tests/BobMacCaptureTests/` for
both mappings, and verify against a `bob` build that includes the execute phase.

## Validation

- `just all` (`cargo fmt --check`, `cargo clippy --all-targets --all-features`,
  `cargo test`) must pass in this repo for the grammar, render, execute, and docs
  phases.
- The mac phase builds and tests through the `bob-mac-capture` repo's own `justfile`.
- Manually exercise a real capture against a scratch vault before landing the execute
  phase: create an area note, run `bob capture '@area^demo+' 'Ship the demo!'` with
  authored task and `FUTURE WORK` section children, then run `bob projects list` and
  `bob projects sync --dry-run` and confirm the new note is recognized as a `wip`
  sub-project of the area.
