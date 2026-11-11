---
tier: epic
title: Notebook section bullets in capture
goal: "In any capture draft whose bullets land directly in a notebook body (a route-only
  `@nb` list or a `@parent^id@ %n` creation item), typing `- #` or `- /` opens a
  Bob-authored section menu listing the notebook's `##` sections (plus sections created
  earlier in the draft), accepting a row writes `- #Git` and drops the caret on the
  first nested note, and parse, menu, dry-run, and commit always agree on which section
  each note lands in.

  "
decisions:
  create_row_return:
    ask: When a lone "＋ New section" row is the only match, should Return accept it?
    choices:
      preselect_clean:
        Preselect it unless it is a near miss; Return opens the first note line
      tab_only:
        Never preselect it (research rule); Tab or arrows accept it, Return submits the
        draft
    default: preselect_clean
    why:
      Return on a deliberate new name should not submit an incomplete draft; near misses
      still ask
    answer: preselect_clean
  typed_capitals:
    ask:
      Should sections created from an unquoted selector keep the capitals you typed
      (#API-keys → API Keys)?
    default: true
    why:
      Acronyms survive without quoting, and every menu row previews the exact title
      first
    answer: true
  memory_decision_records:
    ask:
      Record the section-bullet and notebook-aware-rewrite decisions, and mark the
      records they partly supersede?
    memory:
      - decisions
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: false
  memory_glossary:
    ask: Update the Notebook and Task Drawer glossary entries for section bullets?
    memory:
      - glossary:notebook
      - glossary:task-drawer
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: true
phases:
  - id: resolver
    title: One section resolver, a staged view, and the silent-wrong fixes
    depends_on: []
    size: medium
    description:
      "resolver: extend the notebook section resolver with an Ambiguous outcome and
      optional typed-capitals titles, add the staged section view (disk plus earlier
      draft operations) behind parse and the glued `@nb#` menu, stop the drawer menu and
      drawer probe on notebook note bullets, and fix the list summary noun."
  - id: grammar
    title: Grammar 26 section bullets for @nb lists and %n creation
    depends_on:
      - resolver
    size: large
    description:
      'grammar: lex `- #Section` bullets in notebook bodies, fan their nested notes into
      that section, place plain siblings in the top region (fixing `%n` order), make a
      bare `#` and a bare `@nb#` incomplete, add spans, needs, teaching diagnostics, the
      `\#` escape, docs, and the breaking grammar bump.'
  - id: menu
    title: The notebook section menu and Bob-authored place-menu chrome
    depends_on:
      - grammar
    size: medium
    description:
      "menu: add the `notebook_section_bullet` completion context with title-preferring
      replacements, the nested continuation, provenance groups, note counts,
      consequences, refused rows, and create-row rules; add additive `place_menu` chrome
      to drawer, glued notebook section, and section-bullet responses."
  - id: alias
    title: Notebook-aware slash alias in capture-rewrite
    depends_on:
      - menu
    size: small
    description:
      "alias: let `capture-rewrite` read notebook route frontmatter only when a typed
      `/` sits at a first-level bullet start, so `- /` becomes `- #` exactly where the
      section menu opens and stops firing on notebook note detail; update the help
      contract and alias tips."
  - id: mac
    title: Bob Mac Capture place menu, highlighting, and grouped preview
    depends_on:
      - grammar
      - menu
      - alias
    size: large
    description:
      "mac: in bob-mac-capture, generalize the drawer menu into one place menu painted
      from Bob's chrome, open it for section bullets and glued notebook sections, keep
      the empty-notebook hint, group the notebook batch preview by section, regenerate
      fixtures, update protocol docs, and watch macOS CI to green."
  - id: memory
    title: Decision records and glossary
    depends_on:
      - grammar
      - menu
      - alias
    size: small
    description:
      "memory: write the section-bullet and notebook-aware-rewrite decision records,
      mark the partly superseded records, update the Notebook and Task Drawer glossary
      entries, and regenerate the decisions docs page."
proposed_by: bbugyi200.athena.1b9
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-11 09:22:44
status: wip
---

- **PROMPT:**
  [prompts/202611/notebook_section_bullets.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/notebook_section_bullets.md)

# Plan: Notebook section bullets in capture

## Why and what

Capture can already file several notes into a task's drawers in one draft
(`decisions:capture-bullets-mirror-the-task-block`). Notebooks are the one destination
whose only structure is `##` sections, yet a list draft cannot pick a section per note.
Worse, the bullet-start `#` there is silently wrong today (all verified on grammar 25):

- `@dev_tools` + `- #git` + `  - note` writes a literal `- #git [created::…]` tag note
  with `note` as its detail, and `- #` / `- /` open nothing.
- `Parent @dev_tools` + `- #fu` opens the **drawer** menu ("Starts FU on the new task").
- `@dev^nvim@ %n` with `- KEYMAPS` / `  - x` / `- plain` writes `plain` under
  `## Keymaps` because `seed_body` writes in draft order.
- `Note @dev_tools#` and the header `@dev_tools#` commit silently to the top region.
- Sections a `%n` item seeds earlier in the draft are invisible to `@dev_nvim#`
  completion; duplicate `##` headings silently resolve to the first.

This epic implements the consolidated research
`research:202611/notebook_section_groups/notebook_section_groups.md`, which Bryan
accepted in full. That means option **B** for the `/` alias, ALL-CAPS kept permanently
as a `%n` compatibility path, and one-line notes inside a section bullet for v1. Out of
scope, already tracked: `bob-cli-k9` (`@@nb` bullet-led items write a checkbox) and
`bob-cli-k8` (`created:` UTC offset). This plan adds no `@@` change and no new CLI verb.

**Name.** A first-level `- #Section` bullet with notes nested under it is a **section
bullet**, the notebook counterpart of a drawer bullet. Codes, spans, and docs use that
name; "section group" is an accepted alias in docs and memory.

### The finished interaction

```text
@dev_tools                          ← route-only header: bullets land in the notebook body
- #Git                              ← typed "- /" → "- #" opens the menu; "gi" + Return
  - Use worktrees for investigations
- #"API Design"                     ← titles a bare selector cannot spell are quoted
  - Preserve meaning across versions
- #docker                           ← "＋ New section “Docker”", created at end of file
  - Pin base images by digest
- Prefer fd over find               ← a plain sibling: top region, before the first ##
```

```text
 Section · 📓 Developer tools · dev
 IN THIS NOTEBOOK
   Git                                              8 notes
   API Design                                       3 notes
 EARLIER IN THIS DRAFT
   Docker                                           new
 ───────────────────────────────────────────────────────────
 ＋ New section “Dockr”                    did you mean Docker?
 Adds after “Use worktrees for investig…” in ## Git   ⏎ choose · ⎋ close
```

```text
📓 4 notes → Developer tools
   › Git                  Use worktrees for investigations
   › API Design           Preserve meaning across versions
   › Docker  New section  Pin base images by digest
   · top                  Prefer fd over find
```

During `%n` creation the caption reads `📓 New notebook · Nvim · dev`, and every section
lives under **Earlier in this draft**.

## The contract

Every phase implements a slice of this contract. Phases must not reopen it. Anything
genuinely blocked goes in a `PROPOSED FOLLOW-UP:` note.

### Where a bullet-start `#` names a section

A first-level bullet's leading `#` (opener rule unchanged: an ASCII letter, `"`, `“`, or
a bare `#` before whitespace or end of line; `#1`, `##`, `#-` stay prose) means:

