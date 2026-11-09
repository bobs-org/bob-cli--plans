---
tier: epic
title: Notebooks — a home for notes under areas and projects
goal: 'Bob gains a `type: "[[notebook]]"` note kind for task-free note bullets (optionally
  under `##` sections) that always hangs under an area, project, or notebook. Capture
  writes notes into notebooks through the existing `@nb` forms, creates notebooks
  with `@parent^id@ %n <summary>`, and every area or project shows a beautiful generated
  `## Notebooks` catalog. 26 existing notes, 11 of them rescued from the zorg dead
  end, become the first notebooks.

  '
decisions:
  collapse_assist:
    ask: Should `#`, `/`, `^`, `.`, `+`, or `,` typed right after a complete `@@route`
      collapse it to `@route#`/`^`/`+`?
    default: true
    why: Keeps a typed @@foo#bar, @@foo+baz, or @@foo^id working while @@ keeps one
      meaning
    answer: true
  memory_glossary_notebook:
    ask: Should the plan add the new Notebook glossary strand?
    memory:
    - glossary:notebook
    default: true
    requested: We should add an excellent new "notebook" glossary term to describe
      this concept.
    answer: true
  memory_notebook_decision:
    ask: Should the plan add the notebook decision record and mark capture-sigils
      superseded in part?
    memory:
    - decisions:notebook-routes-write-notes
    - decisions:capture-sigils-say-where-what-how
    default: true
    corrects: '`%c`, `%n`, `%t`, and `%l` at an item edge declare clip, Pomodoro-note,
      template, and listen directives.'
    answer: true
  memory_glossary_corrections:
    ask: Should the plan update the Capture, Project Note Capture, and Capture Alias
      strands for notebooks?
    memory:
    - glossary:capture
    - glossary:project-note-capture
    - glossary:capture-alias
    default: true
    corrects: item may create a task (default `mac_inbox.md`, an [[inbox-file]]),
      bullet, sub-bullet, project note (a [[project-note-capture]]), or record note
    answer: true
phases:
- id: notebook-core
  title: Notebook recognition, bob notebooks list and check, and the contract doc
  depends_on: []
  size: large
  description: 'notebook-core: recognize `type: "[[notebook]]"`, add the notebooks
    module (one vault scan, path-aware parents, owner and root through the existing
    Root Parent walk, title, summary, sections, counts), the `bob notebooks list`
    and `check` commands with beautiful human output and JSON, teaching refusals wherever
    a notebook is offered as a task or reference home, and `docs/notebooks.md`.

    '
- id: notebook-catalog
  title: The generated Notebooks catalog, bob notebooks sync and create, nightly step
  depends_on:
  - notebook-core
  size: large
  description: 'notebook-catalog: render and place the machine-owned `## Notebooks`
    catalog in every area or project with notebooks beneath it, add `bob notebooks
    sync` (locked, committed, byte-stable) and its lock-free nightly step, add `bob
    notebooks create` on a shared new-notebook renderer, and report catalog state
    in list and check.

    '
- id: capture-notebook-notes
  title: Capture writes notes into notebooks (the destination decides)
  depends_on:
  - notebook-core
  size: large
  description: 'capture-notebook-notes: add the memoized notebook route policy and
    teach capture that a notebook route writes notes: `@nb`, `@nb#section` (exact,
    created on demand, near misses refused), `@nb^id`, `@nb#section^id` (new on every
    route), `@nb+id` sub-notes, lists, draft-wide `@@nb`, URLs, `--route`, teaching
    refusals, parse modes, previews, notebook capture-targets rows, and the grammar
    bump.

    '
- id: capture-notebook-create
  title: Create notebooks with @parent^id@ %n
  depends_on:
  - capture-notebook-notes
  - notebook-catalog
  size: large
  description: 'capture-notebook-create: make `%n` on a `^`-sealed project capture
    create `<parent>_<id>.md` as a notebook whose summary is the item text, seed notes
    and ALL-CAPS sections from its bullets, make it visible to later items in the
    same draft, rewrite the affected catalogs in the same all-or-nothing batch, and
    fix the stale `%n` sentence tracked by bob-cli-j3.

    '
- id: capture-notebook-assist
  title: Notebook completion, block IDs, and the @@ collapse assist
  depends_on:
  - capture-notebook-create
  size: large
  description: 'capture-notebook-assist: give capture-complete notebook route rows,
    `##` section rows with a create row, `@nb^` ID suggestions marked `destination_kind`,
    `@nb+` note-bullet rows with ID minting through capture-task-id, an enabled `%n`
    row on sealed items, and (per collapse_assist) the capture-rewrite `@@route` collapse.

    '
- id: plugins-notebook-catalog
  title: Beautiful catalog card, live chips, and the notebook header in Obsidian
  depends_on:
  - notebook-catalog
  size: large
  description: 'plugins-notebook-catalog: in bob-ledger-tools, style the `[!notebooks]`
    catalog as a hue-accented card with a tree rule in Reading view and Live Preview,
    add live note-count, recency, add-summary, and stale-sync chips from `bob notebooks
    list -f json`, render a breadcrumb and child-notebook header inside notebooks,
    and trigger a debounced `bob notebooks sync` on notebook frontmatter changes.

    '
- id: plugins-notebook-gestures
  title: Obsidian block-ID prompt, pickers, and Create notebook here
  depends_on:
  - notebook-catalog
  size: medium
  description: 'plugins-notebook-gestures: make the block-ID prompt suggest IDs for
    plain note bullets, classify notebooks (with a notebook icon) in navigation, filter
    the parent picker by the notebook parent rules, and add a Create notebook here
    command backed by `bob notebooks create`.

    '
- id: mac-notebook-capture
  title: Bob Mac Capture presents notebooks
  depends_on:
  - capture-notebook-assist
  size: large
  description: 'mac-notebook-capture: decode unknown block-ID intents as unsupported
    instead of link, show `kind: "notebook"` targets with a book icon and summary,
    keep notebooks out of File-under, render notebook note, sub-note, and creation
    previews and success lines, and present note-bullet rows in the `+` picker.

    '
- id: notebook-vault
  title: Notebook type note, template, Base, and the wave-1 migration
  depends_on:
  - capture-notebook-notes
  - notebook-catalog
  size: medium
  description: 'notebook-vault: once every host runs a bob with notebook capture,
    add the `notebook.md` type note, `notebooks.base`, and `_templates/new_notebook.md`,
    then migrate the wave-1 manifest (type, quoted parent, summary, H1; nothing else),
    run `bob notebooks sync` and `check`, and prove the success criteria.

    '
- id: notebooks-wrapup
  title: End-to-end proof, docs audit, memory, and the wave-2 bead
  depends_on:
  - notebook-vault
  - mac-notebook-capture
  - plugins-notebook-catalog
  - plugins-notebook-gestures
  size: medium
  description: 'notebooks-wrapup: walk every notebook gesture end to end in a temporary
    vault, audit every notebook doc and help page, apply the accepted memory decisions,
    and file the wave-2 migration task bead.'
proposed_by: bbugyi200.apollo.research.3x.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-09 12:11:24
status: wip
bead_id: bob-cli-j8
---

- **PROMPT:** [prompts/202611/bob_notebooks.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/bob_notebooks.md)
- **BEAD:** [bob-cli-j8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-j8/README.md)

# Plan: Notebooks — a home for notes under areas and projects

## Why

Bob has a note kind for every kind of obligation (areas, projects, references, records,
articles) but none for knowledge that has none. A fresh read-only census of `~/bob`
(2026-11-09) shows what that costs:

- 664 vault-root notes; 503 are untyped, and 430 of those have `parent:` chains that
  never reach an area or project (most end at the retired zorg hub `org`). Every Bob
  consumer is blind to them.
- Bryan maintains "Related notes" indexes by hand in `dev.md`, `meet.md`, `ks.md`,
  `linux.md`, `dog.md`, `lang.md`, and `dev_tools.md`. Nothing keeps them current.
- He already builds notebooks by hand (`legacy_dad`, `dev_ideas`, `gtd_ideas`,
  `vet_log`, and others). The vault task `bob.md ^notebooks` (in progress) asks for this
  feature, and the blocked tasks `^n-notebook-directive` and `^h2-to-notebook` depend on
  it.

