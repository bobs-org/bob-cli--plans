---
tier: epic
title: Drawer bullets and several notes per @file+id capture
goal: "In Bob capture, a `#` typed right after a top-level bullet's `- ` opens a drawer
  menu that expands to the drawer's ALL-CAPS title, and `@file+id` with no note text on
  its line files every top-level bullet as its own note, with drawer bullets merging
  into that task's drawers. Completion, `capture-parse`, preview, and commit always
  agree.

  "
decisions:
  continue_bullet:
    ask:
      Should accepting a bullet drawer row also open a nested `  - ` bullet when the
      drawer has no notes yet?
    default: true
    why: Picking FUTURE WORK lands the caret where its first note goes
    answer: true
  memory_decisions_record:
    ask:
      Add a capture-bullets decision record and mark capture-sigils-say-where-what-how
      superseded in part?
    memory:
      - decisions
    default: true
    corrects:
      "A `#` standing alone at an item edge names a [[task-type]]: `#fix` types a new
      task or retypes the selected existing task in the same transaction."
    answer: true
  memory_drawer_glossary:
    ask:
      Update the Task Drawer glossary entry for bullet-start drawer names and drawer
      bullets?
    memory:
      - glossary:task-drawer
    default: true
    corrects: a free-standing `#fix` at an item edge types the task
    answer: true
phases:
  - id: drawer-bullets
    title: Drawer bullets in the capture grammar
    depends_on: []
    size: medium
    description:
      "drawer-bullets: make a first-level bullet's leading `#` a drawer shortcut instead
      of a type marker, recognize drawer bullets on new tasks and new project notes,
      render and merge them through the commit resolver, report them in capture-parse,
      and bump the grammar to 4."
  - id: note-bullets
    title: Several notes on one existing task
    depends_on:
      - drawer-bullets
    size: medium
    description:
      "note-bullets: let a note-less `@route+id` (or `@route+id#drawer`) item file each
      first-level bullet as its own note, file drawer bullets into the target's drawers,
      and report notes and drawers in JSON and human output."
  - id: bullet-menu
    title: Bullet drawer menu, docs, and memory records
    depends_on:
      - note-bullets
    size: medium
    description:
      "bullet-menu: open the drawer completion menu at a first-level bullet's `#`,
      replace with the bare ALL-CAPS title (plus the optional nested-bullet
      continuation), finish the docs, and write the accepted memory records."
  - id: mac-capture
    title: Bob Mac Capture presentation
    depends_on:
      - note-bullets
      - bullet-menu
    size: medium
    description:
      "mac-capture: paint drawer-bullet spans, summarize multi-note sub-bullet previews
      from the new `notes`/`drawers` fields, and prove the bullet drawer menu end to end
      with real-bob fixtures."
proposed_by: bbugyi200.apollo.c0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-01 12:53:05
status: wip
---

- **PROMPT:**
  [prompts/202611/capture_drawer_bullets.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/capture_drawer_bullets.md)

# Plan: Drawer bullets and several notes per `@file+id` capture

## Why

Today a `#` right after a capture bullet's `- ` opens the **task-type** menu, and an
`@file+id` item whose own line has no note text is refused ("task text is required")
even when bullets follow it. Bryan wants the drawer menu at that position, with the
selected drawer expanding to its ALL-CAPS bullet (`FUTURE WORK`). He also wants an
`@file+id` followed by bullets to add several notes to one task in one operation.

