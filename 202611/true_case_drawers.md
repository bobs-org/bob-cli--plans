---
tier: epic
title: "True-case drawers: bold headers that keep their real names"
goal: "A task drawer is a `- **Title**` bullet whose title keeps its authored
  capitalization everywhere: on disk, in capture, in Obsidian and Bob Mac Capture, and
  through promotion, reversal, and rename. Every surface draws it as a quiet,
  recognizable drawer, and ALL-CAPS stops being structure after a gated, multi-host-safe
  migration.

  "
decisions:
  drawer_glyph:
    ask:
      Which glyph should replace the bullet on a drawer header in Obsidian and the Mac?
    choices:
      tray:
        Open tray (a shallow ⊔ with an inner shelf line); the Mac draws SF Symbol tray
      drawer_front:
        Drawer front (wide rounded box, centered handle); the Mac draws SF Symbol
        cabinet
    default: tray
    why: An open tray stays distinct from the square task checkbox even at small sizes
    answer: tray
  memory_drawer_record:
    ask:
      Add a "Drawers Keep Their Real Names" decision and mark Capture Bullets Mirror The
      Task Block superseded in part?
    memory:
      - decisions:drawers-keep-their-names
      - decisions:capture-bullets-mirror-the-task-block
    corrects: ALL-CAPS with notes is a drawer
    default: true
    answer: true
  memory_glossary:
    ask:
      Rewrite the Task Drawer and Project Note Capture glossary strands for bold,
      true-case drawers?
    memory:
      - glossary:task-drawer
      - glossary:project-note-capture
    corrects:
      A direct non-checkbox ALL-CAPS child bullet of a `#task` line, with its notes
      nested
    default: true
    answer: true
phases:
  - id: drawer-contract
    title: Dual-read drawer recognizer, readers, and outside-text escape
    depends_on: []
    size: large
    description:
      "drawer-contract: one recognizer that reads `- **Title**` and legacy ALL-CAPS
      drawers, the true-case drawer model and Unicode key, every bob-cli vault reader
      retargeted, the `__…__` escape for imported text, DV/DH vectors, and the
      task-drawers contract page, with every writer left unchanged."
  - id: plugin-readers
    title: One plugin recognizer twin and dual-read plugin readers
    depends_on:
      - drawer-contract
    size: large
    description:
      "plugin-readers: one byte-identical recognizer twin replaces every drifted plugin
      copy, the DV/DH vectors are copied, and every plugin reader accepts both drawer
      forms; deployed with plugin writers unchanged."
  - id: reader-rollout
    title: Every host reads bold drawers before any writer flips
    depends_on:
      - drawer-contract
      - plugin-readers
    size: small
    description:
      "reader-rollout: a host-currency script plus a gated check that bob on athena,
      apollo, and the Mac and the vault's deployed plugins all read bold drawers."
  - id: bob-writers
    title: Every bob-cli writer emits bold drawer headers
    depends_on:
      - reader-rollout
    size: medium
    description:
      "bob-writers: every bob-cli writer creates `- **Title**` and `- **Notes**`
      headers, leaves existing headers untouched, and flips the writer vectors."
  - id: drawer-rename
    title: Drawer rename respells and inventories legacy drawers
    depends_on:
      - bob-writers
    size: medium
    description:
      "drawer-rename: `bob task rename-drawer` writes NEW verbatim, allows case-only
      respelling, and gains a read-only legacy inventory for the migration."
  - id: capture-drawers
    title: One place resolver and bold drawer drafts in capture
    depends_on:
      - bob-writers
    size: large
    description:
      "capture-drawers: one place resolver for drawers, project sections, and notebook
      sections; bold draft forms in all four container contexts; `drawer_marker` spans,
      `**Title**` menu rows, preview drawer annotations, the `*` rewrite, catalog
      spelling preference, and grammar 30."
  - id: obsidian-drawers
    title: Obsidian drawer rendering, typing, and bold writers
    depends_on:
      - bob-writers
    size: large
    description:
      "obsidian-drawers: the drawer glyph and true-case title in Live Preview and
      reading view, the `*` typing assists and Toggle Drawer, bold plugin writers, and
      verbatim promotion and reversal."
  - id: mac-drawers
    title: Bob Mac Capture drawer rendering and typing
    depends_on:
      - capture-drawers
    size: large
    description:
      "mac-drawers: the Mac renders bob's drawer spans and preview annotations, dims the
      delimiters, drives the `*` assist through capture-rewrite, and stops uppercasing
      drawer titles."
  - id: vault-migration
    title: Reviewed migration of legacy ALL-CAPS drawers
    depends_on:
      - drawer-rename
      - capture-drawers
      - obsidian-drawers
    size: small
    description:
      "vault-migration: confirm every host writes bold, inventory the legacy drawers,
      and apply Bryan's per-line rename or release decisions through a gate."
  - id: cutover
    title: Retire ALL-CAPS drawer recognition
    depends_on:
      - vault-migration
    size: medium
    description:
      "cutover: once no unreleased legacy drawer remains, remove ALL-CAPS recognition,
      the legacy casing helpers and vectors, and the ALL-CAPS import escape in bob-cli
      and the plugins; grammar 31."
  - id: memory
    title: Memory records for true-case drawers
    depends_on:
      - cutover
      - mac-drawers
    size: small
    description:
      "memory: apply the accepted memory decisions with /sase_memory_write and
      regenerate the glossary page."
proposed_by: bbugyi200.apollo.research.5x.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-22 17:36:44
status: wip
---

- **PROMPT:**
  [prompts/202611/true_case_drawers.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/true_case_drawers.md)

# Plan: True-case drawers

## Context

Bryan wants drawers to stop needing capitals, so a drawer can keep the real
capitalization of its section name (`API Design`, `iOS / OAuth 2.0`, `C++ Notes`,
`Café`). This is a stepping stone toward giving every project note a distinct, lossless
task representation (`^task-to-project`): today promotion turns `API DESIGN` into
`## Api Design` and reversal uppercases `## Future Work` and refuses `## C++ Notes`
outright, so no round trip can be lossless.

Read `research:202611/true_case_drawers/true_case_drawers.md` (with
`sase artifact read`) before starting any phase. Bryan endorsed every requirement it
recommends. Memory edits are the exception: they happen only in the `memory` phase,
under the accepted memory decisions. Bryan's own roadmap note asks for the same marker:
"Use `**Title Text**` for this! The `*` character should expand to `**<cursor>**` when
typed at the start of a sub-bullet!"

### Sequencing with `bob-cli-on`