This epic implements the research report
`research:202611/bob_notebooks/bob_notebooks.md`. Bryan agreed with every requirement it
recommends (A1–A13): the destination decides what a route writes, `@@` stays draft-wide,
notebooks are created with `@parent^id@ %n`, the summary is required on creation,
notebooks never hold tasks, "only note bullets" is a shape rather than a byte filter,
every area or project lists the notebooks in its subtree, Markdown holds no numbers, the
parent rules, `URL @nb` writes a plain note, H2 extraction is designed for but not
built, and migration never renames files. Every phase worker reads the report sections
relevant to the phase with
`sase artifact read research:202611/bob_notebooks/bob_notebooks.md "<why>"`. **This plan
is the binding contract wherever the two differ.**

### Design calls this plan makes beyond the research

1. **The catalog writes no colour.** The research wrote `> [!notebooks|blue]`. Instead
   bob-ledger-tools resolves the host note's hue live with its existing
   `pomodoroHomeChainHue`, so the accent matches Stats colours and a `color:` change
   never rewrites Markdown.
2. **The `@@` collapse covers `/` and `\` as well.** The research left `/` out,
   believing `/` is route text inside `@@`. It is not: `parse_global_destination_token`
   (`capture_language/draft.rs`) accepts only `[A-Za-z0-9_-]`, and `@@cash/` stays
   literal today only because no `#` menu opens there. So the `/`→`#` alias Bryan asked
   about works on the collapse too.
3. **Notebook section matching** is exact title or whole slug. A near miss is refused
   with "did you mean". Anything else creates the section. Areas and projects keep
   today's prefix matching and pre-heading fallback.
4. **The wave-1 manifest was re-verified against the vault** and differs from the
   research:
   - Dropped `the_monthly`: 5 checkbox lines and 3 zorg open-task lines would fail
     `check`.
   - Dropped `sase_blog_0_legacy_notes`: it has no bullets (one numbered list) and calls
     itself obsolete.
   - Added `java` and the `vim` › `vim_plugins` › `nvim_plugins` tree. Both are rescued
     from the zorg dead end. The research's worry about `nvim_ref` attribution does not
     apply: it holds plain `- [ ]` reading checkboxes and no `#task` lines, and it stays
     under the untyped `nvim`.
5. **Notebook capture targets are opt-in.** `scan_capture_targets` keeps returning only
   task-capable notes. A separate entry point adds notebooks, used only by
   `bob capture-targets` and capture-complete route rows. Gmail routes, reference
   parents, gkeep, the gcal root check, Highlights, transfer assist, link-task pickers,
   and capture-related therefore stay closed without auditing each one.
6. **Recency comes from `[created::]` stamps, not mtimes.** File mtimes are checkout
   times in a git-synced vault.
7. **`%c` and `%t` compose with notebook notes.** Notes are bullets, and `%c` already
   captures clipboard content beneath a bullet. A screenshot into a notebook is a
   natural gesture.
8. **Sub-notes (`@nb+id`) carry no `[created::]`**, matching today's sub-bullets.
   Top-level and section notes carry one, matching section bullets.
9. **The research's open questions, answered:**
   - New notes **append**, so notebooks read oldest to newest like section bullets.
   - Non-root projects get their own catalog slice (A8).
   - All 26 manifest rows migrate now. The 15 already-parented notes cost only a
     frontmatter edit, and they make every catalog complete.
   - The hand-written "Related notes" indexes stay; the migration never rewrites note
     bodies, and the phase report lists the entries the catalogs now duplicate.
   - `_templates/new_note.md` is unchanged; `bob-cli-j7` owns its link-form bug.

## Vocabulary

| Term                   | Meaning                                                                                                        |
| ---------------------- | -------------------------------------------------------------------------------------------------------------- |
| **Notebook**           | A note with `type: "[[notebook]]"`. Never "notes file", "record", or "folder" in copy.                         |
| **Note** (note bullet) | A top-level `-` bullet in a notebook, with everything nested under it.                                         |
| **Section**            | A `##` heading in a notebook.                                                                                  |
| **Owner**              | The nearest area or project on a notebook's parent chain.                                                      |
| **Root**               | The owner's Root Parent, from the existing walk (`pomodoro_homes/resolve.rs` `resolve_note`).                  |
| **Catalog**            | The bob-written `## Notebooks` block in an area or project note. Use "catalog" only for this in notebook copy. |
| **Host**               | An area or project note that carries (or should carry) a catalog.                                              |

## The contract

### The file

```markdown
---
parent: "[[dev]]"
template: "[[new_notebook]]"
type: "[[notebook]]"
summary: Linux machine setup, shell shortcuts, and troubleshooting notes.
created: 2026-11-09T09:14:00-05:00
---

# Linux

#dev notes on Linux ← optional preamble paragraph (legacy notes keep theirs)

- Use <C-x><C-e> to edit the current command in $EDITOR [created::2026-11-09] ^edit-cmd
  - nested detail is free-form: code, links, images, sub-bullets

## Troubleshooting

- Check which process owns a port before restarting services [created::2026-11-09]
```

- Key order on files bob creates: `parent`, `template`, `type`, `summary`, `created`.
  This mirrors `render_project_note`. The migration adds keys without reordering
  existing ones.
- `type: "[[notebook]]"`. Every type form `frontmatter_type_names` accepts is valid
  (scalar, flow list, block list, bare `[[notebook]]`).
- **Title:** the first H1 before the first bullet or `##`, with wikilinks reduced to
  their display text and emphasis stripped. Otherwise the first `aliases:` entry.
  Otherwise the stem through `humanize_stem` (`highlights_ref/clip_url.rs`). Notebook
  identity is the file stem; a reparent changes placement, never the name.
- **`summary:`** is one plain line describing scope, not contents. It is written as a
  YAML string, quoted only when needed (reuse the records engine's typed YAML string
  serializer, `src/native/records/`).

| Rule                                                                                                                                              | Kind    | Enforced by                                                                                                         |
| ------------------------------------------------------------------------------------------------------------------------------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------- |
| Exactly one `parent` that resolves (path-aware) to an area, a project of any status, or a notebook                                                | hard    | creation refuses; `check` errors                                                                                    |
| Acyclic; the chain reaches an area or project                                                                                                     | hard    | `check` errors                                                                                                      |
| No checkbox line anywhere outside fenced code (`- [ ]`, `* [x]`, `1. [-]`, any status character, any depth)                                       | hard    | capture never writes one; `check` errors                                                                            |
| Areas, projects, references, and record notes never name a notebook as `parent` (reference tasks live in the parent, and notebooks hold no tasks) | hard    | `check` errors; parent pickers hide notebooks for those kinds                                                       |
| Block IDs unique within the file                                                                                                                  | hard    | capture and ID minting refuse duplicates; `check` errors                                                            |
| Lives at the vault root, because capture routes are root stems                                                                                    | hard    | `check` errors (`notebook_not_at_root`)                                                                             |
| `summary` present, one line, at most 140 characters                                                                                               | soft    | creation requires non-empty; `check` warns; the catalog shows an add-summary chip                                   |
| Shape: at most one H1 before the first bullet, one optional preamble paragraph, top-level bullets, `##` sections                                  | soft    | `check` warns (numbered lists, `====` dividers, H3+, stray paragraphs), so legacy notes migrate untouched           |
| `parent` written as quoted `"[[stem]]"`                                                                                                           | soft    | `check` warns on the unquoted `[[x]]` form (YAML reads it as a nested list)                                         |
| Transparent to root, clock, and type walks; never a Pomodoro home, Stats root, Ready lane, `## Tasks` grouping target, or review item             | derived | already true: these walks treat unknown types as `NoteKind::Other`. Add regression tests; do not add a second walk. |

### `bob notebooks` (Vault section of `bob --help`, bare command defaults to `list`)

Read `sase/memory/cli_rules.md` with `/sase_memory_read` before touching the CLI: every
long option needs a short alias, options stay alphabetical, and a bare group defaults
only to a read-only, flags-only member.