| Where the item's first-level bullets land                                        | Meaning                                                                                                                                          |
| -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| Under a task (new task, new project note, note-less `@route+id`)                 | Drawer, unchanged                                                                                                                                |
| Directly in a notebook body: a route-only `@nb` list, or a `@parent^id@ %n` item | **Section bullet (new)**                                                                                                                         |
| Under an `@nb#Section` list header                                               | Refused, `section_bullet_position`: "this list already files every bullet into ## Git; drop `#git` from `@dev_tools#git` to use section bullets" |
| Under one notebook note (`Text @nb`, `@nb+id` fan-out, an `@@nb` item)           | Opaque note detail: no menu, no drawer, no diagnostic (fixes bug 1)                                                                              |
| `@area` / `@project` task lists                                                  | Unchanged (`drawer_bullet_position`)                                                                                                             |

Whether a route is a notebook is always Bob's `NotebookRoutePolicy` lookup, never the
Mac's guess.

### What a section bullet means and writes

- A section bullet is a first-level bullet holding only a selector: `- #git`, `- #Git`,
  `- #"API Design"`, `- #“API Design”`, or `- #coc.nvim`. The selector after `#` uses
  exactly the glued `@nb#…` selector grammar and the same resolver.
- **Each nested bullet is one note** in that section. It gets a `[created::]` stamp and
  may end in `^id` under the existing `@nb^id` rules. The bullet line itself is never a
  note. Notes are one line in v1. A third-level line under a section-bullet note is
  refused with code `section_bullet_depth` and copy that teaches the header form: "notes
  in a section bullet are one line; for a note with detail, use a `@dev_tools#git`
  header list". The `%n` copy names the new route.
- **Ownership is by indentation.** A plain first-level sibling is a top-region note
  (before the first `##`) whatever its draft order. It keeps its nested detail as today.