The task-requirements epic `bob-cli-on` is still landing (`.11.1.6`, `.12`, `.15.1.4`,
`.15.1.5`, and `.17` were open at plan time). Its work does not conflict with this work
in meaning: requirements are checkbox children and drawers are not. But the two epics
edit the same files, vectors, and memory: the promotion seed (`.12`), the task-split and
close-card twins (`.11`), Mac fixtures (`.15`), and `glossary:task-drawer` (`.17`). So
this epic runs strictly after `bob-cli-on` closes:

- **Approve with a wait.** Approve this plan with the wait option `bead=bob-cli-on`
  (`sase plan approve <plan> --kind epic --wait bead=bob-cli-on`, or the same wait in
  the approval gate). No phase launches before the epic closes.
- **Check again in the first phase.** `drawer-contract` step 0 re-reads the bead and
  stops without changes if it is still open.
- **Preserve the epic's constraints.**
  - Drawers hold notes, never requirements or an alternate status. `req_in_drawer`
    stays.
  - The notes drawer is born with the first note
    (`notes_drawer_birth = with_first_note`).
  - Promotion moves only the outer requirement boundary.
  - The retired `REQUIREMENTS` drawer is not revived.
  - Rust owns the semantics. JS twins pass vectors that Rust owns, and the Mac renders
    Bob's decisions.

### The design in one paragraph

A drawer is a bullet whose whole body is one bold span, `- **Future Work**`, directly
under a task. Its title is stored verbatim. Its identity is the case-folded key
(`FUTURE WORK`), so `Notes`, `NOTES`, and `notes` stay one drawer. Capture keeps
`- #slug` and `#"Title"` as typing shortcuts, resolved through one shared place resolver
that drawers, project sections, and notebook sections all use. Accepting a menu row
writes the canonical `- **Title**`, so the draft reads like the block it writes. In
Obsidian and on the Mac, a small drawer glyph replaces the bullet and the true-case
title is set semibold. The words themselves never change. With plugins off (GitHub,
mobile, raw source), it reads as a bold header above its notes.

### Design calls made here beyond the research

The research leaves these choices open or slightly inconsistent. Each one serves the
brief: intuitive, reliable, beautiful.

1. **Readers first, writers second.** The research ships dual-read and bold writers in
   one release. But every host (athena, apollo, the Mac) runs its own `bob`, and each
   follows the daily release train at its own pace. A host that updates first would then
   write `- **Notes**` that a host still on the old release reads as a loose note, and
   that host would add a second `NOTES` drawer. So readers ship first, in bob-cli
   (`drawer-contract`) and in the plugins (`plugin-readers`). An explicit
   `reader-rollout` gate then confirms that every host and the deployed plugins read
   bold. Only after that does any writer flip.
2. **ALL-CAPS drafts are refused while legacy drawers are still read.** The research
   says `- FUTURE WORK` with nested notes becomes "a note, plus a warning". While
   readers still recognize ALL-CAPS, though, that "note" lands on disk as a legacy
   drawer. In project-note capture it would silently turn into a `## Tasks` row instead
   of a section. So until `cutover`, capture refuses it with `caps_drawer_retired` and a
   fix-it hint. The hint names the existing drawer's spelling when the key matches
   ("write `- **Notes**`"). After `cutover`, the line is ordinary content. It gets a
   non-blocking warning only when its key matches a known drawer name.
3. **Legacy spellings never seed a new title.** The drawer catalog (version 3) records
   each spelling's marker. Creation prefers explicit spellings (bold drawers and project
   `##` headings). A key with only ALL-CAPS spellings falls back to the notebook create
   rule, so the system never re-shouts.
4. **Migration decides rename or release for each line.** Each legacy drawer is either
   renamed to a spelling Bryan supplies or released. A released line becomes an ordinary
   note at cutover, with no bytes written. Lines like `DONE`, which may never have been
   drawers, cost nothing. The cutover condition is "no unreleased legacy drawer
   remains", not a vault-wide rewrite.
5. **One import escape for both forms.** While legacy drawers are still read, importers
   write any would-be drawer line, ALL-CAPS or whole-bold, as `__…__`. That renders as
   the same bold text and is never a drawer. `cutover` drops the ALL-CAPS half.
6. **Bob annotates, the Mac renders.** Preview block lines and drawer objects carry an
   additive `drawer` / `marker` annotation, so the Mac never sniffs `**`
   ([[decisions:mac-capture-is-a-thin-client]]).
7. **One ink family.** A drawer's glyph wears its owner task's type-hue ink, the same
   ink the requirement `⌗⌗` run uses. So a block reads as one family: square for the
   task, diamond for requirements, glyph for drawers, dot for notes.
8. **The title mapping is a Rust contract.** The drawer ⇄ `##` title mapping ("DH") is
   owned by bob-cli and shared through vectors. Obsidian promotion and reversal pass
   those vectors instead of keeping their own casing functions.

## Ground rules for every phase

- **Rust is the authority.** Rules live in bob-cli first. Each plugin copies a bob-cli
  fixture byte for byte. Its README says "update bob-cli first, then copy", like
  `task_reqs` and `task_split` today. The Mac stays a thin client: it never parses
  grammar or decides drawer eligibility. Its only local editing is keystroke handling
  around bob's rewrite answers and spans.
- **Other repositories.** Open `bob-plugins` and `bob-mac-capture` through `/sase_repo`
  and work only at the path it prints. Read each repo's `AGENTS.md` first.
- **bob-plugins.**
  - Edit `src/` fragments and keep each one at 1000 lines or fewer. Add new fragments
    rather than grow near-cap ones.
  - Run `npm run build` and `npm test`. Add any new test file to the `test:unit` list.
  - Deploy only the plugins you changed, with `bob plugins sync -p <id>`. Never pass
    `-F` unprompted.
- **Bob Mac Capture.**
  - Compile every Swift change, using the scratch build and `just typecheck-tests` as
    that repo's `AGENTS.md` describes. Then run `just portable-check` and the macOS CI
    check script.
  - After every push, watch the exact-SHA macOS run to green with `/sase_monitor`.
- **Docs and CLI.**
  - Follow the `docs.md` and `cli_rules.md` reference memory (read them with
    `/sase_memory_read`).
  - Run `just docs-gen` and `just docs-check`. The gates are `just check` for bob-cli
    and the per-repo gates above.
- **Releases.** Never edit the Cargo.toml version or CHANGELOG.md. Mark a CLI, JSON,
  vault-format, or grammar contract break with `!` in the commit subject.