| Command                                                                                    | Purpose                                                                                                               | Exit                                    |
| ------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------- | --------------------------------------- |
| `bob notebooks [list] [-b DIR] [-f human\|json] [-r ROOT]`                                 | Tree of notebooks grouped by root, with titles, summaries, counts, and issue markers                                  | 0                                       |
| `bob notebooks check [-b DIR] [-f human\|json]`                                            | Findings grouped by notebook, plus a summary line such as `✓ 26 notebooks in 7 roots — 0 errors, 4 warnings`          | 1 on any error, 0 otherwise, 2 on usage |
| `bob notebooks sync [-b DIR] [-d] [-f human\|json]`                                        | Write every catalog; `-d` prints per-note diffs and writes nothing                                                    | 0, or 1 on a damaged catalog            |
| `bob notebooks create PARENT ID -s SUMMARY [-t TITLE] [-b DIR] [-d] [-f human\|json] [-o]` | Create a notebook plus its catalog rows in one guarded write; `-o` opens it through the existing Obsidian-open helper | 0, or 2 on refusal                      |

Human output is polished and colourful, using the `style.rs` palette and Bob's root
hues. Each root gets a header, nested notebooks hang from tree glyphs, summaries are
dimmed, and counts are right-aligned (`45 notes · 2 § · 2d`). Every JSON object carries
`schema_version: 1`, and `docs/reference/json-outputs.md` documents it. Minimum `list`
JSON:

```json
{
  "ok": true,
  "schema_version": 1,
  "notebooks": [
    {
      "stem": "git",
      "path": "git.md",
      "title": "Git",
      "summary": "Git commands, …",
      "parent": "dev_tools",
      "parent_kind": "notebook",
      "owner": "dev",
      "root": "dev",
      "breadcrumb": ["dev", "Developer tools", "Git"],
      "children": [],
      "notes": 6,
      "sections": [{ "title": "Config", "notes": 2 }],
      "last_note": "2026-11-09",
      "issues": [
        { "code": "notebook_shape", "severity": "warning", "line": 14, "message": "…" }
      ]
    }
  ],
  "catalogs": [
    { "host": "dev", "path": "dev.md", "state": "current", "notebooks": 17 }
  ],
  "counts": { "notebooks": 26, "roots": 7, "errors": 0, "warnings": 4 }
}
```

- `catalogs[].state` is `current`, `stale`, `missing`, or `damaged`. `notebook-core`
  emits `catalogs: []`; `notebook-catalog` fills it.
- `last_note` is the latest `[created::]` date anywhere in the file, or `null`.

### The catalog

**Scope.** Every area or project note lists the notebooks whose owner lies in its own
subtree, stopping at descendants that are roots themselves (an area at any depth, a
`root: true` project, or an outermost project). For a root this is exactly the requested
recursive rollup. A non-root sub-project also gets its own slice.

- Notebooks nest under their parent notebook.
- Notebooks owned by a non-root sub-project appear in that project's catalog and, in its
  root's catalog, under a `🧩 [[sub_project]]` group line.
- Order is alphabetical by title (case-insensitive) within each level. The host's own
  notebooks come first, then the 🧩 groups by stem.
- An invalid notebook (no owner, cycle) appears in no catalog. `check` reports it.

**Bytes.** The content is bob's; the _position_ is Bryan's.

```markdown
## Notebooks

<!-- bob:notebooks:begin v1 -->

> [!notebooks]
>
> - **[[ai|AI]]** — Definitions and notes on LLMs, AI assistants, and agentic coding
>   tools.
> - **[[dev_tools|Developer tools]]** — Command-line tools and dotfile setup I rely on:
>   chezmoi, tmux, git, and more.
>   - **[[git|Git]]** — Git commands, configuration, and workflow tips.
> - **[[lang|Languages]]** — Notes on the programming languages I use.
>   - **[[rust|Rust]]** — Rust features, tools, and crates worth knowing.

<!-- bob:notebooks:end -->
```

- A missing summary writes the row without the ` — …` tail.
- Titles and summaries are emitted as inert text: escape Markdown link, emphasis,
  wikilink, and HTML openers; never emit HTML.
- **First write:** the block goes immediately before the first `## Tasks` heading. With
  none, it goes at the end of the note, separated by one blank line.
- **Later writes:** only the bytes between the fences are rewritten, wherever Bryan
  moved the block (the gmail precedent: `gmail/note/parse.rs` and `merge.rs`). Output is
  byte-stable, and nothing is written when nothing changed.
- **Damaged fences** (a begin without an end, two begins, an end before its begin):
  refuse that host, report `notebook_catalog_damaged`, and never guess.
- **Empty subtree:** delete the fenced region. Delete the `## Notebooks` heading too,
  but only when its section then holds nothing but blank lines.
- `## Notebooks` is never a section-bullet target (`@dev#notebooks`). Add it beside the
  status-group headings in `is_bullet_section` (`capture/sections.rs`) and section
  completion. It sits outside `## Tasks`, so status grouping never touches it.

**Writers.**

- `bob notebooks sync`, run by hand and nightly.
- `bob notebooks create`.
- The capture batch that creates a notebook.
- The plugin's debounced trigger, which runs `bob notebooks sync`.
- `bob projects sync` must never list a notebook on a Sub-projects line. It scans only
  `[[project]]` children today; add a regression test.

### Capture: the destination decides

**The rule.** An item's route is its own `@route`, otherwise the draft's `@@route`. If
that route is a notebook, the item writes a note. A `^`-sealed `@parent^id@` with `%n`
creates a notebook. Everything else is unchanged.

Examples below assume `dev_tools.md` is a notebook and `dev.md` an area.

| Draft                                                                                 | Result                                                                                                                       |
| ------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `Try mergiraf @dev_tools` or `@dev_tools Try mergiraf`                                | Top-level note `- Try mergiraf [created::DATE]`, appended after the last top-level note before the first `##`                |
| `Try mergiraf @dev_tools#git`                                                         | Note in `## Git` (exact title or whole-slug match)                                                                           |
| `Try mergiraf @dev_tools#gi` when `## Git` exists                                     | Refused, `notebook_section_near_miss`: "no section `gi` in dev_tools.md — did you mean `#git`? Write `#\"Gi\"` to create it" |
| `Try mergiraf @dev_tools#merge-tools` with no near miss                               | Creates `## Merge Tools` at the end of the note; the preview says "new section"                                              |
| `Try mergiraf @dev_tools#"Merge tools"`                                               | Exact; creates `## Merge tools` verbatim when missing                                                                        |
| `Try mergiraf @dev_tools^mergiraf`                                                    | Named note `… [created::DATE] ^mergiraf`                                                                                     |
| `Try mergiraf @dev_tools#git^mergiraf`                                                | Named note in `## Git`. `#sec^id` is new grammar on **every** route: on an area it is a section bullet with a block ID       |
| `Faster with -j @dev_tools+mergiraf`                                                  | `- Faster with -j` appended as the last child of the bullet `^mergiraf` at any depth                                         |
| `@dev_tools` alone, then `- a` / `- b`                                                | Two top-level notes; `- text ^id` names one                                                                                  |
| `@dev_tools#git` alone, then bullets                                                  | Each bullet becomes a note in `## Git`                                                                                       |
| `@dev_tools+mergiraf` alone, then bullets                                             | Each bullet becomes a child note (`decisions:capture-bullets-mirror-the-task-block`)                                         |
| `Idea A` / `Idea B` / `@@dev_tools`                                                   | Both items become top-level notes                                                                                            |
| `Try mergiraf @dev_tools` plus first-level bullets                                    | One note whose nested bullets are the item's bullets, verbatim                                                               |
| `https://example.com/post @dev_tools`                                                 | A plain note holding the URL. No reference job; `URL @owner` stays the reading-queue route                                   |
| `Screenshot of the diff @dev_tools %c`                                                | A note with the clipboard content beneath it (`%c` composes; so does `%t`)                                                   |
| `bob capture -r dev_tools 'text'` / `-r dev_tools --section T` / `-r dev_tools -t ID` | `--route` follows the destination: top-level note, exact section (created on demand), or sub-note                            |
| `Neovim config notes and plugin picks @dev^nvim@ %n`                                  | Creates `dev_nvim.md` (summary = the item text, `# Nvim`) and rewrites the affected catalogs in the same batch               |
| …the same item plus `- First note` and `- PLUGINS` with nested bullets                | Plain bullets seed top-level notes; the ALL-CAPS bullet with nested bullets becomes `## Plugins` with those notes            |
| `Lua for my config @dev_nvim^lua@ %n`                                                 | Nested notebook `dev_nvim_lua.md`                                                                                            |
| `Fix the bug @dev` / `remembered it %n` / `Ship it @dev^ship@`                        | Unchanged: task, Pomodoro note, project note                                                                                 |

**Teaching refusals.** Each refusal names the owner route, so the fix is one edit.