- **Placement.** Top-region notes append after the last top-region note, and section
  notes after the last note of their section. Missing sections are created at end of
  file in first-mention order, repeated bullets for one section merge in source order,
  and existing section order never changes. For `@nb` lists this falls out of the
  existing `plan_notebook_note` writer once each note is a `NotebookNote` with a
  section. For `%n` it is a rewritten seed builder.
- **Refusals** are Bob-written and teach only the current grammar. All are usage errors
  at commit, parse diagnostics with precise ranges, and never partial writes:
  - `section_bullet_text`: text, or a `^id`, after the selector on the bullet line.
    Copy: "`- #git` names a section, so it takes no text: nest notes beneath it
    (`  - …`), or write `\#git` to keep the tag".
  - `empty_section_bullet`: no notes at commit. While typing, parse reports mode
    `incomplete` with `needs: ["section_notes"]` and no error.
  - `section_bullet_position`: under an `@nb#Section` header.
  - `section_bullet_depth`: a third-level line, as above.
  - `notebook_section_near_miss`: unchanged code, now also on bullets.
  - `notebook_section_ambiguous` (new): duplicate headings, see the resolver below.
  - `notebook_section_required`: a bare `- #`, a bare glued `Note @nb#`, or a bare
    `@nb#` list header. Parse reports `needs: ["section"]` with an
    `interactive_placeholder` span over the `#`. It never falls back to the top region
    and is never written (fixes bug 3).
- **Escape.** `- \#git` keeps a literal `#git` note.
- **ALL-CAPS.** In `%n` creation, an ALL-CAPS first-level bullet with nested notes still
  seeds a `##` section (permanent compatibility, now with correct sibling placement and
  merging with `#` bullets of the same key). In `@nb` lists an ALL-CAPS bullet stays an
  ordinary note with detail. Completion never inserts ALL-CAPS for notebooks.
- **One transaction.** The whole draft remains one guarded all-or-nothing write. That
  covers notebook files, any new notebook, and owner `## Notebooks` catalogs, with the
  existing preimage guards and re-resolution at submit. Completion state is never write
  authority.

### One resolver and one staged view

- `resolve_notebook_section` returns `Existing`, `Create`, `NearMiss`, or
  **`Ambiguous`** (with every matching title and 1-based line). Precedence:
  1. A quoted selector matches its exact title case-insensitively, otherwise it creates
     verbatim.
  2. An unquoted selector matches the exact title case-insensitively, otherwise a unique
     whole slug.
  3. Two or more headings tied at the winning step are `Ambiguous`.
  4. Otherwise the near-miss rules apply: a strict slug prefix, or edit distance ≤ 2,
     which stays the notebook threshold.
  5. Otherwise `Create`.

  Matching is never by prefix or fuzz at write time. The ambiguous copy is:
  "dev_tools.md has 2 sections titled “Git” (lines 9 and 23); rename one, then capture
  again".

- **Create titles.** These come from `-` → space plus Title Case as today. When
  `typed_capitals` is on, words containing an uppercase letter keep exactly what was
  typed. Quoted selectors are always verbatim.
- **Staged view.** One function, which every consumer must use, returns a route's
  sections at a draft offset: the disk H2s (title, 1-based line, top-level note count),
  then sections that _valid_ earlier operations in this draft create, in source order,
  each with provenance `notebook` or `draft`. The earlier operations are seeds from an
  earlier `%n` item, earlier glued `@nb#New` items and headers, and (after `grammar`)
  earlier section bullets, including earlier bullets inside the same `%n` item. Later
  operations, failed items, and the bullet under the caret never appear. Draft sections
  that resolve to an existing section dedupe into it. Parse diagnostics, both menus, and
  the `%n` provisional sections read it. The writer already resolves against staged
  bytes, so commit agrees by construction.
- **Round-trip invariant.** This is the epic's decisive test. Splice any enabled menu
  row at Bob's range, then `capture-parse` and `capture --dry-run` the draft. Both must
  report exactly the section and `section_action` the row advertised. Commit bytes must
  equal dry-run bytes.

### Protocol (all additive except the grammar bump)

- **`capture-parse`.** A section-bullet selector is a `section` span with
  `notebook: true`. It covers the `#` through the selector and carries `title` (the
  resolved or to-be-created heading) and an additive `section_action` (`existing` |
  `create`). Glued notebook `section` spans gain `section_action` too. The new need is
  `section_notes`. The diagnostic codes are listed above.