Design principle: **the draft reads exactly like the task block it produces.** A
first-level bullet that lands directly under a task _is_ that task's child. An ALL-CAPS
title with notes nested beneath it is a drawer (the vault's own definition of a drawer).
Every other bullet is a note. `#name` is only a typing shortcut that completes to the
title. Commit-time resolution reuses the existing drawer resolver, so the menu, the
`capture-parse` spans, the `--dry-run` preview, and the commit never disagree.

## Shared design contract

### Vocabulary

- **First-level bullet**: an authored child line at column zero (`- `, `* `, or `+ `).
  **Nested bullet**: a two-space child of the nearest first-level bullet. The two-level
  authored grammar is unchanged (see Non-goals).
- **Under-task position**: the item's first-level bullets become direct children of a
  task. This covers:
  1. A new task: plain text, `@route`, `@route^id`, and the new-task `@route:id` forms.
  2. A new project note (`@route^id@`, `@route:id@`), whose existing writer already
     turns ALL-CAPS bullets into `## Title Case` sections and merges repeats.
  3. An existing task named by `@route+id` (leading, or trailing on any line) when the
     item's parent line carries **no note text** and the marker has **no glued
     `#drawer`**. This one lands in phase `note-bullets`.
- **Not under-task**:
  - `@route+id <note>`: bullets nest under the note, unchanged.
  - `@route+id#drawer` with no note text: bullets land inside that drawer (phase
    `note-bullets`).
  - `%n` Pomodoro notes and `@route#Section` bullets.
  - Pomodoro-close Work Log bullets: a separate grammar, untouched by this epic.

### `#` at a first-level bullet start (lexical, every position)

A `#` that begins a first-level bullet's body names a drawer. It is never a type marker.
Type markers keep working at every line's trailing edge, at a nested bullet's start, and
on the parent line. The opening predicate matches today's type marker: an ASCII letter,
`"` or `“`, or a bare `#` followed by whitespace or end of line. `#` followed by a
digit, `##`, and `#-` stay prose, and `\#` keeps a literal `#`. Slugs and quoted
literals use the glued selector's lexer (`is_drawer_slug`, `lex_exact_name_after_hash`,
`normalize_drawer_literal`).