- **No silent vault rewrite.** Code changes only what a gesture writes. An existing
  drawer header is never respelled, re-marked, or moved in the background, and
  `bob task reconcile` stays out of it. Bulk changes happen only in `vault-migration`,
  through a gate.
- **One contract page.** `docs/task-drawers.md`, created in `drawer-contract`, is the
  behavior contract. Each later phase updates the sections it ships and links details
  out, for example capture grammar to `docs/capture.md`.
- **Two different drawers.** The property-card "drawer" of `bob-cli-pf`
  (bob-ledger-tools foot drawer) is a different thing. Name everything here "task
  drawer" (`bob-task-drawer-*` classes, `data-bob-drawer*` attributes), and never reuse
  or restyle property-card drawer names.
- **Discovered work.** Phase workers record `PROPOSED FOLLOW-UP:` notes on their own
  phase bead and never create beads. Phases outside `memory` never edit memory.

## Design of record

### 1. What a drawer is

A **drawer header** is a list item that meets all of these:

1. It is a direct child of a real `#task` line, by the `note_tasks` rules, outside
   fences and quotes, and its body has no checkbox. The managed-line matchers run first:
   the Schedule, Work, Rotten, and Cancel logs, Depends-On, BLOCKS, and LINKS / LINKED
   FROM. On a project note's own `^prj` task, the note's eligible `##` sections stay its
   drawers, as today (`project_h2_drawers`).
2. Its trimmed body is exactly `**` + title + `**`. Nothing may sit outside the
   delimiters: no trailing text, `^id`, tag, or field.
3. The title is nonempty and a single line, with no whitespace just inside the
   delimiters. It contains no `*`, has at least one Unicode alphanumeric character, and
   does not end with `:`. Internal colons are fine (`**Phase 1: Auth**`). The colon rule
   keeps bold leads such as `- **Note:** …` and `- **TL;DR:** …` as notes, and it
   matches the data: none of the vault's unique project `##` titles end in `:`.
4. Its key is not reserved. Reserved means today's managed set plus `TASKS`, plus every
   structural `##` name `is_structural_h2_key` knows (Tasks, the status groups,
   Pomodoros, Events, Highlights, and the generated catalogs such as Projects and
   Notebooks). This way a promoted drawer can never collide with a Bob-owned section. A
   reserved title in bold is a note.
5. **Children are optional for recognition.** An empty drawer is valid, so a hand-typed
   header can be filled instead of duplicated, and an empty `## Future Work` reverses to
   a valid drawer. Creation is a separate policy: capture never creates an empty drawer.

Three more rules:

- **Escape.** `__Title__` renders as the same bold text and is never a drawer. That is
  the one-character escape.
- **Depth.** A whole-bold bullet deeper than a direct child stays a note, which keeps
  `###` ⇄ nested bold header free for a later stone.
- **Legacy form (dual-read only).** Until `cutover`, a body that matches today's
  ALL-CAPS grammar is also a drawer. That grammar is today's `is_drawer_title`, renamed
  `is_legacy_drawer_title` and kept byte for byte.

**Model.** The recognizer returns the authored title (display and serialization, never
rebuilt from the key), the key, a `marker` (`bold`, `legacy_caps`, or `section` for a
`^prj` `##`), and the header and title byte ranges. Recognition
(`recognize_drawer_header`) and creation policy (`can_create_drawer`) are separate APIs.

**Pomodoro names.** `is_pomodoro_name` gets its own predicate before the drawer grammar
changes, with unchanged behavior. Pomodoro names and task types stay ALL-CAPS; sharing
that vocabulary with drawers was incidental.

### 2. Identity and the Notes role

- **Key.** `drawer_key(title)` collapses whitespace, then applies Unicode uppercase:
  `str::to_uppercase` in Rust and `toUpperCase` in JS. The vectors pin non-ASCII folds
  (`Straße` → `STRASSE`, `Café` → `CAFÉ`), so the twins cannot diverge.
- **Twins.** Same-task case twins stay `ambiguous_drawer`. Display never comes from the
  key.
- **The notes role.** It splits into `NOTES_KEY = "NOTES"` for every comparison and
  `NOTES_TITLE = "Notes"` for writers. A legacy `NOTES`, a `**Notes**`, and a
  `**NOTES**` are all the notes drawer, by key. Until `bob-writers`, writers keep
  today's `NOTES` spelling. The JS twins follow the same split.

### 3. Dual-read, writers, and cutover

| Window                                                                          | Readers       | Writers            | Capture drafts                          |
| ------------------------------------------------------------------------------- | ------------- | ------------------ | --------------------------------------- |
| After `drawer-contract` / `plugin-readers`                                      | bold + legacy | legacy (unchanged) | unchanged                               |
| After `reader-rollout` → `bob-writers` / `capture-drawers` / `obsidian-drawers` | bold + legacy | bold               | bold forms; ALL-CAPS-with-notes refused |
| After `vault-migration` → `cutover`                                             | bold only     | bold               | ALL-CAPS is ordinary content            |

Writers never convert an existing header. A legacy `- NOTES` stays as it is until
`rename-drawer` or the migration converts it, even when a capture files into it.

### 4. Outside text never becomes a drawer

Today `gcal/description/bullets.rs` defuses an ALL-CAPS description line by wrapping it
in `**…**`. Under the bold marker, that exact branch would manufacture drawers. So:

- **The helper.** A shared `defuse_drawer_header(body)` sits beside the requirement
  checkbox escape in `task_text_escape`. When imported or pasted text would become a
  drawer header in a direct-child position under the union recognizer, it writes the
  title as `__title__`. That covers both ALL-CAPS and whole-bold lines.
- **Where it applies.** It replaces gcal's `**…**` defuse, and the check runs after
  inline rendering, so a description line that is one bold span (`<b>Agenda</b>`) lands
  as `- __Agenda__`. The same rule applies in gkeep, Gmail, and highlights child text
  and in `%c` clipboard notes.
- **The contract sentence.** "Pasted or imported text never becomes a requirement or a
  drawer."
- **Re-sync.** gcal's span hash covers rendered bodies, so untouched Google spans
  re-render on their next sync. That is the intended behavior.
- **After cutover.** `cutover` narrows the helper to whole-bold lines, because ALL-CAPS
  is just text by then.

### 5. Capture: one place resolver, drafts that mirror the block