| Marker or shape on a notebook route                                                                                  | Code                                                   | Copy, in Bob's voice                                                                            |
| -------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------- |
| `s:<N>`, `$<N>`, free `#type`, `!`, `=…` session suffixes, `%l`                                                      | `notebook_marker_unsupported` (names the marker)       | "dev_tools.md is a notebook; notes take no schedule. Capture the task in its home with `@dev`." |
| `@nb:id`, `&`, `*`, `~`, `<`, `>` transfers, a lone `@nb` with no bullets (task pull)                                | `notebook_holds_no_tasks`                              | "dev_tools.md is a notebook and holds no tasks to link, pull, or relate — use `@dev`"           |
| `@nb+id#drawer`                                                                                                      | `notebook_marker_unsupported`                          | "notes have no drawers"                                                                         |
| `@nb+id` whose target is a task line, a heading, inside a code fence, missing (with did-you-mean), or a duplicate ID | reuse the sub-bullet target errors, with notebook copy |                                                                                                 |
| `@nb^id@` without `%n`                                                                                               | `notebook_project_seal`                                | "dev_tools.md is a notebook and holds no projects; add `%n` to create a child notebook"         |
| `%n` with a plain route                                                                                              | `pomodoro_note_conflict`                               | unchanged                                                                                       |
| `@@nb#…`, `@@nb^…`, `@@nb+…`                                                                                         | `invalid_global_destination`                           | New copy: "`@@` declares a draft-wide route and takes no place; put it on the item: `@nb#sec`"  |
| A notebook used by an item **before** the item that creates it                                                       | `notebook_used_before_created`                         | "create dev_nvim before using it: move the `@dev^nvim@ %n` item up"                             |

**Blocks and block IDs.**

- New notes go through `format_bullet_line`, plus `append_block_id` for a named note.
  The item's own first-level bullets nest under the note with the existing sub-bullet
  indentation.
- A sub-note is `- body` with no stamp. It is inserted after the target's existing
  nested lines.
- `reject_duplicate_block_id` already checks every `^id` in the note, task or not.

**One resolver.**

- A memoized `NotebookRoutePolicy`, shaped like `RecordRoutePolicy`
  (`capture_language/record.rs`, built by `record_policy_for_draft` in
  `capture_record.rs`), serves `capture-parse`, `capture-complete`, `--dry-run`, and
  commit.
- It starts with a lexical preflight that collects each distinct candidate route,
  including the `@@` route and routes staged by creation items. Then it does one
  frontmatter read per route (plus the `##` titles of notebook routes). It is zero I/O
  when the draft has no route.
- The grammar never touches the filesystem. Items are claimed as notebook items before
  task parsing (as `classify_record_item` does), and after `@@` inheritance for
  route-less items (`inherit_global_destination` in `capture_language/draft.rs` applies
  only to tasks today). URL items are claimed before `claim_reference_item`.
- The written notebook and every ancestor read for a creation are batch read guards. A
  file retyped between plan and commit therefore aborts the whole capture.

**Wire additions** (all additive; document in `docs/reference/json-outputs.md`):

| Surface                   | Addition                                                                                                                                                                                                                                                                          |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `capture-parse`           | Modes `notebook_note`, `notebook_sub_note`, `notebook_create`. Spans reuse the existing labels (`route`, `section`, `task_block_id`, `sub_bullet_block_id`, `project_note_marker`, `directive`) with a `notebook` modifier                                                        |
| dry-run items             | `kind` `notebook_note` / `notebook_sub_note` / `notebook_create`, plus a `notebook` object: `route`, `title`, `breadcrumb`, `section`, `section_action` (`existing`, `create`, or null), `block_id`, `parent_block_id`. Creation adds a `note_files[]` entry (`action: "create"`) |
| human dry-run and success | `📓 Note → Developer tools › Git` and `📓 Note → Developer tools › new section Merge Tools` for notes; `📓 Note → Developer tools › ^mergiraf` for sub-notes; `📓 New notebook dev_nvim.md · in dev` for creation                                                                 |
| `capture-targets`         | `kind: "notebook"` rows sorted after projects, with `title`, `summary`, `owner`, and `breadcrumb`                                                                                                                                                                                 |
| `capture-complete`        | `destination_kind: "notebook"` on block-ID fields; `item_kind: "note"` on `@nb+` rows; notebook route-row badge and detail                                                                                                                                                        |

**Grammar and release.**

- `capture-notebook-notes` bumps `GRAMMAR_REVISION` (22 at planning; bump from whatever
  HEAD says). Retyping a file changes what an existing `@route` writes, and `#sec^id`
  changes the parse of `@route#sec^id` everywhere. Use a `feat(capture)!:` subject.
  While there, fix the stale `grammar: 21` mentions (`capture_parse.rs` help,
  `capture_complete/cli.rs`, `docs/reference/cli/capture-{parse,complete}.md`,
  `json-outputs.md`).
- `capture-notebook-create` bumps it again from HEAD.
- Never edit the `Cargo.toml` version or `CHANGELOG.md`
  (`decisions:bob-cli-releases-ride-a-daily-train`).
- No vault note is retyped until every host runs a bob with notebook capture; see
  `notebook-vault`.

### Compatibility

| App / bob                                    | Behaviour                                                                                                                                                                                                                                                                               |
| -------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| New / new                                    | Full notebook experience                                                                                                                                                                                                                                                                |
| Old Mac / new bob                            | Notebook route rows appear with the generic icon and work. File-under may list notebooks; choosing one is refused by bob with teaching copy (reference parents never accept notebooks). Notebook previews use the generic header. `@nb^` keeps the `new` intent, so the ID picker works |
| New Mac / old bob                            | No notebook data; nothing changes                                                                                                                                                                                                                                                       |
| Old bob on a host after the vault is retyped | Would write tasks into notebooks, which is why `notebook-vault` gates the retype on every host                                                                                                                                                                                          |

- `CaptureGrammarGate.minimumRevision` stays where it is. Notebook UI appears only when
  bob returns notebook data.
- Coordinate with epic `bob-cli-j6`. Its Mac phase `bob-cli-j6.2` is in progress, and
  its `.`→`&` fallback already shipped in bob. The collapse assist must claim a glued
  `.` after `@@route` ahead of that fallback.
- `bob-cli-j7` (`bob projects sync` ignores `[[x.md|x]]` parents) stays independent. The
  notebooks resolver is path-aware regardless.

### Out of scope (designed for, not built)

- **H2 extraction** (`bob notebooks extract`, an Obsidian "Extract section to notebook"
  command) and moving note bullets between notebooks. The vault task
  `bob.md ^h2-to-notebook` tracks this. It must reuse `bob task move`'s vault-wide
  block-link repair, because about 137 inbound block links point into `## Legacy Notes`
  sections. Nothing in this epic may assume a notebook's notes never move.
- Wave-2 and wave-3 migrations. `notebooks-wrapup` files the bead.
- `@@nb#…`, `@@nb^…`, and `@@nb+…` in the strict grammar. They stay teaching errors.

## Wave-1 migration manifest

All 26 rows migrate. Rows marked **R** rescue notes whose chains end at `org` today; the
other 15 already sit under the right area or notebook and gain a type, a summary, and a
catalog row, so every catalog is complete on day one. Parents migrate before their
children (`dev_tools` before `git`, `linux` before `athena`, `lang` before its
languages, `vim` before `vim_plugins` before `nvim_plugins`, `dog` before its two
children).

