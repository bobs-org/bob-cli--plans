---
tier: epic
title: Rename the article note type to prose
goal: "The Bob note type for authored writing is named prose everywhere a person reads
  it: the type link, the bob prose CLI, Bob Prose in Obsidian, the docs, and the vault.
  Every legacy article note, bob articles call, plugin id, and JSON consumer keeps
  working unchanged, and prose-section extract/inline is planned on top of the rename.

  "
decisions:
  type_name:
    ask: Which name should replace article for the note type?
    choices:
      prose:
        Reads right as a note, a section, and a CLI noun; a mass noun, so copy says
        prose note
      page:
        Countable, but every note is a page, refs already mean PDF pages, page section
        reads oddly
    default: prose
    why:
      The only candidate that works at file and section scale with no vault or CLI
      collision
    answer: prose
  numbering_default:
    ask: Should prose notes keep numbering their headings by default?
    choices:
      keep_on:
        Default unchanged; turn numbering off per note with the new toggle command
      flip_off:
        Numbers only with section_numbers true; a ! break; sase_blog_contents opts in
        first
    default: keep_on
    why:
      Keeps the house style with no contract break; extraction will write false
      explicitly
    answer: keep_on
  memory_glossary:
    ask: Add a Prose Note glossary term (legacy alias article note) to SASE memory?
    memory:
      - glossary:prose-note
    default: false
    answer: true
  memory_decision:
    ask:
      Add a decision record for the prose type's name, ownership rule, and permanent
      aliases?
    memory:
      - decisions:prose-notes-are-owned-prose
    default: false
    answer: true
phases:
  - id: engine-rename
    title: Rename the Rust article engine to prose
    depends_on: []
    size: small
    description:
      "engine-rename: move the article engine module and its Rust identifiers to prose
      names, with byte-identical CLI, JSON, help, and docs output."
  - id: prose-cli
    title: bob prose with permanent legacy aliases
    depends_on:
      - engine-rename
    size: medium
    description:
      "prose-cli: one predicate reads both type spellings, writers emit [[prose]], bob
      prose becomes canonical with bob articles as a hidden alias, copy and docs move to
      prose, and a section-numbers flag plus a host probe ship; wire vocabulary stays
      frozen."
  - id: prose-plugins
    title: Bob Prose in Obsidian
    depends_on:
      - prose-cli
    size: medium
    description:
      "prose-plugins: four plugins detect both spellings, display names and command
      titles change under unchanged ids, a Toggle section numbers command lands, plugin
      docs are fixed, and the bundles deploy with a currency marker."
  - id: prose-vault
    title: Host gate, vault cutover, and canonical plugin calls
    depends_on:
      - prose-cli
      - prose-plugins
    size: small
    description:
      "prose-vault: pass the host-currency gate, retype the one live note and rename the
      vault's type files, switch plugin calls to bob prose, and apply only accepted
      memory decisions."
  - id: prose-sections
    title: Plan prose-section extract and inline
    depends_on:
      - prose-vault
    size: xlarge
    description:
      "prose-sections: author and propose the epic for bob prose extract/inline and its
      Obsidian keymap, bound by the research requirements, with the research's open
      questions embedded as decisions."
proposed_by: bbugyi200.apollo.research.6i.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-26 10:56:22
status: wip
---

- **PROMPT:**
  [prompts/202611/prose_note_type.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/prose_note_type.md)

# Plan: Rename the article note type to prose

## Context

Bryan wants the `article` note type renamed to something generic, because he will use it
for any file whose job is to hold prose. Two goals sit behind the name:

- Soon, a section inside an area, project, or prose note should implicitly be a **prose
  section**, so that one keymap can turn it into a full prose note (and back).
- Eventually, tasks and projects should be reversible with little or no loss. Projects
  can have article notes as children today, and that blocks reversal.