| Draft line (first-level)                      | Outcome                                                                                                                                                                                                 |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `- #`                                         | Incomplete. `capture-parse` reports `needs: ["drawer"]` with an `interactive_placeholder` span, the same shape as glued `@route+id#`; `bob capture` refuses with copy teaching `- #future-work` or `\#` |
| `- #future-work` + nested bullets, under-task | **Drawer shortcut**: the line is the drawer bullet `- FUTURE WORK` (title chosen by the resolver; slugs take the catalog's most-used spelling)                                                          |
| `- #future-work` with no nested bullet        | Incomplete: `capture-parse` reports `needs: ["drawer_notes"]` with no error diagnostic, so the menu can stay open; `bob capture` refuses `empty_drawer_bullet` (capture never creates an empty drawer)  |
| `- #requirements Postgres 17`                 | Error `drawer_bullet_text`: a drawer bullet holds only its name, so nest the note beneath it; a task type goes at the end of a line (`… #fix`)                                                          |
| `- #requirements` when not under-task         | Error `drawer_bullet_position`, with copy per position (see Refusal copy)                                                                                                                               |
| `- #"bad:name"`                               | Existing `invalid_drawer`                                                                                                                                                                               |
| `- \#idea`                                    | Plain bullet text `#idea`                                                                                                                                                                               |

This is a breaking grammar change: `- #fix …` on a first-level bullet used to type the
item. Bump `GRAMMAR_REVISION` from 3 to 4 (`src/native/capture_language/mod.rs`). Bob
Mac Capture's gate is `grammar >= 3`, so it keeps working. Commit with `!` and a
`BREAKING CHANGE:` footer. Per the release-train decision, never touch the `Cargo.toml`
version or `CHANGELOG.md`.

### Drawer bullets

In an under-task position, a first-level bullet is a **drawer bullet** when, after
item-wide markers are stripped, its whole body is one of:

- a **drawer shortcut** (`#slug`, or `#"TITLE"` with straight or curly quotes), or
- an **ALL-CAPS drawer title** (`capture_drawers::is_drawer_title`, no checkbox) **with
  at least one nested bullet**. A childless ALL-CAPS bullet stays a plain bullet,
  exactly as today. Phase `note-bullets` adds one exception: it is refused when it
  repeats a drawer the target already has.

A drawer bullet's nested bullets are its notes. Resolution is the commit resolver, never
a parallel rule:

- A slug resolves through `resolve_drawer_for_write(.., exact = false, ..)`.
- A quoted shortcut or a literal title resolves exactly, so `- FUTURE WORK` ≡
  `- #"FUTURE WORK"`.
- On a `^prj` target, `resolve_h2_for_write` resolves against the note's `##` sections.

Drawer bullets therefore get the same refusals (`reserved_drawer`, `ambiguous_drawer`,
`drawer_near_duplicate` for slugs, `invalid_drawer`) and the same non-fatal
`drawer_near_catalog` warning as `@route+id#drawer`. Two drawer bullets with the same
key in one item merge into the first one, keeping the notes in draft order.

### Several notes on one task (phase `note-bullets`)

Given:

```markdown
- [ ] #task Upgrade Postgres [created::2026-07-31] ^bar
  - REQUIREMENTS
    - 2 GB RAM
```

the draft

```text
@foo+bar
- Ping Alice about the cutover
- REQUIREMENTS
  - Postgres 17 minimum
- FUTURE WORK
  - Try logical replication
- Benchmarks look fine
  - p99 under 40 ms
```

writes

```markdown
- [ ] #task Upgrade Postgres [created::2026-07-31] ^bar
  - REQUIREMENTS
    - 2 GB RAM
    - Postgres 17 minimum
  - Ping Alice about the cutover
  - FUTURE WORK
    - Try logical replication
  - Benchmarks look fine
    - p99 under 40 ms
```

How each kind of bullet lands:

- **Plain notes**: each plain first-level bullet and its nested bullets form one note.
  It lands at today's plain-child anchor: the end of the task block, before the earliest
  direct Schedule, Work, or Rotten Log, using the existing child indentation. Notes land
  in draft order.
- **Existing drawers**: a drawer bullet naming an existing drawer appends its nested
  bullets at that drawer's end (`drawer_insertion`). A childless literal ALL-CAPS bullet
  whose key matches an existing drawer is refused `empty_drawer_bullet` ("`REQUIREMENTS`
  is already a drawer on ^bar; nest the notes to add beneath `- REQUIREMENTS`"), so a
  duplicate header is never written.
- **New drawers**: a drawer bullet naming a missing drawer is created at the plain-child
  anchor with its notes, through the existing create-then-re-resolve path. A later
  bullet in the same item that names it merges in.
- **`^prj` targets**: drawers are the note's `##` sections, through
  `plan_prj_section_capture`. A missing section is created as `## Title Case`.
- **Same-line note**: `@route+id <note>` + bullets is unchanged. The bullets nest under
  the note, ALL-CAPS bullets stay literal, and a drawer shortcut there is
  `drawer_bullet_position`.
- **Glued drawer, no note text**: `@route+id#requirements` + bullets files each
  first-level bullet (with its nested bullets) as its own note inside `REQUIREMENTS`,
  creating the drawer once when it is missing. ALL-CAPS bullets there are plain text,
  and a drawer shortcut is `drawer_bullet_position`. The `drawer_text` need and the "add
  the note text" error now apply only when no first-level note exists.
- **Item-wide markers**: a separate `#type` token retypes the target, exactly as on a
  body-bearing sub-bullet today. `&`/`*`/`~` keep the target task as owner. `s:<N>`,
  `$<N>`, `%c`, and `--clip` are refused `multi_note_marker` because each one decorates
  a single note.
- **Unchanged forms**: a marker-only `@route+id` (placeholder rows only) stays Ensure
  Next, `@route+id!` with bullets keeps its refusal, and forced
  `--route/--task/--task-ref` mode is unchanged.
- **Atomicity**: every note plans against staged content inside the existing batch
  planner, so one refusal rolls the whole draft back.

### Output contract (additive JSON, phase `note-bullets`)

On a note-less `sub_bullet` item (the multi-note form), the item JSON:

- keeps `kind: "sub_bullet"`, with `text: ""` and `task_line: ""`, and omits
  `sub_bullets`;
- adds `notes`: one entry per note in draft order, each with `text`, `task_line`,
  optional `sub_bullets` (rendered like today's field), and `drawer`
  (`{title, key, created}` or null);
- adds `drawers`: every drawer the item filed into, in first-touch order, each
  `{title, key, created}`;
- sets `drawer` to that single entry when exactly one drawer received every note, and to
  null otherwise.

Batch-level `task_blocks` keeps its contract and naturally shows every added line. For
the example above, `notes` holds four entries (`Ping Alice…`, `Postgres 17 minimum` in
REQUIREMENTS with `created: false`, `Try logical replication` in FUTURE WORK with
`created: true`, and `Benchmarks look fine` with
`sub_bullets: ["\t- p99 under 40 ms"]`). `drawers` holds REQUIREMENTS then FUTURE WORK.

Human output mirrors the draft order:

```text
[dry-run] ok would capture  foo.md
  under [ ] Upgrade Postgres  ^bar · 4 notes
  - Ping Alice about the cutover
  REQUIREMENTS
  	- Postgres 17 minimum
  FUTURE WORK        new drawer
  	- Try logical replication
  - Benchmarks look fine
  	- p99 under 40 ms
```

Drawer titles use the existing cyan style and the existing `new drawer` tag. A glued
drawer item reads `under … ^bar · REQUIREMENTS · 2 notes`.

### Bullet drawer menu (phase `bullet-menu`)

- **Field**: the cursor sits inside a drawer shortcut token, including a bare `#`, at a
  first-level bullet start in an under-task position. The response:
  - uses context `drawer`;
  - sets `replacement` to the whole token (`#` through its end, including a partial or
    closed quoted literal);
  - sets `query` to the decoded name, plus `initial_selection: "none"` for an empty
    query;
  - carries a `target`.
- **Other positions**: a bullet-start `#` that is not under-task returns an empty
  success with a null context. It never opens the type menu; `capture-parse` explains
  why.
- **Rows** (shared `capture_complete/drawers.rs` machinery):
  - _Existing target_ (`@route+id`, no note text): `task`, `recent`, and `new` groups,
    with the glued menu's detail and consequence copy
    (`Adds after "2 GB RAM" in REQUIREMENTS`, `Creates FUTURE WORK under ^bar`). A
    `^prj` target uses its `##` sections.
  - _New task_: `recent` + `new` only. Consequence:
    `Starts FUTURE WORK on the new task`. `target` is
    `{route: "<destination>.md", block_id: null, text: <parent body>}`, where the
    destination is the item's route, global `@@route`, or `mac_inbox.md`.
  - _New project note_: `recent` + `new`, consequence `Creates ## Future Work`, `target`
    = the new note stem with `block_id: "prj"`.
- **Row replacement**: every row's `replacement` is the bare ALL-CAPS title
  (`FUTURE WORK`), never `#…`. A row is valid only when resolving that title exactly
  against the target yields the row's intent: the same existing drawer (by line) or a
  creation of that key. Otherwise the row carries the resolver's `refused` message
  (reserved, ambiguous, or a heading ALL-CAPS cannot spell, with copy pointing at
  `@route#Heading`). `cursor_after` is absolute.

> [!decision] continue_bullet When the token is the last thing on its line and the next
> physical line of the item is not a nested bullet, `replacement` is `TITLE\n  - `, so
> `cursor_after` lands on the fresh nested bullet. An unused placeholder row is ignored
> at commit. Shell completion never carries the continuation.

> [!decision] continue_bullet = no `replacement` is always the bare title.

- **Shell completion** (`shell.rs`) lists the titles in the existing `drawers` group.

### Refusal copy (authored by Bob; wording may be polished, codes may not)

- `drawer_bullet_position`:
  - under a same-line note: "`- #requirements` names a drawer, but these bullets nest
    under `<note>`; file the note into it with `@foo+bar#requirements <note>`, or put
    the note on its own bullet";
  - in a glued drawer item: "this item already files every bullet into REQUIREMENTS;
    drop `#requirements` from `@foo+bar` to use drawer bullets";
  - elsewhere: "only bullets that land directly under a task can name a drawer; write
    `\#` to keep a literal `#`".
- `drawer_bullet_text`: "`- #requirements` names a drawer, so it takes no text: nest
  `Postgres 17` beneath it (`- REQUIREMENTS` / `  - Postgres 17`); a task type goes at
  the end of a line (`… #fix`)".
- `empty_drawer_bullet`, for a childless shortcut: "`- #future-work` has no notes
  beneath it; nest them as `  - …` (capture never creates an empty drawer)".
- `multi_note_marker`: "`s:3` decorates a single note: put the note on the `@foo+bar`
  line to schedule it (`@foo+bar Call Morgan s:3`)". The same shape covers `$<N>` and
  `%c`.

## Phase `drawer-bullets`

Implement the lexical rule, drawer bullets on new tasks and new project notes, and
`capture-parse` reporting. Existing targets keep today's behavior in this phase: a
note-less `@route+id` with bullets is still refused.

1. **Lexer**:
   - Add one shared scanner (suggested: `capture_language/drawer_bullet.rs`) that
     classifies a first-level authored line's leading token as a drawer shortcut
     (complete, incomplete bare `#`, invalid, or escaped) with absolute spans. Execution
     (`item.rs`), the editor (`editor_parse.rs`/`editor_classify.rs`), and completion
     all use it.
   - In `type_marker.rs`, `scan_line_with_positions` must skip that token on first-level
     lines, even when it is also the trailing edge. As a result,
     `task_type_completion_at` no longer opens there.
2. **Model**: carry the shortcut on `AuthoredSubBullet` (for example
   `drawer: Option<DrawerSelector>`), so planning knows a bullet's spelling. Classify
   under-task positions from the resolved item kind.
3. **Execution parse** (`item.rs`): raise `drawer_bullet_text`,
   `drawer_bullet_position`, `invalid_drawer`, the incomplete bare `- #` refusal, and
   the lexical childless-shortcut `empty_drawer_bullet`, all before planning. Keep
   `orphaned_nested_bullet` and the placeholder rules unchanged. Today a lone
   `- #requirements` is eaten as a type marker and orphans its nested line; it must now
   be a drawer bullet.
4. **Planning** (`capture/plan.rs`, wherever new-task child lines render):
   - For new tasks, resolve each drawer bullet with an empty drawer list and the cached
     catalog (`load_drawer_catalog`, best effort as in `sub_bullet.rs`). Render the
     resolved title, merge repeated keys, refuse reserved keys, and push the existing
     `drawer_near_catalog` structured warning.
   - For new project notes, normalize shortcuts to the ALL-CAPS title before the
     existing project-note writer runs. That writer keeps its section conversion,
     merging, and structural-heading rules.
5. **`capture-parse`**:
   - Emit a `drawer` span (with `title`/`key` when valid, computed lexically like the
     glued span) on a shortcut. The Mac app already requests completion on `drawer`
     spans and the `drawer` need.
   - Emit a new additive `drawer_bullet` span (`title`, `key`) on a literal ALL-CAPS
     drawer bullet's text in under-task positions.
   - Emit the needs and diagnostics from the shared contract table, with
     `reserved_drawer` reported lexically as for glued selectors.
6. **Grammar 4**: bump `GRAMMAR_REVISION`, then update every `grammar: 3` assertion and
   doc mention.
7. **Docs**: `docs/capture.md`:
   - In "Grammar at a glance", add rows for drawer bullets and the shortcut.
   - Fix the `#` row and the "`#` is not one marker" table (add `- #requirements` at a
     first-level bullet start).
   - Add drawer bullets to "Authored sub-bullets".
   - In "Typing tasks with `#<type>`", say the first-level bullet start is a drawer
     position.
   - In the `capture-parse` reference, document the new span, need, and codes.

   Also update `docs/reference/json-outputs.md` (spans and needs), the
   `capture`/`capture-parse` `--help` text (`src/native/capture/cli.rs`,
   `src/native/capture_parse.rs`), then `just docs-gen` and `just docs-check`.

8. **Tests**:
   - Unit tests in `capture_language/tests/{grammar,draft,completion}.rs`:
     - lexer outcomes from the table;
     - no type completion at a first-level start, while nested starts still type;
     - `\#`, `#1`, and `##` stay prose.
   - CLI tests (a new `tests/cli/capture/drawer_bullets.rs`, registered in
     `tests/cli/capture/mod.rs`):
     - new task `- #future-work` + notes renders `- FUTURE WORK`;
     - catalog spelling for a slug;
     - repeat merge;
     - reserved refusal;
     - childless shortcut refusal;
     - `drawer_bullet_text`;
     - same-line note `drawer_bullet_position`;
     - the project-note shortcut becomes a `## Title Case` section;
     - `Plan trip` + a childless `- NYC` stays a plain bullet;
     - updated `capture_type_marker.rs` expectations for the retired bullet-start type.
9. **Gate**: `just check` and `just docs-check` pass.

## Phase `note-bullets`

Implement "Several notes on one task" and the output contract.

1. **Mode**:
   - Parse layer: a `CaptureKind::SubBullet` item with an empty parent body and at least
     one nonempty first-level bullet becomes the multi-note form. Today
     `missing_text_error` fires there, while `capture-parse` already accepts it;
     execution and editor must now agree.
   - With a glued drawer and no note text, the editor drops `Need::DrawerText` once a
     first-level note exists.
   - Make existing targets an under-task position for drawer bullets in this mode.
2. **Writer** (`capture/plan.rs` SubBullet dispatch, `capture/sub_bullet.rs`):
   - Plan the notes in draft order against staged content, reusing
     `plan_sub_bullet_capture` / `plan_prj_section_capture` per note or per drawer group
     (factor shared parent resolution if that helps).
   - Add the header-only creation case only if a code path still needs it; a childless
     drawer bullet is either refused or a plain note, so normally none does.
   - Apply the retype first, as today, and pass `effective_target` to every note.
   - Refuse `multi_note_marker` and the existing-drawer `empty_drawer_bullet`.
3. **Output** (`capture/output.rs`, `capture/batch.rs`): add the `notes`/`drawers` JSON
   fields, the item `drawer` rule, and the human rendering from the contract. Leave
   every same-line-note and single-item JSON byte-identical.
4. **Docs**:
   - `docs/capture.md`: add a "Several notes at once" subsection to "Sub-bullets under
     existing tasks", with the worked example, rules, and refusals. Add
     grammar-at-a-glance rows for `@route+id` + bullets and `@route+id#drawer` +
     bullets, and document the JSON fields under "Input, stdin, and JSON output".
   - `docs/reference/json-outputs.md`: document `notes` and `drawers`.
   - `bob capture --help`: add a sub-bullet paragraph.
   - Run `just docs-gen` and `just docs-check`.
5. **Tests** (extend `tests/cli/capture/sub_bullet.rs` and `prj_drawers.rs`, or add
   `note_bullets.rs`):
   - the worked example, byte for byte, plus its JSON and human output;
   - insertion before managed logs;
   - two bullets naming one missing drawer (one header, `created: true` once);
   - an existing-drawer childless refusal;
   - slug near-duplicate and reserved refusals;
   - a `^prj` target with existing and new sections;
   - glued drawer + bullets;
   - a same-line note still nests (and ALL-CAPS stays literal there);
   - `#type` retype + notes;
   - `&` owner + notes;
   - `s:`/`$`/`%c` refusals;
   - a placeholder-only `@route+id` still Ensure Next;
   - a batch rollback when a later item fails.
6. **Gate**: `just check` and `just docs-check` pass.

## Phase `bullet-menu`

1. **Completion field** (`capture_language/completion.rs`): detect the bullet shortcut
   field through the shared scanner and classify the position from the editor item.
   Not-under-task positions return an empty success.
2. **Menu** (`capture_complete/drawers.rs`, `engine.rs`, `shell.rs`, `render.rs`): build
   targets for existing tasks (task or `^prj`), new tasks, and new project notes. Apply
   bare-title replacements with exact-resolution validity, the consequence copy and
   `target` object from the contract, and the `continue_bullet` decision. Keep the glued
   `@route+id#` menu byte-identical.
3. **Docs**:
   - `docs/capture.md`: add the bullet menu to the `capture-complete` section.
   - `docs/completion.md`: document the drawer menu at a bullet start.
   - `docs/reference/json-outputs.md`: record the bare-title replacement, the nullable
     `target.block_id`, and the continuation.
   - `capture-complete --help` (`capture_complete/cli.rs`): add a sentence on bullet
     drawer completion.
   - Run `just docs-gen` and `just docs-check`.
4. **Tests** (`capture_complete/tests/drawers.rs`,
   `tests/cli/capture/complete_drawers.rs`):
   - every row group per target kind;
   - the bare-title replacement and `cursor_after`;
   - continuation on and off at line end and before an existing nested line;
   - a refused reserved or ambiguous row;
   - same-line-note and glued-drawer positions returning an empty success;
   - nested `  - #` still offering task types;
   - zsh `drawers` group titles;
   - the commit-agreement test: committing the inserted title reproduces each row's
     `consequence` (accepting a row and committing agree).
5. **Memory**: use `/sase_memory_write`, acting only within accepted memory decisions.

   > [!decision] memory_decisions_record Add the decisions strand
   > `capture-bullets-mirror-the-task-block`, titled "Capture Bullets Mirror The Task
   > Block", with the summary "A first-level capture bullet is its task's child:
   > ALL-CAPS with notes is a drawer, a leading # names one, and note-less @file+id
   > files each bullet as a note."
   >
   > - **Applies to**: bob-cli and Bob Mac Capture.
   > - **Claim**: the shared design contract above.
   > - **Why**: the draft reads like the note it writes; one resolver serves menu and
   >   commit; and it fills the refused note-less `@file+id` + bullets gap.
   > - **Rejected alternatives**: keeping bullet-start types; a `- #drawer text`
   >   shorthand (it cannot survive the bare-title rewrite); repeating `@file+id` per
   >   note; treating ALL-CAPS bullets as plain notes under existing tasks (duplicate
   >   headers); spreading `s:`/`$`/`%c` over every note.
   > - **Cost**: the grammar 4 break for `- #fix`; multi-note refusals; drawer notes
   >   stay one line under the two-level grammar.
   > - **Reopens when**: authored bullets gain a third level, or Bryan wants
   >   bullet-start types back.
   > - **Evidence**: this epic's archived plan and its phase commits.
   >
   > Link `[[decisions:capture-sigils-say-where-what-how]]` and
   > `[[glossary:task-drawer]]`.

   > [!decision] memory_decisions_record Then, on `capture-sigils-say-where-what-how`,
   > set `metadata.status: superseded-in-part` and
   > `superseded_by: [decisions:capture-bullets-mirror-the-task-block]`, and append one
   > back-link sentence saying that a `#` at a first-level bullet start now names a
   > drawer. Leave the accepted body otherwise untouched.

   > [!decision] memory_drawer_glossary In `glossary/task-drawer.md`:
   >
   > - add that a `#` right after a first-level capture bullet's `- ` completes to the
   >   drawer's title, and that an ALL-CAPS first-level bullet with nested notes is a
   >   drawer bullet (merging into an existing drawer under note-less `@route+id`);
   > - narrow "a free-standing `#fix` at an item edge types the task" to exclude
   >   first-level bullet starts;
   > - link the new record.

   Run `sase memory init` after editing.

6. **Gate**: `just check` and `just docs-check` pass.

## Phase `mac-capture`

Work in the linked `bob-mac-capture` repository, opened with `/sase_repo` (read its
`AGENTS.md` first). Bob stays the only grammar authority (thin-client decision), so
Swift only decodes and paints.

1. Paint `drawer_bullet` spans with the `drawer` palette. Confirm that `drawer` spans on
   bullet shortcuts and the `drawer` need already open `CaptureDrawerMenuView`. Show a
   caption for a `target` whose `block_id` is null (the new-task text and route).
2. Make sure accepting a row whose `replacement` contains `\n  - ` splices it verbatim
   and puts the caret at `cursor_after`, with no single-line assumptions in the
   insertion path.
3. Decode the additive `notes` and `drawers` fields with `decodeIfPresent`. Extend
   `CaptureSubBulletPresentation` to read "4 notes under ^bar", with drawer capsules
   (`new drawer` when created) and a VoiceOver phrase. Single-note output stays
   unchanged.
4. Keep `CaptureGrammarGate.minimumRevision` at 3.
5. Regenerate the affected fixtures from a real bob built at this epic's head:
   - a multi-note dry run;
   - an empty bullet drawer menu on an existing task, on a new task, and with a typed
     query.

   Add CaptureCore and hosted tests for the menu, insertion, and header.

6. Agent hosts have no Swift toolchain, so macOS CI is the gate. Report the CI result in
   the phase notes.

## Non-goals and follow-ups

- **Deeper authored nesting**: the authored grammar stays two levels (`AuthoredDepth`
  has about 80 uses across 12 files). Notes filed inside a drawer bullet are therefore
  one line each. Each phase worker records a `PROPOSED FOLLOW-UP:` note for a
  third-level authored bullet grammar, which would let drawer notes carry details.
- **Clipboard as notes**: no clipboard-as-notes (`%c` stays refused on multi-note
  items).
- **Forced picker mode**: no change to forced `--route/--task/--task-ref`.
- **Work Log bullets**: no change to Work Log close bullets.
- **No new commands or options**: no new CLI commands or options; every `capture-*` wire
  change is additive except the grammar-4 meaning of a bullet-start `#`.

## Verification

Each phase passes `just check` and `just docs-check` in bob-cli. The final state is also
verified by hand in a scratch vault (`BOB_DIR` pointing at a temp dir) with
`bob capture --dry-run`, `-f json`, `bob capture-parse -f json`, and
`bob capture-complete -f json` over:

- the worked example;
- each refusal in the contract table;
- a new task, a new project note, and a `^prj` target.

The `mac-capture` phase is verified by macOS CI.