| Root  | File                 | Title (H1)          | New `parent`        | `summary`                                                                     |
| ----- | -------------------- | ------------------- | ------------------- | ----------------------------------------------------------------------------- |
| dev   | `dev_tools`          | Developer tools     | `"[[dev]]"`         | Command-line tools and dotfile setup I rely on: chezmoi, tmux, git, and more. |
| dev   | `git`                | Git                 | `"[[dev_tools]]"`   | Git commands, configuration, and workflow tips.                               |
| dev   | `linux`              | Linux               | `"[[dev]]"`         | Linux machine setup, shell shortcuts, and troubleshooting notes.              |
| dev   | `athena`             | Athena              | `"[[linux]]"`       | Upkeep, update, and recovery notes for the athena home server.                |
| dev   | `networking`         | Networking          | `"[[dev]]"`         | Networking protocols, fediverse and ActivityPub notes, and port diagnostics.  |
| dev   | `ai`                 | AI                  | `"[[dev]]"`         | Definitions and notes on LLMs, AI assistants, and agentic coding tools.       |
| dev   | `dev_ideas`          | Dev ideas           | `"[[dev]]"`         | Open, uncommitted ideas and proposals for my developer tooling.               |
| dev   | **R** `mac`          | Mac                 | `"[[dev]]"`         | macOS questions, tools, shortcuts, and new-laptop setup steps.                |
| dev   | **R** `lang`         | Languages           | `"[[dev]]"`         | Notes on the programming languages I use.                                     |
| dev   | **R** `rust`         | Rust                | `"[[lang]]"`        | Rust features, tools, and crates worth knowing.                               |
| dev   | **R** `python`       | Python              | `"[[lang]]"`        | Python language notes and tooling.                                            |
| dev   | **R** `shell`        | Shell               | `"[[lang]]"`        | Bash and zsh idioms, builtins, and completion setup.                          |
| dev   | **R** `lua`          | Lua                 | `"[[lang]]"`        | Lua language notes and references.                                            |
| dev   | **R** `java`         | Java                | `"[[lang]]"`        | Java and Kotlin idioms, Q&A, and tips collected while writing Java.           |
| dev   | **R** `vim`          | Vim                 | `"[[dev]]"`         | Vim and Neovim editing tips, settings, and command tricks.                    |
| dev   | **R** `vim_plugins`  | Vim plugins         | `"[[vim]]"`         | Plugins that work in both Vim and Neovim, with setup notes.                   |
| dev   | **R** `nvim_plugins` | Neovim plugins      | `"[[vim_plugins]]"` | Neovim-only plugins worth knowing: what each does and where it lives.         |
| sase  | **R** `sase_ideas`   | SASE ideas          | `"[[sase]]"`        | Open ideas and proposals for SASE, captured in zorg (Oct 2025 – Feb 2026).    |
| gtd   | `gtd_ideas`          | GTD ideas           | `"[[gtd]]"`         | Open ideas and experiments for improving my GTD habits and system.            |
| love  | `word_of_day`        | Word of the day     | `"[[love]]"`        | Kelly's and my word-of-the-day list, with definitions.                        |
| love  | `dog`                | Dogs                | `"[[love]]"`        | Our dogs' vet, food, and care notes.                                          |
| love  | `vet_log`            | Vet log             | `"[[dog]]"`         | Dated log of vet visits.                                                      |
| love  | `bugyi_dog_policy`   | Dog cost policy     | `"[[dog]]"`         | Our self-insurance policy for deciding on costly vet care.                    |
| abby  | `legacy_dad`         | Legacy dad notes    | `"[[abby]]"`        | Zorg-era notes on being Abby's dad, moved here 2026-11-02.                    |
| cash  | `taxes`              | Taxes               | `"[[cash]]"`        | Where tax documents live, plus per-year filing notes.                         |
| kelly | `kellys_salary_info` | Kelly's salary info | `"[[kelly]]"`       | Reference numbers for Kelly's pay and benefits; verify yearly.                |

**Edits per file, and nothing else:**

- add `type: "[[notebook]]"`;
- rewrite `parent` to the quoted value above (the zorg files use unquoted `[[x]]`;
  `legacy_dad` uses `"[[abby|dad]]"`);
- add `summary`;
- insert `# <Title>` as the first body line when the file has no H1 before its first
  bullet.

`legacy_dad` already has `# Legacy [[abby]] Notes` after its preamble. Replace that one
line in place with `# Legacy dad notes`. Bodies, preambles, `^z-…` IDs, `tick::`
children, and the `zorg_*` provenance keys all stay. No file is renamed.

**Known soft warnings to expect, not fix:**

- `vet_log` has `====` day separators.
- `python` has one non-bullet zorg done line.
- `athena` has one nested zorg `o` line.
- `mac` has a `## Legacy Notes` section with indented code fences.

## Phase details

**Every bob-cli phase:**

- Read `sase/memory/cli_rules.md` and `sase/memory/docs.md` with `/sase_memory_read`
  before touching CLI help or docs.
- Run `just` (fmt, clippy, tests) green, `just docs-check` when docs change, and
  `just docs-gen` when CLI help changes.
- Keep parse, complete, dry-run, and commit on one resolver.
- Cite `docs/notebooks.md` as the contract hub.

**bob-plugins phases:**

- Open the repo with `/sase_repo` and read its `AGENTS.md`.
- Edit `src/` fragments only (each at most 1000 lines). Never hand-edit a built
  `main.js`.
- Run `npm run build`, `npm run build:check`, `npm test` (add new test files to the
  `package.json` list), `npm run validate`, and the docs inventory checks.
- Finish with `bob plugins sync`.

**The Mac phase:**

- Open `bob-mac-capture` with `/sase_repo` and follow its `AGENTS.md`. Never land
  uncompiled Swift.
- Run `just portable-check` and the Mac typecheck path, then `Scripts/macos-ci-check`.
- After the push, watch the exact-SHA macOS run to green with `/sase_monitor`.

### notebook-core

- **Recognition:**
  - add `frontmatter_is_notebook` next to `frontmatter_is_area`/`_project`/`_article` in
    `src/native/projects/scan.rs`, and re-export it;
  - add `"notebook"` to `RESERVED_RECORD_IDS` (`src/native/records/schema.rs`) and to
    the reserved list in `docs/records.md`.
- **Module** `src/native/notebooks/` (`mod.rs`, `scan.rs`, `model.rs`, `rules.rs`,
  `cli.rs`, `output.rs`):
  - **One walk** of the vault: the `is_excluded_directory` exclusions plus `done/` and
    `_templates/`. Read frontmatter for every `.md` (the child-kind rule needs refs and
    record notes); read full bodies only for notebooks.
  - **Parents** are resolved path-aware, the way `HomeResolver` does it
    (`wikilink_path_target`, then `NoteIndex` / `VaultLinkResolver`). `[[x.md|x]]`,
    quoted, and unquoted forms all work.
  - **Owner and root** come from the existing `pomodoro_homes` resolver, so they agree
    with Stats. Never write a second walk.
  - **Per-notebook model:** the fields in the JSON above, sections with per-section note
    counts, children, and issues. A cycle is detected and reported, never looped.
- **Rules:** every hard and soft rule in the contract, with stable codes.
  - Errors: `notebook_parent_missing`, `notebook_parent_ambiguous`,
    `notebook_parent_kind`, `notebook_cycle`, `notebook_checkbox`,
    `notebook_child_kind`, `notebook_duplicate_block_id`, `notebook_not_at_root`.
  - Warnings: `notebook_summary_missing`, `notebook_summary_long`,
    `notebook_summary_multiline`, `notebook_shape`, `notebook_parent_unquoted`.
- **CLI:** `bob notebooks list` and `check`, wired as a group in `src/runner.rs` (Vault
  section; `default: list`, modelled on `records`). This covers:
  - `NativeCommand` variants and dispatch in `src/native.rs`;
  - `bob __catalog` rows (`src/native/catalog/table.rs`);
  - completion providers (`completion/tree.rs`, `kinds.rs`), including a notebook-stem
    provider for `-r`;
  - help fixtures, and the alphabetical and section-contiguity tests.
- **Teaching refusals where a notebook is offered as a home:**
  - `classify_dest` (`src/native/task_move/plan.rs`) and therefore inbox routing;
  - `check_non_candidate_note` (`src/native/parent_notes.rs`), which covers Gmail
    routes, reference and Highlights parents, gkeep, and the gcal root check.
  - Both now say "`x.md` is a notebook and holds no tasks — use its owner `@dev`"
    instead of the generic not-an-area-or-project copy.
  - Leave `scan_capture_targets` unchanged in this phase.
- **Docs:**
  - create `docs/notebooks.md`: what a notebook is, the file, title, rules, and the
    `list` and `check` CLI. Later phases append the catalog, sync, create, and capture
    sections;
  - register it in `docs/_meta/contracts.yml` and the mkdocs nav;
  - add a Notebook entry to `docs/reference/vault-format.md` "Note types" and to
    `docs/reference/glossary.md`;
  - run `just docs-gen` for the CLI reference.
- **Tests:**
  - a rules table test;
  - title derivation (H1 with wikilinks, aliases, humanized stem);
  - the three parent forms;
  - cycles and ambiguity;
  - checkbox detection that ignores fenced code;
  - the child-kind rule;
  - JSON goldens;
  - help snapshots;
  - regressions: a task in an untyped child of a notebook roots to the notebook's owner,
    and `projects sync` ignores notebooks.