- **`capture` JSON.** The shape is unchanged: per-note `notebook.section` and
  `notebook.section_action` already exist. Notes fanned out of one section bullet carry
  the existing `fanout` context (`line`, `position`, `count`, `marker: "#Git"`). Human
  list summaries count notes and say `note(s)` for notebook lists:
  `list  dev_tools.md · 4 notes`.
- **`capture-complete`.**
  - New context `notebook_section_bullet`. Its replacement range covers only the
    selector text after `#`, and the `#` stays in the draft.
  - The glued `@nb#` stays context `section`.
  - Both contexts emit one notebook row model. It keeps today's
    `{replacement, title, level}` and the create row's
    `{kind: "notebook_section_create", label, title}`, and adds `kind`
    (`notebook_section` or `notebook_section_create`), `group` (`notebook` | `draft` |
    `new`), `line`, `note_count`, `detail`, `consequence`, `creates`, `refused`, and
    (bullet context only) `cursor_after`.
  - New additive top-level `place_menu` object on `drawer`, notebook `section`, and
    `notebook_section_bullet` responses: `kind` (`drawer` | `notebook_section`),
    `title`, `caption`, `groups` (`[{id, title}]`, with the `new` group untitled),
    `new_row_prefix`, `empty_hint`, `results_label`, and `footer_hint`. Drawer chrome
    reproduces today's Mac copy exactly ("Drawer", "On this task", "Recent", "＋ New
    drawer", "⏎ choose · ⎋ close", and the `^id text (route)` caption). Older apps
    ignore it, and the Mac falls back to its hard-coded drawer copy when it is absent.
- **`capture-rewrite`.** This is option B. It reads notebook route frontmatter read-only
  from the configured vault (`BOB_DIR`), and only when the typed key is `/` at a
  first-level bullet start. Every other rewrite stays filesystem-free. This replaces the
  "never touches the filesystem" sentence in its help.
- **Grammar 25 → 26.** This ships as a `feat(capture)!:` commit on the release train
  (`decisions:bob-cli-releases-ride-a-daily-train`; never edit `Cargo.toml`'s version or
  `CHANGELOG.md`). It is breaking: `@nb` + `- #…` changes from a literal tag to a
  section bullet, a bare `@nb#` becomes incomplete, and `%n` sibling placement changes.

## Phase resolver: one section resolver, a staged view, and the silent-wrong fixes

No grammar bump. Commit as `fix(capture)`/`feat(capture)` as fits.

- **Resolver** in `src/native/capture_language/notebook.rs` (`resolve_notebook_section`,
  `NotebookSectionResolution`): add `Ambiguous` and the precedence above. Handle it
  everywhere the enum is matched:
  - the parse near-miss diagnostics in `editor_view_for_item`, which become a
    `notebook_section_ambiguous` diagnostic on the selector or header range;
  - the writer in `src/native/capture/notebook_note.rs` (`plan_notebook_note`), as a
    usage error with the same code;
  - `notebook_section_replacement` / `notebook_section_candidates` in
    `src/native/capture_complete/candidates.rs`, where each duplicate heading becomes a
    refused row carrying the ambiguous copy and is never offered as a replacement.

  `--section` forced routes resolve through the same function.

- > [!decision] typed_capitals
  >
  > Create titles keep typed capitals in words that contain an uppercase letter
  > (`#API-keys` → `API Keys`, `#git` → `Git`, `#coc.nvim` → `Coc.nvim`). This applies
  > to glued selectors, the near-miss `quoted_title`, and (via `grammar`) section
  > bullets. ALL-CAPS `%n` compatibility bullets keep today's Title Case.

  > [!decision] typed_capitals = no
  >
  > Create titles stay Title Case exactly as today.

- **Staged view.** Add the single staged-sections function (contract above) in
  `capture_language`. Build it from the per-item claim pipeline with the
  `NotebookRoutePolicy` (`claim_notebook_members` outcomes and
  `discover_notebook_creates` seeds), skipping items that fail to claim. Do not
  re-derive syntax in a second lexer. Give each section an origin (disk line or draft
  offset) so replacement round-trips stay identity-based. Use it for:
  - parse near-miss and ambiguity diagnostics;
  - the glued `@nb#` menu, which today calls `read_target` on disk only. That fixes bug
    5: after `Neovim notes @dev^nvim@ %n` + `- KEYMAPS` / `  - x`, a later
    `More @dev_nvim#` lists `Keymaps`;
  - `NotebookRoutePolicy` provisional create sections. Keep
    `provisional_create_sections` and the writer's `seed_sections` on one shared
    derivation.
