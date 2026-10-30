---
tier: epic
title: (( task wikilink picker for Bob Mac Capture
goal: 'Typing `((` anywhere free text is allowed in a Bob Mac Capture draft opens
  a large, fuzzy, keyboard-first picker over every task in the vault. Choosing a task
  replaces the `((` token with a valid Obsidian block link such as `[[foo#^bar]]`.
  A task without a block ID is named first through a prompt prefilled with a good
  suggested ID. bob-cli owns the trigger grammar, the candidates, the link text, and
  the ID write; the app only presents them.

  '
decisions:
  history:
    ask: Should (( also offer Done and Cancelled tasks, listed after every open task?
    choices:
      all: Every task in the vault; completed history gets its own last section
      open_only: Open tasks only; a shorter list, but finished work can't be linked
    default: all
    why: You asked for any task, and notes often point back at finished work
    answer: all
  link_form:
    ask: Which link form should a picked task insert?
    choices:
      shortest: '[[note#^id]] when the name is unique, a path only when needed, like
        bob''s own links'
      full_path: Always [[dir/note#^id]]; longer, but never relies on a unique note
        name
    default: shortest
    why: Matches your [[foo#^bar]] example and the Task Links and Depends-On links
      bob writes
    answer: shortest
phases:
- id: id-suggestions
  title: Block-ID suggestions skip link URLs
  depends_on: []
  size: small
  description: 'id-suggestions: in bob-cli, make the shared block-ID suggestion generator
    ignore Markdown link destinations and bare URLs, so Keep-captured tasks stop suggesting
    `https-keep-google-com`; update the pinned tests and the documented algorithm.'
- id: bob-contract
  title: bob-cli wikilink_task completion contract
  depends_on: []
  size: medium
  description: 'bob-contract: in bob-cli, detect the `((query` field, serve the vault-wide
    `wikilink_task` snapshot with backend-formatted `[[target#^id]]` replacements,
    groups, disabled reasons, and a picker descriptor, add `wikilink_replacement`
    to `capture-task-id`, keep shell completion out of it, and document the contract.'
- id: mac-picker
  title: Mac Task Wikilink Picker
  depends_on:
  - id-suggestions
  - bob-contract
  size: medium
  description: 'mac-picker: in bob-mac-capture, add the `.wikilinkTask` picker source
    that decodes `wikilink_task`, auto-opens on typed `((`, filters locally, inserts
    Bob''s link, runs the Add block ID prompt in wikilink mode against `capture-task-id`,
    and covers it with presentation, panel, and fake-bob tests.'
- id: mac-design-docs
  title: Mac picker design review and docs
  depends_on:
  - mac-picker
  size: small
  description: 'mac-design-docs: in bob-mac-capture, render light/dark design fixtures
    of the card and ID prompt, review and polish them, show Markdown links in task
    text by their label, and document the picker, keys, and capability row.'
proposed_by: bbugyi200.apollo.b3
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-29 17:02:43
status: done
bead_id: bob-cli-e7
---

- **PROMPT:** [prompts/202610/wikilink_task_picker.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/wikilink_task_picker.md)
- **BEAD:** [bob-cli-e7](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-e7/README.md)

# Plan: `((` task wikilink picker for Bob Mac Capture

## Why and what

Bob Mac Capture already completes `[[` wikilinks inline: notes, headings, and block IDs
that already exist. It has no quick way to link a draft to a **task**, which is what
Bryan usually wants to point at, and many tasks have no block ID yet. This epic adds the
Roam/Logseq-style `((` gesture. `((` is a **draft-only trigger, never stored syntax**:
what gets captured is always a plain Obsidian block link, so the vault, Obsidian,
`bob task archive` link repair, and every Bob reader keep working unchanged.

```text
Draft:   Follow up on ((        ← typing the second "(" opens the picker
Pick:    "Stop idling when no agents run"   (sase_cleanup.md, ^no-idle)
Draft:   Follow up on [[sase_cleanup#^no-idle]]|   ← caret lands after "]]"
```

**Naming.** The wire context is `wikilink_task` (alongside `wikilink_note`,
`wikilink_heading`, and `wikilink_block`). In docs and UI it is the **Task Wikilink
Picker**. Never call it a "Task Link" or a "task block link": both are glossary terms
for the ledger links under today's Pomodoros (`glossary:task-link`), and confusing the
two would mislead.

### Architecture rule (decision `mac-capture-is-a-thin-client`)

bob-cli alone decides where `((` is a field, what the query is, which tasks appear,
their order, groups and guards, the exact replacement bytes, `cursor_after`, and the
block-ID write. The app filters Bob's snapshot locally (the same presentation-only
precedent as `:` / `&` / `~`), renders it, and splices Bob's strings verbatim. The
contract grows additively: new fields only, `schema_version` unchanged, and the app
decodes them with `decodeIfPresent`.

## Experience design (the spec both halves implement)

### Trigger

- `((` opens the picker when it starts a token: at line start, or after a space or tab.
  It works in any capture item's free text (task text, sub-bullets, bulk-draft lines),
  the same places `[[` note completion works.
- It is **not** a field inside code spans, inside an existing `[[…]]`, in Work Log text
  under an `x` close (entries reject block links, the same suppression `wikilink_block`
  has), on whole-item action items that carry no free text (`=…` starts, `x…` closes,
  `+N` / `-N` adjustments, `++` / `--` shifts, `!`, `/`, `:`, `^` items), or as part of
  `(((`. Mid-word `f((x` never triggers.
- The picker **auto-opens only when the edit wrote the second `(`**: typing it, pasting
  it, or a fast burst since the last settled analysis. Backspacing into an existing
  `((foo`, Delete, or moving the caret into it shows a compact reopen chip **Link a task
  ⇥** instead. Tab, Down, or Ctrl-N on the chip opens the picker; Escape hides the chip.
- Text typed before the picker appears seeds the filter (Bob's `query`).

### The card (reuses the shared picker card; nothing new in shape)

- **Filter bar**: a scope capsule showing `((` in the wikilink delimiter color with the
  caption **Link to Task**; placeholder _Search every task by text, note, or ^id_; a
  monospaced trailing count (`4,186 tasks`, `12 of 4,186`) held to a fixed width so the
  field never jumps.
- **Empty filter, grouped in Bob's order**: **This note** (the draft item's
  destination), **In Progress**, **Next**, one section per note for other open tasks
  (header `▣ route.md` with its Inbox/Area/Project icon), then **Completed history**.
  `#hide` rows are subdued and sort last within their section. A non-empty filter
  becomes one ranked flat list (spaces are AND terms) with open matches ahead of
  completed ones at equal score.

> [!decision] history = open_only Bob omits the `completed` group entirely, the card has
> no Completed history section, and `--allow-closed` is never passed. Everything else is
> unchanged.

- **Rows** (34pt, single line): status glyph (orange In Progress, blue Next, grey open,
  dim check/cross for Done/Cancelled); rich task text (`code` monospaced on a faint
  fill, `[[links]]` accent-tinted, fuzzy matches semibold accent); a trailing **locator
  drawn as the link it will insert**: `sase_cleanup` in the wikilink target color, `#^`
  in the delimiter color, `no-idle` in the block-ID color. ID-less rows show `mac_inbox`
  plus a dim `plus.circle` and **Add ID**. Rows Bob guards are dimmed and show Bob's
  reason as the badge. Completed rows show a quiet **Done** / **Cancelled** capsule.
- **Detail strip** (fixed 80pt): the full task text (two lines), a metadata line (status
  · note icon and label · `› section`), and the action line:
  - identified: `↩ inserts [[sase_cleanup#^no-idle]]`, link-colored like the editor's
    wikilink spans;
  - ID-less: `↩ names it ^try-opus-blender in mac_inbox.md, then links it`;
  - guarded: Bob's `disabled_reason` verbatim.
- **Footer key hints**:
  `↑↓ Move · ↩ Insert link · ⌘↩ Insert & Capture · ⌃↩ Capture & Open · esc Clear/Cancel`
  (Control-Return drops first under 620pt).
- **Editor**: dims to 50% while picking, with an accent wash over Bob's `marker_range`
  (the `((query` token) so it is obvious what gets replaced. Tapping the editor cancels.
- The panel grows once on open and never resizes while filtering. VoiceOver announces
  "Task wikilinks, 4,186 tasks" on open, match-count changes, and each keyboard move.
  Rows are buttons labeled with the full text and the link they insert. Fills strengthen
  under Increase Contrast, and color is never the only signal.

### Keys

| Key                                                                       | Behavior                                                                                                                                                         |
| ------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Printables, Cmd-A/C/V/X/Z, Ctrl-A/E, ←/→                                  | Native filter edits; the draft is untouched while filtering                                                                                                      |
| Return / keypad Enter / Tab / click                                       | Insert Bob's replacement over Bob's range; caret to `cursor_after`; back to the editor                                                                           |
| Command-Return                                                            | Insert, then capture                                                                                                                                             |
| Control-Return                                                            | Insert, capture, and open the reported notes                                                                                                                     |
| Shift-Return, Option-Return, Shift-Tab                                    | Consumed                                                                                                                                                         |
| Down/Ctrl-N/Ctrl-J, Up/Ctrl-P/Ctrl-K, Page Up/Down, Home/End, Cmd-Up/Down | Move (wrapping), page, first/last                                                                                                                                |
| Escape / Ctrl-[                                                           | Clear a non-empty filter; otherwise cancel: draft and caret unchanged, the `((` stays literal, auto-open is suppressed for that token, and the chip shows        |
| Backspace on an empty filter                                              | Delete Bob's `trigger_removal_range` (the second `(` plus any typed fragment), leaving one literal `(`, just as Backspace would in the editor; the picker closes |
| Ctrl-S                                                                    | Consumed                                                                                                                                                         |
| Ctrl-C                                                                    | Stash and close, as everywhere                                                                                                                                   |

An accepted insert is followed by an `.edit` analysis **without** completion, so no
inline `[[` list pops up behind the new link. A stale draft (changed since the snapshot)
never inserts; it refetches instead, like every other picker.

### Add block ID prompt (wikilink mode)

Accepting an ID-less row opens the existing Add block ID prompt in a new wikilink mode:

- Title **Add a block ID to link this task**; subtitle **Adds ^id to mac_inbox.md now**.
  This is honest that the note changes immediately, like the `&` / `~` prompts.
- The field is prefilled with Bob's first `block_id_suggestions` entry, fully selected,
  so Return accepts the good default and typing replaces it. The other suggestions
  appear as clickable chips, and Tab / Shift-Tab cycles them.
- A live line **Inserts [[mac_inbox#^try-opus-blender]]** is composed for display only,
  from Bob's per-candidate `link_target` plus the typed ID. The text that is actually
  spliced is always Bob's `wikilink_replacement`.
- Buttons **Add ID & Link** (Return) and **Add ID & Capture** (Command-Return). Return
  runs
  `bob capture-task-id --note-path <note_path> --task-ref <ref> --block-id <id> --format json`,
  adding `--allow-closed` for completed rows. On success it splices Bob's
  `wikilink_replacement` into the saved range and puts the caret after it. On failure
  the prompt stays open with Bob's error, and the draft is unchanged.
- Escape returns to the picker with the filter intact. An older Bob whose result lacks
  `wikilink_replacement` keeps the prompt open with **This Bob is too old for (( links.
  Update Bob first.**

## bob-cli contract (phase `bob-contract`)

### Field detection

Add a new module (for example `src/native/capture_task_wikilinks.rs`) that exposes
`completion_field_at(raw, cursor, current_note_path)`. Call it from
`capture_complete::engine::build_result_inner` right after the existing
`capture_links::completion_field_at` check and before marker completion, so a caret
inside `((query` wins over any marker sigil typed into the query.

- **Opener**: a `((` at line start or after ASCII space/tab, not preceded by another
  `(`, not inside inline/fenced code or any `[[…]]` (reuse
  `capture_links::is_protected_markup_range`), and not in Work Log text (reuse
  `capture_language::cursor_in_close_log_text`).
- **Item gate**: return no field when the cursor's item is a whole-item action with no
  free text (the same items the CLI help says request no candidates), and when the
  template gate claims the position.
- **Query** (unclosed): the bytes from after `((` to the cursor. Spaces are allowed; the
  query must not contain `(`, `)`, `[`, `]`, a backtick, or a newline, otherwise there
  is no field. The replacement range runs from the opener to the end of the
  non-whitespace run at the cursor, so a mid-word caret replaces the whole fragment and
  never swallows the text after it.
- **Closed** `((query))` on one line, with the caret anywhere inside: the range covers
  the whole token including `))`. The query is still the bytes from after `((` to the
  cursor, so a request at the token start always returns the full snapshot the app
  refetches.
- `query` is returned at the top level (like `task_complete`).
- A `picker` descriptor is returned: new `PickerKind::WikilinkTask`, scope `vault`,
  `scope_token: "(("`, `marker_range` set to the replacement range, and
  `trigger_removal_range` running from the second `(` to the range end.

### Candidates

- The pool is `capture_dependency_tasks::discover(bob_dir)`: every task-bearing note,
  open and closed, including hidden tasks, ref notes, daily notes, and `done/` history.
  ID-less tasks are always included; `--all-tasks` has no effect, as with `task_link`.
- Order follows the dependency picker canonical order, with
  `owner_note = current_note_path`: groups `in_this_note`, `in_progress`, `next`, `open`
  (per note), `completed`, hidden last within each group, and stable path/line
  tiebreaks. A query ranks with
  `capture_link_tasks::match_score(dependency_search_fields(..))`, falling back to that
  order on ties. This is what `relation_candidates` does for `~` `To`; reuse its helpers
  instead of copying them.
- Candidate fields use the same names as the relation/dependency rows, so the Mac's
  unified `CaptureCompletionCandidate` decodes them unchanged: `replacement`, `ref`,
  `note_path`, `locator`, `group`, `hidden`, `block_id`, `requires_block_id`,
  `block_id_suggestions`, `disabled_reason`, `status_symbol`, `status_name`,
  `status_type`, `text`, `section`, `depth`, `line`, `task_kind`. Two additive fields:
  `cursor_after` (present when `replacement` is non-empty, in draft-global UTF-8 bytes)
  and `link_target` (the exact target text bob would put before `#^`, for display only).
- `replacement` is `[[<link_target>#^<block_id>]]`. It is empty for ID-less and guarded
  rows.
- `link_target` uses the rules of `task_dependencies::format::canonical_link` **without
  the same-note rule**, because a draft's destination can change after insert (inbox
  routing, a later `@route`). `done/` archive targets always keep the explicit path;
  otherwise the basename is used when it resolves uniquely through the vault
  `NoteIndex`, else the full vault-relative path without `.md`. Add a small cross-note
  sibling helper next to `canonical_link`, so `capture-task-id` and the candidates share
  one formatter.

> [!decision] link_form = full_path `link_target` is always the full vault-relative path
> without `.md` (`[[projects/sase_cleanup#^no-idle]]`). The shared formatter, the
> candidates, and `wikilink_replacement` all change together. Nothing else changes.

- `disabled_reason`, Bob-authored and checked in this order: duplicate block ID in its
  note (`duplicate block ID ^x in note.md`); and, for ID-less rows only, an archived
  `done/` note (`archived tasks can't get an id`), the previous daily snapshot
  (`the previous daily note is read-only`), or a read-only file (`read-only note`).
  These are exactly the cases `capture-task-id` would refuse, so no row offers a doomed
  prompt.

> [!decision] history = open_only Filter out `group == "completed"` rows before ranking.
> `capture-task-id` keeps `--allow-closed` for the other pickers.

### `capture-task-id`

Add `wikilink_replacement` (`[[<link_target>#^<id>]]`, from the same shared formatter)
to the JSON success shape in both `--route` and `--note-path` modes, next to
`complete_replacement` and the relation replacements, and mention it in the long help.

### Everything else in this phase

- Add `CompletionContext::WikilinkTask` to every exhaustive match (`engine.rs` link
  arms, `render.rs` context name `wikilink_task`, `shell.rs`). Shell completion must
  return `None` for this field before any vault scan, as it does for wikilinks.
- Update the `capture-complete` long help, the contexts list, and the examples (add
  `bob capture-complete -c 13 -f json -- 'Follow up ((s'`). Rerun `just docs-gen` for
  the generated help.
- In `docs/capture.md` § `bob capture-complete`: add the context to the list, add a
  `wikilink_task` paragraph (trigger rules, range, query, pool, order, guards,
  descriptor), and a JSON example. In § `bob capture-task-id`: add
  `wikilink_replacement`.
- Tests in `tests/cli/capture/complete_wikilink_task.rs`, modeled on
  `complete_task_complete.rs`, with a fixture vault that has root, nested,
  duplicate-basename, `done/`, ref, and daily notes. Cover: every trigger and
  non-trigger position in the lists above; unclosed and closed ranges; a mid-word caret;
  CRLF drafts; multi-byte text before the token (byte offsets); bulk drafts; order and
  groups; each guard; basename vs path vs archive link targets; `cursor_after`;
  descriptor ranges; read-only (vault untouched); and `capture-task-id`
  `wikilink_replacement` in both modes.
- A **splice-validity test**: for each offered position, splice the first identified
  candidate's replacement and assert that `capture-parse` reports no new diagnostic and
  `capture --dry-run` accepts the draft. This proves Bob never offers a link where the
  capture grammar would reject it.
- Gate: `just check` and `just docs-check`. Do not edit `Cargo.toml` versions or
  `CHANGELOG.md` (decision `bob-cli-releases-ride-a-daily-train`); this is an additive
  `feat(capture)`.

> [!decision] history = all Keep `completed` rows (Done and Cancelled history) last; the
> Mac passes `--allow-closed` when naming one.

## Block-ID suggestions skip link URLs (phase `id-suggestions`)

Today `capture_block_ids::suggest_ids_with_used` treats a Markdown link destination as a
parenthetical phrase, and treats URL text as prose. Every Keep-captured task
(`Try Opus with Blender. [💡](https://keep.google.com/…)`) therefore suggests
`https-keep-google-com` first; about half of the ID-less tasks in the vault start that
way. Fix this in the shared generator so `((`, `:`, `&`, `*`, `~`, `!`, and successor
mints all improve together:

- In `phrase_spans`, a `[label](destination)` link contributes its label as a phrase
  (when the label has words) and never its destination. A `(` that directly follows `]`
  is a link destination, not a parenthetical.
- In prose words, drop bare URLs (`scheme://…` runs and `www.` runs) and link
  destinations before splitting into words.
- Expected: `Try Opus with Blender. [💡](https://keep.google.com/u/0/#NOTE/x)` suggests
  `try-opus-blender` then `opus-blender` (the exact list follows the existing
  verb/stopword rules; pin whatever those rules produce in the test). Existing pinned
  examples are unchanged.
- Update the algorithm description in `docs/capture.md` (the "Suggestions derive from
  the task text" paragraph). Check the successor-mint vectors in
  `docs/task-dependencies.md` §11.7, and update a vector only if it contains a URL.
- Gate: `just check`. bob-plugins' TypeScript `suggestPromptBlockId` is a separate
  implementation and stays out of scope.

## Mac Task Wikilink Picker (phase `mac-picker`)

Read `bob-mac-capture/AGENTS.md` first and open the repo with `/sase_repo`. Mirror the
existing `.taskComplete` source end to end (its commit `57220a2` is the best map of the
files involved), and reuse the `&` / `~` row machinery:

- **CaptureCore decoding** (`CaptureModels.swift`, `CompletionRowContent.swift`): add
  `CaptureCompletionContext.wikilinkTask` for `"wikilink_task"`, the descriptor kind,
  candidate `link_target` (`cursor_after` already decodes), and `wikilink_replacement`
  on the `capture-task-id` result. Use `decodeIfPresent` throughout.
- **Source** (`CapturePickerPresentation.swift`): add `CapturePickerSource.wikilinkTask`
  with trigger byte `(`, scope symbol `((`, caption **Link to Task**, the placeholder
  and accessibility label from the spec, and a `CapturePickerIndex` case. It is **not**
  a relationship modifier and has no owner header, so `dependencyDirection` stays nil
  and no owner rules apply.
- **Index**: build the rows, grouping, and local ranking from `DependencyPickerIndex`
  (it already handles `group`, `hidden`, `locator`, `note_path|ref` keys, disabled
  reasons, and Completed history). Either add a `wikilink` direction (leading sigil
  `((`, no "Already" badge) or a thin `WikilinkTaskPickerIndex` wrapper. Pick whichever
  keeps relationship-only behavior out of this source. Add the action-line copy and the
  link-colored locator from the spec.
- **Routing** (`CapturePanelModel.swift`): add a `handleWikilinkTaskCompletion` modeled
  on `handleTaskCompleteCompletion`, but auto-open only when
  `captureEditWroteSeparator(writtenRange, before: r.start + 2)` holds (the edit wrote
  the second `(`). Otherwise show the chip. Refetch the full snapshot at `r.start + 2`
  when the caret is past it. Seed the filter from Bob's `query`. On accept, insert
  `replacement` over Bob's range and place the caret at `cursor_after`, behind the
  stale-draft guard, then run `.edit` analysis without completion. Backspace on an empty
  filter deletes Bob's `trigger_removal_range` (the parent-task path already does this).
  Escape and the chip follow the shared lifecycle. Command-Return and Control-Return
  insert and then capture (and open), like `!`.
- **ID prompt**: add a `.wikilinkTask(...)` prompt purpose that carries `note_path`,
  `ref`, suggestions, a closed flag, `link_target`, and the picker snapshot to restore.
  Call `capture-task-id --note-path`, with `--allow-closed` for completed rows. Splice
  `wikilink_replacement`, with the old-Bob guard copy from the spec. Add the
  wikilink-mode title, subtitle, live line, and buttons in `BlockIDField.swift` and the
  prompt view.
- **Keys** (`CaptureKeyCommandRouter.swift`) per the table; **view**
  (`CapturePickerView.swift`, `CapturePickerState.swift`, `CapturePanelView.swift`) for
  the locator, the detail strip, the footer hints, and the editor wash over
  `marker_range`.
- **Tests**: `Tests/CaptureCoreTests/WikilinkTaskPickerPresentationTests.swift`
  (grouping, hidden-last, filtering, open-before-completed, disabled rows, action lines,
  locator text); `Tests/BobMacCaptureTests/CaptureWikilinkTaskPanelTests.swift`
  (auto-open only on a written second `(`, the chip on caret moves and Backspace, Escape
  suppression, Backspace leaves `(`, accept/caret/no inline list, stale guard, ⌘↩ /
  ⌃↩, the ID prompt success/failure/Escape/old-Bob paths, and `--allow-closed` for
  history); decoding cases in `CaptureModelTests`. Add
  `Tests/Fixtures/wikilink-task-*.json` and `capture-task-id` fixtures plus `fake-bob`
  routes. Generate the fixtures by running bob-cli built from source (`cargo build` in a
  bob-cli checkout) against a small synthetic vault in a temp dir, never Bryan's vault,
  because the released `bob` may not carry the contract yet.
- **Gates** (non-negotiable, from the repo AGENTS.md): never land Swift you have not
  compiled. Run `just portable-check` (on athena if this host lacks `swift`), run
  `Scripts/macos-ci-check` for the macOS build and tests, and compile a scratch copy on
  the Mac with `bundle.sh` and `just typecheck-tests` as best effort. Format only
  changed lines (`.swift-format`). If the Mac is offline, say so in the final report.

## Mac picker design review and docs (phase `mac-design-docs`)

"Beautiful" is verified on real renders, not assumed:

- Add render cases to `CapturePickerDesignTests` through `RenderFixtureWriter`:
  `wikilink-task-{grouped,filtered,id-less-selected,history,id-prompt}` at 760 and 620
  points, in light and dark. Run `Scripts/macos-ci-check`, download that run's
  `render-fixtures` artifact with `gh run download`, look at every PNG, and fix what
  looks wrong: alignment, truncation (middle-truncate long paths so the filename
  survives), color balance of the locator, and dark-mode contrast. Repeat until the card
  reads as calm and consistent with the `!` and `~` cards.
- In `TaskDisplayText`, render a Markdown link `[label](url)` as its label in the
  existing `link` style, so Keep-captured rows read `Try Opus with Blender. 💡` instead
  of a raw URL. This is display only and benefits every picker. Pin it with
  `TaskDisplayTextTests`.
- Docs (then run `just docs-check`): in `docs/wikilink-completion.md`, add a "Task
  wikilinks (`((`)" section covering the trigger, card, rows, ID prompt, keys, and how
  it differs from `[[^^` block search (it lists tasks, including ID-less ones, and can
  name them). Also add the index bullet in `docs/pickers.md`, the `((` line in
  `docs/keyboard.md` § Capture picker keys, and the capability bullet plus a degradation
  row in `docs/protocol.md` (`wikilink_task` / `wikilink_replacement`: the picker with
  link inserts; an older bob: no completion, so `((` stays plain text; upgrade bob).
- Curating a docs image through `Scripts/docs_media.py` needs a green master run. Leave
  that for a later `just docs-media` refresh; do not block on it.

## Rollout and risks

- **Order**: the bob phases land first, then the Mac phases, and Bryan installs the app
  after a bob release that carries the contract. With an older bob, `((` is plain text
  (no context). In the short window where a newer bob meets an older app, the app routes
  the unknown `wikilink_task` context to its inline list. This is accepted, because
  nobody types `((` before the feature exists.
- **Snapshot size**: the pool matches the `~` picker (about 4,200 tasks, ~1.8 MB of
  JSON, ~0.6 s today). That is proven acceptable there, and filtering is local, so
  keystrokes never respawn `bob`.
- **Link durability**: `bob task archive` and task moves already repair block links to
  tasks they move (`collect_done/link_repair.rs`), so inserted links stay valid.

## Non-goals

- `((` inside Obsidian itself (bob-plugins), or in the relation card, the Name Pomodoro
  field, or other secondary fields.
- Link aliases (`[[note#^id|text]]`), embeds (`![[…]]`), and heading links.
- Any change to the inline `[[` completion or the `[[^^` vault block search.
- A new capture-parse need or span: an unresolved `((foo` is ordinary text and previews
  and captures as written.