### notebook-catalog

- **Renderer and region:**
  - exact bytes, scope, ordering, escaping, placement, in-place rewrite, deletion, and
    damaged-fence handling as specified above;
  - a pure function from the scan model to a per-host plan, so `sync`, `create`,
    capture, and `check` all share it;
  - CRLF and BOM preserved.
- **`bob notebooks sync`:**
  - Model it on `vault_rewrite::run` (`src/native/vault_rewrite/mod.rs`): take the
    vault-sync lock, pre-sync, check staleness, write atomically, make one commit
    (`notebooks: sync catalogs (N notes)`), then post-sync.
  - Do **not** copy `bob projects sync`'s unguarded `fs::write`.
  - `-d` prints unified diffs per host and writes nothing.
  - A run that changes nothing writes nothing and commits nothing.
- **Nightly:**
  - Add a `notebooks` step to `STEPS` in `src/native/nightly.rs` between `period notes`
    and the final `vault-sync`.
  - The step calls a lock-free core, because nightly already holds the lock.
  - Update the step help text, the root `about` string, the catalog confirm text, the
    help fixtures, and the nightly order tests (`tests/cli/vault_sync.rs`,
    `tests/cli/periods.rs`).
- **`bob notebooks create PARENT ID -s SUMMARY [-t TITLE]`:**
  - A shared `render_new_notebook` writes the contract file. The title is `-t`, else the
    humanized ID.
  - The ID uses the block-ID alphabet. The stem is `<parent>_<id>` with `-`→`_`, exactly
    `project_note_basename`.
  - The parent must be an area, a project of any status, or a notebook. The summary must
    be non-empty; over 140 characters is a warning.
  - A case-insensitive collision refuses.
  - The new file and every affected catalog host are written in one guarded transaction
    (`capture/commit.rs` `write_staged_files`, which inherits the no-replace install),
    then committed.
- **Reporting:**
  - `list -f json` fills `catalogs[]`;
  - `check` adds `notebook_catalog_stale` (warning: run `bob notebooks sync`) and
    `notebook_catalog_damaged` (error).
- **Section exclusion:** `## Notebooks` is excluded from section bullets and section
  completion for areas and projects.
- **Docs:** the catalog, sync, create, and nightly sections in `docs/notebooks.md`;
  `docs/vault-git-sync.md` or the nightly docs if they list steps; and the CLI
  reference.
- **Tests:**
  - goldens on a fixture tree: nested notebooks, a non-root sub-project group, a
    `root: true` project that stops the rollup, and an area under an area;
  - first placement with and without `## Tasks`;
  - in-place rewrite after the block moves;
  - deletion;
  - damaged fences;
  - idempotence (the second run is a byte-for-byte no-op);
  - dry-run;
  - `create` refusals;
  - the nightly step order.

### capture-notebook-notes

Read `docs/capture.md` (Routing and insertion, Global destination declaration, Section
bullets, Sub-bullets under existing tasks, Lists, Record notes),
`docs/reference/capture-syntax.md`, and the two governing decision records:
`decisions:capture-sigils-say-where-what-how` and
`decisions:capture-bullets-mirror-the-task-block`.

- **Policy:** `NotebookRoutePolicy`, as specified, threaded through
  `CaptureParseOptions` and `EditorParseOptions` beside `record_policy`. Build it once
  per invocation in `capture_parse.rs`, `capture/plan.rs` (plus the template path in
  `capture/mod.rs`), and `capture_complete`.
- **Grammar** (`src/native/capture_language/`):
  - `CaptureKind::NotebookNote { route, section, block_id }` and
    `CaptureKind::NotebookSubNote { route, parent_block_id }`;
  - claims before task parsing, after `@@` inheritance;
  - the `#sec^id` token form on every route (`tokens.rs`
    `is_task_block_id_marker_candidate` and the section token path). A quoted `#"…^…"`
    stays literal;
  - list headers;
  - the refusals table, with diagnostics in `editor_parse.rs` / `editor_classify.rs`;
  - the `invalid_global_destination` copy.
- **Planner** (`src/native/capture/notebook_note.rs`):
  - placement: append after the last note of the target region (pre-`##` region or
    section), so notebooks read oldest to newest like section bullets today;
  - section resolution: exact title or whole slug; a near miss is a selector that is a
    prefix of an existing section slug or within edit distance 2 of one (reuse the
    existing edit-distance helpers); otherwise create `## Title Case` from the slug, or
    verbatim from `#"…"`;
  - a notebook section ends at the next heading of level ≤ 2;
  - sub-note target lookup over note bullets at any depth;
  - `%c` beneath the note;
  - URL items;
  - `--route`, `--section`, and `-t` on notebooks.
- **Targets:** add `scan_capture_targets_with_notebooks`. `bob capture-targets` and
  `capture_complete/candidates.rs` `route_candidates` use it; every other caller keeps
  `scan_capture_targets`. Notebook rows carry `kind: "notebook"`, `title`, `summary`,
  `owner`, and `breadcrumb`.
- **capture-related:** notebook items get no Similar-tasks band.
- **Output and docs:**
  - output: the wire additions and human preview lines above;
  - `docs/capture.md`: a new "Notebook notes" section, rows in the grammar-at-a-glance
    tables, and a `#sec^id` sentence in Section bullets;
  - `docs/reference/capture-syntax.md`, `json-outputs.md`, and
    `docs/reference/compatibility.md` (grammar paragraph);
  - the capture section of `docs/notebooks.md`.
- **Tests** (CLI integration on a temporary vault with `BOB_DIR` and `BOB_NOW`):
  - every row of both capture tables, including `@@`, lists, URLs, `%c`, `--route`, and
    `#sec^id` on an area;
  - parse modes and spans;
  - dry-run and commit parity;
  - an all-or-nothing draft that mixes a task item and a note item;
  - a notebook retyped between plan and commit aborts;
  - duplicate IDs;
  - the grammar assertions.

### capture-notebook-create

- **Grammar:**
  - `%n` at an item edge on a `^`-sealed project capture becomes
    `CaptureKind::NotebookCreate { parent, id }`, whether the marker leads or trails;
  - lift the `%n`+route conflict for exactly this shape: the strict path at the
    `item.rs` route check, and `apply_note_directive` in `editor_parse.rs`;
  - `@parent:id@ %n` and `@parent^id@#name %n` are refused;
  - `%n` with a plain route and a bare `%n` are unchanged.
- **Planner** (`src/native/capture/notebook_create.rs`):
  - The parent must be an area, a project of any status, or a notebook, on disk or
    created earlier in the draft.
  - The file is `<parent>_<id>.md`. A disk or staged collision (case-insensitive) is
    refused with `notebook_exists`, naming the route to use instead.
  - The summary is the item text. An empty summary is refused with
    `notebook_summary_required`; over 140 characters is a warning diagnostic.
  - Content comes from `render_new_notebook`. Plain first-level bullets seed top-level
    notes with stamps (`- text ^id` names one). An ALL-CAPS bullet with nested bullets
    becomes `## Title Case` holding those notes, and duplicate titles merge (the
    `render_project_note` rule).
  - Task-ID bullets (` :id`), move bullets (`- ^id`, `- :id`, `- ^^`), `s:`, `$`, free
    `#type`, `%c`, `=…`, `!`, `&`, `*`, `~`, `<`, and `@@` absorption are refused.
- **Batch:**
  - Later items in the draft see the new route as a notebook through the policy's staged
    routes. An earlier item that uses it is refused with `notebook_used_before_created`.
  - The affected catalog hosts are rewritten in the same all-or-nothing batch with the
    `notebook-catalog` renderer.
  - The parent and owner chain are read guards.
- **Output:**
  - mode `notebook_create`;
  - a `note_files[]` entry;
  - the creation preview and success line;
  - catalog host edits appear as ordinary file changes.
- **Docs:**
  - add a "Creating notebooks" section to `docs/capture.md`;
  - update `capture-syntax.md`, `json-outputs.md`, and `docs/notebooks.md`;
  - fix the `%n` sentence tracked by `bob-cli-j3` (capture-parse modes paragraph, about
    `docs/capture.md:8327`: `#` after the seal opens Pomodoro-name completion);
  - grep for other `#`→`%n` slips near seals;
  - record the fix on `bob-cli-j3` per `sase/memory/sase_beads.md` (read it with
    `/sase_memory_read`).