- **Bug 1.** The drawer probe (`drawer_bullet_probe` in
  `capture_language/completion.rs`) classifies with the lexical `editor_item_at`. Add a
  policy-aware entry point (for example
  `completion_field_at_with(raw, cursor, Option<&NotebookRoutePolicy>)`) and use it from
  `capture_complete/engine.rs`, which already builds the policy. Then a bullet under a
  notebook-claimed item never yields a drawer target, and the field is an empty success
  instead of the drawer menu. `rewrite.rs` keeps the lexical call until `alias`.
- **Bug 7.** Notebook list summaries (human `list` line in
  `src/native/capture/output.rs`) say `note(s)` and count notes.
- **Docs.** Update `docs/capture.md` "Notebook notes" and the capture-complete section,
  plus `docs/notebooks.md` "Capture", for ambiguity, staged sections, and typed
  capitals. Then run `just docs-gen` if any help text changed, and `just docs-check`.
- **Tests.** Resolver unit tests: every precedence step; quoted vs unquoted; ambiguity
  by title and by slug; near-miss at distance 1 and 2 and strict prefix; typed capitals.
  Add CLI tests in `tests/cli/capture/notebook_notes.rs`, `notebook_create.rs`, and
  `complete_drawers.rs`:
  - the bug-1 draft opens no drawer menu, and `Text @nb` + `- #` returns no context;
  - the ambiguous duplicate is refused in parse, complete, dry-run, and commit;
  - bug 5's staged rows appear, while later and failed items' sections do not;
  - the list noun.

## Phase grammar: grammar 26 section bullets for @nb lists and %n creation

This phase is large because it crosses draft splitting, notebook claiming, the editor
parse, the `%n` writer, output, and docs. Plan it first.

- **Lexer.** Add a section-bullet scanner beside `capture_language/drawer_bullet.rs`
  (same opener, `Complete` / `Incomplete` / `Invalid` / `Escaped` hit shape, the
  `lex_exact_name_after_hash` quoting) that yields a notebook selector (`text`, `exact`)
  using the glued `@nb#` selector rules and not the drawer slug rules.
- **`@nb` lists.** In the draft splitter, each first-level list bullet is its own item
  whose parent line is the bullet body. In `claim_list_notebook`
  (`capture_language/notebook.rs`), a route-only list item whose body is a section
  bullet fans out its nested bullets into one
  `CaptureKind::NotebookNote { section, exact_section, block_id }` outcome each, with
  `FanoutContext` (model the fan-out on `fanout_notebook_plus`). Everything else in the
  contract follows:
  - text after the selector → `section_bullet_text`;
  - no notes → `empty_section_bullet`;
  - under an `@nb#Section` header → `section_bullet_position`;
  - third level → `section_bullet_depth`;
  - bare `#` → `notebook_section_required`;
  - `\#` → a literal note.

  A route-only header with a bare hash (`@nb#`) and the glued bare `Note @nb#` both
  become `notebook_section_required` (bug 3).

- **`%n` creation.** Replace `seed_body` / `seed_sections` in
  `src/native/capture/notebook_create.rs` with one seed plan derived in
  `capture_language`, shared with `provisional_create_sections`, the editor view, and
  the staged view. The plan holds top notes first (in source order, with detail), then
  sections in first-mention order. Each section has its merged one-line notes from `#`
  bullets and ALL-CAPS compatibility bullets, keyed through the resolver, and a
  `%n`-internal near miss or ambiguity refuses. Validate section bullets in
  `refuse_notebook_create_children`. A later same-draft `@dev_nvim#lsp` must resolve to
  the seeded section.
- **Parse.**
  - In `editor_view_for_item` and `notebook_create_editor_view`, add the `section` spans
    (`notebook: true`, `title`, `section_action` from the staged view) and the `section`
    / `section_notes` needs (add `Need::SectionNotes`, wire `section_notes`).
  - Add the diagnostics with precise ranges.
  - Under a notebook item, bullets never carry `drawer_bullet_target`.
- **Output.** Human per-note lines read `📓 Note → n/N  Developer tools › Git` and
  `› new section Docker`, and list summaries count notes. Keep the JSON shape.
- **Grammar.** Set `GRAMMAR_REVISION` to 26 in `capture_language/mod.rs`, and update
  every test and fixture that pins 25.
- **Docs.**
  - `docs/capture.md`: "Notebook notes", the capture-parse needs and diagnostics prose,
    and the authored-bullet rules.
  - `docs/notebooks.md`: "Capture", with an example draft and the written file.
  - `docs/reference/capture-syntax.md`: rows for `- #Section` in a notebook body and the
    `\#` escape.
  - The capture-parse `long_about` in `capture_parse.rs`, which today says `- #` names a
    drawer.
  - Then run `just docs-gen` and `just docs-check`.
