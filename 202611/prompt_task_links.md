---
tier: epic
title: (( task links in Bob's prose prompts
goal: Typing (( in every Bob prose prompt that already completes [[ (S1–S16) opens
  the LINK band's tasks mode. It inserts bob's exact [[target#^id]] link to any vault
  task, names an ID-less task in place with one explicit write, never submits the
  prompt early, and never moves focus out of the field.
decisions:
  naming:
    ask: Should picking an ID-less task name it in place (one explicit vault write)
      and then link it?
    choices:
      inline: Enter on an ID-less row opens a NAME stage; a second Enter adds ^id
        and links it
      identified_only: ID-less rows stay disabled; prompts never write; NAME, naming
        API, Mac step dropped
    default: inline
    why: Matches the editor and Mac (( flows; without it about half the tasks need
      a detour
    answer: inline
  decision_record:
    ask: Record the prompt (( rules as a new decisions strand (decisions:prompts-link-tasks)?
    memory:
    - decisions:prompts-link-tasks
    default: false
    answer: false
phases:
- id: cli-task-links
  title: 'bob capture-complete --task-links: (( in plain prose, with shared prose
    vectors'
  size: small
  depends_on: []
  description: 'cli-task-links: add a prose (( detector without capture-grammar checks,
    the -T/--task-links option on capture-complete -L, byte-identical prose trigger
    vectors, tests, and capture-complete docs.'
- id: bip-task-api
  title: 'block-id-prompt api.taskLinks v1: prose detector, ranked session, and safe
    ID naming'
  size: medium
  depends_on:
  - cli-task-links
  description: 'bip-task-api: expose the existing (( engine as api.taskLinks v1. It
    adds the prose fieldAt detector replaying the shared vectors, a snapshot session
    with ranked rows, guards (including subjects) and indexing progress, and an editor-free
    validate/name write via vault.process or the open editor.'
- id: nav-task-band
  title: 'proseField v2: the tasks mode of the LINK band, the NAME stage, and the
    real-Obsidian spike'
  size: medium
  depends_on:
  - bip-task-api
  description: 'nav-task-band: add the tasks mode to nav''s proseField controller
    in new fragments (model, renderer, mode state machine). It covers the list and
    NAME stages, beforeinput-fed ID editing, the leftover (( hint, and API v2, plus
    tests, preview states, a real-Obsidian spike, and the block-id-prompt docs section.'
- id: prompt-subjects
  title: 'Task-bearing prompts S1–S12: subject guards, host safety, prose lint'
  size: medium
  depends_on:
  - nav-task-band
  description: 'prompt-subjects: wire subjects for S1–S11 (S12 none) and audit picker-host
    key offering during NAME. It adds the per-family host-safety tests and extends
    the prose-field lint with subjects.'
- id: card-surfaces
  title: Intention Card, Record Card, and Task from email (S13–S15)
  size: medium
  depends_on:
  - nav-task-band
  description: 'card-surfaces: make (( and NAME survive the Intention Card repaints
    and key ladder, the Record Card typed stages, and the (( field-mode link stage,
    and verify the email task modal.'
- id: mac-pull-in-tasks
  title: 'Mac Pull-In reason: the tasks mode and NAME stage, plus Return never confirms
    while the band is open'
  size: medium
  depends_on:
  - cli-task-links
  description: 'mac-pull-in-tasks: request -T in the Pull-In LINK band and render
    wikilink_task rows. It adds the BlockIDField NAME stage backed by capture-task-id
    and dims the reason row''s own task. It also fixes Return confirming the Pull-In
    while the band is open, plus docs and macOS CI.'
- id: task-walkthrough
  title: Real-Obsidian walkthrough, mention round-trip, and screenshots
  size: small
  depends_on:
  - prompt-subjects
  - card-surfaces
  description: 'task-walkthrough: run the isolated real-Obsidian e2e over S1, S6,
    S10, S11, S14, and S15, then confirm the mention chips after bob task reconcile,
    attach screenshots, and run the docs check.'
proposed_by: bbugyi200.apollo.ff
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-21 17:39:45
status: done
bead_id: bob-cli-oz
---

- **PROMPT:** [prompts/202611/prompt_task_links.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/prompt_task_links.md)
- **BEAD:** [bob-cli-oz](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-oz/README.md)

# Plan: `((` task links in Bob's prose prompts

## Summary

Epic `bob-cli-om` made `[[` complete notes, headings, and blocks in every Bob-owned
prose field: S1–S16, the Work/Rotten/Cancel/Schedule reasons, the intentions, the record
text, the notebook summary, the email task, and the Mac Pull-In reason. It listed `((`
as a non-goal because accepting an ID-less task writes a block ID. Bryan now wants `((`
in those same fields, so a task can be referenced directly instead of finding its note
first with `[[`.

`((` already works in two places:

- **The Obsidian editor.** block-id-prompt's Task Wikilink Picker, a modal card.
- **The Mac capture panel.** A picker card fed by `bob capture-complete`'s
  `wikilink_task` context.

Both insert bob's plain block link `[[target#^id]]`, and both can name an ID-less task
before linking it. A link like that inside a Work, Cancel, Schedule, or Rotten Log entry
is exactly a Task Mention (`docs/guides/see-task-mentions.md`). Prompt Work summaries go
to the task's own `🛠️ **WORK LOG**` drawer, never under a Pomodoro Task Link. The
capture-close refusal of block links in Work Log bullets is therefore a capture-grammar
rule and does not apply.

The design in one paragraph: `((` is a **second mode of the same LINK band**. It uses
the same slot, height, keys, and Enter rule, and the header reads `🔗 LINK · tasks`.
Candidates, ranking, guards, and the ID write come from block-id-prompt's existing
engine through a new `api.taskLinks`. That engine is already pinned to bob's order by
shared vectors. Naming an ID-less task happens in place, in a NAME stage drawn inside
the band, while focus never leaves the prompt's field. It is the only vault write, it
happens only on an explicit Enter, and it adds exactly what the editor's `((` adds. The
Mac Pull-In reason gets the same mode through a new `bob capture-complete --task-links`.

## Design principles

1. **One band, one vocabulary.** `[[` and `((` share the band frame, the row grammar,
   the keycaps, the selection treatment, and the Enter rule. A prompt never shows two
   popups and never shows a modal over a modal.
2. **The link is bob's link.** Every inserted string is byte-identical to what the
   editor's `((` and bob's `wikilink_task` would insert: `[[target#^id]]`, where target
   is the stem when it is unique, else the vault path without `.md`. This is the same
   principle the `[[` epic used for `link_text = path`: every surface inserts exactly
   what bob would.
3. **Focus never leaves the field.** Several hosts commit on blur (S6), switch handles
   on focus (S10), or route keys through picker scopes. The NAME stage is therefore
   drawn in the band and fed through the field's own `beforeinput`/`keydown`, with no
   second `<input>`.
4. **Writes are explicit, minimal, and atomic.** The only write is naming, and only on
   Enter in the NAME stage. It adds ` ^id` to one task line, plus the freshness stamp
   that the editor's `((` naming already applies. The line is re-located by identity
   first. The prompt's own subject task is never named.
5. **Fails open.** Without block-id-prompt, or with an old one, `((` stays literal text.
   A thrown ranker shows no band. A failed write leaves the NAME stage open with the
   reason. The host's submit is never blocked while the band is closed.

## Scope

### The rule

`((` completes in every field where `[[` completes (the `bob-cli-om` allowlist S1–S16),
because it follows the field's `links` setting. A host can opt out with `tasks: false`.
Every field that stays literal for `[[` stays literal for `((`. The Mac event-note
editor stays excluded: gcal escapes `[` in private notes, so the inserted link would be
corrupted.

### Surfaces

Repo prefixes are the same as in `bob-cli-om`:

- **nav** = bob-plugins `plugins/bob-navigation-hotkeys/src`
- **bip** = `plugins/block-id-prompt/src`
- **ldg** = `plugins/bob-ledger-tools/src`
- **mail** = `plugins/bob-mail/src`
- **mac** = bob-mac-capture

The `sourcePath` each surface already passes for `[[` also feeds `((`'s "this note"
group. The new column is **Subjects**: the task(s) the prompt writes to. Those rows are
disabled with "The task this prompt is about" (badge `this prompt`). The guard exists
for two reasons. Naming one would change its line under the host's pending write, and
bob's stale-safe refs are `line:digest`. A self-mention is also meaningless.

| ID  | Surface                                          | Where                                         | Subjects                                                    | Phase             |
| --- | ------------------------------------------------ | --------------------------------------------- | ----------------------------------------------------------- | ----------------- |
| S1  | Unlink / task-link-open Work summary             | bip `090-prompt-modals.js`                    | the unlinked task / the opened task                         | prompt-subjects   |
| S2  | Move-to-Next summary                             | nav `196-task-link-lane-modal.js`             | every moved task                                            | prompt-subjects   |
| S3  | Lane-release summary                             | nav `190-summary-modals.js`                   | the released task                                           | prompt-subjects   |
| S4  | Batch keep reason                                | nav `192-freshness-refresh-summary-modal.js`  | every kept entry                                            | prompt-subjects   |
| S5  | Rotten keep / decay Stay reason                  | nav `195-rotten-keep.js` (192, `060`)         | the kept or decaying entry                                  | prompt-subjects   |
| S6  | Successor inline cancel reason (dense)           | nav `206d-successor-picker-verdict-render.js` | the row being cancelled                                     | prompt-subjects   |
| S7  | Task Card cancel-reason stage                    | nav `380-picker-cancel-and-lane.js`           | the card's target task(s)                                   | prompt-subjects   |
| S8  | Task Card lane-release-reason stage              | nav `380`                                     | same                                                        | prompt-subjects   |
| S9  | Task Card schedule-work-log stage                | nav `370-picker-schedule-work-log.js`         | same                                                        | prompt-subjects   |
| S10 | Task Card schedule-review reason + summary       | nav `360` + `185`                             | same                                                        | prompt-subjects   |
| S11 | Task Card scheduled inline reason                | nav `360`/`410`, `185a`                       | same; `((` honours the existing `minOffset`                 | prompt-subjects   |
| S12 | Notebook create Summary                          | nav `685-plugin-notebook-create.js`           | none                                                        | prompt-subjects   |
| S13 | Task from email                                  | mail `040-task-modal.js`                      | none                                                        | card-surfaces     |
| S14 | Intention Card intention, summary, reflection    | ldg `297c*`                                   | none                                                        | card-surfaces     |
| S15 | Record Card text, suggest, list, and link stages | ldg `242o`–`242sa`                            | none; the link stage opens `((` only at offset 0            | card-surfaces     |
| S16 | Mac Pull-In cancel reason                        | mac `CapturePullInChoiceView.swift`           | the reason row's own task (same `note_path` and `block_id`) | mac-pull-in-tasks |

## Design

### Architecture

```text
 host prompt ──attach──▶ proseField v2 (nav)                       one keyboard owner, one band
                           ├─ recall mode   USED BEFORE (unchanged)
                           ├─ notes mode    `[[`  (unchanged LINK band)
                           ├─ tasks mode    `((`  list stage ─Enter on ID-less─▶ NAME stage
                           └─ hint line     unresolved `[[…]]` + leftover `((…` (informational)
                                    │ feature-detect, per band open
                                    ▼
                         block-id-prompt api.taskLinks v1 (bip)    the existing `((` engine
                           ├─ fieldAt()       pure prose detector (shared prose vectors)
                           ├─ open() session  TaskWikilinkIndex snapshot → orderForWikilinkPicker
                           └─ session.name()  validate → relocate → planTargetTaskUpdate → atomic write
 Mac Pull-In reason ──▶ bob capture-complete -L -T -S <note>  (bob decides; Mac renders)
                    ──▶ bob capture-task-id                    (bob writes the ID)
```

**Why the engine stays in bip and the band in nav.** bip already owns `((` in Obsidian:
the token grammar (`022`), the guards (`031`), the order pinned to bob (`032`,
`scripts/fixtures/task_wikilinks/vectors.json`), the warm index (`036`), and the naming
write (`110`, `070`, `140`). nav already owns the one keyboard arbiter and the band. nav
feature-detects bip at each band open, the same way it already detects
`pomodoroLinkPicker`, so neither plugin needs the other to load first.

### Prose field rules for `((` (shared vectors)

These rules are bob's `capture_task_wikilinks` token rules minus every capture-grammar
check. Prose has no items, close-log text, or whole-item actions.

- **Opener.** `((` at the start of a physical line, or right after a space or tab. It
  must not be followed by a third `(`. It must not be inside bob's protected markup: an
  inline code span, or a `[[…]]` link, where an unclosed `[[` covers the rest of the
  line. The last valid token containing the caret wins.
- **Query.** The text from after `((` up to the caret. It may contain spaces, which
  makes a multi-term AND filter. It stops at `(`, `)`, `[`, `]`, `` ` ``, or a newline.
- **Closed token `((q))`.** The caret may sit anywhere up to the end of `))`. The query
  is clamped to the closer, and the replacement covers `((q))`.
- **Unclosed token.** The replacement runs from `((` to the end of the non-whitespace
  run at the caret. A mid-token caret (`((ap▏i`) therefore replaces the whole token,
  while ` tomorrow` after a space always survives.
- **Controller rules**, which the vectors do not cover:
  - `minOffset`: openers before it never open.
  - `mode: "field"`: only an opener at offset 0 opens.
  - The same opening rules as `[[`: a user `input` event with a collapsed caret; never
    paste, drop, yank, IME composition, or a programmatic recall fill. Caret-only moves
    never open the band.
- **Different from the editor on purpose.** The editor opens only when the edit wrote
  the second `(`, because its picker is a modal that takes focus. The band is non-modal,
  so `((` follows `[[`: any user input that leaves the caret in a `((` field opens it,
  unless dismissal memory holds.

### Candidates and order

These come from bip's `TaskWikilinkIndex` and
`orderForWikilinkPicker(rows, query, ownerPath = sourcePath)`, unchanged.

- **Empty `((`.** bip's group order: this note, in progress, next, other open, then
  completed history. Hidden (`#hide`) rows sort last.
- **With a query.** One ranked flat list, with completed tasks mixed in by score.
- **Guards**, in precedence order:
  1. unlinkable note name;
  2. **subject**;
  3. duplicate ID;
  4. archived ID-less (`done/`);
  5. ID-less in the previous daily note.

  Disabled rows stay selectable but never insert. Closed tasks outside `done/` can be
  named (editor and Mac parity).

- **Indexing.** bip builds its index at layout-ready in idle chunks. If `((` is typed
  before it finishes, the band shows the rows it has so far, with
  `Indexing · 1,200 of 6,500 notes` in the header count slot. The band refreshes on
  progress and keeps the selection by key.

### Visual design

The band is the LINK band: same slot, same fixed height (roomy is a header, five 24px
rows, and a detail strip; dense is three 22px rows), the same 120 ms first fade, the
same selection bar and fill, and the same edge bump. Only row content and strings
change. A `((` session never animates between stages.

```text
 Cancel — why?
 ┌──────────────────────────────────────────────────────────────────────────┐
 │ Superseded by ((api do▏                                                  │
 └──────────────────────────────────────────────────────────────────────────┘
  🔗 LINK · tasks                                                     1 of 4
 ▎[/]  Write the API docs                                     api#^api-docs
  [ ]  Publish the API docs site                ⊕ projects/api#^publish-api-docs
  [*]  Review API docs with Sam                            api#^review-docs
  ✓    Draft the API docs outline                           api#^outline      ← dimmed
  ·
  ↵ [[api#^api-docs]]                                                ⇥ insert
```

```text
  🔗 LINK · tasks › this note                                       1 of 214     ← empty ((
 ▎[ ]  Reply to Dana about the offsite      this note   ⊕ 2026/2026-11-21#^reply-dana
  [/]  Write the API docs                   in progress            api#^api-docs
  [*]  Fix the flaky sync test              next                   sync#^flaky
  [*]  Book the offsite venue                                   offsite#^venue
  [ ]  Draft the Q1 plan                    open                    plans#^q1
  ↵ names it ^reply-dana in 2026/2026-11-21.md, then links it         ⇥ insert
```

```text
  🔗 LINK · tasks › name it                                in projects/api.md     ← NAME stage
 ▎[ ]  Publish the API docs site                                       ⊕ Add ID
       ^ publish-api-docs▏                                               ✓ free
         publish-docs · api-docs-site                               ⇥ next idea
  ·
  ·
  ↵ adds ^publish-api-docs, then links [[api#^publish-api-docs]]       esc back
```

```text
  🔗 LINK · tasks
            No task matches “api dcos” — keep typing, or esc
  ↵ closes this list · nothing is inserted                            esc close

 ▎[/]  Write the API docs                   this prompt            api#^api-docs     ← subject, dimmed
  ⊘ The task this prompt is about                                     esc close

  ⚠ ((api docs)) isn't a link yet — type in it to pick a task                      ← leftover hint
```

**Row grammar.** This mirrors the editor picker row and the Mac `CompletionRowContent`
`.wikilinkTask` row.

- **Status pill.** A mono, three-character pill in the icon column: `[ ]` todo, `[/]` in
  progress (accent), `[*]` next (accent, muted), `[?]` blocked (`--color-orange`), `✓`
  done (`--color-green`), and `✕` cancelled (faint).
- **Book icon.** Reading tasks (`task_kind` `ref`) add a muted Lucide `book-open` icon
  after the pill.
- **Primary text.** bip's display text, rendered from safe parts:
  - `` `code` `` on a faint fill;
  - `[[links]]` and Markdown `[labels](url)` in accent, by their label;
  - URLs as their host;
  - matched spans emphasised as in the LINK band;
  - never `innerHTML`.
- **Group capsule.** Empty query only. The first row of each run in the visible window
  wears a quiet capsule: `this note`, `in progress`, `next`, `open`, or `history`. Runs
  read like inline section labels without spending rows.
- **Other capsules.** `Hidden`, or a guard's short badge (`this prompt`, `duplicate ID`,
  `archived`, `read-only`).
- **Locator.** Right-aligned mono, drawn as the link it will insert: the target in
  `--text-muted`, `#^` in `--text-faint`, and the ID in the accent. An ID-less row shows
  `⊕` plus the suggested ID in italic faint. Middle truncation eats the target's folders
  first, so the ID always survives.
- **Dimming.** Done, cancelled, hidden, and disabled rows dim to 60%.

**Header.** `🔗 LINK · tasks`, followed by `› <group>` for an empty query, or
`› name it` in the NAME stage. The right side shows `n of N`, `Indexing · x of y notes`,
or `in <note>.md` in the NAME stage.

**Detail strip** (strings shared with the editor and the Mac):

| State                 | Left side                                                           | Right side  |
| --------------------- | ------------------------------------------------------------------- | ----------- |
| Identified row        | `↵ [[target#^id]]` (mono)                                           | `⇥ insert`  |
| ID-less row           | `↵ names it ^<suggestion> in <note>.md, then links it`              | `⇥ insert`  |
| Disabled row          | `⊘ <full reason>`                                                   | `esc close` |
| Zero rows             | `↵ closes this list · nothing is inserted`                          | `esc close` |
| NAME stage            | `↵ adds ^<id>, then links [[target#^<id>]]`                         | `esc back`  |
| NAME, write in flight | `Adding ^<id>…`, shown only after 150 ms so fast writes never flash | —           |

Dense (S6) shows `⇥ insert` only, and `esc back` in NAME.

**NAME stage layout.** The same frame and height.

- **Slot 1:** the chosen task, pinned and never selectable, with an `⊕ Add ID` capsule.
- **Slot 2:** the ID line. It starts with a `^` sigil, then the ID in `--font-monospace`
  with a drawn 1px accent caret that blinks and is steady under
  `prefers-reduced-motion`. A prefilled suggestion renders selected (accent fill) until
  the first keystroke replaces it.
- **Slot 3:** the other suggestions, faint and separated by `·`, with the current one
  emphasised. On the right, `⇥ next idea`.
- **Validation glyph.** On the ID line's right: `✓ free`, or `✕` with the short reason
  (`already in api.md`, or `letters, digits, - only`). The full reason goes to the
  detail strip.
- **Dense layout.** Slot 1 is the task, slot 2 is the ID line with the glyph, and slot 3
  is the suggestions.

**Leftover hint.** This extends the unresolved-link hint and uses its debounce and slot.
When the band is closed and the draft still holds an unaccepted `((…` token, one faint
line reads `⚠ ((api docs)) isn't a link yet — type in it to pick a task`. It is
informational only, and Enter still submits. A link inserted by naming counts as
resolved for this session, so the hint never flags `[[api#^new-id]]` while the metadata
cache catches up.

**Theming.** Classes are `bob-task-band-*` for the pill, locator, ID line, suggestions,
and validation, on top of the `bob-link-band-*` frame, in nav `styles.css`. Use only
theme variables. Raise contrast under `prefers-contrast: more`. It must be legible in
light, dark, and high contrast.

### Keyboard contract

The routing order is unchanged: **IME → band → recall → host**. While the tasks mode is
open, `isLinkOpen()` returns `true` in either stage. Hosts that already gate Enter on it
(Record Card `242sa`, Intention Card `297cd`) are correct without changes.

**List stage:**

| Key                                                             | Behaviour                                                                                                                                                                                                                                                                              |
| --------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ↓ / Ctrl+N, ↑ / Ctrl+P                                          | Move the selection by key. It clamps with the edge bump and never wraps.                                                                                                                                                                                                               |
| Enter (also Ctrl/Cmd/Alt+Enter)                                 | An identified row inserts its link and keeps the prompt open, arming the held-Enter swallow. An ID-less row opens NAME, and the swallow holds until Enter's `keyup`. A disabled row stays put and pulses the detail strip. Zero rows close the band and insert nothing. Never submits. |
| Tab                                                             | Same as Enter without the swallow. With zero rows, passes to the host.                                                                                                                                                                                                                 |
| Shift+Tab                                                       | Passes to the host; the band closes on blur.                                                                                                                                                                                                                                           |
| Esc / Ctrl+[                                                    | Closes the band and remembers `(mode, opener, query)`. One layer per press.                                                                                                                                                                                                            |
| `q`                                                             | Types.                                                                                                                                                                                                                                                                                 |
| ← → Home End, Backspace through `((`, Shift+Enter in a textarea | Native. The band closes when the caret leaves the token.                                                                                                                                                                                                                               |
| Click a row                                                     | `mousedown` calls `preventDefault`. The click acts like Tab on that row.                                                                                                                                                                                                               |
| Hover                                                           | Tint only.                                                                                                                                                                                                                                                                             |

**NAME stage:**

| Key / input                                         | Behaviour                                                                                                                    |
| --------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `a–z A–Z 0–9 -`                                     | Edit the ID at its caret. The first keystroke replaces a selected suggestion.                                                |
| Space, `_`                                          | Types `-`.                                                                                                                   |
| Any other character                                 | Refused. The ID line shakes once (no motion under reduced motion), and the glyph reads `letters, digits, - only`.            |
| Backspace / Delete; Ctrl or Alt+Backspace           | Delete a character, or delete back to the previous `-`.                                                                      |
| ← → Home End; Ctrl/Cmd+A                            | Move the ID caret; select the whole ID.                                                                                      |
| Tab / ↓ / Ctrl+N; Shift+Tab / ↑ / Ctrl+P            | Cycle bob's suggestions into the line, selected.                                                                             |
| Paste                                               | Sanitized into the ID. Nothing pastes into the field.                                                                        |
| Enter (also Ctrl/Cmd/Alt+Enter), fresh keydown only | Validate, write, insert the link, and close the band. Arms the swallow. A failure stays open with the reason. Never submits. |
| Esc / Ctrl+[                                        | Back to the list with the query and the selected row restored. One layer per press.                                          |
| Blur                                                | Cancels naming and closes the band without writing.                                                                          |
| Any key while the write is in flight                | Swallowed. The write times out after 3 s with "Couldn't add ^id — try again".                                                |

**How NAME input works.** On entry, the controller snapshots the field's value and
selection. While NAME is open:

- It cancels every `beforeinput` on the field (`insertText`, `insertFromPaste`, the
  `delete*` types, and `insertReplacementText`) and routes that input to the ID-line
  model.
- Navigation keys arrive through the existing keydown paths: the `ownKeys` capture
  listener, or the host offering keys to `handleKeydown` first.
- The controller's own link insertion is flagged, so it is never intercepted.
- If anything still changes the field (an IME composition, or a host writing the value),
  the field is restored to the snapshot on the next `input`. A host repaint that
  replaces the element arrives through `rebind`, and NAME survives it.
- On exit, the caret is restored to the snapshot.

The field is byte-identical from NAME entry to the one link insertion.

### Naming (the one write)

> [!decision] naming = inline The flow below is in scope.

> [!decision] naming = identified_only Skip this section, the NAME stage,
> `session.name`, and the Mac's `capture-task-id` step. ID-less rows get the guard "Name
> it first — this prompt can't add IDs" (badge `needs ID`). Their locator shows
> `⊕ needs ID`, and the detail strip reads
> `⊘ Name it first — Ctrl+6 on the task, or (( in the editor`.

Enter in the NAME stage runs these steps:

1. `session.validate(row, id)`, which is synchronous:
   - the ID matches bip's `BLOCK_ID_RE`;
   - the row is still ID-less and not disabled;
   - the ID is not already used in the note, checked against the snapshot.
2. `await session.name(row, id)` runs bip's existing pieces outside the editor source
   path:
   - Read the target's current text. If the note is open in a Markdown editor, its
     buffer wins. Two open editors that disagree refuse with "open twice with different
     text".
   - Re-locate the line with `taskWikilinkLineForIdentity`. If the line is gone or
     ambiguous, stop with "That task moved or changed — pick it again" and return to the
     refreshed list.
   - If the line already has an ID, return to the list. The row is now identified, so
     the next Enter links it.
   - Check for a duplicate with `blockTokenMatches`.
   - Plan with
     `planTargetTaskUpdate(content, line, { newBlockId, activationEligible: false, now, stampLine, freshDateText })`.
     That is the editor's `((` naming without Pomodoro activation.
   - Apply it. An open note gets a one-line `editor.replaceRange` after a preimage
     check. Otherwise use `vault.process`, which re-runs the relocation and the plan on
     the fresh data inside the callback and leaves the data unchanged when the identity
     no longer matches.
3. On success, the result is `{ ok, link, notePath, id }`. `link` is `[[target#^id]]`,
   with the target re-checked through
   `metadataCache.getFirstLinkpathDest(target, sourcePath ?? "")` and falling back to
   the full path. The controller then:
   - checks that the field still equals the snapshot and the token still sits at its
     range;
   - inserts the link with one `execCommand("insertText")`, so one Ctrl+Z restores
     `((query`;
   - places the caret after `]]` and closes the band;
   - shows a Notice `Added ^id to projects/api.md`, because a vault write must be
     visible;
   - announces "Linked Publish the API docs site".

   If the field changed anyway, it inserts nothing and the Notice names the ID, so a
   fresh `((` finds the task identified.

Undo never removes the ID. Cancelling the prompt afterwards keeps the ID, which is
harmless and the same as in the editor and on the Mac. The docs say so.

### Accessibility

- Use the LINK band's editable-combobox pattern unchanged: the listbox, option rows,
  `aria-activedescendant`, and a polite live region.
- Announcements:
  - "Task 2 of 37: Write the API docs, in progress, api#^api-docs".
  - Disabled rows add their reason.
  - Entering NAME: "Name a block ID for Publish the API docs site: publish-api-docs.
    Enter adds it and links."
  - Validation changes, debounced by 300 ms.
- In the NAME stage, `aria-activedescendant` points at the ID line, which is
  `role="option"` with `aria-label="Block ID publish-api-docs"`.
- Clear every attribute on close and on `detach`.

### Reliability rules (release criteria for every phase)

- **T1.** The field changes only through the user's own typing in the list stage and the
  single link insertion. In the NAME stage it is byte-identical. No text outside the
  `((` token is ever replaced.
- **T2.** The prompt never submits while the band is open, in either stage. A held or
  repeated Enter can never move from the list to NAME to a write, or from an accept to a
  submit.
- **T3.** One native undo restores the typed `((query`. Insertion uses `execCommand`,
  with the existing `setRangeText` fallback.
- **T4.** The vault write happens only on an explicit NAME Enter. It adds only ` ^id`
  plus the editor-parity freshness stamp, to one re-located line. It never touches a
  subject task, or an archived, previous-daily, read-only, or duplicate-ID task.
- **T5.** The write is atomic: `vault.process`, or the open editor's buffer after a
  preimage check. Never a whole-file write from a stale read.
- **T6.** It fails open. No bip, or an old bip, means `((` stays literal. A thrown
  ranker or index means no band. A failed write keeps NAME open with the reason. Hosts
  keep working unchanged when nav or bip is old.
- **T7.** The band stays quiet in code spans, in `[[…]]`, on `(((`, on a mid-word `f((`,
  on paste, during IME, with a non-collapsed selection, and before `minOffset`.
- **T8.** Privacy: task text, paths, and IDs are never logged, persisted, or sent
  anywhere.
- **T9.** Performance: keystroke-to-rows p95 ≤ 12 ms over 5,000 tasks, session open
  (snapshot plus first rank) ≤ 50 ms, and no vault reads while filtering. Enforce all
  three in `test:perf`.
- **T10.** No drift:
  - the prose trigger vectors are byte-identical in bob-cli and bob-plugins, and both
    replay them;
  - the order stays pinned by the existing `task_wikilinks/vectors.json`;
  - every task-bearing surface declares its subjects, which the prose lint checks.
- **T11.** Host safety: naming a different task in the subject's own note between prompt
  open and submit never makes the host's write refuse or go stale. Test this per host
  family.

### APIs

**bip `api.taskLinks` v1** is new. Add it to bip's frozen `api` next to
`pomodoroLinkPicker`, and replace it with a null-backed copy on unload.

```js
const bip = app.plugins.plugins["block-id-prompt"]?.api;
const taskLinks =
  bip && bip.taskLinks && Number(bip.taskLinks.version) >= 1 ? bip.taskLinks : null;
taskLinks.fieldAt(text, caret, { minOffset, mode }); // → { start, end, query, closed } | null (pure, UTF-16)
const session = taskLinks.open({ sourcePath, subjects }); // snapshot with open-editor overlays
session.rank(query); // → { rows, total, indexing: null | { done, total } }
session.linkFor(row); // → "[[target#^id]]" | null (re-checked target; null for ID-less/disabled)
session.suggestions(row); // → Promise<string[]> (content-aware; [0] is the prefill)
session.validate(row, id); // → { ok: true } | { ok: false, short, message }
session.name(row, id); // → Promise<{ ok: true, link, notePath, id } | { ok: false, message, stale }>
session.onChange(listener); // → unsubscribe (index progress or metadata change while open)
session.close();
```

Each row carries:

- `key`, `notePath`, `line`, `blockId | null`, and `needsId`;
- `status` (`todo | active | next | blocked | done | cancelled`) and `glyph`;
- `taskKind`, `titleParts` (`[{ text, kind: text | code | link | url | match }]`), and
  `section`;
- `group` (`this-note | in-progress | next | open | history`) and `hidden`;
- `linkTarget`, `suggestedId`, and `disabled: null | { short, full }`.

`subjects` is an array (or a function returning one) of `{ path, line?, blockId? }`. An
entry matches by `blockId` when one is given, else by `path` plus a 0-based `line`.

**nav `api.proseField` v2.** This is additive, and `attach` is unchanged.

- **New options:**
  - `tasks`, which defaults to the field's `links` value; `false` keeps `((` literal;
  - `subjects`, as above.
- **New handle methods:** `isTaskOpen()`, `isNaming()`, and `setSubjects(next)`.
- **Changed handle method:** `isLinkOpen()` is true while either mode is open.
- **Compatibility.** Hosts never need to check the version. An older nav simply ignores
  `subjects`, and `logRecall.attach({ links: true, … })` hosts get `((` with no change.

**bob `capture-complete -T, --task-links`.** It requires `--links-only`. A `((` field at
the caret returns the existing `wikilink_task` response: the same shape, rows, guards,
`picker` descriptor, `replacement`, and `cursor_after`. Rows are grouped against
`--source-note` when one is given; without it there is no "this note" group. `[[` fields
win when both could apply. Everything else stays an empty success. The flag is additive,
so the commit is `feat(capture): …` with no `!`.

## Phases

General rules for every bob-plugins phase (same as `bob-cli-om`):

- Open bob-plugins with `/sase_repo` and read its `AGENTS.md`.
- Edit only `src/` fragments and register new ones in `src/fragments.json`. Keep every
  fragment at or under 1000 lines. nav's `189b-prose-field-controller.js` is at 836, so
  tasks-mode code goes in new fragments.
- Add new test files to `test:unit` or `test:perf`.
- Run `npm run build`, `npm test`, `npm run validate`, and `npm run docs:check` when
  docs change.
- Bump the changed plugins' manifest versions (minor) and run `npm run docs:gen`.
- Finish with `bob plugins sync`, following `docs/develop/deployment.md` if it refuses.
- If phases running in parallel touch the same plugin, rebase and rebuild `main.js`.
  Never hand-merge generated output.
- **Docs budgets.**
  - `docs/bob-navigation-hotkeys.md` sits about 17 lint words under its 6000-word
    reference cap. Put the full "`((` in prompts" section in `docs/block-id-prompt.md`,
    which owns `((`. Give nav's Wikilink section a single cross-link sentence, and trim
    elsewhere on that page if the lint needs it.
  - `docs/develop/plugin-apis.md` already warns above 8000 words, tracked by
    `bob-cli-ob`. Add only concise API rows there.

### Phase cli-task-links — bob-cli

1. **Prose detector.** In `src/native/capture_task_wikilinks.rs`, add
   `prose_completion_field_at(raw, cursor)`.
   - It reuses `is_valid_opener`, `token_from_opener`, and `field_at_token`. It skips
     `cursor_in_close_log_text`, `editor_item_at`, and `is_whole_item_action`.
   - Do not change the capture-draft behaviour of `completion_field_at` or
     `wikilink_task_spans`.
2. **Flag.**
   - Add `task_links_arg()` (`-T, --task-links`, `.requires("links-only")`) to
     `capture_complete/cli.rs`, keeping the options sorted. Help text: "Also complete
     `((` task links in plain prose (requires --links-only)".
   - Thread it through the request.
   - In `engine.rs` `links_only_result`, keep the `[[` path first. Then, when the flag
     is set and the prose detector finds a field, build the same response as the
     capture-path `wikilink_task` branch, with owner = `--source-note` or `""`. Factor
     that branch into a shared helper rather than copying it.
3. **Prose vectors.** Add `tests/fixtures/task_wikilinks/prose_trigger_vectors.json`
   with `version: 1` and a `source`. Each case has `name`, `before`, `after`, and
   `expect: null | { query, replaced, closed }`, all as strings, never offsets. Cover at
   least:
   - line start, after a space, and after a tab;
   - mid-word `f((x`, `(((x`, and `\((x`;
   - inside backticks, inside `[[a ((b]]`, and after an unclosed `[[a ((b`;
   - a closed `((a))` with the caret before, inside, and right after `))` (expect
     non-null), and one character past `))` (expect null);
   - a mid-token caret in `((ap▏i more` (replaced `((api`);
   - a query with spaces;
   - query stops at `[`, `)`, and `` ` ``;
   - two tokens on one line (last wins);
   - a token on the second line of a multi-line value;
   - emoji and RTL neighbours;
   - prose that capture would treat as grammar (`x ((foo`, `- 1 ((foo`, `@inbox ((foo`),
     which **does** open here.

   A Rust test replays the file through `prose_completion_field_at`.

4. **Tests.**
   - `-T` without `-L` is a clap error.
   - A `((` field returns `wikilink_task` with guards and `cursor_after`.
   - With `-S`, the "this note" group is present; without it, it is absent.
   - `[[` still wins.
   - A plain `-L` (no `-T`) still leaves `((` empty.
   - The capture path is unchanged.
5. **Docs.** Update `docs/capture.md`'s `capture-complete` section (the `-L` paragraph
   gains `-T`) and run `just docs-gen`. Never edit `Cargo.toml`'s version or
   `CHANGELOG.md`.

Acceptance:

- `just all` is green.
- `bob capture-complete -L -T -S projects/api.md -c 22 -f json -- 'Superseded by ((api do'`
  returns `context: wikilink_task` with replacement `{14,22}`.
- The vectors test passes.

### Phase bip-task-api — bob-plugins (block-id-prompt)

- **Detector.** `taskLinks.fieldAt` is a pure, UTF-16 port of the prose rules.
  - Build it from `022`'s `findTaskWikilinkToken` per physical line, plus bob's
    protected-markup check (inline code and `[[…]]`).
  - Add `minOffset` and `mode: "field"`, which accepts only an opener at offset 0.
  - Never split a surrogate pair.
  - Copy bob-cli's prose vectors byte-identically to
    `scripts/fixtures/task_wikilinks/prose_trigger_vectors.json`, assert the shared
    `source`, and replay every case.
- **Session.** `open({ sourcePath, subjects })`:
  - Take one `TaskWikilinkIndex.snapshot({ ownerPath: sourcePath, overlays })` per
    session.
  - `rank(query)` runs `orderForWikilinkPicker` and maps rows to the API row shape.
  - `titleParts` comes from the picker view's existing title-part builder (`088`). Move
    it into a shared pure fragment if it is not already one, so the picker and the API
    cannot drift.
  - Guards follow the precedence above. Add a subject guard to
    `pickerDisabledReasonFor`'s family with the wording "The task this prompt is about"
    and the short badge `this prompt`.
  - `indexing` reports build progress. `onChange` relays index progress and changes to
    already-snapshotted notes, so the caller can re-rank.
  - Nothing in `rank` reads the vault.
- **Naming.** `suggestions(row)` uses `resolvePromptBlockIdSuggestion` /
  `suggestPromptBlockId` against the note's text (open buffer first), so it reserves
  every `^id` in the note. `validate` and `name` follow "Naming (the one write)" above.
  - Factor the editor-free pieces out of `submitLinkTaskBlockId` and
    `applyTargetTaskPlan`. Do not fork them. The editor's `((` path must keep its
    current behaviour, and its tests must stay green.
  - Use `vault.process` for closed notes. The editor's existing `vault.modify` path is
    out of scope.
- **Lifecycle.** Freeze `taskLinks` into `plugin.api` beside `pomodoroLinkPicker`. On
  unload, replace it with a null-backed copy (`fieldAt` → null, `open` → an empty
  session) and close live sessions.
- **Tests** (`scripts/test-block-id-prompt-task-links-api.cjs`):
  - vector replay;
  - `minOffset` and field mode;
  - row mapping and `titleParts`;
  - guard precedence including subjects;
  - indexing progress;
  - `linkFor` re-check fallback;
  - `name()` happy path through both the editor and `vault.process`;
  - stale and ambiguous relocation;
  - a duplicate ID;
  - an already-identified line;
  - two disagreeing editors;
  - the identity changing inside the `process` callback;
  - unload safety.

  Add a perf gate (T9) to `test:perf`, reusing `task-wikilink-index-perf-*.cjs`.

- **Docs.**
  - `docs/develop/plugin-apis.md`: the `taskLinks` v1 row and its contract.
  - `docs/block-id-prompt.md`: one sentence saying the engine is shared with prompts.
    The full prompt section lands in nav-task-band.

### Phase nav-task-band — bob-plugins (bob-navigation-hotkeys)

- **Fragments.**
  - `189c-prose-task-model.js` (pure, never throws):
    - the ID-line model (`value`, `caret`, `selected`, and the ops insert, backspace,
      delete-word, delete, move, select-all, set-suggestion, and sanitize);
    - the row-to-view mapping and group runs;
    - the crumb, the detail strings, and the leftover-hint messages.
  - `189d-prose-task-band.js`: the renderer, for both stages, on the `bob-link-band-*`
    frame.
  - `189e-prose-task-mode.js`: `createProseTaskMode(ctx)`, the tasks-mode state machine,
    which `189b` delegates to.
  - Changes to `189b` are limited to the mode switch, key routing, the `beforeinput`
    listener, the v2 options and handle, and `isLinkOpen` covering both modes.
  - Bump `createProseFieldApi` to `version: 2`.
- **Behaviour.** Implement Visual design, Keyboard contract, Naming, Accessibility, and
  T1–T9 exactly.
  - Look up bip's `taskLinks` lazily at each potential open.
  - Open a session when the band opens, and close it when the band closes, on `detach`,
    and on unload.
  - Keep the `[[` dismissal memory and the `((` dismissal memory separate, keyed by
    mode.
  - Recall hides while either mode is open.
  - `rebind` preserves the stage, the selected key, the ID-line state, and the field
    snapshot.
- **Hint.** Add the leftover `((…` message to the unresolved hint. Keep a session set of
  just-named `path#^id` that the block resolver treats as resolved.
- **Styles.** `bob-task-band-*` in nav `styles.css`, next to the LINK band block.
- **Real-Obsidian spike** (`scripts/obsidian-*-e2e.cjs` harness pattern). Run it before
  the phase is done, on S1 (an input) and on a throwaway `<textarea>` host, and record
  the results in the phase bead notes. Prove that:
  - `beforeinput` cancellation keeps the field byte-identical through typing, paste, and
    Backspace in NAME;
  - an IME composition in NAME is restored;
  - one Ctrl+Z after a NAME accept restores `((query`;
  - Esc and Ctrl+[ in NAME go back to the list, then close the band, then reach the
    modal: one layer per press;
  - a held Enter never moves from the list to NAME to a write;
  - a pop-out window works.

  If `beforeinput` is not reliably cancellable in Electron's `<input>`, fall back to
  keydown interception plus snapshot restore. Document the fallback, and keep the T1
  tests asserting the outcome either way.

- **Tests** (`navigation-hotkeys-harness.cjs`, `modal-harness.cjs`):
  - the opening rules and the vectors' controller-only rules;
  - list states: empty grouped, query, no match, disabled, indexing, refresh on
    `onChange`;
  - accepting an identified row (range, caret, swallow);
  - NAME: entry, the edit ops, refused characters, cycling suggestions, validation,
    pending, success, stale return, failure, blur cancel, and the Esc ladder;
  - `rebind` across stages;
  - `detach` idempotence;
  - ARIA;
  - the leftover hint and the just-named overlay;
  - unchanged `[[` behaviour, with the existing link-band tests green;
  - fail-open without bip.

  Extend the HTML preview script with tasks-mode states (grouped, query, ID-less,
  disabled, no match, indexing, NAME roomy and dense, validation error, leftover hint)
  in light and dark.

- **Docs.**
  - `docs/block-id-prompt.md` gains a new "`((` in prompts" subsection: the scope rule,
    the key tables, NAME, and the write and cancel semantics.
  - `docs/develop/plugin-apis.md`: the `proseField` v2 row.
  - nav's Wikilink section: one cross-link sentence, within budget.

### Phase prompt-subjects — bob-plugins (nav + bip)

- **Subjects.** Pass `subjects` for S1–S11, as in the Surfaces table. Every listed
  surface opts in explicitly; S12 declares none.
  - S1 is wired in bip `090`. S2–S6 are wired at each modal's `attach`.
  - S7–S11 go through `attachPickerLogRecall` (`185`) and
    `attachPickerScheduledReasonLinks` (`185a`), using the Task Card's target task
    identities.
  - When a task identity is unknown, pass `{ path, line }` from the host's own write
    plan.
- **Key routing.** Verify that picker hosts (`ownKeys: false`) offer Tab, Shift+Tab, ↑↓,
  Ctrl+N/P, Enter, Esc, and Ctrl+[ to `handleKeydown` before their own logic while NAME
  is open. That covers S10's Tab field cycling, S9's Alt+P (which keeps priority), and
  S11's date-row navigation. S6's 350 ms Enter guard and blur commit must hold: NAME
  never blurs, and a real blur cancels naming without writing.
- **Host safety (T11).** For each family (bip unlink write, nav lane modals, the
  keep/decay writes, the successor cancel, and the Task Card stage writes), add a test:
  1. open the prompt;
  2. name an ID-less sibling task in the subject's note through the API;
  3. submit;
  4. assert that the host's write lands on the right line and does not refuse as stale.
- **Prose lint.** Extend `scripts/fixtures/prose-fields.json` with
  `subjects: "wired" | "none"` per surface. `scripts/test-prose-field-roles.cjs` then
  asserts that each `wired` wiring file mentions `subjects`. S12–S15 are `none`.
- **Docs.** One line each in `docs/task-card.md` and the other affected pages, saying
  `((` works where `[[` does and that the prompt's own task is shown but disabled.

### Phase card-surfaces — bob-plugins (ledger-tools + mail)

- **S14 Intention Card.**
  - Each paint's `rebind` preserves NAME, including the ID-line caret and the snapshot.
  - `intentionOnKeyDown` offers keys first. In NAME, Tab and Shift+Tab cycle
    suggestions, and Ctrl/Cmd+Enter is consumed.
  - The pending-write swallow and the multi-line paste collapse are untouched in the
    list stage. In NAME, paste goes to the ID line.
  - Check that `bob periods update`'s YAML quoting of `[[…#^…]]` round-trips.
- **S15 Record Card.**
  - The typed-stage handler offers keys first, before the Enter save, `recordTab`, the
    list Enter, and suggest cycling.
  - The document capture Esc/`q`/Ctrl+[ listener calls `backOut()` first: NAME, then the
    list, then the card.
  - `recordTypedInput` repaints must not run in NAME. Cancelled `beforeinput` fires no
    `input`; prove it.
  - In the link stage (field mode), `((` opens only at offset 0. Accepting replaces the
    whole value with `[[target#^id]]`, which `recordCardParseLinkDraft` already accepts
    as a wikilink with a subpath. Add a test.
- **S13 Task from email.** `((` follows its `[[` wiring. Enter continues to do nothing
  while the band is closed.
- **Docs.** `docs/intention-card.md`, `docs/record-card.md`, and `docs/bob-mail.md`: one
  line each.

### Phase mac-pull-in-tasks — bob-mac-capture

Read its `AGENTS.md`. Never land uncompiled Swift. Run `Scripts/macos-ci-check`, then
watch the exact-SHA macOS run to green with `/sase_monitor`.

- **Data.**
  - The pre-filter becomes "the reason contains `[[` or `((`". bob still decides whether
    the caret is in a field.
  - Requests add `--task-links` to the existing
    `capture-complete --links-only [--source-note]` call.
  - Accept the `wikilinkTask` context alongside the three wikilink contexts.
  - On `unsupportedOption("--task-links")`, drop `-T` for the session, cached per
    executable identity like `--links-only`, and keep `[[`.
- **Presentation.**
  - `CapturePullInLinkBand` renders task rows with `completionRowContent`'s
    `.wikilinkTask` case and the shared locator colours (`WikilinkTaskPickerIndex`
    styling).
  - Header: `LINK · tasks · k of n`.
  - An empty `((` shows the group capsule on the first row of each run.
  - The detail line uses the strings from Visual design.
  - The reason row's own task (same `note_path` and `block_id`) is dimmed with "The task
    this prompt is about". This only filters what bob returned.
- **NAME stage.**
  - Return on an ID-less row turns the band into the NAME layout. It uses the reusable
    `BlockIDField`, focused, prefilled with bob's first `block_id_suggestions` entry and
    selected, with the other suggestions cycled by Tab/↑↓.
  - Return runs `capture-task-id --note-path --task-ref --block-id [--allow-closed]`,
    passing `--allow-closed` for DONE and CANCELLED rows. It then splices bob's
    `wikilink_replacement` over bob's `((` byte range as one undo step, puts the caret
    after the link, and returns focus to the reason field.
  - Esc returns to the list with the filter intact.
  - The reason draft is never mutated before the write succeeds. A changed draft shows
    an error and inserts nothing.
  - Moving focus into the name field must not confirm or restore the reason. Verify the
    reason field's focus-loss handling.
- **Fix: Return never confirms while the band is open.**
  `acceptPullInLinkFromKeyboard(onFailure: .confirm)` currently closes the band and then
  confirms the Pull-In when there are zero rows, the edit is stale, or the editor is
  missing. That contradicts the documented "Return closes the band without confirming".
  - Make every such failure close the band and stop.
  - Add a controller-level test that drives Return with zero rows. The existing test
    only calls `closePullInLinkBand()`.
  - Also fix `pullInLinkToken` so the detail preview finds a `((` replacement.
- **Docs.** In this repo: `docs/wikilink-completion.md` (the Pull-In section gains `((`)
  and `docs/keyboard.md`. The bob-cli site mounts them, so do not edit bob-cli's copies.
- **Release.** This needs a released bob with `-T`, which ships on the daily train. It
  fails open until then.

### Phase task-walkthrough — bob-plugins

- Run one isolated real-Obsidian e2e walkthrough over these surfaces:
  - S1;
  - S6 (dense);
  - S10;
  - S11 (after the date token);
  - S14 (reflection);
  - S15 (text stage and link stage).

  In each:
  - type `((`, filter, and accept an identified task;
  - type `((` again, pick an ID-less task, cycle a suggestion, and press Enter;
  - confirm the task line gained exactly ` ^id` plus the stamp;
  - undo once;
  - check the Enter safety and the Esc ladder;
  - submit, and confirm the written text holds `[[target#^id]]`.

- **Mention round-trip.** Run `bob task reconcile`, then check that each linked target's
  `🪢 LINKED FROM` row shows a `💬` mention chip and that no Task Link or relation was
  created.
- Attach light and dark screenshots of both stages as artifacts with `/sase_artifact`.
- Run `npm run docs:check`. If a phase left `docs/bob-navigation-hotkeys.md` over its
  cap, fix it here.

## Decision record

> [!decision] decision_record Use `/sase_memory_write` to add
> `sase/memory/decisions/prompts-link-tasks.md`, with `origin: requested`,
> `metadata.status: accepted`, and `decided: 2026-11-21`, then run `sase memory init`.
>
> - **Applies to:** bob-plugins prose prompts and cards, the Bob Mac Capture Pull-In
>   reason, and bob-cli `capture-complete -T`.
> - **Claim:**
>   - `((` completes wherever `[[` does, as the tasks mode of the same LINK band.
>   - Accepted rows insert bob's `[[target#^id]]`.
>   - Enter and Tab never submit while the band is open.
>   - The only write is naming an ID-less task, on an explicit NAME-stage Enter. It adds
>     ` ^id` plus the editor's freshness stamp to one re-located line, and never to the
>     prompt's own subject task.
>   - Focus never leaves the field.
> - **Why:** cite this plan and `bob-cli-om`. Rejected alternatives:
>   - a modal picker over the prompt (stacked modals, blur commits);
>   - a second focused input for the ID;
>   - identified-only rows;
>   - spawning bob per keystroke in Obsidian.
> - **Cost:** one more API (`taskLinks`), and hosts declare their subjects.
> - **Reopens when:** a prompt needs literal `((`, or naming must happen elsewhere.
>
> Link `[[decisions:mac-capture-is-a-thin-client]]` and
> `[[decisions:q-and-ctrl-bracket-mirror-esc]]`. The task-walkthrough phase does this
> step.

> [!decision] decision_record = no Skip it. Follow `/sase_memory_write`'s
> declined-change rule.

## Non-goals

- **No changes to `((` where it already works.** The editor's Task Wikilink Picker and
  the Mac capture panel's `((` behave as they do today.
- **Pomodoro linking, lane changes, or relations from prompts.** A prompt `((` writes a
  mention, never a Task Link, a LINKS entry, or a status change. The editor's "Link a
  task to this Pomodoro" mode is not used.
- **Creating tasks from `((`.**
- **Undoing an ID write with Ctrl+Z.**
- **The Mac event-note editor**, and every literal or identifier field.
- **Changing capture-draft `((` grammar.**

## Risks

- **`beforeinput` cancellation in Electron.** The spike runs first. The keydown plus
  snapshot-restore fallback keeps T1 true either way.
- **Picker hosts that do not offer NAME keys first.** Mitigations: text mutations go
  through `beforeinput` regardless of host routing; prompt-subjects audits every picker
  stage; and a key a host steals can only fail to edit the ID, never change the field.
- **Host writes going stale after a sibling is named in the same note.** Mitigations:
  the subject guard, one-line in-place writes that never shift line numbers, and the T11
  tests.
- **Metadata-cache lag after naming.** The just-named overlay stops false unresolved
  hints, and `onChange` refreshes the rows.
- **Release train.** The Mac stays `[[`-only until bob ships `-T`.
- **Docs budgets** on the nav page. The `((` docs therefore live in block-id-prompt's
  page.
