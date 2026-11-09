---
tier: tale
title: Add the Project Note Capture glossary term
goal:
  Define the sealed `@<route>^<id>@` / `@<route>:<id>@` project-note capture form as a
  new glossary term, Project Note Capture (aka project capture), grounded in the capture
  contract.
size: small
decisions:
  add_project_note_capture_term:
    ask: Should the plan add the new Project Note Capture glossary strand?
    memory:
      - glossary:project-note-capture
    requested:
      Can you help me add an excellent new "project note capture" (aka "project
      capture") glossary term that describes the `@foo[:^]bar@` capture syntax?
    default: true
    answer: true
  link_from_capture_strand:
    ask:
      Should the Capture strand name project-note capture where it lists what an item
      may create?
    memory:
      - glossary:capture
    default: false
    answer: true
proposed_by: bbugyi200.athena.1af
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-09 10:08:17
status: wip
---

# Add the "Project Note Capture" glossary term

## Goal

Add one memory-web strand, `glossary:project-note-capture` (keyword **Project Note
Capture**, aliases `project capture` and `project-note capture`). It defines the sealed
capture marker `@<route>^<block-id>@` / `@<route>:<block-id>@` (Bryan's shorthand is
`@foo[:^]bar@`), which creates a brand-new project note instead of a task. This is a
memory-only change: no Rust, docs, or plugin edits.

## Why this term, and why these names

- The form is one of capture's largest grammar families: two forms, a Pomodoro suffix,
  task-ID bullets, move bullets, `- ^^`, offers, and a set of refusals. But no glossary
  term names it. The Capture strand mentions "project note" only in passing, and the
  Successor Offer strand mentions `@route:id@` without explaining it.
- Keyword: Bryan's phrase. Aliases: his short form `project capture`, plus
  `project-note capture`. That hyphenated spelling is the one the codebase uses: the
  refusal text `project-note capture cannot be combined with %c`, "sealed project-note
  item" throughout `docs/capture.md`, and the `project_note` parse mode. The glossary
  web links mentions implicitly (`closure: mentions`), so both spellings should resolve.
  No other strand contains either alias today, so adding them changes no existing
  closure.

## Source of truth (read these before writing)

The strand must agree with these sources. Where the draft below disagrees with them, the
sources win: fix the strand, not the docs.

- `docs/capture.md`:
  - the syntax-table rows for `@route^block-id@` and `@route:block-id@`;
  - the error/example table rows for `@cash^goog-exit@`;
  - `### Project notes`, `#### Moving parent tasks in`, `#### Choosing parent tasks in`,
    and `#### Offer a new project's tasks`.
- `docs/reference/capture-syntax.md`, `## Project captures`.
- Grammar: `src/native/capture_language/tokens.rs` (the seal parse around the
  `CaptureKind::ProjectNote` construction), `src/native/capture_language/markers.rs`
  (the project-note error builders), and `ProjectNoteForm { Create, Offer }` in
  `src/native/capture_language/model.rs`.
- Writer: `src/native/capture/project_note.rs` and `src/native/capture_project_note.rs`;
  CLI behavior tests are in `tests/cli/capture/project_note.rs` and
  `tests/cli/capture/project_note_offer.rs`.
- Neighbor strands, to match their style and avoid restating them: `capture`,
  `bulk-capture`, `project-note`, `project-task`, `successor-offer`, `task-drawer`. Read
  them with
  `sase memory read glossary:capture glossary:project-note glossary:successor-offer -r "<why>"`.
- Decision `decisions:capture-bullets-mirror-the-task-block`.

## Changes

### 1. Create `sase/memory/glossary/project-note-capture.md`

> [!decision] add_project_note_capture_term

Use your `/sase_memory_write` skill first; this plan's accepted
`add_project_note_capture_term` decision authorizes the edit. Follow the house strand
shape:

- frontmatter has only `keyword` and `aliases` (as in `inbox-file.md` and
  `task-drawer.md`);
- no H1 title, because the renderer adds one;
- prose wrapped near 88 columns;
- links are bare `[[slug]]` for glossary terms and `[[decisions:<slug>]]` for decisions;
- every other `[[...]]` (frontmatter values like `"[[project]]"`) goes inside inline
  code, so it is never scanned as a link.

Start from this draft. After checking it against the sources above, keep it, or tighten
it if a sentence proves wrong or redundant:

```markdown
---
keyword: Project Note Capture
aliases:
  - "project capture"
  - "project-note capture"
---

A [[capture]] item whose route marker is sealed with a trailing `@`, `@<route>^<id>@` or
`@<route>:<id>@` (optionally followed by `#<pomodoro>`), so it creates a brand-new
[[project-note]] instead of a task. The marker may lead the item or trail any of its
lines. bob writes `<route>_<id>.md` at the vault root (each `-` in the ID becomes `_`)
with `parent: "[[<route>]]"`, `type: "[[project]]"`, `status: wip`, and the
`template: "[[new_project]]"` line, in the same shape as the Obsidian **Create project
note from task** command, which instead promotes an existing task. The item's parent
text becomes the [[project-task]] line `- [ ] #task #prj <text> #hide ^prj` (`[?]` once
the item resolves a schedule), which is never linked. `<route>.md` must already be an
area or non-terminal project note, a target note that already exists or is staged
refuses the capture, and `bob projects sync` later adds the note to the parent's
Sub-projects line.

The sigil before the ID picks the form. The `^` (create) form links only the item's
explicit `:` bullets. The `:` (offer) form creates the same note, then offers its open,
unhidden, not-yet-linked `## Tasks` rows as [[successor-offer]]s of the new `^prj`,
linking none until picked with `-p` (Bob Mac Capture's Pull-In card). A `#<pomodoro>`
after the seal names the Pomodoro those links land under, created on demand, and is
refused when the item has nothing to link or offer.

First-level bullets mirror the new note
([[decisions:capture-bullets-mirror-the-task-block]]). Each becomes a `## Tasks` row: a
trailing ` :<id>` names it, makes it Next, and adds its [[task-link]] under the Pomodoro
its type places (or the `#<pomodoro>` one), while ` ^<id>` only names it. An ALL-CAPS
bullet with nested notes becomes a `## Title Case` section instead. A bullet that is
only `^<id>` or `:<id>` moves an existing open top-level task out of the parent note
(`:` also links it), and `- ^^` chooses several from a Move card before writing. The new
note, the moved blocks, and today's ledger write in one all-or-nothing batch.

`%c`, `--clip`, `=<X>` and `=x` session suffixes, `!`, and `@@` declarations are
refused, and the retired `+` seals (`@route^id+`, `@route:id+`) teach the `@` spelling.
Not `@route^id` (an ordinary task with a block ID), `@route:id` (a task linked into a
Pomodoro), `@route+id` (a note under an existing task), or record-note capture
(`@dir/^stem`). `capture-parse` reports mode `project_note`, or `pomodoro_project_note`
once an explicit `:` link exists. See `docs/capture.md` ("Project notes") and
`docs/reference/capture-syntax.md` ("Project captures").
```

Claims to check one by one (each is a sentence in the draft):

1. The leading marker is allowed only on the item's first physical line; a trailing
   marker may sit on any of its lines, including a "sealing child line" (see "Routing
   and insertion" and "Choosing parent tasks in").
2. The filename is lower-cased route + `_` + ID, with `-` → `_` and the ID's case kept.
   The note is at the vault root, and its frontmatter keys are as listed.
3. `^prj` is `[ ]`, or `[?]` when a schedule resolves (`s:<N>` or a `$<N>` roll). It is
   never `[*]` and never linked.
4. The parent must exist and be an area or non-terminal project note. A note that
   already exists or was staged earlier in the batch fails with
   `project note already exists`.
5. The offer form offers only open Ready, Next, and Pending `## Tasks` rows. It folds
   `#hide` rows and already-` :`-linked rows, never offers `^prj` or the placeholder,
   and links nothing without `-p`.
6. `#<pomodoro>` placement and the unused-name refusal (`unused_project_note_pomodoro`
   and its offer variant).
7. The bullet rules: ` :<id>` / ` ^<id>`, ALL-CAPS-with-children → `## Title Case`, bare
   `^<id>` / `:<id>` moves, and `- ^^`.
8. The refusals listed: `%c`, `--clip`, `=<X>`/`=x`, `!`, `@@`, and retired `+`.
9. The `capture-parse` modes `project_note` and `pomodoro_project_note`. The docs say
   the latter applies "when an explicit `:` link exists" and also "when a `:` move is
   present". If one short phrase can't cover both, say "once the item links a task".

If a claim needs more than a clause to state correctly, drop the detail rather than grow
the strand. `docs/capture.md` remains the full contract. Keep the strand within roughly
10% of the draft's length.

### 2. Optionally link from the Capture strand

> [!decision] link_from_capture_strand

In `sase/memory/glossary/capture.md`, change the phrase "project note," in the list of
what an item may create to "project note (a [[project-note-capture]]),". Change nothing
else in that strand.

> [!decision] link_from_capture_strand = no

Leave `capture.md` untouched.

### 3. Regenerate

Run `sase memory init`. It rewrites:

- the `<!-- sase:strands -->` roster in `sase/memory/glossary.md`, which should now list
  `Project Note Capture (project capture, project-note capture)` between
  `Project Note (prj note)` and `Project Task (prj task)`;
- the matching roster line in the generated `AGENTS.md`;
- any generated stats in `sase/memory/README.md`.

Never hand-edit those generated files.

## Verification

1. `sase memory show glossary:project-note-capture` renders the strand, and its
   `## Linked References` / related-terms section resolves Capture, Project Note,
   Project Task, Successor Offer, Task Link, and the decision without any
   unresolved-link warning.
2. `sase memory show "glossary:project capture"` and
   `sase memory show "glossary:project-note capture"` both resolve to the new strand.
3. `sase memory show glossary:successor-offer` and `sase memory show glossary:capture`
   print the same closures as before, unless the `link_from_capture_strand` decision was
   accepted, in which case only Capture gains the new term.
4. `git status --short` shows the new strand plus generated changes (and `capture.md`
   only under the opt-in decision). Nothing under `src/`, `docs/`, or `tests/` changes.
5. `grep -n "Project Note Capture" AGENTS.md sase/memory/glossary.md` finds the roster
   entry in both.

No Rust build or test run is needed: the change touches only memory files and their
generated outputs.

## Out of scope

- Editing `docs/capture.md` or `docs/reference/capture-syntax.md`. One drift exists: the
  reference page says a project-note ` :id` child links into "the current/next
  Pomodoro", while the contract says the Pomodoro the task's type places. If you confirm
  it, file it as a separate bug through `/sase_new_task` instead of fixing it here.
- Glossary terms for neighboring concepts (record-note capture, task promotion).
- Any change to other strands beyond the opt-in Capture link.