- **Tests.** Cover every row of the position table and every refusal. Also cover:
  - placement: siblings before any `##` regardless of order, repeated bullets merging,
    new sections at end of file in first-mention order, and existing order untouched;
  - `%n` bug 2;
  - text robustness: mixed case, acronyms, emoji, non-Latin quoted titles, smart quotes,
    escaped quotes, CRLF/BOM, fenced `##`, and a trailing note `^id`;
  - dry-run bytes equal commit bytes;
  - a retyped or renamed notebook between parse and commit aborting atomically;
  - the `@area` list and task drawer behavior being unchanged.

## Phase menu: the notebook section menu and Bob-authored place-menu chrome

- **Field.** Add `notebook_section_bullet_completion_at(raw, cursor, policy)` in
  `capture_language/completion.rs`, modeled on `DrawerBulletCompletion`: the query
  (decoded for quotes), a replacement range over the selector after `#`, and
  `continuation` (true when the token ends its line and no nested bullet follows). The
  engine tries it before the drawer probe, using the policy-aware entry point from
  `resolver`, and `%n` items count as notebook bodies.
- **Rows** (`capture_complete/candidates.rs`, `model.rs`; one row model for both
  notebook contexts, read from the staged view):
  - **Spelling.** Bullet rows insert a title-preferring selector: the bare title when it
    opens the bullet scanner and round-trips through the resolver to that heading,
    otherwise the quoted exact title (`"API Design"`). Glued rows keep today's bare
    title → slug → quoted spelling.
  - **Continuation.** When `continuation` is true, append `\n  - ` and set
    `cursor_after` past it.
  - **Groups.** Rows are ordered by group: `notebook` rows, then `draft` rows, then the
    create row. Within a group, an empty query keeps document or source order. A typed
    query ranks exact (title or slug), then prefix, then substring, with order as the
    tie-break.
  - **Detail.** `8 notes` / `1 note` / `empty` for notebook rows, `new` for draft rows,
    and `did you mean Docker?` on a near-miss create row.
  - **Consequence.**
    - `Adds after “Use worktrees for investig…” in ## Git` (note text ellipsized);
    - `First note in ## Git` for an empty section;
    - `Adds to ## Docker, new in this draft`;
    - `Creates ## Docker at the end of Developer tools`;
    - `Seeds ## Keymaps in the new notebook` during `%n`.
  - **Create row.**
    - It is shown only for a nonempty query with no exact match.
    - It is **hidden while the query is a strict prefix of any listed title or slug**.
    - A near-miss query gets the quoted spelling so the row commits as a create.
    - Its title shows the resolver's create title, so `typed_capitals` is visible before
      commit.
  - **Refused rows.** Ambiguous duplicates, and titles no selector can spell, stay
    listed with `refused` set and Bob's reason.
  - **Read failure.** If the notebook cannot be read, return a bounded warning, no rows,
    and **no** create row.
  - **Glued rows.** The glued `@nb#` menu gets the same groups, detail, consequence,
    refused, and create-row rules, without a continuation.
- **Selection.** An empty query opens with `initial_selection: "none"`. A typed query
  selects the first enabled non-create row.

  > [!decision] create_row_return = preselect_clean
  >
  > When the create row is the only enabled row and is not a near miss, it is selected,
  > so Return inserts `#docker` plus the nested line. A near-miss create row is never
  > preselected.

  > [!decision] create_row_return = tab_only
  >
  > The create row is never preselected. Tab or the arrow keys pick it, and Return
  > submits the draft.

- **Chrome.** Build `place_menu` (contract above). The notebook caption is
  `📓 <title> · <owner>` from `NotebookRouteInfo`, or
  `📓 New notebook · <title> · <owner>` for a staged create. The empty hint is
  `Type a section name`, the title is `Section`, and the groups are `In this notebook` /
  `Earlier in this draft`. The drawer chrome lives in `capture_complete/drawers.rs`, and
  drawer rows stay byte-identical.
- **Docs.** In `docs/capture.md` ("bob-capture-complete"), cover the new context, the
  row fields, and `place_menu`. Update the capture-complete `long_about`, then run
  `just docs-gen` and `just docs-check`.
- **Tests.** Cover the round-trip invariant for every enabled row in both contexts, for
  an existing notebook and a `%n` item, including continuation splicing. Also cover:
  - the create-row rules (hidden on prefix, the near-miss quoted spelling, initial
    selection under the chosen `create_row_return`);
  - staged rows (earlier `%n`, earlier bullets in the same list and the same `%n` item,
    and the exclusion of later and failed items);
  - refused rows and the read-failure behavior;
  - `place_menu` on all three contexts;
  - no menu at `Text @nb`, `@nb+id`, `@@nb`, or nested positions.

## Phase alias: notebook-aware slash alias in capture-rewrite