- **Tests:**
  - creation under an area, a project, a done project, and a notebook;
  - seeding;
  - every refusal;
  - in-draft use after creation, and the refusal when it comes before;
  - the catalog rewritten in the same batch;
  - dry-run parity;
  - an aborted run leaves no file behind.

### capture-notebook-assist

- **Routes:** notebook rows carry the 📓 badge, the summary as detail, and the
  breadcrumb as label.
- **Sections:** on a notebook route, only `##` sections, in document order. When the
  query matches no section exactly, a final create row reads "New section “Query”". It
  inserts the bare slug when that would create the section, and the quoted `#"Query"`
  when the slug is a near miss.
- **Block IDs:**
  - `@nb^` keeps intent `new` and adds `destination_kind: "notebook"`. Suggestions come
    from the item text, avoiding every ID in the file (`collect_used` already counts
    non-task IDs; verify with a test).
  - `#sec^` gets the same field.
- **`@nb+`:**
  - Rows are note bullets with `item_kind: "note"`: ID'd bullets first, then ID-less
    ones carrying `requires_block_id` and `block_id_suggestions`.
  - Extend `bob capture-task-id` (`plan_assignment` in `capture_task_id.rs`) so it can
    mint an ID on a note bullet in a notebook. This is additive; the command name stays.
- **Directives:** the `%n` row is enabled on `^`-sealed items
  (`capture_complete/ directives.rs` greys it out on `pomodoro_note_conflict` today).
- > [!decision] collapse_assist
  >
  > Add the capture-rewrite rule `collapse_global_destination`. A `#`, `/`, or `\` glued
  > to a complete `@@route` (`is_route_token`) at the caret becomes `@route#`; `^` or
  > `.` becomes `@route^`; `+` or `,` becomes `@route+`. The normal picker then opens.
  >
  > - It is lexical and works for every route.
  > - It claims a glued `.` before the `.`→`&` fallback.
  > - `@@` absorption is untouched.
  > - Undo restores the typed text.
  > - Pasted and CLI text is never rewritten.
  > - Document it in `docs/capture.md` under `bob capture-rewrite`, and adjust the
  >   rewrite tests that pin `@@r.`, `@@cash,`, and `@@cash/` as literal.
  >
  > When false, skip the rule; the refusal copy still teaches `@nb#…`.
- **Docs:** the capture-complete and capture-rewrite sections of `docs/capture.md`, plus
  `json-outputs.md`.
- **Tests:**
  - every completion context above;
  - ID minting on a note bullet;
  - the collapse matrix, covering each character and the routes `@@r`, `@@r.`, and
    `@@cash/`;
  - non-firing cases: incomplete route, templates, paste.

### plugins-notebook-catalog

- **Card CSS** in bob-ledger-tools `styles.css`, on
  `.callout[data-callout="notebooks"]`:
  - a soft card with a `lucide-notebook-tabs` icon;
  - the callout title hidden, because the H2 names the block;
  - semibold titles, with summaries in `--text-muted` after the em dash;
  - nested rows hung from a thin tree rule;
  - 12–16 px row rhythm and a visible keyboard focus;
  - checked in light and dark themes, Reading view and Live Preview, and at mobile
    widths.
- **Hue:** a post-processor (Reading) and a CodeMirror decoration (Live Preview) set
  `data-root-hue` on the callout from `pomodoroHomeChainHue` of the host note. They use
  the `--bob-area-*` tokens and fall back to a neutral accent. Model both on the
  note-ready `## Tasks` heading chip (`220-plugin-heading-and-review.js`).
- **Live chips** (rendered, never written): fetch `bob notebooks list -f json` with
  `bobSpawnJson` behind a stale-while-revalidate cache modelled on
  `241-plugin-reading-board-data.js` (vault-generation counter, debounced revalidate).
  Per row:
  - right-aligned tabular `48 notes · 3 §`;
  - `2d` from `last_note`;
  - an amber **add summary** chip that opens the notebook's properties.

  On the `## Notebooks` heading, a subtle `↻ sync` chip appears when the host's catalog
  state is not `current`; clicking it runs `bob notebooks sync` and revalidates.

- **Notebook header:**
  - Inside a notebook, render (never write) a breadcrumb such as
    `📓 dev › Developer tools › Git`, where every crumb is a link and the root is tinted
    with its hue.
  - Add a row of child-notebook chips.
  - Both appear in Reading view and Live Preview.
  - The parent-picker ancestry helpers (`208-parent-note-picker-model.js`) are reusable.
- **Sync trigger:**
  - Run `bob notebooks sync` fire-and-forget, the way the gmail and gcal triggers run,
    on a metadataCache change to a notebook's `parent`, `summary`, or `type`, or to any
    note's `type` to or from notebook.
  - Debounce by 3 s, run at most one at a time, and skip while a sync is running.
- **Wrap-up:** a settings toggle for the chips, plugin docs, manifest version, tests
  (CSS contract test, chip model, breadcrumb model), and `bob plugins sync`.

### plugins-notebook-gestures

- **Block-ID prompt:**
  - Suggest IDs for plain note bullets: generalize `promptBlockIdExtractTaskBody` in
    **both** byte-identical copies of `015-prompt-block-id-naming.js` (block-id-prompt
    and bob-navigation-hotkeys).
  - Strip `[created::]` and other inline fields before suggesting.
  - Unblock the null paths in `095-prompt-suggestion-resolver.js`.
  - Update `scripts/test-prompt-block-id-naming.cjs`,
    `test-block-id-prompt-prefill.cjs`, and `docs/block-id-defaults.md`.
- **Navigation:** `getChildNoteInfo` returns `kind: "notebook"` with a `notebook-tabs`
  icon in the child-note picker and any other kind display.
- **Parent picker** (Ctrl+Shift+-): for an area, project, reference, or record note,
  hide notebooks. For a notebook, offer only areas, projects, and notebooks. Show the
  notebook icon.
- **Create notebook here** command (active note must be an area, project, or notebook):
  - a modal with a required summary, an ID pre-filled from the summary by the same
    suggestion function, and an optional title;
  - runs `bob notebooks create <stem> <id> -s … [-t …]` and opens the new note;
  - refusals show as notices.
- **Task move:** confirm with a test that Ctrl+Shift+M and inbox routing still never
  offer notebooks.

### mac-notebook-capture

- **Decoding:**
  - `CaptureBlockIDIntent` gains an unsupported case for unknown wire values, instead of
    `.link`. Update the pinned test in
    `Tests/CaptureCoreTests/BlockIDDecodingTests.swift`.
  - An unsupported intent opens plain new-ID typing, never the link picker.
  - Decode `destination_kind` and `item_kind` with `decodeIfPresent`.
- **Icons:**
  - `kind == "notebook"` maps to `book.closed` in every kind map: `noteKindIcon`,
    `noteKindHeaderIcon`, `ParentTaskPickerPresentation.noteKindSubtitle`, the Similar
    band, and pull-in.
  - Route rows show the summary as detail.
- **File-under:** the reference-parent list keeps only inbox, area, and project kinds.
- **Previews:**
  - notebook note headline `📓 Note → <title> › <section>`, with a "new section" badge;
  - sub-note `📓 Note → <title> › ^id`;
  - creation `📓 New notebook <path> · in <parent>`, with the `note_files[]` body
    preview reusing the record preview's View Markdown disclosure;
  - success lines;
  - `friendlyKindLabel` gives "Notebook note" and "Notebook".
- **`+` picker:** `item_kind: "note"` rows render without a checkbox glyph. Choosing an
  ID-less row mints its ID through `bob capture-task-id` before insertion, exactly as
  tasks do.
- **Coordination:** rebase on whatever `bob-cli-j6.2` (Mac alias escapes) has landed.
  The collapse assist needs no app code: the app already applies bob's capture-rewrite
  edits.
- **Docs:** the app's `docs/capture-panel.md` and `docs/keyboard.md` where they list
  kinds or aliases.

### notebook-vault

- **Precondition:**
  - Every host (athena, apollo, and the Mac; read `sase/memory/tailnet.md` with
    `/sase_memory_read`) must run a bob with notebook capture: `bob notebooks check -h`
    succeeds, and `bob capture --dry-run 'x @<fixture notebook>'` against a temporary
    `BOB_DIR` plans a note.
  - Update a lagging Linux host with `just install` from current master.
  - For the Mac, ask Bryan through `/sase_questions` to run `just install-all`.
  - Retype nothing until all three pass.