**One place resolver** lives in a new `capture_language/place.rs`. It has three scope
adapters: `TaskDrawer`, `ProjectSection` (a `^prj` note's `##` sections), and
`NotebookSection`. Records can join later. It brings these together:

- **One lexer.** It serves the bullet-start and glued `#`. The two `scan_leading`
  functions in `drawer_bullet.rs` and `section_bullet.rs` are already identical, and
  they merge.
- **One ladder.**
  1. A quoted exact title.
  2. A key match on the selector's key forms.
  3. A unique slug, in the scopes that use it today.
  4. Ambiguity, naming the lines.
  5. Reserved or structural refusal.
  6. Near-miss refusal, teaching the quoted override.
  7. Create.
- **One create-title rule.**
  - A quoted literal is kept verbatim, with whitespace collapsed.
  - An unquoted slug takes the catalog's preferred explicit spelling, in scopes with a
    catalog (TaskDrawer and ProjectSection).
  - Otherwise it takes the notebook rule (`format_notebook_section_create_title`): words
    with typed capitals keep them, and the rest get Title Case, so `#API-keys` gives
    `API Keys` and `#git` gives `Git`.
  - The creation row always previews the exact title that will land.
- **Scope policies.** These carry today's deliberate differences unchanged: near-miss
  thresholds (drawer prefix or edit distance 1, notebook ≤ 2), reserved sets, nouns, and
  catalog use. Converging them is a later, explicit vector change.

Two boundaries hold. Shared selection never unifies mutation geometry: writers stay per
destination. And the dead `select_drawer*` / `match_drawer` / `suggest_drawer` helpers
go.

**Draft forms**, extending [[decisions:capture-bullets-mirror-the-task-block]]. A
"container position" is a first-level bullet under a new task, a new project note, a
note-less `@route+id`, a `%n` notebook seed, or a record capture.

| Draft                                                                  | Writes                                                                                                                                                                                                                                                                                                |
| ---------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `- **Future Work**` + nested notes, under a task                       | That drawer, with the title verbatim. It merges by key into an existing drawer, which keeps its spelling, and the preview names it.                                                                                                                                                                   |
| `- **Future Work**` with no nested notes                               | Refused with `empty_drawer_bullet`: "`- **Future Work**` names a drawer, so nest its notes beneath it (`  - …`); write `__Future Work__` for a bold note".                                                                                                                                            |
| `- #future-work` / `- #"Q1: Goals"` + nested                           | The shortcut, through the ladder. Accepting a menu row rewrites the bullet to `- **Title**`, plus `\n  - ` when no nested bullet follows.                                                                                                                                                             |
| `@foo+bar#api-design note` / `#"API Design"` / `--drawer "API Design"` | The same ladder. Creation preserves case.                                                                                                                                                                                                                                                             |
| Project-note capture `- **Design**` + nested                           | `## Design`, verbatim (today it is ALL-CAPS → Title Case).                                                                                                                                                                                                                                            |
| `%n` seed or record capture `- **Git**` + nested                       | A `## Git` seed section or that record section group, verbatim.                                                                                                                                                                                                                                       |
| Route-only `@nb` list `- **Git**` + nested                             | Unchanged: a note with detail. `- #Git` selects sections ([[decisions:notebook-routes-write-notes]]).                                                                                                                                                                                                 |
| `- FUTURE WORK` + nested, in any container position                    | Until `cutover`: refused with `caps_drawer_retired`, "`- FUTURE WORK` no longer makes a drawer: write `- **Future Work**`, or pick one from `- #`". The hint uses the existing drawer's spelling when the key matches. After `cutover`: ordinary content, with a warning only on a catalog-key match. |

The `*` blocking sigil does not collide: a `*` followed by `*` is already literal. The
completion invariant holds: the applied replacement, the parse, the preview, and the
commit all reach the same place with the same title.

**Wire changes.** All land in `capture-drawers` unless noted.

- **Grammar.** `GRAMMAR_REVISION` goes up by one (29 → 30 at plan time), as a `!`
  commit. ALL-CAPS-with-notes stops being a drawer draft, and quoted literals keep their
  case.
- **Spans.** A `drawer_marker` span covers each `**` delimiter. `drawer_bullet` covers
  the title.
- **Menu rows.** `capture-complete` drawer rows put `**Title**` in `replacement`.
- **Drawer objects.** In `capture-parse`, dry-run, and `bob capture-drawers`, they carry
  the verbatim `title`, the `key`, and an additive `marker`. `drawer-contract` adds
  `marker` to `bob capture-drawers`.
- **Preview lines.** `task_blocks[].lines[]` header lines carry an additive
  `drawer: {title, key, marker, empty}` object.
- **Catalog.** `DRAWER_CATALOG_VERSION` goes 2 → 3 in `drawer-contract`, with each
  spelling's marker recorded. `capture-drawers` applies the preference rule from design
  call 3.
- **Rewrite.** A new `capture-rewrite` rule, `expand_drawer_marker`, is described in §6.
- **Human output.** It prints verbatim titles in the section style and never uppercases.

### 6. Typing the marker

Bryan's `*` → `**|**`, scoped so it fires only where a drawer can start:

- **Where it fires.** Only on an empty bullet body in a drawer position. In Obsidian
  that is a direct child of a task line. In capture it is any container position, which
  bob decides through `capture-rewrite` `expand_drawer_marker`: the rule fires only when
  a hypothetical `- **x**` + nested would resolve as a container bullet there. Italic
  notes elsewhere are untouched.
- **Expansion.** `*` gives `- **|**`, whether or not Obsidian's `autoPairMarkdown`
  already inserted a closing `*`. It composes the same way the requirement `[` assist
  composes with `autoPairBrackets`. If prototyping still shows auto-pair friction, fall
  back to triggering on `- *` + Space, which conflicts with nothing because CommonMark
  forbids a space after an opening italic `*`.
- **Smart Enter.** Enter with the caret at the end of the title, before the closing
  `**`, steps past it and opens a nested bullet one level deeper.
- **Overtype.** A `*` typed just before the closing `**` steps over both stars.
- **Atomic Backspace.** Inside an empty `**|**`, Backspace removes all four stars, back
  to `- `.
- **Undo.** Undo after the expansion restores `- *`. That is the path to a first-level
  italic note.
- **Never rewritten.** Paste, templates, and `bob capture` itself never rewrite.
- **Toggle Drawer** (Obsidian command). It wraps or unwraps a direct child bullet,
  refusing checkbox lines, managed lines, trailing-colon titles, and reserved names, as
  the requirement toggle does. Its hotkey is optional and must not conflict.

### 7. Rendering: a quiet label, distinct from everything else

**Principle: the plugin adds a glyph and never changes the words.** True case, semibold,
everywhere. Never `text-transform: uppercase` or small caps, because they would hide the
very casing this change exists to show.

**Obsidian** lives in bob-ledger-tools, in a new fragment beside
`264-plugin-task-reqs.js`. It is attribute-only, like requirements, with zero extra DOM
nodes.

- **Attributes.** A Live Preview `Decoration.line` and a reading-view post-processor
  stamp `data-bob-drawer`, `-key`, `-count`, `-empty`, `-owner-state`, and the owner's
  `data-bob-type-hue` / `-state` (reuse the requirement ink pipeline).
- **Glyph.** The list bullet becomes the drawer glyph, drawn as a CSS mask in the
  owner's type-hue ink, softened toward `--text-muted`.
- **Title.** Weight 600. The native `**` reveal at the caret stays, with no custom
  concealment.
- **States.**
  - A folded drawer shows a muted, tabular `· n` after its title.
  - An empty drawer draws a hollow, dashed glyph and a muted title.
  - Under a closed owner (`[x]` or `[-]`), the glyph and title desaturate.
- **Labels.** The tooltip and accessible label name the thing: "Drawer · Future Work · 2
  notes · in Upgrade Postgres".
- **Setting.** A "Drawer marks" toggle (default on) sits beside the requirement marks
  setting.
- **Shape grammar.** Square for a task, ring for a Task Link, diamond for a requirement,
  dot for a note, the glyph for a drawer. Each is distinct in silhouette and in
  grayscale.

> [!decision] drawer_glyph = tray The glyph is an open tray: a wide, shallow ⊔ about
> 0.95em across, with a short inner shelf line. Its stroke weight matches the
> requirement glyph at the same font size. The Mac draws the outline SF Symbol `tray`
> (not `tray.fill`).

> [!decision] drawer_glyph = drawer_front The glyph is a drawer front: a rounded
> rectangle about 1.5:1, with a short centered handle bar and no fill, so it never reads
> as a checkbox. The Mac draws SF Symbol `cabinet`.

**Mac** (bob-mac-capture):

- **Draft editor.** The `drawer_bullet` title is semibold in the section color family.
  The `drawer_marker` delimiters are dimmed to the tertiary label color and go to full
  ink while the caret touches them. They are never hidden, which avoids the `req_marker`
  caret defect recorded on `bob-cli-on` (note #6).
- **Preview stack.** A line that bob annotates with `drawer` renders as the chosen SF
  Symbol plus the semibold verbatim title, with no `**`. An empty drawer is muted.
- **Menus and capsules.** Both show verbatim titles. Drop the two `.uppercased()` calls
  in `CaptureTransferPresentation.swift` and `CaptureMergeInPresentation.swift`.

**CLI and cards.** Verbatim titles in the section style. Requirement and successor cards
carry titles through unchanged.

**Plugins off, GitHub, mobile.** A bold header above its notes: the honest fallback, and
better than shouting.

### 8. Promotion, reversal, rename, and what "lossless" means here

**The DH contract** is Rust-owned; its vector group is `DH` in
`tests/fixtures/task_drawers/vectors.json`.

- **Drawer to heading.** `heading_for_drawer` maps a bold drawer to its title verbatim,
  so `- **API Design**` becomes `## API Design`. A legacy ALL-CAPS drawer keeps today's
  Title Case until `cutover`, because its intended case is unknowable.
- **Heading to drawer.** `drawer_for_heading` maps `## X` to the verbatim title. A
  heading the recognizer cannot hold is refused with a named reason, never dropped
  silently: `not_a_drawer_title` (contains `*`, ends with `:`, or has no alphanumeric)
  or `reserved`.
- **Round trip.** For every valid title T, the round trip returns T byte for byte. A
  pure round-trip test asserts it on its own, outside the promotion transformation.

**Promotion and reversal** live in bob-navigation-hotkeys and land after `.12`. They
pass the DH twin:

- Bold drawers become verbatim `##` sections, and the notes drawer becomes `## Notes`.
- `## Title` reverses to `- **Title**`.
- Reversal stops refusing mixed-case and punctuated headings.

**rename-drawer:**

- **NEW.** It is written verbatim: a bold header, or a verbatim `##`.
- **Case-only respell.** It becomes legal when the keys are equal. It converts a legacy
  header to bold with NEW's spelling, or respells a bold header. Today
  `rename-drawer 'API DESIGN' 'API design'` exits 2 with `same_drawer`.
- **Inventory mode.** A read-only inventory lists:
  - every legacy ALL-CAPS drawer (open, closed, and `done/`);
  - every whole-bold direct child that is now an empty drawer, such as old gcal
    `**WHY?**` lines;
  - bold trailing-colon leads, as information only.
- **Merges and duplicates.** Distinct-name merges stay explicit. Duplicate identities
  are reported, never coalesced silently.

**Lossless, honestly.** This slice delivers title isomorphism for the bodies it already
supports: `- **API Design**` ⇄ `## API Design`, in both directions, with refusals
instead of drops. It does not deliver the universal project ⇄ task representation. The
gaps the research lists stay open, such as generated status groups inside `## Tasks`,
frontmatter, prose, `###`, and merged-note provenance. They need their own spec.

### 9. Rollout gates and migration

**The host-currency script.** `scripts/dev/bob-host-currency` (created in
`reader-rollout`) takes a set of commits and plugin markers. It:

1. Finds the earliest `v*` release tag that contains every commit.
2. Asks athena, apollo, and the Mac for `bob --version` over Tailscale SSH, using the
   `tailnet.md` reference memory, with a 10 s timeout.
3. Greps each deployed vault plugin bundle for its marker.
4. Prints one colored table and exits 0 only when every row is current. An unreachable
   Mac reports `unknown`.

**Gates that use it.**

- **Readers.** `reader-rollout` checks the `drawer-contract` commits and the
  `plugin-readers` markers.
- **Writers.** `vault-migration` step 0 checks the `bob-writers`, `drawer-rename`, and
  `capture-drawers` commits and the `obsidian-drawers` markers.

**When a check fails.**

- If no release tag contains the commits yet, wait for the train with `/sase_monitor`.
  Poll with a bounded timeout of at most 36 h.
- If a host is behind, propose its update through `/sase_gate`, using the install path
  `docs/guides/install-a-tagged-release.md` describes for that host.
- If the Mac is offline, ask Bryan through `/sase_gate` to confirm the Mac's version,
  and record his answer.

**Migration.** The census taken on 2026-11-22 counted about 7 legacy task drawers:
`NOTES` (several, some in `done/`), `NOTES:`, `OPTIONS:`, `MOTIVATION`, and `DONE`. The
`bob-cli-on` NOTES filing may have added more before `bob-writers` landed. For each
inventory line the migration proposes either `rename → <spelling>` or `release`:

- NOTES-keyed lines are proposed as `Notes`, the role's spelling.
- Others get a suggestion that Bryan confirms or edits: `Options`, `Motivation`.
- `DONE` is proposed for `release` unless Bryan says it is a drawer.
- There is no automated recasing, ever. Bryan supplies every non-role spelling in the
  gate.

## Phases

### drawer-contract — recognizer, model, readers, outside text (bob-cli; plans first)

This phase is `large`, so plan it before implementing.

0. **Preconditions.**
   - Run `sase bead read bob-cli-on -r "<why>"`. If it is not `CLOSED`, change nothing.
     End with an incomplete status that names its open phases.
   - Read epic notes #2 and #4, the `^prj` NOTES home that duplicates `## NOTES` at EOF
     with tab-indented children. If that bug survived the epic, fix it first:
     - Route the `^prj` notes home through `project_h2_drawers`, keyed by `NOTES_KEY`.
     - Write a column-0 `## Notes` with column-0 bullets, and never a duplicate heading.
     - Add RL and RW `^prj` vectors.
   - If the docs contract tests from epic notes #1 and #5 still fail on master, fix them
     here, following the `bob-cli-hs` precedent. This phase registers a new contract
     page in the same pinned lists.
1. **Recognizer and model** (§1, §2).
   - Add `recognize_drawer_header`, `can_create_drawer`, and `is_legacy_drawer_title`
     (today's grammar, renamed).
   - Switch `drawer_key` to the Unicode fold, extend the reserved set, and add `marker`
     plus byte ranges on `Drawer`.
   - Split `NOTES_KEY` from `NOTES_TITLE`. Writers keep today's spelling in this phase.
   - Decouple `is_pomodoro_name` first.
2. **Retarget every vault reader** to the dual-read recognizer:
   - `capture_drawers::drawers` and the catalog scan;
   - `task_children`, `task_reqs`, and the block-layout zones and tidy;
   - the writers' parent checks and `task_merge` containers;
   - `task_drawer_rename` lookups;
   - every other `is_drawer_title` caller. Grep for it, and for `- NOTES`,
     `NOTES_TITLE`, and `to_ascii_uppercase` near drawer code.

   Capture draft parsing does not change here; `capture-drawers` owns it. Bump
   `DRAWER_CATALOG_VERSION` to 3 and record each spelling's marker.

3. **Outside text** (§4). Add `defuse_drawer_header`, flip the gcal defuse to `__…__`,
   and apply it in the gkeep, Gmail, highlights, and `%c` child text. Prove that no
   imported ALL-CAPS or whole-bold line yields a drawer, using cases drawn from the DV
   vectors.
4. **Vectors.**
   - Create `tests/fixtures/task_drawers/{vectors.json,README.md}`. Group `DV` covers
     recognition: bold, legacy, empty, trailing colon, `__…__`, `*` in the title,
     trailing text, `^id`, reserved and structural names, managed lines, checkbox,
     fences, depth, Unicode folds, and case twins. Group `DH` covers the title mapping
     (§8).
   - Add bold-drawer reading cases to the RL, RQ, and SP fixtures (zones,
     `req_in_drawer`, NOTES by key) without changing any writer expectation.
5. **CLI.** `bob capture-drawers` JSON gains an additive `marker`. Human output stays
   verbatim.
6. **Contract page.**
   - Create `docs/task-drawers.md` (§§1–4, §8) and register it in
     `docs/develop/contracts.md`.
   - Update the `tools/docs` contract tests' registry count, llms.txt list, and README
     snapshot.
   - Link the page from `docs/capture.md`, `docs/task-requirements.md`, and
     `docs/reference/vault-format.md`.
7. **Gates and commit.** Run `just check`, `just docs-gen`, and `just docs-check`. The
   commit is a `feat` (readers only, additive JSON).

### plugin-readers — one predicate twin, dual-read readers (bob-plugins; plans first)

This phase is `large`, so plan it before implementing.

1. **One twin.** Write one recognizer fragment (recognize, key, legacy flag, DH helpers)
   and copy it byte for byte into block-id-prompt, bob-ledger-tools,
   bob-navigation-hotkeys, and task-status-cycler.
   - It replaces every drawer predicate: the four `taskReqIsDrawerTitle` copies (this
     ends the ledger-tools regex bug, where `\/-?` is a character range that accepts
     `NOTE: X` and `KEY:: VALUE`), `isTaskSplitDrawerTitle`, `taskBlockIsNotesDrawer`,
     and any other ALL-CAPS check.
   - Add a twin-copy check script modeled on `scripts/test-task-reqs-twin-copies.cjs`.
2. **Vectors.** Copy the DV/DH fixtures and the updated RL, RQ, and SP fixtures byte for
   byte, each with an "update bob-cli first, then copy" README. Every twin passes them.
3. **Dual-read readers.**
   - The layout twins find the notes drawer by key.
   - The requirement guards and structural-container checks recognize both forms.
   - The task-split drawer scope uses the recognizer's title, not the raw parent body.
     This fixes the `**…**` header that `257-task-split-apply.js` (`parentDrawer`) would
     otherwise show.
   - Picker titles and subtitles stay verbatim.
   - The promotion read path (`parseProjectSectionBulletTitle`, wherever `.12` left it)
     accepts bold drawers verbatim and legacy drawers with today's Title Case.
4. **Writers stay unchanged.** That covers the NOTES writers, reversal, and the seed.
5. **Ship.** Run `npm run build` and `npm test`, then run `bob plugins sync -p <id>` for
   each changed plugin. Record the deployed bob-plugins commit and a marker string for
   each bundle on the phase bead, for `reader-rollout`.

### reader-rollout — every host reads bold first (bob-cli dev script; gated)

1. **The script.** Write `scripts/dev/bob-host-currency` (§9) with its usage text. Pass
   `just check-scripts`, and commit it as `chore`.
2. **Run it.** Check the `drawer-contract` commits and the `plugin-readers` markers.
3. **Handle a failure** as §9 describes: monitor the train, gate the host updates, and
   get Bryan's confirmation through a gate when the Mac is offline. Bryan's confirmation
   stands in for an unreachable host.
4. **Close.** Store the passing table as a SASE artifact and close the phase. Every
   writer phase depends on this gate.

### bob-writers — bold headers from every bob-cli writer (bob-cli)

1. **One writer helper.** It renders `{indent}- **{title}**`.
   - `ensure_drawer` uses it on the task path.
   - The `^prj` path writes `## {title}` verbatim.
2. **Notes writers.** `task_block_layout` (ensure, scoop, insert, toggle) and
   `bob task tidy` write `- **Notes**` via `NOTES_TITLE = "Notes"`.
3. **Audit the remaining writers.** Grep for every other header writer, including
   `task_merge` containers and requirement carry and split landings. Captures already
   reach drawers through `ensure_drawer`, so until `capture-drawers` lands, an ALL-CAPS
   draft title is written as typed inside the bold marker.
4. **Existing headers stay untouched** (§3). Add a test for it.
5. **Vectors.** Flip the RL, RW, and SP writer expectations to the bold header.
   `obsidian-drawers` re-copies them.
6. **Docs.** Update `docs/task-requirements.md` (Notes), `docs/task-drawers.md`
   (writers), and `docs/reference/vault-format.md`.
7. **Gates and commit.** Run the gates. The commit is `feat(drawers)!`, because vault
   bytes and preview lines change.

### drawer-rename — respell, verbatim NEW, legacy inventory (bob-cli)

1. **Read first.** Read `cli_rules.md`. Keep options alphabetical, add short aliases,
   and color the human output.
2. **NEW.** Write NEW verbatim (§8). Refuse a NEW that `can_create_drawer` rejects, with
   its reason.
3. **Case-only respell.** Allow it when the keys are equal. It converts legacy headers
   to bold and respells bold headers. Repair heading links on `^prj` sections, as today.
4. **Inventory.** Add a read-only inventory mode (for example `-l, --legacy`, taking no
   positionals) with the §8 contents, in human and JSON form.
5. **Keep today's guarantees.** Dry run by default, one commit on `--apply`, catalog
   invalidation, and idempotence (a second run reports nothing).
6. **Tests, docs, and commit.** Update `docs/task-drawers.md` and the CLI reference. The
   commit is `feat(task)!`, because NEW's casing rule changes.

### capture-drawers — one place resolver, bold drafts (bob-cli; plans first)

This phase is `large`, so plan it before implementing.

1. **Extract `place.rs`** (§5) as a behavior-preserving refactor first. The existing
   drawer, `^prj` section, and notebook section suites pass unchanged.
2. **Switch casing** to the create-title rule and the catalog preference. Quoted
   literals and `--drawer` become verbatim.
3. **Draft forms** (§5 table) in all four container contexts:
   - task drawers;
   - project-note capture (`capture_project_note.rs`);
   - `%n` seeds (`section_bullet.rs`, `all_caps_as_section`);
   - record groups (`records/render.rs`).

   Retire `is_project_section_title` and `format_project_section_title` from these
   paths. The legacy helper survives only for the DH legacy mapping until `cutover`.

4. **Refusals.** Add `caps_drawer_retired` and reuse `empty_drawer_bullet`, with the §5
   copy. Leave `req_in_drawer` unchanged.
5. **Wire** (§5).
   - Add `drawer_marker` spans and `**Title**` menu replacements.
   - Add the preview `drawer` line annotation and `marker` on drawer objects.
   - Add the `expand_drawer_marker` rewrite rule (§6), which mirrors
     `expand_req_marker`.
   - Bump `GRAMMAR_REVISION`.
6. **Tests.**
   - Completion invariant tests: the menu replacement, `capture-parse`, the dry-run
     preview, and the commit agree for every row kind.
   - Childless and ALL-CAPS refusals.
   - Merge-by-key into legacy and bold drawers.
   - Non-ASCII and punctuated titles (`C++`, `Q&A`, `Café`, `Q1: Goals`).
7. **Docs.** Update `docs/capture.md` (drawer bullets, sub-bullets, project notes,
   section bullets), `docs/reference/capture-syntax.md`,
   `docs/reference/json-outputs.md`, `docs/projects.md`, `docs/notebooks.md`,
   `docs/records.md`, and `docs/task-drawers.md`. Regenerating Mac fixtures belongs to
   `mac-drawers`.
8. **Commit.** It is `feat(capture)!`.

### obsidian-drawers — render, type, write (bob-plugins; plans first)

This phase is `large`, so plan it before implementing.

1. **Rendering** (§7, with the glyph from `drawer_glyph`) in bob-ledger-tools.
   - Use new fragments, the CSS masks, and the setting toggle, in Live Preview and
     reading view, with light and dark themes.
   - Do a taste pass with real-Obsidian screenshots (light and dark; open, folded,
     empty, and closed-owner drawers next to requirements and Task Links) and store them
     as SASE artifacts.
2. **Typing** (§6). Put the assists beside the requirement `[` assist from
   `bob-cli-on.10`, and add the Toggle Drawer command.
3. **Writers.**
   - The layout-twin notes writers write `- **Notes**`. Re-copy the RW vectors.
   - Promotion and reversal pass the DH twin (§8). `formatProjectSectionTitle` and
     `formatProjectReversalSectionTitle` survive only on the legacy path until
     `cutover`.
   - The receipts name verbatim titles.
4. **Tests.**
   - Unit tests over the vectors.
   - A real-Obsidian e2e script modeled on `obsidian-task-reqs-e2e.cjs`: render, type,
     toggle, and the promotion → reversal round trip of `API Design`, `C++ Notes`,
     `Q1: Goals`, and `Café`.
5. **Docs and ship.**
   - Update the bob-plugins docs, plus bob-cli `docs/projects.md` (promotion) and
     `docs/task-drawers.md` (rendering and typing).
   - Run `npm test`, then sync the changed plugins.

### mac-drawers — render and type in Bob Mac Capture (bob-mac-capture; plans first)

This phase is `large`, so plan it before implementing.

1. **Decode.** Decode the `drawer_marker` spans and the `drawer` preview annotation,
   additively. Older fixtures keep decoding.
2. **Render** (§7). Draw the editor title and dim the delimiters, with full ink while
   the caret is adjacent. Draw the preview glyph and semibold title using the
   `drawer_glyph` symbol. Drop both `.uppercased()` calls, and show verbatim menu and
   capsule titles.
3. **Typing** (§6). Drive the `*` expansion through bob's `expand_drawer_marker`. Smart
   Return, overtype, and atomic Backspace are span-driven keystroke handling, like the
   requirement marker's Backspace.
4. **Fixtures.** Regenerate the drawer fixtures from bob, for example
   `drawer-capture-plain-child.json`.
5. **Docs.** Update `docs/protocol.md` and `docs/capture-previews.md` in
   bob-mac-capture.
6. **Gates.** Run the repo gates, and watch exact-SHA macOS CI to green with
   `/sase_monitor`.

### vault-migration — reviewed migration of legacy drawers (vault; gated)

0. **Preparation.**
   - Read the `obsidian.md` reference memory.
   - Run the writer gate (§9). Every host must write bold before any legacy drawer is
     retired.
1. **Inventory.** Run the `drawer-rename` inventory against `~/bob` and store it as an
   artifact. Compare it with the plan-time census (§9) and explain every difference.
2. **Proposal.**
   - Build a per-line proposal table: `rename → <spelling>` or `release` (§9).
   - List the whole-bold empty drawers and trailing-colon leads for review. For each,
     the choice is to keep it as a drawer or convert it to `__…__` (a preimage-checked
     edit).
3. **Gate.** Dry-run every rename and store the outputs. Propose the apply batch through
   `/sase_gate`, with Bryan supplying or confirming each spelling and release. The
   gate's `next` successor verifies the outcome.
4. **Verify.** The verifier:
   - re-runs the inventory and confirms that only released lines remain;
   - stores the approved rename/release record as a SASE artifact, which `cutover`
     reads;
   - confirms that a second dry run makes no changes.
5. **Commits.** Vault edits follow the vault's normal git sync.

### cutover — retire ALL-CAPS recognition (bob-cli + bob-plugins)

0. **Precondition.** The live inventory must be a subset of the migration record's
   released set. Otherwise, stop and report.
1. **bob-cli.**
   - Remove `is_legacy_drawer_title` and the `legacy_caps` marker.
   - Remove the legacy DH mapping and `format_project_section_title` wherever it only
     served the legacy path.
   - Remove the legacy DV cases. Re-express them as "is a note" cases.
   - Narrow `defuse_drawer_header` to whole-bold lines.
   - Turn `caps_drawer_retired` into the catalog-key-match warning (§5).
   - Bump `GRAMMAR_REVISION`, and update the contract page and the capture docs.
2. **bob-plugins.** Re-copy the vectors. Remove the legacy branches from the twin,
   promotion, and reversal. Run the tests and sync.
3. **Commits.** Make a `feat(drawers)!` commit in each repo.

### memory — records (bob-cli memory)

Use `/sase_memory_write`, then run `sase memory init`. Regenerate
`docs/reference/glossary.md` with `just docs-gen`, and run `just docs-check`. Describe
what actually shipped, including the `drawer_glyph` answer, and cite this epic's commits
and the research ref.

> [!decision] memory_drawer_record Add the decisions-web record
> `drawers-keep-their-names`, titled "Drawers Keep Their Real Names", in the records'
> shape:
>
> - **Applies to:** bob-cli, bob-plugins, Bob Mac Capture, and the vault.
> - **Claim:**
>   - A drawer is a direct non-checkbox child of a task whose whole body is one bold
>     span, `- **Title**`.
>   - Its title keeps its authored spelling everywhere, and its identity is the
>     case-folded key.
>   - Empty drawers are recognized, but capture never creates one.
>   - Imported or pasted text never becomes one (`__…__`).
>   - Capture's `#` shortcuts complete to `**Title**`.
>   - ALL-CAPS is not structure.
> - **Why:** true case and punctuation, an honest plugins-off fallback, a lossless
>   `## Title` mapping, and Bryan's own `**Title Text**` request.
> - **Rejected alternatives:**
>   - `#tags` (no spaces, globally indexed, they mark the task);
>   - a stored bullet-start `#`;
>   - `§`;
>   - `- ##` list headings;
>   - a trailing colon, a field, a comment, or a callout;
>   - an emoji prefix.
> - **Cost:** bold gains structural meaning (fenced by the recognizer and the `__`
>   escape), the grammar break, and the dual-read window and migration.
> - **Reopens when:** Bryan wants nested `###`-style headers, or bold leads keep
>   colliding.
>
> Then mark `decisions:capture-bullets-mirror-the-task-block` with
> `metadata.status: superseded-in-part` and
> `superseded_by: decisions:drawers-keep-their-names`. Add a back-link line stating that
> a drawer bullet is now `- **Title**` and that the menu inserts `**Title**`.

> [!decision] memory_drawer_record = no Leave both records unchanged.

> [!decision] memory_glossary Rewrite two glossary strands for what shipped.
>
> - **`glossary:task-drawer`:**
>   - the `- **Title**` definition, true case, and the key;
>   - the examples `Notes`, `Future Work`, `@foo+bar#notes`, and `- #future-work` →
>     `- **Future Work**`;
>   - the retired ALL-CAPS form;
>   - a one-line rendering note.
>
>   Keep the task drawer distinct from the property-card drawer, and keep the `^prj`
>   section, requirement, and managed-line distinctions.
>
> - **`glossary:project-note-capture`:** replace the ALL-CAPS sentence. A `- **Title**`
>   bullet with nested notes becomes a verbatim `## Title` section.

> [!decision] memory_glossary = no Leave both glossary strands unchanged.

## Acceptance evidence

The land agent checks these across the epic:

- **Vectors.** The DV, DH, RL, RW, RQ, and SP fixtures are byte-identical across bob-cli
  and every plugin twin, and every twin-copy check is green.
- **Gates.** The `reader-rollout` and writer-gate tables are stored as artifacts. No
  writer phase merged before the reader gate closed.
- **Scratch-vault walkthrough** with the installed release:
  - `@foo+bar` + `- **API Design**` + nested writes `- **API Design**`.
  - The `- #api-design` menu row writes `**API Design**` when the catalog knows it.
  - A childless bold bullet and an ALL-CAPS-with-notes draft are refused with the §5
    copy.
  - A gcal `<b>Agenda</b>` description line lands as `- __Agenda__`.
  - `bob capture-drawers` prints verbatim titles.
- **Round trip.** Obsidian promotion → reversal returns `API Design`, `C++ Notes`,
  `Q1: Goals`, and `Café` byte for byte.
- **Screenshots** of the Obsidian rendering (light and dark) and the Mac rendering.
- **Migration.** The migration record and the final inventory (released lines only).
  After `cutover`, `rg` finds no ALL-CAPS drawer predicate outside history.
- **Bumps.** Grammar and catalog bumps landed in `!` commits. `CHANGELOG.md` and the
  Cargo version were untouched.
- **Follow-ups.** Phase `PROPOSED FOLLOW-UP:` notes are triaged into task beads where
  warranted.