- `expand_hash_alias` in `capture_language/rewrite.rs` uses the lexical field. Add a
  policy-carrying variant (`rewrite_draft_with(raw, cursor, options)`). The
  `src/native/capture_rewrite.rs` adapter builds
  `notebook_policy_for_draft(bob_dir, draft)` **only** when a cheap exported lexical
  predicate says the byte before the cursor is `/` at a first-level bullet start (a
  `- /` / `* /` / `+ /` line in a list or `%n` item). Every other rewrite stays
  filesystem-free.
- Allow the alias where the `notebook_section_bullet` field opens at the cursor
  (`replacement.0 == cursor`). With the policy, a bullet under a notebook note no longer
  fires (today `Parent @dev_tools` + `- /` wrongly fires into the drawer position).
  Pasted `/`, doubled `//`, Undo, and Escape keep today's behavior. A typed `/` and a
  typed `#` converge on the same menu.
- Thread the same policy into the `sigil_alias_tip` proof (`alias_tip.rs`) so a typed
  `- #` in a notebook body can teach `/`.
- Rewrite the `long_about` contract sentence in `capture_rewrite.rs` ("purely lexical …
  never touches the filesystem …") to state the one bounded read. Update
  `docs/capture.md` ("bob capture-rewrite", "Sigil alias") and run `just docs-gen` /
  `just docs-check`.
- Tests: `tests/cli/capture/rewrite.rs` positive cases (`@nb` list, `%n` item) and
  negatives (`@area` list, `Text @nb`, `@nb+id`, `@nb#Section` header, nested bullet,
  unrouted draft reading no files), plus the alias tip.

## Phase mac: Bob Mac Capture place menu, highlighting, and grouped preview

All of this phase happens in the linked `bob-mac-capture` repo. Open it with
`/sase_repo` and read its `AGENTS.md`. Bob stays the only grammar authority
(`decisions:mac-capture-is-a-thin-client`): the app decodes and paints, and every string
comes from Bob except the fallback drawer copy.

- **Decode.**
  - `place_menu` on `CaptureCompletionResponse`.
  - The new row fields on `CaptureCompletionCandidate` (`kind`, `group`, `line`,
    `note_count`, `detail`, `consequence`, `creates`, `refused`, `cursor_after`).
  - `notebook` and `section_action` on `CaptureSpan`.
  - The `notebook_section_bullet` case in `CaptureCompletionContext`.

  Use `decodeIfPresent` throughout, so an older bob keeps working.

- **Place menu.**
  - Generalize `CaptureDrawerMenuPresentation` / `CaptureDrawerMenuView` into one place
    menu painted from `place_menu` (title, caption, group headers, new-row prefix,
    results label, footer hint). Today's drawer copy remains the fallback when
    `place_menu` is absent.
  - Choose it for any response that carries `place_menu`, and for `drawer` without it.
    Keep the branch order and viewport-height and accessibility parity in
    `CapturePanelView`.
  - Icons: drawers keep `tray.full`; notebook rows use a section glyph tinted with the
    section color; draft rows add a small "new" capsule; the create row keeps the dashed
    `plus.square.dashed` treatment.
  - Refused rows are dimmed, with Bob's reason in the footer.
  - An empty response that carries `empty_hint` keeps the menu open with that calm hint
    instead of closing (`applyNonActiveTaskCompletion` closes empty responses today).
- **Triggers.** Make sure `needs: ["section"]` at a bullet start and a caret inside a
  `section` span request completion (the trigger sets near
  `CapturePanelModel.swift:20387-20470`). `section_notes` must not block typing.
  Accepting a row reuses the generic `cursor_after` splice. The drawer continuation test
  proves it, so add a section-bullet twin.
- **Selection.** Honor `initial_selection`, and keep the active row by candidate
  identity (`group` + `title` + `kind`) across refreshes. Stale replies are dropped as
  today.
- **Highlighting.** Section-bullet selectors already arrive as `section` spans (purple).
  Additionally render `section_action: "create"` spans with a subtle dotted underline,
  and include "new section" in the VoiceOver and tooltip text.
- **Preview.** Group consecutive notebook-note results by `notebook.route` into one
  block: the header `📓 N notes → Title`, then one row per note with `› Section`, a
  `New section` capsule when `section_action == "create"`, and `· top` for top-region
  notes. Keep `CaptureNotebookPresentation` as the single headline source.
- **VoiceOver.** Announce the destination, title, existing or new, and any refusal,
  using Bob's words only.
- **Fixtures and tests.** Regenerate the notebook and drawer fixtures with
  `Scripts/generate-notebook-fixtures.sh` / `generate-drawer-fixtures.sh` against a bob
  built from bob-cli master containing `grammar`, `menu`, and `alias`, and record the
  SHA in the fixture READMEs. Add `fake-bob` drafts for the section-bullet menu, the `/`
  alias, and the grouped preview. Extend `CaptureDrawerMenuPresentationTests` (or its
  renamed successor), `CompletionRowContentTests`, `CaptureNotebookPresentationTests`,
  `CaptureDrawerMenuHostedTests`, and `CaptureHashAliasDoubleTapTests`.
- **Docs.**
  - `docs/capture-panel.md`: the "Drawer menu" section becomes "Place menu", and fix its
    stale grammar-3 note.
  - `docs/capture-previews.md`: "Notebook notes".
  - `docs/protocol.md`: compatibility rows for grammar 26, `notebook_section_bullet`,
    and `place_menu`.
  - `docs/keyboard.md` (the `/` alias) and `docs/typing-tips.md` (`hash_menu`).
  - Then run `just docs-check`.
- **Gate.** There is no Swift toolchain on agent hosts. Run `just portable-check`
  locally. Run `Scripts/macos-ci-check` before landing, push, and watch the exact-SHA
  macOS CI run to green with `/sase_monitor`.

## Phase memory: decision records and glossary

Use `/sase_memory_write` for every change below. Each change is authorized only by the
matching memory decision.

- > [!decision] memory_decision_records
  >
  > - Add the strand `decisions/notebook-section-bullets.md` (aliases: section bullets,
  >   section groups). Its claim: in a notebook body (a route-only `@nb` list or a
  >   `@parent^id@ %n` item), a first-level `- #Section` bullet files its nested
  >   one-line notes into that `##`. The selector stays in the draft, one resolver and
  >   one staged view serve every surface, and there are no empty sections. Its rejected
  >   alternatives are the ALL-CAPS rewrite, same-line text, `## ` draft lines, `@@nb#`,
  >   and vault-wide catalogs. Its cost is grammar 26, and it reopens on a third
  >   authored level. It links `notebook-routes-write-notes` and
  >   `capture-bullets-mirror-the-task-block`, and cites this epic's commits and the
  >   research report.
  > - Add the strand `decisions/rewrite-reads-notebook-routes.md`. Its claim:
  >   `capture-rewrite` may read notebook route frontmatter, read-only, only to decide a
  >   bullet-start `/`. It is otherwise lexical. Rejected alternatives are lexical
  >   over-fire (A) and a semantic assist in `capture-complete` (C).
  > - Mark `notebook-routes-write-notes` `superseded-in-part` by the rewrite record,
  >   with a back-link that retires its V4 "Rewrite is purely lexical" reasoning.
  > - Mark `capture-sigils-say-where-what-how` `superseded-in-part` by the
  >   section-bullet record, with a back-link stating that in a notebook body a `#` at a
  >   first-level bullet start names a `##` section.
  > - Never edit accepted bodies beyond those marks.
- > [!decision] memory_glossary
  >
  > - `glossary:task-drawer`: scope its bullet-start sentence to bullets under a task,
  >   and add that in a notebook body `- #` names a section instead.
  > - `glossary:notebook`: add one sentence on section bullets.
- After the edits, run `sase memory init`, `just docs-gen` (regenerates
  `docs/develop/decisions.md` from the decisions web), and `just docs-check`.

## Acceptance matrix

Each row runs through parse, complete, dry-run, and commit, and the four must agree.

| Check                                                                                                                                         | Owner phase |
| --------------------------------------------------------------------------------------------------------------------------------------------- | ----------- |
| No drawer menu on notebook note bullets; `@area` and task drawer menus unchanged                                                              | resolver    |
| Ambiguous duplicates refused everywhere; staged `%n` sections listed by glued `@nb#`                                                          | resolver    |
| Section bullets in `@nb` lists and `%n`; siblings in the top region regardless of order; merging; end-of-file creation in first-mention order | grammar     |
| Bare `#`, bare `@nb#`, empty, same-line text, wrong position, third level, `\#`                                                               | grammar     |
| Text robustness (case, acronyms, emoji, non-Latin, quotes, CRLF/BOM, fences, `^id`)                                                           | grammar     |
| Every enabled row round-trips to its advertised section and action                                                                            | menu        |
| Create row hidden on prefix, quoted on near miss, selection per `create_row_return`                                                           | menu        |
| `- /` becomes `- #` exactly where the section menu opens, and nowhere under a note                                                            | alias       |
| Place menu, highlight, and grouped preview render from Bob's data; macOS CI green on the exact SHA                                            | mac         |
| Records and glossary match the shipped behavior; docs-check passes                                                                            | memory      |