**Required reading for every phase:**
`research:202611/prose_rename_and_section_extraction/prose_rename_and_section_extraction.md`
(read it with `sase artifact read`). Bryan endorsed every requirement it recommends,
with one exception: memory edits happen only under this plan's accepted memory
decisions. This epic carries the report's Phase 1 (the rename) in full. Its last phase
plans the report's Phase 2 (extract and inline). The report's Phase 3 (reversible
projects) is out of scope. See [Out of scope](#out-of-scope).

### The name

Bryan proposed `prose` but was unsure, and asked for a harder look. Five independent
researchers checked about 25 candidates and all chose `prose`. This plan's own pass
agrees. The deciding tests:

1. **It names the membership rule.** A prose note is a file whose body is free-form
   writing. "Article", "essay", "post", and "memo" name genres, and genres are exactly
   what made `article` too narrow.
2. **It works at every scale the roadmap needs.** "Prose note" names the file, "prose
   section" names the extraction unit, and `bob prose extract` reads naturally. Most
   candidates fail at the section scale ("page section", "doc section", "writing
   section").
3. **It completes the structural contrast** among task-free notes: a notebook holds
   bullets, a record holds schema fields, a prose note holds continuous writing.
4. **It collides with nothing that matters.** No `src/` identifier, CLI noun, glossary
   term, vault file, or `bob ref` kind uses it. Bob already uses the word in its plain
   sense (the plugins' `proseField` prompt control, "prose links", "never changes
   prose"), and that sense points the same way.

Other candidates and why they lost:

- **Countable but overloaded:** `page` (every note is a page, and refs talk about PDF
  pages), `doc`/`document` (collides with `~/bob/docs.md`, the repo's `docs/`, and the
  docs site), `entry` (reads like a diary entry or a `DirEntry`).
- **Genre words** (same problem as `article`): `essay`, `memo`, `post`, `draft`,
  `writeup`, `paper`, `story`, `letter`.
- **Already taken in Bob:** `note`, `text`, `section`, `sheet` (Record Card), `card`,
  `leaf` (Leaf Project), `copy` (Copy Chip), `body`.
- **Ambiguous, ornate, or wrong:** `writing` (reads like a note about writing),
  `freeform` (an adjective, and it promises no formatting when bob does format),
  `longform` (short letters aren't long-form, and an Obsidian plugin has that name),
  `folio`, `manuscript`, `composition`.

The one cost is that `prose` is a mass noun. Copy always says "prose note", "prose
notes", or "prose section", and never "a prose" or a bare "prose" for the type.
`bob prose <verb>` follows the `bob ref` and `bob mail` precedent.

> [!decision] type_name = page Read `page` wherever this plan says `prose` as the type's
> name. That covers `type: "[[page]]"`, the CLI group `bob pages` (plural, like
> `bob notebooks`), the module `src/native/pages/`, the contract `docs/pages.md`, the
> CLI page `reference/cli/pages.md`, the plugin display name "Bob Pages", the vault
> files `page.md`, `pages.base`, and `_templates/new_page.md`, the probe
> `page-type-read`, the marker `bob-page-type-v1`, and the copy "page note(s)" and "page
> section". Everything else in this plan is unchanged.

### Definition to record

A **prose note** is a note whose body is free-form authored Markdown that the author
owns: paragraphs, headings, lists, tables, quotes, and code. bob may format its layout
and (unless turned off) its section numbers, but never changes a word or adds structure.
A prose note holds no `#task` lines, no `^prj`/`^ref` anchors, and no second type. The
test is the note's **primary role**: a project or area that contains prose sections
stays a project or area. Do not define the type as "any file that contains prose",
because every area, project, and ref contains prose (research A1).

### Verified current state (bob-cli `4dd787bc`)

- **The engine.** It lives in `src/native/articles/` (14 files, about 5.7k lines) and is
  matched by `frontmatter_is_article` (`src/native/projects/scan.rs:364`). Adoption
  writes `TYPE_LINE = type: "[[article]]"` (`marker.rs:116`). The mixed-type check is
  `types.iter().any(|name| name != "article")` (`marker.rs:47`, `:88`). The CLI is
  `bob articles` (`runner.rs:1284`, `articles/cli.rs`), with subcommands adopt, format,
  list, and outline. Bare `bob articles` runs list.
- **Other type-sense sites.**
  - Capture section targeting (`capture/sections.rs`, through `destination_is_article`).
  - The completion kind `Kind::ArticleNote` (`completion/kinds.rs`, `providers.rs`,
    `present.rs`, `tree.rs`).
  - Catalog rows `articles-list` and `articles-outline`, the slot `article-note`, and
    two `Excluded` entries (`catalog/table.rs`).
  - The nightly step `articles` (`nightly.rs:42`).
  - The reserved record ids (`records/schema.rs:15,19,2301`) and the built-in type list
    (`props/roles.rs:352`).
  - `bank/month.rs`, which imports `split_physical_lines` from the engine.
  - Help fixtures (`tests/fixtures/help/root-*.txt`) and `tests/cli/articles.rs`.
- **Wire contract.** ArticlePlan is `schema_version: 1` and growth-only, and clients
  reject unknown versions (`docs/articles.md`). `summary.articles` is a wire key, and
  the refusal codes include `not-article`. The only client is the `bob-articles` plugin.
- **Plugins (bob-plugins).**
  - `bob-articles` (id kept; name "Bob Articles"; commands `format-article` and
    `convert-to-article`; spawns `bob articles <action> --stdin …`).
  - `bob-command-mode` (`170-sheet.js:22` `cmSheetArticleType`, `160-sheet-rank.js:70`
    and `:658`, and the slot list in `040-catalog.js:38`).
  - `bob-navigation-hotkeys` (`208-parent-note-picker-model.js:23`).
  - `bob-ledger-tools` (`242d-plugin-album-entrypoints.js:14`).
  - Many test vectors use `type: [[article]]` only as a neutral fixture type.
- **Vault (`~/bob`).**
  - `[[article]]` appears only in `sase_blog_contents.md` (the one live note, which uses
    numbered `##`/`###` headings), `_templates/new_article.md`, `AGENTS.md:15`, and
    write-once `.bob/ref-sources/**` history.
  - The type note `article.md`, `articles.base`, and the `type.md` embed complete the
    set.
  - No hotkeys bind `bob-articles:*`, and the plugin has no `data.json`.
  - `~/bob/articles.md` is an unrelated, zorg-imported reading list of web articles.
- **Elsewhere.** Bob Mac Capture's only hits are the web-article Refs kind, so it is
  unaffected. Chezmoi mentions only the `bob-articles` plugin id in `plugins.sync`,
  which stays.
- **Docs.** `docs/reference/vault-format.md` omits the type entirely.
  `docs/reference/plugins/*.md` is mounted from `bob-plugins/docs/`. Precedent for a
  rename with redirects and a hidden alias: commit `d0603a5d` (`bob themes` →
  `bob types`). It kept catalog row ids, changed paths and labels, and added mkdocs
  redirects.
- **Hosts.** Hosts install bob from master, ahead of the crates.io train.
  `scripts/dev/bob-host-currency --probe … --plugin id=marker` checks athena, apollo,
  and the Mac (precedent: the true-case drawers rollout,
  `plan:202611/true_case_drawers.md` §9).

### Governing decisions

- `decisions:bob-cli-releases-ride-a-daily-train`: no `Cargo.toml` or `CHANGELOG.md`
  edits. Use `!` only for a real CLI or JSON break. Hosts track master.
- `decisions:reference-sources-are-preserved-write-once`: never edit the
  `.bob/ref-sources/**` copies that mention `[[article]]`.
- `decisions:mac-capture-is-a-thin-client`: the Mac gets nothing to change. It renders
  what bob returns, and no Mac contract names the type.
- `decisions:notebook-routes-write-notes`: notebooks stay bullet collections. A prose
  note is a different kind, never a notebook variant.
- `decisions:decision-rules-are-read-at-planning-time`: the `prose-sections` planner
  must read the roster again and cite what governs extraction, including
  `decisions:drawers-keep-their-names`, `decisions:task-reqs-are-met-with-their-task`,
  `decisions:unblocked-tasks-are-offered`, and `decisions:projects-block-from-prj-line`.

No decision governs the article type's name, and nothing in the roster contradicts the
rename.

## Rules every phase follows

1. **The rename is semantic and site by site, never a `sed`.** "Article" means four
   things across these repos: (1) this note type, which is renamed; (2) the
   web-capture/ref kind (`highlights_ref`, `RouteHint::Article`, `kind: "article"`,
   `web_article`, the Mac Refs kinds, `~/bob/articles.md`), which stays; (3) the HTML
   `<article>` element, which stays; (4) the English word ("a/an" in `capture_parse.rs`,
   the stop-word list in `capture_related/tokenize.rs`), which stays.
2. **One predicate, two spellings, forever.** `[[prose]]` is canonical and `[[article]]`
   is a permanent legacy spelling of the same type. Both are read through one helper. A
   note that lists both spellings is not `mixed-type`. Writers emit only `[[prose]]`.
3. **Aliases are permanent and silent** (`cli_rules`). `bob articles …` becomes a hidden
   alias of `bob prose …` with byte-identical stdout, stderr, exit codes, and JSON, and
   no deprecation output. Help, diagnostics, logs, and commit subjects print only the
   canonical path.
4. **Wire vocabulary is frozen; only display copy changes** (research: "JSON contract
   version" and "Plugin identity"). These stay exactly as they are:
   - ArticlePlan `schema_version: 1` and every JSON key, including `summary.articles`.
   - Every refusal and finding code (`not-article`, `mixed-type`, `holds-anchor`, …).
   - The catalog row ids `articles-list` and `articles-outline`, and the slot
     `article-note`.
   - The plugin id `bob-articles` and its command ids `format-article` and
     `convert-to-article`.
   - The runtime CSS class `.bob-article` and the plugin's settings keys.

   So the rename needs no `!`. The one exception is
   [numbering_default = flip_off](#decisions-in-this-plan).

5. **Copy convention.** Say "prose note", "prose notes", or "prose section", never a
   bare or countable "prose" for the type. The plugin display name is "Bob Prose".
6. **Neutral fixtures stay.** Test vectors that use `type: [[article]]` only as an
   arbitrary type (the `pomodoro_homes` and `pomodoro_clock` vectors, the ledger-tools
   notebook tests, the Rust `pomodoro_homes` tests) stay as they are. They now also
   exercise the legacy spelling.
7. **Wait for hosts before switching.** Nothing in the vault is retyped, and no plugin
   calls `bob prose`, until `scripts/dev/bob-host-currency` shows every host is current
   (`prose-vault` step 0).

## Decisions in this plan

- **`type_name`**: see [The name](#the-name).
- **`numbering_default`.** With `keep_on`, nothing about defaults changes, and the new
  `--section-numbers` flag and the Toggle section numbers command give each prose note a
  one-keystroke opt-out. This follows the research's origin rule: hand-created prose
  keeps `true`, and extraction (in `prose-sections`) writes `section_numbers: false`
  explicitly.

> [!decision] numbering_default = flip_off In `prose-cli`, `parse_options` treats an
> absent `section_numbers` as `false`. Before committing that change, add
> `section_numbers: true` to the frontmatter of `~/bob/sase_blog_contents.md`, following
> the vault `AGENTS.md` commit rules. That keeps its numbers under every bob version.
> The marker block in `docs/prose.md` says "false by default", and N9 is reworded to
> match. The commit subject is `feat(prose)!: …`, with a `BREAKING CHANGE:` footer
> saying that prose notes without `section_numbers` are no longer numbered. In
> `prose-vault`, the new template carries no numbering key.

- **`memory_glossary`, `memory_decision`.** Bryan approves memory edits explicitly. Both
  default to off, and `prose-vault` step 3 runs only the accepted ones.

## Phases

### engine-rename: rename the Rust article engine to prose (bob-cli)

A pure refactor with no observable change. After this phase, every remaining "article"
inside the engine is a string literal, a legacy wire name, or a legacy-spelling test,
which makes `prose-cli`'s copy pass mechanical.

1. `git mv src/native/articles src/native/prose` (its `tests/` subdirectory moves with
   it). Change `mod articles;` to `mod prose;` in `src/native.rs`, and update every
   `crate::native::articles::` path. Today those are in `bank/month.rs`,
   `capture/sections.rs`, `completion/providers.rs`, `completion/tree.rs`, and
   `nightly.rs`.
2. Rename the Rust identifiers:
   - Types: `ArticlePlan` → `ProsePlan`, `ArticleOptions` → `ProseOptions`,
     `ArticleContext` → `ProseContext`, `ArticleTypeInsert` → `ProseTypeInsert`, and
     `AdoptDecision::AlreadyArticle` → `AlreadyProse`.
   - Functions: `frontmatter_is_article` → `frontmatter_is_prose` (and its re-export in
     `projects/mod.rs`), `with_article_type` → `with_prose_type`,
     `destination_is_article` / `article_heading_matches` → `destination_is_prose` /
     `prose_heading_matches`, completion `article_notes` → `prose_notes`, and
     `run_articles_step` → `run_prose_step`.
   - Enum variants: `Kind::ArticleNote` → `Kind::ProseNote` and
     `NativeCommand::Articles` → `NativeCommand::Prose`.
   - Rename any other `article`-named private items in the module the same way.
3. `git mv tests/cli/articles.rs tests/cli/prose.rs` and update `tests/cli/main.rs`.
4. **Must not change:** `COMMAND_NAME` (`"articles"`), any string literal that is
   printed, serialized, or matched (including `"[[article]]"`), refusal codes, catalog
   rows and slots, or the nightly step name. Don't rename serialized struct fields. If a
   field rename helps readability, pin it with `#[serde(rename = "…")]` and prove the
   JSON is unchanged.
5. **Verify:**
   - `just all` passes.
   - `just docs-gen` leaves `git status` clean apart from the rename itself, so help
     goldens and the generated CLI reference are byte-identical.
   - `rg -n -i article src/native/prose` lists only literals, comments, and tests.
6. Commit as `refactor(prose): rename the article engine module to prose`.

### prose-cli: bob prose with permanent legacy aliases (bob-cli)

1. **One predicate.** In `src/native/projects/scan.rs`, add `PROSE_TYPE = "prose"`,
   `LEGACY_ARTICLE_TYPE = "article"`, and `is_prose_type_name(name)`. Make
   `frontmatter_is_prose` match either spelling, scalar or list, with today's
   normalization.
   - Route `marker.rs`'s mixed-type checks (`:47`, `:88`) and the `mixed-type` message
     through the helper. The message names the spelling the note actually uses.
   - Capture section targeting (`destination_is_prose`) picks up the change
     automatically. Add a capture test that `@note#Section` heading-identity matching
     works on a `[[prose]]` note.
2. **Writers.**
   - `TYPE_LINE` becomes `type: "[[prose]]"`.
   - `bob prose adopt` on a note of either spelling is the existing "already" no-op,
     with copy "already a prose note". It does not retype legacy notes.
   - Add `prose` to the reserved record ids (all three lists in `records/schema.rs`;
     keep `article`) and to the built-in type list in `props/roles.rs`. Without that, a
     `[[prose]]` note reads as a custom record type and raises the schema-discovery
     diagnostic.
   - Confirm with tests that `props` and `note_catalog` classify a `[[prose]]` note
     exactly as they classify `[[article]]` today (kind `note`). Add no new wire kind.
3. **CLI.**
   - `COMMAND_NAME = "prose"`. Add `Alias { from: "articles", to: &["prose"] }` to
     `ALIASES` in `runner.rs`, in sorted order.
   - Rewrite the runner entry's `about`, the group's `about`/`long_about`/`after_help`,
     and each subcommand's copy for prose notes. Name the legacy spelling once in
     `long_about`: "`[[article]]` notes still count".
   - Keep root-help order alphabetical within the Vault section (… `props`, `prose`,
     `query` …), and regenerate `tests/fixtures/help/root-*.txt`.
   - Add an alias test modeled on the `themes` → `types` one. For `list`, `outline`,
     `adopt` (preview), and `format -s … -f json`, `bob articles …` and `bob prose …`
     must give identical stdout, stderr, and exit codes.
4. **Copy.** Every user-visible string in `src/native/prose/` moves to prose-note copy:
   notices ("Prose note formatted · …"), refusal and finding messages (`not-article` →
   "not a prose note; run `bob prose adopt`", `holds-anchor` → "prose notes cannot be
   projects or refs"), and list and outline labels. Codes stay unchanged. Also update:
   - **Nightly:** the step name becomes `prose notes`, the blurb becomes "Report prose
     notes that need formatting", and the `about` text and step list change to match.
   - **Catalog** (`catalog/table.rs`):
     - Paths become `["prose", …]`. Labels become "Prose notes" and "Prose note
       outline".
     - Keywords gain `prose` and keep `articles` so Command Mode still finds the rows.
     - Row ids, the `article-note` slot, and `KNOWN_SLOTS` stay.
     - The `Excluded` paths become `["prose", "format"]` and `["prose", "adopt"]`, with
       reasons that name Bob Prose's commands.
   - **Completion:** the candidate group label for prose notes and the doc comments.
5. **`--section-numbers` flag.** `bob prose format NOTE -n|--section-numbers <on|off>`
   writes `section_numbers: true|false` into the note's frontmatter. It replaces an
   existing `section_numbers:` line, including one with a `bad-option` value, or inserts
   one directly after the `type:` line, using adopt's insertion helper as the model. It
   then numbers or strips headings per N9, all in one plan.
   - It conflicts with `--all` and `--check` (clap errors).
   - It works with `--stdin`, where the plan's line edits carry the frontmatter edit so
     the plugin applies one undoable transaction, and with `--dry-run`.
   - Setting the current value on an already formatted note reports unchanged.
   - The JSON contract only grows: no new keys, and `schema_version` stays 1.
   - Tests: on→off strips owned tokens, off→on restores them, idempotence, the stdin
     plan, insertion versus replacement, and the conflicts.
6. **Host probe.** Add a `prose-type-read` probe to `scripts/dev/bob-host-currency`:
   - It builds a throwaway vault under the host's temp directory holding one
     `type: "[[prose]]"` note with an `## Intro` section.
   - It passes only when the installed bob runs the canonical `bob prose` path and
     treats the note as a prose note, for example when
     `bob prose outline -b "$vault" probe -f json` lists `Intro`.
   - Update the usage text, the known-probe list, and `probe_detail`, then pass
     `just check-scripts`.
7. **Docs** (read the `docs.md` reference memory first).
   - `git mv docs/articles.md docs/prose.md` and retitle it "Prose notes". It opens with
     the [definition](#definition-to-record) and states the legacy `[[article]]`
     spelling and the frozen wire vocabulary (rule 4) in a short "Names and
     compatibility" section. Document `--section-numbers` and keep the N/F rules intact.
     `prose.md` (not `prose-notes.md`) names the type for both scales, so a later
     prose-sections contract needs no URL move.
   - Add mkdocs redirects `articles.md: prose.md` and
     `reference/cli/articles.md: reference/cli/prose.md`. Update the contracts entry in
     `docs/_meta/contracts.yml` (stem `prose`), the mkdocs nav (optional contracts list,
     generated contracts nav, and "bob prose" under Vault), and the `docs/README.md`
     table.
   - Update the type-sense mentions in `docs/develop/contracts.md`,
     `docs/reference/json-outputs.md` (command names and links; keep
     `schema_version: 1`), `docs/vault-git-sync.md:431` (the nightly list),
     `docs/capture.md:8068` (section targeting: `[[prose]]`, with `[[article]]` still
     accepted), and `docs/command-catalog.md:94` (`article-note` is the legacy slot name
     for prose notes).
   - Add a "Prose notes" section to `docs/reference/vault-format.md`, next to Notebook
     notes. It is the only first-class type missing there.
   - Keep `https://bobs-org.github.io/articles/` in `docs/_meta/url_contract.txt`; the
     redirect makes it resolve. Never delete an entry.
   - Run `just docs-gen` (this regenerates `docs/reference/cli/prose.md` and the CLI
     index) and `just docs-check`.
8. **Verify** with `just all` and `just docs-check`. Then review
   `rg -n -i article src/native/prose src/native/catalog src/native/completion src/native/nightly.rs`:
   every remaining hit must be legacy wire vocabulary, the legacy spelling, or a test of
   it.
9. **Commit** as `feat(prose): rename the article note type to prose`. Under `flip_off`
   it is `feat(prose)!:` with the footer. Record the commit SHA on the phase bead for
   `prose-vault`.

### prose-plugins: Bob Prose in Obsidian (bob-plugins)

Open the repo with `/sase_repo` (`sase repo open bob-plugins`) and read its `AGENTS.md`.
Edit the `src/` fragments and run `npm run build`; never hand-edit a generated
`main.js`.

1. **bob-articles.**
   - **Manifest:** `name: "Bob Prose"`, a prose-note description, and a version bump per
     bob-plugins convention. `id` stays `bob-articles`.
   - **Detection:** `articleHasArticleType` returns true for `prose` or `article`,
     scalar or list. Internal `article*` identifiers can stay; don't rename helpers into
     the existing `proseField` namespace.
   - **Copy:** command names become "Format prose note" and "Convert note to prose
     note…" (ids unchanged). Update notice fallbacks (for example "bob skipped this
     prose note.") and the settings heading and descriptions. The "Article typeface"
     label becomes "Prose typeface"; its storage key and the `.bob-article` class stay.
   - **Spawn argv stays `["articles", …]`** in this phase, so every host's bob keeps
     working. `prose-vault` switches it after the gate.
   - **New command `toggle-section-numbers`** ("Toggle section numbers"), available only
     on prose notes:
     - It reads the current `section_numbers` value from the metadata cache: `false`
       turns numbering on, anything else turns it off.
     - It runs `bob articles format --stdin PATH --section-numbers on|off -f json` and
       applies the plan exactly as Format does, with the same staleness checks and one
       undoable transaction. The notice comes from bob's plan.
     - If bob rejects the flag (an older host bob), it shows bob's error and changes
       nothing.
2. **bob-command-mode.**
   - `cmSheetArticleType` accepts both spellings, and prose notes get the existing
     note-kind rows.
   - Row titles become "Format this prose note" and "Outline this prose note". The
     `format-article` command ref, the `articles-outline` row id, and the `article-note`
     slot stay.
   - Add a "Toggle section numbers" vault row for prose notes that points at
     `bob-articles:toggle-section-numbers`.
3. **bob-navigation-hotkeys / bob-ledger-tools.** Add `"prose"` next to `"article"` in
   `PARENT_RULE_NON_RECORD_TYPES` (`208-parent-note-picker-model.js`) and in the album
   entrypoint type list (`242d-plugin-album-entrypoints.js`). Each list gets a test
   showing `[[prose]]` behaves like `[[article]]`. The neutral fixtures stay (rule 6).
4. **Marker.** Add the comment
   `// Marker searched by the prose-vault phase: bob-prose-type-v1` to one source
   fragment of each changed plugin, so it lands in each bundle.
5. **Docs.** In `docs/bob-articles.md` (the filename follows the id):
   - Retitle it "Bob Prose" and describe prose notes.
   - Fix the wrong "formats reading notes" sentence.
   - List the renamed commands and the new toggle.

   Update the command-mode doc line that lists "article notes". Run `npm run docs:gen`
   and `npm run docs:check`.

6. **Tests.** In `scripts/test-bob-articles.cjs`, add `prose` scalar, list, and
   mixed-spelling detection, plus the toggle command (on, off, metadata read, bob error
   path). Add a command-mode sheet case for a prose note. Run `npm run build`,
   `npm test`, and `npm run validate`.
7. **Deploy** with `bob plugins sync`, following the refusal handling in `AGENTS.md`;
   never `-F` on your own. Record the bob-plugins commit and the marker on the phase
   bead.

### prose-vault: host gate, vault cutover, and canonical plugin calls (bob-cli, bob-plugins, vault)

0. **Gate.** Run `scripts/dev/bob-host-currency --probe prose-type-read`, with
   `--plugin <id>=bob-prose-type-v1` for each of `bob-articles`, `bob-command-mode`,
   `bob-navigation-hotkeys`, and `bob-ledger-tools`. The `tailnet.md` reference memory
   names the hosts. On failure, follow the drawers precedent (§9):
   - **A host's bob is behind:** propose its update through `/sase_gate`, using that
     host's usual install path (`docs/guides/install-everything.md`).
   - **A host lacks the plugin markers:** propose `bob plugins sync` there through
     `/sase_gate`.
   - **The Mac is offline:** ask Bryan through `/sase_gate` to confirm its state, and
     record the answer.
   - **Waiting:** wait with `/sase_monitor`, bounded at 36 h.

   Store the passing table as a SASE artifact (`/sase_artifact`). Change nothing in the
   vault before it passes.

1. **Vault cutover** in `~/bob`. Follow the vault `AGENTS.md`: inspect `git status`
   first, stage and commit only your files with `git add <files> && git commit`, and
   never use `/sase_git_commit` there. Make these edits:
   - `sase_blog_contents.md`: change only `type: "[[article]]"` to `type: "[[prose]]"`.
   - `git mv article.md prose.md`. Keep `parent` and `created`, add
     `aliases: [article]`, and replace the body with: "Prose notes link to this note as
     their `type`." plus the [definition](#definition-to-record), ending "Maintain them
     with `bob prose`; notes typed `[[article]]` still count."
   - `git mv _templates/new_article.md _templates/new_prose.md`, then set
     `template: "[[new_prose]]"` and `type: "[[prose]]"`.
   - `git mv articles.base prose.base`.
     - The filter becomes an `or` of `note.type == link("prose")` and
       `note.type == link("article")`.
     - The display name becomes "Prose note" and the view becomes "📝 Prose notes".
   - `type.md`: change the heading to `## Prose notes ([[prose.base]])` with
     `![[prose.base]]`.
   - `AGENTS.md:15`: "Prose notes use `type: "[[prose]]"` (legacy `[[article]]` still
     counts); use `bob prose adopt` and `bob prose format` to maintain section numbers
     and layout."
   - Never touch `.bob/ref-sources/**` or `articles.md` (the web-article reading list).
   - **Verify:**
     - `bob prose list` shows `sase_blog_contents`.
     - `bob prose format -c sase_blog_contents` exits 0, so its numbers are unchanged.
     - `rg -l '\[\[article\]\]' ~/bob -g '!.bob/**'` lists only `AGENTS.md` and
       `prose.md` (the legacy mentions).
2. **Canonical plugin calls.** In bob-plugins, the bob-articles spawn argv becomes
   `["prose", …]` (format, adopt, and the toggle). Update the spawn tests, then run
   `npm run build`, `npm test`, and `bob plugins sync`.
3. **Memory.** Use `/sase_memory_write` and change only what an accepted decision
   covers. With both decisions off, the declined-change rule applies.

> [!decision] memory_glossary Add the glossary strand `glossary:prose-note`:
>
> - Frontmatter: keyword "Prose Note", aliases "prose note" and "article note".
> - Body: the [definition](#definition-to-record), plus the legacy `[[article]]`
>   spelling and `bob articles` alias.
> - What it is not: a [[notebook]], a [[record-note]], a [[reference-note]], a
>   [[task-drawer]], or an [[annotation-section]]. A [[project-note]] or [[area-note]]
>   with prose sections stays one.
> - A pointer to `docs/prose.md`.

> [!decision] memory_decision Add the decision strand
> `decisions:prose-notes-are-owned-prose` (status accepted). It uses the usual sections:
> Applies to (bob-cli, bob-plugins, the vault), Claim, Why, Cost, and Reopens when.
>
> - **Claim:** the type is named `prose` and defined by ownership and primary role.
>   `[[article]]` and `bob articles` are permanent aliases. Wire vocabulary keeps its
>   legacy names. "Sections implicitly use the type" means derived extraction
>   eligibility, never in-place formatting or a stored section type.
> - **Evidence:** cite the research report and the `prose-cli` commit.

After any memory edit, run `sase memory init`, `just docs-gen` (it regenerates the docs
glossary and decisions pages), and `just docs-check`.

### prose-sections: plan prose-section extract and inline (planning handoff)

This phase is `xlarge`. Use `/sase_plan` to author and propose an **epic** plan for the
research report's Phase 2. Do not implement directly. Before planning:

- Read the research report (Phase 2 and adjustments A2, A3, A5, A6).
- Read `docs/prose.md` as `prose-cli` left it.
- Read the decisions roster with `sase memory read roster:decisions`.

**Binding requirements** (Bryan endorsed these; carry them into the child plan):

- **Architecture.**
  - `bob prose extract NOTE --section IDENTITY` and `bob prose inline NOTE` are a
    Rust-owned, preimage-guarded, all-or-nothing, link-repairing planner in
    `src/native/prose/`, under a **new**, separately versioned conversion contract.
    ArticlePlan is never repurposed.
  - Base the writes on `vault_rewrite`, adding tested create/trash handling and
    crash-recovery evidence.
  - The word-token invariant holds across every multi-file plan.
  - The Obsidian plugin is a thin keymap client.
- **Eligibility is derived, never stored** (A2).
  - **Eligible:** an authored, non-structural section (or the subtree at the cursor) in
    an area, project, or prose note, and bullet-only sections too.
  - **Never eligible:** `## Tasks` and its status groups, generated catalogs, subtrees
    holding `#task` lines, `^prj`/`^ref` anchors, or conflict markers, and notebooks,
    records, refs, and daily notes in v1.
  - Use the real Markdown classifier.
  - Refuse with a named reason, and never convert the surrounding note.
- **Ship the inverse with the extract** (A3).
- **Provenance** (A5): extraction records the origin parent and the heading identity.
  Missing provenance means root prose, which is never poured into a task.
- **Lossless, testably** (A6): `inline(extract(S)) == S` byte for byte, apart from
  bob-owned number tokens and seam blank lines. Anything that would be dropped (unknown
  frontmatter keys, colliding block ids) refuses or previews.
- **Extract details.**
  - Write `<owner>_<slug>.md` with `parent`, `type: "[[prose]]"`, `created`, `title`
    (the exact heading bytes), the provenance fields, and an explicit
    `section_numbers: false`.
  - Promote headings, refusing on depth overflow.
  - Repair links in every direction, reusing the inbound machinery.
  - Refuse ambiguous duplicate headings and footnote or definition references that cross
    the boundary.
- **The stub left behind** defaults to the heading plus `- [[stem|Title]]`, with an
  embed bullet as the alternative.

**Embed as decisions in the child plan** (the research's open questions):

- The stub form: a link bullet or an embed bullet.
- The keymap: a new `Ctrl+Shift+E` (verified unbound), or overloading the task/project
  conversion gesture by cursor context.
- The intro prose above a project's first `##`: extract it as `## Overview`, or refuse.
- Whether to add a generated `## Prose` host catalog. If yes, it obligates the reversal
  catalog-skip fix.
- A memory decision for a **Prose Section** glossary term, written per
  `/sase_memory_write` (default off, since Bryan approves memory explicitly).

Keep the report's Phase 3 out of the child plan. Do design the provenance fields so
Phase 3 can consume them. Close this phase once the child plan is proposed (it reports
Staged).

## Out of scope

- **The report's Phase 3, reversible projects.** It covers fixing the pre-existing
  reversal blockers (`###` status groups under `## Tasks`, `[!notebooks]`/`[!albums]`
  catalog skips, req/subtask asymmetry, project-only frontmatter) and the carry/re-adopt
  rules for prose children. Those blockers are independent of prose, and their reversal
  design needs its own plan.
- Renaming the plugin id or directory `bob-articles`. That would need a manual vault
  cutover and buys nothing functional; do it later as a standalone chore if wanted.
- An ArticlePlan `schema_version` bump, or renaming wire keys and refusal codes.
- The Note Composer extract stopgap. It would be one-way and would sidestep Bob's link
  repair, which A3 rules out.
- Any Bob Mac Capture change. It was verified unaffected.

## Done when

- `bob prose …` is canonical.
- `bob articles …` behaves byte-identically and appears in no help text.
- Notes of both spellings format, list, outline, and capture-target identically.
- Every JSON payload is unchanged apart from display strings.
- Obsidian shows Bob Prose with renamed commands and a working Toggle section numbers
  command.
- The vault has `prose.md`, `prose.base`, and `_templates/new_prose.md`, and
  `sase_blog_contents.md` is typed `[[prose]]` with its numbering intact.
- The plugin calls `bob prose`.
- Every host passed the currency gate.
- The prose-sections epic is proposed.
- Only the accepted memory decisions changed memory.