- **Vault conventions:** read the `obsidian` reference memory with `/sase_memory_read`
  first. Vault changes are made directly in `~/bob`, and `bob vault-sync` carries them.
- **Vault files:**
  - `notebook.md` type note: `parent: "[[type]]"`, `created:`, and two sentences in the
    house style of `area.md` and `article.md`, followed by an embedded
    `![[notebooks.base]]`.
  - `notebooks.base`: every notebook with title, summary, parent, and created, grouped
    by parent.
  - `_templates/new_notebook.md`: Templater, with the parent set to the active file as
    `"[[stem]]"`, `template`, `type`, an empty `summary` to fill, `created`, and an H1
    from the file name.
  - Add `summary` as a text property in `.obsidian/types.json` if it is absent.
- **Migration:**
  - Snapshot each manifest file's digest, block-ID set, and `#task` count.
  - Pre-check every row: no checkbox lines, and no area, project, ref, or record child
    names it as parent. Skip and report any row that fails.
  - Apply the per-file edits, parents first, one root per batch with a `bob vault-sync`
    after each batch.
  - Keep the hand-written "Related notes" indexes (dev, ks, linux, lang, dog, dev_tools,
    vim): the migration never rewrites note bodies. List each entry the new catalogs now
    duplicate in the phase report so Bryan can prune by hand.
- **Prove:**
  - `bob notebooks sync`: catalogs appear in dev, sase, gtd, love, abby, cash, and
    kelly;
  - `bob notebooks check` reports zero errors;
  - every migrated notebook appears in exactly one root catalog;
  - every block-ID set is unchanged;
  - vault-wide `#task` counts are unchanged;
  - a second `sync` is a no-op.
- **Report:** put the warning list in the phase report.

### notebooks-wrapup

- **End-to-end walkthrough** in a temporary vault (`BOB_DIR`, `BOB_NOW`):
  - `bob notebooks create`;
  - capture with each note form;
  - `@parent^id@ %n` with seeded sections;
  - `@@nb` bulk;
  - `sync`, `check`, and `list -f json`;
  - a refused task marker;
  - the collapse rewrite through `bob capture-rewrite --cursor`.

  Paste the transcript into the phase report.

- **Docs audit:**
  - `docs/notebooks.md` reads as the hub;
  - every notebook mention in `capture.md`, `capture-syntax.md`, `json-outputs.md`,
    `vault-format.md`, `compatibility.md`, `records.md`, and the CLI reference agrees
    with shipped behaviour;
  - `just docs-check` passes.
- **Memory**, through `/sase_memory_write`, applying only the accepted decisions. Follow
  the house strand shape: frontmatter `keyword` and `aliases`, no H1, prose wrapped near
  88 columns, and links as `[[slug]]` / `[[decisions:slug]]` with frontmatter-looking
  `[[…]]` kept in inline code. Run `sase memory init` afterwards.
  - > [!decision] memory_glossary_notebook
    >
    > Create `glossary:notebook` (aliases `notebook note`) from this draft. Tighten it
    > against shipped behaviour:
    >
    > ```markdown
    > A Bob vault Markdown file with frontmatter `type: "[[notebook]]"` that keeps a
    > named collection of related notes (observations, explanations, ideas, logs)
    > without the obligations of a new [[area-note]] or [[project-note]]. Its body is
    > note bullets, optionally grouped under `##` sections below an H1 title; each
    > bullet may carry nested detail, a `[created::]` stamp, and a stable `^id`. It
    > never holds a `#task` or any checkbox, so it is never a task's residence, a
    > Pomodoro home, a Ready lane, a review item, or a [[root-parent]]; root, clock, and
    > type walks pass through it to the nearest area or project above it (its owner).
    >
    > Its single `parent` names an area note, a project note, or another notebook, and
    > the chain always reaches an area or project. A one-line `summary:` states what
    > belongs there. Every area or project lists the notebooks beneath it, down to the
    > next root parent, in a machine-owned `## Notebooks` catalog that
    > `bob notebooks sync` maintains: its content is bob's, its position is Bryan's.
    >
    > In [[capture]], the destination decides. On a notebook route, `@nb` adds a
    > top-level note, `@nb#section` adds one to that `##` section (exact match, created
    > on demand), `@nb^id` or `@nb#section^id` adds a named note, `@nb+id` adds a child
    > of the note bullet `^id`, and a draft-wide `@@nb` sends every unrouted item there
    > as a note. Task-only markers are refused with a rewrite to the owner's route. A
    > [[project-note-capture]] seal plus the `%n` directive creates one:
    > `@parent^id@ %n <summary>` writes `<parent>_<id>.md`; a bare `%n` stays a Pomodoro
    > note.
    >
    > Not a [[reference-note]], a record note, a [[task-drawer]], a [[period-note]], the
    > Pomodoro note `%n` writes, or an Obsidian folder. See `docs/notebooks.md` and
    > [[decisions:notebook-routes-write-notes]].
    > ```
  - > [!decision] memory_notebook_decision
    >
    > Create `decisions:notebook-routes-write-notes` ("A Notebook Route Writes Notes;
    > `@@` Stays Draft-Wide") in the house shape of existing decision strands (read two
    > first).
    >
    > - **Applies to:** bob-cli, bob-plugins, Bob Mac Capture, and the vault.
    > - **Claim:**
    >   - notebooks hold only note bullets and never a checkbox;
    >   - on a notebook route, the existing `@nb` forms write notes;
    >   - `@@` stays the draft-wide declaration and never takes a place suffix;
    >   - `@parent^id@ %n` creates a notebook;
    >   - bob owns each area or project's `## Notebooks` catalog content, and Bryan its
    >     position.
    > - **Why:** cite the research verifications V1–V6. The rejected alternatives are:
    >   - item-local `@@nb`;
    >   - suffixed `@@` forms, which would reverse the 2026-10-31 removal
    >     (`bob-cli-ey.1.1.1`);
    >   - a `#notebook` tag;
    >   - sub-areas;
    >   - folders;
    >   - record types;
    >   - a Dataview-only index;
    >   - a `^^` seal.
    > - **Cost:** retyping changes what `@x` writes (the grammar bump); hosts need the
    >   new bob before any retype.
    > - **Reopens when:** Bryan wants tasks in notebooks, or a second `@@` scope.
    > - **Evidence:** the research ref, this plan, and the landing commits.
  - > [!decision] memory_notebook_decision
    >
    > Mark `decisions:capture-sigils-say-where-what-how` with
    > `metadata.status: superseded-in-part`. Append
    > `decisions:notebook-routes-write-notes` to its `superseded_by`, keeping the
    > existing targets. Add one back-link sentence: "`%n` on a `^`-sealed project
    > capture now creates a notebook; see [[decisions:notebook-routes-write-notes]]."
    > Change nothing else in the accepted body.
  - > [!decision] memory_glossary_corrections
    >
    > In `glossary:capture`, extend the list of what an item may create with "a note in
    > a [[notebook]], or a notebook".
  - > [!decision] memory_glossary_corrections
    >
    > In `glossary:project-note-capture`, add one sentence: with `%n`, the `^` seal
    > creates a [[notebook]] instead (parent area, project, or notebook; the item text
    > becomes its `summary`). Keep the rest.
  - > [!decision] memory_glossary_corrections
    >
    > Only if `collapse_assist` shipped: in `glossary:capture-alias`, drop `@@` routes
    > from the `/` literal list. Name the `@@` collapse beside `@@` absorption among the
    > rewrites that are not aliases.
- **Wave-2 bead:** file one task bead through `/sase_new_task` ("Notebook migration wave
  2"). It carries the census and the decisions Bryan must make:
  - `work` and `work_ideas` (211 bullets);
  - the Google-era meeting and `*_cls` notes (archive, or notebooks under `job`);
  - the zorg design history (under `bob`?);
  - `people`, `shortcuts` (strip its `# ----- TABLE -----` pseudo-H1), `define`, and
    `remarkable`;
  - `the_monthly` (triage its checkboxes) and `sase_blog_0_legacy_notes`;
  - the `eat` hub (coordinate with record notes);
  - `tick::` children in the idea queues.
