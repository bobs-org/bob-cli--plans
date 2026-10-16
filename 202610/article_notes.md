---
tier: epic
title: "Article notes: bob-owned section numbers and calm layout"
goal: "Vault notes typed `[[article]]` get house-style section numbers (1. / 1.1 /
  1.1.1) and a layout-only format from one bob engine. It runs from `bob articles`, from
  agents, and from Obsidian on save. Heading links follow every renumber, no word ever
  changes, and articles read like well-set documents. `sase_blog_contents.md` is the
  first adopted article.

  "
decisions:
  memory_glossary:
    ask: Add an "Article Note" glossary term once the feature ships?
    memory:
      - glossary:article-note
    default: false
    answer: false
  memory_decision:
    ask:
      Record a decision that article section numbers are bob-owned source text in house
      style?
    memory:
      - decisions:article-numbers-are-bob-owned
    default: false
    answer: false
phases:
  - id: engine
    title: Pure article formatter core
    depends_on: []
    size: medium
    description:
      "engine: bob-cli src/native/articles/ pure core: article marker and refusals,
      block scanner, section levels and numbering with the ownership token, layout
      rules, word invariant, idempotency, golden fixtures, and a read-only vault corpus
      test."
  - id: links
    title: In-note links, findings, outline, and the plan contract
    depends_on:
      - engine
    size: medium
    description:
      "links: heading identity and in-note link refresh, the report-only findings, the
      outline model (words, stubs, stale numbers), and the ArticlePlan JSON v1 model
      with bob-owned notice text."
  - id: capture
    title: Number-aware capture section selectors in articles
    depends_on:
      - engine
    size: small
    description:
      "capture: in article notes, `@route#prefix` and `--section` match headings by
      de-numbered identity as well as raw title, so captures keep finding renumbered
      sections."
  - id: cli
    title: bob articles group with format, list, and outline
    depends_on:
      - links
    size: medium
    description:
      "cli: register the `bob articles` Vault group (bare = list) with format (check,
      dry-run, stdin, json, all), list, and outline; vault discovery, preimage-checked
      atomic writes, colored output, completion, help, docs/articles.md, and CLI tests."
  - id: adopt
    title: bob articles adopt with preview
    depends_on:
      - cli
    size: small
    description:
      'adopt: `bob articles adopt NOTE` inserts `type: "[[article]]"` plus the first
      format, previews by default with replaced-number reporting, writes with -w, and
      serves the plugin through --stdin JSON.'
  - id: inbound
    title: Inbound heading links, nightly check, and memory
    depends_on:
      - adopt
    size: medium
    description:
      "inbound: rewrite links from other notes when an article's headings change (CLI
      writes, plugin plans), report unrepairable ones, add a report-only nightly step,
      and apply any accepted memory decisions."
  - id: vault
    title: Vault type note, template, base, and first adoption
    depends_on:
      - inbound
    size: small
    description:
      "vault: add article.md, articles.base, the type.md block, the new_article
      template, and the vault AGENTS.md line, then adopt sase_blog_contents.md with bob
      and commit the vault."
  - id: plugin
    title: bob-articles plugin that formats on save and leave
    depends_on:
      - inbound
    size: medium
    description:
      "plugin: new bob-plugins bob-articles plugin: runtime article class, format on
      save, leave, and command as one undoable transaction, inbound link application,
      the convert-to-article preview modal, notices, settings, tests, deploy, and
      enable."
  - id: look
    title: Article reading look and live section signals
    depends_on:
      - plugin
    size: medium
    description:
      "look: scoped article typography, muted tabular section numbers in Live Preview
      and Reading view, hanging numbers on wide panes, debounced stale-number hints, and
      the section status-bar item."
proposed_by: bbugyi200.apollo.research.1o.linker.w0
decided_by: agent
decided_via: cli
create_time: 2026-10-16 10:53:01
status: wip
---

# Plan: Article notes — bob-owned section numbers and calm layout

## Context

Bryan wants to mark specific vault notes as **articles**: long-form content with
sections, such as blog posts and design docs. The first one is
`~/bob/sase_blog_contents.md`, the prose draft of the `sase_blog` project. bob should
then treat these notes differently, starting with auto-formatting and with section
numbers that start at 1 and nest for sub-sections.

The research report on this topic compared five designs and recommended one. Read it
with `sase artifact read file:explicit:4c8f1cf8b674728e9e4d3429 "<why>"`. Bryan accepted
every requirement it recommends. This plan adopts those requirements, fills in the
details they left open, and splits the work into phases.

The example note today has frontmatter with `parent` and `created` but no `type`. It
opens with a preamble that contains `[1]: TODO …` pseudo-footnotes. Its body has one `#`
heading, seven `##` headings, two `####` headings that skip a level, and one `##`
heading with a trailing space.

### Design principles

1. **One engine.** bob-cli (Rust) is the only implementation of the article rules. The
   Obsidian plugin is a thin client: it sends a buffer to bob and applies the plan bob
   returns. This is the same rule as `decisions:mac-capture-is-a-thin-client`.
2. **Layout, never language.** Formatting may change heading markup, section numbers,
   blank lines, and trailing whitespace. It never changes the sequence of words, and a
   runtime invariant enforces that before every write.
3. **Numbers are source text that bob owns.** Section numbers are written into the
   Markdown. That way Obsidian's Outline pane, heading search, `[[note#` autocomplete,
   `git diff`, agents, and PDFs all mean the same thing by "§1.3".
4. **Explicit opt-in; report rather than guess.** Only `type: "[[article]]"` turns the
   feature on. When bob is unsure about something, it reports it instead of fixing it.
5. **Never while typing.** Writes happen on save, when you leave the note, on a command,
   or from the CLI, and each write is a single undoable step.
6. **Calm beauty.** Articles get a reading measure, serif prose, and muted tabular
   numbers. Signals appear only when something changed or is about to.

## Design

### 1. Marker and options

```yaml
---
parent: "[[sase_blog]]"
type: "[[article]]"
created: 2026-10-15T15:10:41-04:00
# Both optional:
# title: "Why I built SASE"     # publication title (default: the file name)
# section_numbers: false        # keep the layout rules, remove bob's numbers (reversible)
---
```

- **One type** covers blog posts, design docs, and essays. There are no other knobs.
- **Matching uses the existing frontmatter type matcher** in
  `src/native/projects/scan.rs`, which already handles scalar, flow-list, block-list,
  and bare-wikilink forms. Two changes:
  - Generalize the hard-coded `project`/`area` bare-wikilink cases in
    `flow_list_contains`/`block_list_contains` into one rule that works for any type
    name.
  - Add `frontmatter_is_article` next to `frontmatter_is_project` and
    `frontmatter_is_area`. Existing callers must keep their exact behavior; add a test
    for that.
- **Refusals.** In each of these cases bob skips the note with a reason code and a
  message you can act on. It never formats part of a note.

  | Code                   | Condition                                                                                                                   | Message direction                             |
  | ---------------------- | --------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------- |
  | `not-article`          | `type` does not include `[[article]]`                                                                                       | "run `bob articles adopt`"                    |
  | `frontmatter-unclosed` | the opening `---` never closes                                                                                              |                                               |
  | `mixed-type`           | `type` also names another kind, such as `[[project]]`                                                                       |                                               |
  | `holds-anchor`         | a `^prj` or `^ref` block anchor                                                                                             | "articles cannot be projects or refs"         |
  | `holds-tasks`          | `#task` task lines; the message lists their line numbers                                                                    | "move them to the parent note (Ctrl+Shift+M)" |
  | `conflict-markers`     | Git `<<<<<<<` / `>>>>>>>` lines                                                                                             |                                               |
  | `excluded-directory`   | the path is under `_conflicts`, `_templates`, `_generated`, `.obsidian`, or `.git` (`ALWAYS_EXCLUDED_NOTE_DIRECTORY_NAMES`) |                                               |

  Task lines, anchors, and conflict markers count only when they appear outside fenced
  code, inline code, math, and comments. A blog post about bob may legitimately show
  `- [ ] #task` inside a code fence. Refusing on tasks also prevents a collision: bob's
  task machinery looks for a section titled exactly `Tasks`, and numbering would rename
  it.

- **`section_numbers`** accepts `true` or `false`. Any other value produces a
  `bad-option` finding, and numbering stays on.
- **`title`** feeds the `title-heading` finding and the `list` and `outline` headers.
- **The CSS class is never written to YAML.** The plugin adds it at runtime.

### 2. Section numbering

| Rule                 | Behavior                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| N1 What is a section | **Counts:** an ATX heading (`#`–`######`, indented 0–3 spaces) in the top-level body, or a single-line Setext heading. An empty heading (`##`) is still a section. **Does not count:** anything inside frontmatter, fenced code, `$$` math blocks, `%%` comment blocks, `<!-- -->` comments, HTML blocks (which run until the next blank line), blockquotes and callouts, or list-item and footnote continuation lines (indented lines that follow a list item before a line at column 0). |
| N2 Preamble          | Text before the first section is the unnumbered lede.                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| N3 Levels            | The base is the shallowest section level in the note, and it is kept. A section's new level is one more than the new level of the nearest earlier section whose _original_ level was shallower; if there is none, the new level is the base. This can only pull headings up, never push them down. Each repair is counted as `promoted`.                                                                                                                                                   |
| N4 Counting          | depth = new level − base + 1. Each depth has its own counter, and a shallower heading resets the deeper counters. Every depth is numbered, starting at 1. A section deeper than depth 4 gets a `deep-section` finding.                                                                                                                                                                                                                                                                     |
| N5 Format            | Depth 1 is written `1.`; deeper sections are `1.1`, `1.1.1`. The heading line is `#…# NUMBER TITLE` with one space after each part. The title's bytes, including inline markup, are kept exactly.                                                                                                                                                                                                                                                                                          |
| N6 Ownership         | Before numbering, bob removes exactly one leading token matching `^(\d{1,3}\.\|\d{1,3}(\.\d{1,3})+\.?)(?:[ \t]+\|$)` from the title. A bare number without a dot is never removed, so `2024 taxes` and `30 Burger` keep their numbers.                                                                                                                                                                                                                                                     |
| N7 Escape            | Write `1\.5 million users`. It renders as `1.5 …` and never matches N6.                                                                                                                                                                                                                                                                                                                                                                                                                    |
| N8 Links             | See §4.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| N9 Numbers off       | With `section_numbers: false`, bob removes the owned tokens and writes none. Turning numbering back on restores them. Both states are idempotent.                                                                                                                                                                                                                                                                                                                                          |

**The token rule matches Bryan's house style.** About 3,400 headings across his docs,
research, and plans repos use `1.` at the top and `1.1` below. It also catches the
numbers agents type by habit: `## 1.4 New` counts as stale and is replaced.

**One risk remains:** `## 1.5 million users` loses its `1.5`. Four things contain it:

- The adoption preview lists every token it will replace.
- When a replaced token's part count does not match the heading's depth (for example
  `3.` on a depth-2 heading), the `number-replaced` finding names it in the save notice.
- A single `u` undoes the format.
- git keeps the history.

**Worked example.** After adoption, `sase_blog_contents.md` begins:

```markdown
[b]: TODO: Update commit/line counts!

# 1. The Abstractions

## 1.1 Gates -> Plans -> Epics

TODO

…

## 1.3 Agent CLIs

TODO

### 1.3.1 Skills, **memory**, provider interface, NOT hooks

TODO
```

The notice reads
`Article formatted · 10 sections numbered · 2 headings promoted (#### → ###)`. The
preamble and the pseudo-footnote lines stay byte-identical. Every section sits under
`1.` because the draft has a single top-level section. That is accurate: deciding
whether to promote the H2s is the author's call, not the formatter's.

### 3. Formatting rules

**Fixed automatically.** Every edit is line-level, and lines bob does not change stay
byte-identical.

| Rule | Behavior                                                                                                                                                                                                                                               |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| F1   | Exactly one blank line between the frontmatter and the body. The frontmatter's own bytes are not touched. This matches the one rule enabled in Obsidian Linter.                                                                                        |
| F2   | Each section heading line becomes `#…# NUMBER TITLE`. Indentation, extra spaces or tabs after the hashes, closing `#`s, and trailing whitespace are removed. `## C#` keeps its `#`.                                                                    |
| F3   | Skipped heading levels are closed by pulling headings up (N3).                                                                                                                                                                                         |
| F4   | Section numbers (N4–N6, N9).                                                                                                                                                                                                                           |
| F5   | One blank line before and after every section heading. There is no extra blank line before a heading that opens the body (F1 already supplies one), and none after a heading that ends the file.                                                       |
| F6   | A run of two or more blank lines collapses to one when the next non-blank line starts a top-level block (at column 0 and not continuing a list). Runs inside fences, math, comments, or HTML, and runs between items of the same list, are left alone. |
| F7   | Trailing spaces and tabs are removed, except inside fences, math, comments, and HTML blocks. Strict line breaks are off in this vault, so two trailing spaces add nothing; a `\` hard break still works.                                               |
| F8   | No leading blank lines, and exactly one final newline. Each line keeps its LF or CRLF ending; inserted lines use the ending of the file's first line. A UTF-8 BOM is kept.                                                                             |
| F9   | Heading links inside the note are refreshed (§4).                                                                                                                                                                                                      |
| F10  | A single-line Setext heading becomes ATX and is counted as `setext`. A multi-line Setext heading is left alone and reported as `setext-multiline`. A `---` after a list item, a quote, or a table is a thematic break, not a Setext underline.         |
| F11  | One blank line before and after each top-level fenced code block. A fence can interrupt a paragraph, so this never changes how the note renders. It does not apply inside lists or quotes, or after an unclosed fence.                                 |

When a code fence never closes, everything after its opening line is left untouched and
the note gets an `unclosed-fence` finding.

**Reported, never fixed (findings).** Each finding has a `code`, a `severity` (`warning`
or `info`), a 1-based `line`, a `message`, and a `hint`. bob writes all of that text.

| Code                     | What it reports                                                                                                                                                                                                                                                                                                                              | Suggested fix                           |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------- |
| `pseudo-footnote`        | A `[label]: …` line at column 0 that is not a footnote (`[^label]:`) and whose destination is not a URL or path (`://`, `mailto:`, `#`, `/`, `./`). It renders as text only by accident, or silently becomes a hidden link definition that turns `SASE[1]` into a link. The example note's `[1]`, `[2]`, `[a]`, and `[b]` lines all qualify. | Use `[^label]`                          |
| `broken-heading-link`    | A heading link inside the note that matches no heading                                                                                                                                                                                                                                                                                       | Fix the link or the heading             |
| `ambiguous-heading-link` | A heading link inside the note that matches more than one heading                                                                                                                                                                                                                                                                            | Fix the link or the heading             |
| `inbound-heading-link`   | Another note links to a heading this article no longer has (phase `inbound`)                                                                                                                                                                                                                                                                 | Repoint the link                        |
| `deep-section`           | A section deeper than depth 4                                                                                                                                                                                                                                                                                                                | Consider flattening                     |
| `unclosed-fence`         | A code fence that never closes                                                                                                                                                                                                                                                                                                               |                                         |
| `title-heading`          | The first section repeats the file name or `title:`; Obsidian already shows the inline title                                                                                                                                                                                                                                                 | Delete the heading                      |
| `number-replaced`        | A replaced token whose part count did not match its heading's depth                                                                                                                                                                                                                                                                          | Escape it, as in `1\.5`, if it was text |
| `setext-multiline`       | A multi-line Setext heading                                                                                                                                                                                                                                                                                                                  | Convert it by hand                      |
| `bad-option`             | An invalid `section_numbers` value                                                                                                                                                                                                                                                                                                           |                                         |

**Never done.** These are rejected outright, not deferred:

- Reflowing paragraphs or adding semantic line breaks.
- Title-casing headings.
- Smart typography.
- Aligning table pipes.
- Normalizing emphasis, list, or thematic-break markers.
- Moving or renumbering footnotes.
- Adding block IDs to headings.
- Generating a table of contents.
- Reordering sections.
- Editing frontmatter beyond F1 and the single `type` line that `adopt` adds.

### 4. Heading identity and link refresh

A heading's **identity** is its title with bob's number token (N6) removed, compared
case-insensitively with whitespace collapsed. bob compares both the raw title and a
plain form with emphasis, inline-code, and wikilink markup stripped, because Obsidian's
link text can drop that markup. Three features share this one rule:

- in-note link refresh;
- inbound link refresh (phase `inbound`);
- capture section selectors (phase `capture`).

Put the rule in one `pub(crate)` helper next to the token grammar.

**Resolving a heading fragment.** bob tries three steps in order:

1. **The pre-format text.** Use the first heading whose text _before this format_ equals
   the fragment. That is exactly what Obsidian resolved before the edit, so it stays
   correct even when two sections share a title (for example two "Examples").
2. **The identity.** Otherwise, use the single heading whose identity equals the
   fragment's identity. This repairs links left stale by earlier runs.
3. **Report.** Otherwise, report `broken-heading-link`, or `ambiguous-heading-link` when
   several headings match. bob never guesses.

For a nested path such as `#A#B`, each later segment is resolved inside the subtree of
the segment before it.

**Rewriting a link.** bob replaces or inserts only the fragment's leading number token:
`[[#Beads]]` becomes `[[#1.2 Beads]]`, and `[[#1.4 Beads]]` becomes `[[#1.5 Beads]]`.
The rest of the fragment, the alias, and the link form stay byte-identical. When
numbering is off, links lose their tokens the same way.

**Links covered:**

- `[[#…]]`;
- `[[note#…]]` and `[[path/note#…]]` when they name this note;
- embeds, `![[…#…]]`;
- Markdown links `[x](#…)` and `[x](note.md#…)`, keeping their percent-encoding.

**Never touched:** block links (`#^id`), and any link inside code, math, or comments.

**The word invariant.** Before any write, bob compares the sequence of non-whitespace
tokens in the input and the output. Before comparing, it applies these steps to both
texts:

- On section-heading lines, remove the hashes, any closing hashes, and the single number
  token.
- Drop Setext underline lines.
- Remove the number token from heading-link fragments.

If the sequences differ, bob writes nothing and reports error `words-changed`, with the
first differing token in context. bob then formats the output a second time; any edits
from that second pass mean error `not-idempotent`.

### 5. Architecture

```text
             ┌─────────── bob-cli: the only implementation ───────────┐
 bob-articles│ articles::format(text, &ArticleContext) -> ArticlePlan │◀── bob articles format/adopt
 plugin ────▶│  scan blocks → sections & levels → numbers → layout    │    (Bryan, agents, nightly)
 (buffer on  │  → in-note links → findings/outline → verify           │
  stdin, JSON│ vault layer: discovery, inbound links, atomic writes   │
  plan back) └────────────────────────────────────────────────────────┘
```

- **The core is pure:** text in, plan out, no I/O. The CLI adds vault discovery,
  preimage-checked atomic writes, and inbound-link rewriting. The plugin adds a single
  CodeMirror transaction.
- **Suggested layout under `src/native/articles/`.** Keep every file under about 1,000
  lines.
  - `mod.rs`: module docs and dispatch.
  - `marker.rs`: the article marker and refusals.
  - `scan.rs`: line regions.
  - `sections.rs`: sections, levels, numbering, the ownership token, and identity.
  - `layout.rs`, `links.rs`, `findings.rs`, `outline.rs`.
  - `verify.rs`: the word invariant and the idempotency check.
  - `plan.rs`: the model and its serde shape.
  - `cli.rs`, `render.rs`, `vault.rs`.
  - `tests/`: Markdown fixture pairs.
- **Reuse what exists. Do not duplicate it.**
  - `src/native/markdown.rs`: `fence_marker`, `closes_fence`, `atx_heading`,
    `setext_underline`, `strictly_closed_frontmatter_end`, `split_line_ending`, and the
    HTML-comment and blockquote helpers. Move the snapshot's private
    `multiline_comment_lines` here if the scanner needs it.
  - `note_snapshot::resolve::parse_link_target`, for link splitting.
  - `vault_links::{NoteIndex, VaultLinkResolver}`, for resolving notes and link targets
    with near-miss suggestions.
  - `collect_done::archive::atomic_write`.
  - The `similar` crate, for diffs and line hunks.
  - `style::Styler` and `env::bob_dir`.
- **Dead code between phases.** The repo gate rejects dead code (`-D unused`). The
  `engine` and `links` phases mark the new module with
  `#![allow(dead_code)] // wired by the cli phase`, following the existing precedent;
  phase `cli` removes it.

### 6. ArticlePlan JSON (schema_version 1)

Every mode emits this one shape: one note, `--all`, `--stdin`, and `adopt`.

```json
{
  "schema_version": 1,
  "notes": [
    {
      "path": "sase_blog_contents.md",
      "status": "changed",
      "reason": null,
      "sha256": "<sha256 of the input text>",
      "edits": [{ "line": 18, "delete": 1, "insert": ["# 1. The Abstractions", ""] }],
      "changes": {
        "numbered": 10,
        "renumbered": 0,
        "unnumbered": 0,
        "promoted": 2,
        "setext": 0,
        "links": 0,
        "layout": 9
      },
      "replaced": [{ "line": 30, "token": "3.", "number": "1.2" }],
      "notice": "Article formatted · 10 sections numbered · 2 headings promoted (#### → ###)",
      "findings": [
        {
          "code": "pseudo-footnote",
          "severity": "warning",
          "line": 7,
          "message": "…",
          "hint": "Use [^1] footnotes"
        }
      ],
      "outline": {
        "title": "sase_blog_contents",
        "numbering": true,
        "words": 253,
        "preamble": { "line": 5, "words": 186 },
        "sections": [
          {
            "number": "1.",
            "written": null,
            "stale": true,
            "depth": 1,
            "level": 1,
            "title": "The Abstractions",
            "line": 18,
            "end_line": 19,
            "output_line": 18,
            "words": 0,
            "stub": false
          }
        ]
      },
      "inbound": []
    }
  ],
  "summary": { "articles": 1, "changed": 1, "unchanged": 0, "skipped": 0, "errors": 0 }
}
```

**Per-note status and edits:**

- `status` is one of `changed`, `unchanged`, `skipped`, or `error`. `reason` is
  `{code, message}` when the status is `skipped` or `error`.
- `edits` are 1-based line hunks against the input text: `delete` input lines are
  replaced by the `insert` lines, which carry no line terminators.

**Outline fields:**

- `line` and `end_line` refer to lines in the input text; `output_line` refers to the
  formatted text.
- `title` is the de-numbered title, as raw bytes.
- `written` is the number token already in the source, or null when there is none.
  `number` is the number bob will write, or null when numbering is off.
- `stale` means `written` differs from `number`.
- `stub` means the section's own body is empty or holds only `TODO`/`TBD` markers.

**Replaced tokens and inbound links:**

- `replaced` lists every number token that a write removes or replaces.
- `inbound` stays empty until phase `inbound`. After that it carries, for each rewrite,
  `{path, line, sha256, before, after}`, where `sha256` is the linking file's hash.

**Contract rules:**

- bob owns every display string: `notice`, `message`, and `hint`.
- Clients only move the JSON. They must reject any `schema_version` other than 1.
- The contract only grows; existing fields never change meaning.

### 7. CLI surface

`bob articles` is a runner group in the **Vault** section, like `bob stats`. Its stated
membership rule: _every `bob articles` command reads or rewrites notes whose type is
`[[article]]`_. A bare `bob articles` runs `list`, which is read-only and takes only
flags, as the CLI rules allow. Every option has a short alias, options are listed
alphabetically, and output is in color when stdout is a TTY.

```text
$ bob articles --help
Format and inspect article notes (frontmatter type: "[[article]]")

Every `bob articles` command reads or rewrites notes whose type is [[article]].
bob owns their section numbers (1. / 1.1 / 1.1.1) and layout; formatting never
changes a word. Bare `bob articles` runs `list`.

Usage: bob articles [COMMAND]

Commands:
  adopt    Mark a note as an article; preview unless --write
  format   Number sections and tidy layout in article notes
  list     List article notes with sections, TODO stubs, and findings (default)
  outline  Print an article's numbered section outline

Examples:
  bob articles                                List article notes
  bob articles adopt sase_blog_contents       Preview adopting a note
  bob articles adopt -w sase_blog_contents    Adopt it
  bob articles format sase_blog_contents      Format one article
  bob articles format -a -c                   Exit 1 if any article needs formatting
  bob articles outline sase_blog_contents     Numbered outline with word counts

See docs/articles.md for the rules and the JSON contract.
```

- **`format [NOTE]...`.** Options: `-a/--all`, `-b/--bob-dir`, `-c/--check`,
  `-d/--dry-run` (prints a colored unified diff), `-f/--format human|json`, and
  `-s/--stdin`.
  - **Writes by default**, like `bob projects sync`.
  - **`--stdin`** reads the text of the note named by NOTE from stdin, prints its plan,
    and writes nothing.
  - **Exit codes:** 0 on success; 1 when `--check` finds notes that would change; 2 on a
    usage error or when a note cannot be formatted. Naming a note that is not an article
    is that kind of error, and it prints the refusal reason. With `-f json`, the full
    plan is always printed on stdout, whatever the exit code, so clients read `status`
    and `reason` instead of parsing stderr.
  - **Human output is silent when there is nothing to do.** A changed note prints one
    summary line, such as
    `✓ sase_blog_contents · 4 sections renumbered · 1 link refreshed`. Warning findings
    print beneath it.
- **`outline NOTE`** prints a colored tree like the one below. Numbers are dim, titles
  bright, and word counts right-aligned; TODO stubs are highlighted. When a number on
  disk is stale, the outline shows the number bob will write, marks it `•`, and adds a
  footer saying so. `-f json` prints the plan's `outline` object, so an agent can act on
  "rewrite §1.3" using that section's exact line range.

  ```text
  sase_blog_contents · article · 253 words · 10 sections · 9 TODO
    ¶      Preamble                                                 186 w
    1.     The Abstractions                                           0 w
    1.1    Gates -> Plans -> Epics                                   TODO
    1.3.1  Skills, **memory**, provider interface, NOT hooks         TODO
  ```

- **`list`** prints a header (`Articles · N · ~/bob`) and a table with the columns
  ARTICLE, SECTIONS, WORDS, TODO, FINDINGS, and STATE. STATE is one of `✓ formatted`,
  `● needs format`, or `skipped: <reason>`. The footer looks like
  `3 articles · 1 needs formatting`. `-f json` emits the plan's `summary` plus the
  per-note fields minus `edits`.
- **`adopt NOTE`** (phase `adopt`) previews by default and writes with `-w/--write`,
  like `bob ref migrate-tasks`. Its options are `-b`, `-f`, `-s`, and `-w`.
- **Completion.** Add a read-only `ArticleNote` provider for `format`/`outline` NOTE
  slots, showing each article's stem and `article · N sections`. `adopt` reuses
  `VaultNote`.

### 8. The Obsidian experience (`bob-articles` plugin)

**Triggers and write safety**

- **What triggers a format:**
  - **Saving:** `:w` or Ctrl+S. The plugin wraps the `editor:save-file` command
    callback, as Linter's lint-on-save does.
  - **Leaving the note:** when the note stops being the active file, or its leaf closes.
  - **The command** "Bob: Format article".
  - **Never** on a timer or on keystrokes.
- **Save path.**
  1. Send the buffer with `bob articles format --stdin NOTE -f json`, with a timeout of
     about 1.5 s.
  2. If the buffer has not changed since it was sent, apply the edits as **one**
     CodeMirror transaction. First narrow each hunk to its smallest changed span, so
     cursors and selections map naturally.
  3. Apply any `inbound` rewrites to other files.
  4. Call the original save.
  5. If anything fails or times out, still call the original save. A save is never lost
     or blocked.
- **Leave path.** Wait until Obsidian has saved the note, then read the file, send it to
  bob, and write with `vault.process(file, data => data === sent ? formatted : data)`.
- **Inbound rewrites.** Write each other file through `vault.process`, after first
  checking that the target line still equals `before`.
- **Undo guard.** If you undo a format, the plugin does not format that note again until
  you make a fresh edit (an input, delete, or paste user event).
- **Notices appear only on change.** The first line is bob's `notice`. A second line
  shows up to two warning findings, plus "+N more" when there are others.
- **Refusals are explained once.** When bob skips an article for any reason other than
  `not-article` (for example `holds-tasks`), the save still happens. bob's refusal
  message appears once per note per session, so formatting never stops silently.

**The look** (every rule scoped under `.bob-article`, which the plugin adds at runtime)

- **Measure and rhythm.**
  - `--file-line-width` about `40rem` (roughly 70 characters), and body line height
    about 1.65.
  - More space above H1 and H2 than below them.
  - `text-wrap: balance` on headings and `text-wrap: pretty` on paragraphs.
  - Section breaks come from margins. No rules, borders, or badges.
- **Typeface.**
  - Article prose uses a serif stack:
    `ui-serif, "Iowan Old Style", Charter, Georgia, serif`.
  - Headings keep the theme's face, for contrast.
  - The setting "Article typeface" offers Serif (the default) or Theme.
- **Numbers.**
  - Shown as `.bob-secno`: `--text-muted` color, normal weight, `tabular-nums`. They
    stay real text, so screen readers and copy-paste get them.
  - In Reading view on wide panes, they hang in the left margin so every title starts at
    the same edge.
  - Narrow panes, print, and Live Preview keep them inline.
- **Stale signal.**
  - In Live Preview, about 400 ms after typing stops, the plugin runs a dry run on the
    buffer.
  - A number that no longer matches gets a dotted accent underline and a tooltip, such
    as "Becomes 1.4 on save".
  - So a number on screen is either correct or visibly stale.
- **Status bar.** `§ 1.3.1` names the section under the cursor, with its title in the
  tooltip. It appears only in article notes.
- **Accessibility.** The stale signal never relies on color alone (it also uses an
  underline and a tooltip). Check contrast in both light and dark themes.

```css
.bob-article {
  --file-line-width: 40rem;
  --line-height-normal: 1.65;
}
.bob-article.bob-article-serif {
  --font-text: ui-serif, "Iowan Old Style", Charter, Georgia, serif;
}
.bob-article .bob-secno {
  color: var(--text-muted);
  font-weight: var(--font-normal);
  font-variant-numeric: tabular-nums;
}
.bob-article .bob-secno.is-stale {
  text-decoration: underline dotted var(--text-accent);
}
```

That CSS shows the direction only. Container queries and exact selectors need a test
pass in light and dark themes, at narrow widths, at 200% zoom, and in print.

### 9. Vault scaffolding

**New files and edits:**

- **`~/bob/article.md`.** A type note with `parent: "[[type]]"` that states the contract
  in a few lines, the way `project.md` and `area.md` do. Metadata Menu's type picker
  lists the children of `type.md`, so adopting a note through Properties needs no new
  UI.
- **`articles.base`.** It filters on `note.type == link("article")` and excludes
  `_templates`. Its table has columns for title link, parent, created, and updated,
  sorted by most recently updated.
- **An "Articles" block in `type.md`.** Add `## Articles ([[articles.base]])` with
  `![[articles.base]]`, next to Projects and Reading List.
- **`_templates/new_article.md`.** It takes `parent` from the active note, using the
  same Templater snippet as `new_project.md`. It also sets
  `template: "[[new_article]]"`, `type: "[[article]]"`, and `created`. The body is
  empty.
- **One line in the vault's `AGENTS.md`:** _after editing a note whose `type` is
  `[[article]]`, run `bob articles format <path>`; it numbers sections and tidies layout
  and never changes words._

**Three ways to adopt a note**, all of which write the same single `type` line:

- the Properties picker;
- the plugin command "Bob: Convert note to article…", which previews the change;
- `bob articles adopt`.

### Out of scope (follow-ups, not this epic)

- Exporting with numbers stripped (`bob articles export --strip-numbers`).
- A `NoteKind::Article` PDF snapshot profile.
- A per-heading `<!-- bob:unnumbered -->` opt-out.
- A footnote conversion tool, and appendix lettering.
- Number fields in the `capture-sections` JSON for the Mac picker.
- Making nightly _write_ articles. This epic adds nightly as a report-only check, as the
  research recommends; it can write once that check has earned trust.

## Phase `engine` — pure article formatter core (bob-cli)

**Build:**

- **The marker** (§1). Generalize the type matcher, add `frontmatter_is_article`, parse
  `title` and `section_numbers`, and detect every refusal as a pure function of
  `(text, vault-relative path)`.
- **`scan.rs`.** Classify each line into one region: frontmatter, fence (including an
  unclosed fence), math, `%%` comment, HTML comment, HTML block, quote or callout, list
  or footnote continuation, indented code, table, paragraph, blank, ATX heading, or
  Setext underline. When a line is ambiguous, protect it.
- **Section numbering and layout.** Sections with N2–N7 and N9, and layout F1–F8, F10,
  and F11 (§2, §3).
- **Checks.** The word invariant (headings and Setext lines for now) and the second-pass
  idempotency check.
- **Output model.**
  - The formatted text and line hunks (via `similar`).
  - Change counts and the replaced tokens.
  - The findings this phase can see: `deep-section`, `unclosed-fence`,
    `number-replaced`, `setext-multiline`, and `bad-option`.
  - The ordered sections with written and computed numbers and their line ranges.
- **Shared helpers.** Expose the token grammar (`strip_section_number`) and the identity
  normalizer as `pub(crate)`, for phases `links` and `capture`.

**Tests.** Every fixture is a pair: an input and its expected output. For each one,
assert that a second run is a no-op and that the word invariant holds.

| Fixture     | What it covers                                                                                                                                                                                                                                                                         |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Blog draft  | A structural replica of `sase_blog_contents.md` with placeholder prose; do not copy Bryan's draft text into the repo. It has a preamble with pseudo-footnotes, one H1, H2 sections, `####` under `##`, a heading with a trailing space, and `TODO` bodies. Expect §2's worked example. |
| Ownership   | `2024 taxes`, `30 Burger`, an agent-typed `1.4`, a typed `3.`, `1.5 million users`, `1.16.0 changelog`, `1\.5`, a number-only `## 1.2`, and an empty `##`.                                                                                                                             |
| Protected   | Headings inside \`\`\` and `~~~` fences of different lengths, fences inside lists, `$$`, `%%` blocks, `<!-- -->`, a `<div>` HTML block, `> [!note]` callouts containing `> ##`, list continuations, and indented code.                                                                 |
| Setext      | Single-line `===` and `---`; multi-line (left alone); `---` after a list item (a thematic break); the frontmatter fence.                                                                                                                                                               |
| Layout      | Blank runs at top level and inside lists; trailing whitespace, kept inside fences; a missing final newline; extra trailing blank lines; a CRLF file; a BOM; tabs after hashes; closing hashes; `## C#`.                                                                                |
| Fences      | F11 at top level, not inside lists; an unclosed fence leaves everything after it untouched.                                                                                                                                                                                            |
| Levels      | `###` before any `##`; the sequence `#`, `##`, `####`, `####`, `##`; depth 5.                                                                                                                                                                                                          |
| Numbers off | `section_numbers: false`, then a round trip back to on.                                                                                                                                                                                                                                |
| Refusals    | Each refusal code. `#task` lines inside a code fence must not refuse.                                                                                                                                                                                                                  |

Also add:

- A seeded **mutation test** using a tiny in-test LCG, with no new dependency. It
  perturbs heading levels and blank runs and asserts idempotency and the invariant.
- An **`#[ignore]` corpus test.** It reads `BOB_ARTICLES_CORPUS`, treats every `.md`
  note's text as an article (bypassing the marker), and asserts no panic, idempotency,
  and the invariant. Skips are fine. Run it once against `~/bob` (it only reads) and
  report the results.

**Done when:** `just check` passes, and the corpus run on `~/bob` shows zero invariant
or idempotency failures.

## Phase `links` — in-note links, findings, outline, and the plan contract (bob-cli)

**Build:**

- **The identity helper and resolution** (§4). This includes nested `#A#B` paths,
  embeds, Markdown links with percent-encoding, aliases, and links that name this note
  through `[[stem#…]]` or `[[path/stem#…]]` (resolved with the context's vault-relative
  path). Extend the word invariant to de-number link fragments.
- **The remaining findings:** `pseudo-footnote`, `broken-heading-link`,
  `ambiguous-heading-link`, and `title-heading`.
- **The outline model:** word counts that skip fenced code, math, and comments; `stub`
  detection; the preamble; `written`, `number`, and `stale`; and `line`, `end_line`, and
  `output_line`.
- **The `ArticlePlan` serde model** with the exact §6 JSON shape, plus bob's `notice`
  text. Render the notice from counts:
  - Clauses: `N sections numbered`, `N renumbered`, `N headings promoted (#### → ###)`,
    `N Setext headings converted`, `N links refreshed`.
  - Join the clauses with `·`.
  - Omit the zero clauses.
  - Use `Article formatted` alone when only layout changed.

**Tests:**

- The research's link cases: `[[#Beads]]` → `[[#7. Beads]]`, and
  `[[stem#2024 taxes|alias]]` → `[[stem#1. 2024 taxes|alias]]`.
- Duplicate titles resolved by pre-format text, and the ambiguous case reported.
- Block links untouched, and links inside code untouched.
- Each finding.
- An outline snapshot for the blog-draft fixture.
- A JSON golden for the plan.

Rerun the corpus test.

**Done when:** `just check` passes and the JSON golden matches §6.

## Phase `capture` — number-aware capture section selectors (bob-cli)

**The problem.** Once an article is numbered, `@sase_blog_contents#agent` no longer
prefix-matches `## 1.3 Agent CLIs`. A stale exact `--section` title, as the Mac picker
sends, misses too.

**The change.** When the destination note is an article,
`heading_matches_bullet_selector` in `src/native/capture/sections.rs` (and its callers)
also matches on identity:

- a prefix selector matches when it is a prefix of the heading's identity;
- an exact selector matches when its de-numbered identity equals the heading's identity.

Today's raw-title match is still tried first. Notes that are not articles keep their
current behavior exactly. `capture-sections` output and completion stay unchanged;
titles keep their numbers. Bob Mac Capture needs no change, because it passes titles
back to bob.

**Tests:**

- `@note#agent` → `1.3 Agent CLIs`.
- `@note#1.3` → `1.3`, not `1.3.1`, because of document order.
- `--section "Agent CLIs"` and a stale `--section "1.4 Agent CLIs"` both reach
  `1.3 Agent CLIs`.
- A non-article note with a numbered heading keeps today's behavior.

**Docs.** Add one paragraph under "Section bullets" in `docs/capture.md`.

## Phase `cli` — `bob articles` with format, list, and outline (bob-cli)

**Registration:**

- Add native commands for `format`, `list`, and `outline`.
- Register a runner `Group` in `src/runner.rs`: Vault section, first alphabetically,
  `default: Some("list")`, and `extra_about` stating the membership rule.
- Wire the completion tree, and add the `ArticleNote` provider.
- Remove the dead-code allowance.

**Behavior** (§7):

- **Note resolution.** Resolve NOTE through `VaultLinkResolver`: a missing note gets
  near-miss suggestions, and an ambiguous one lists its candidates.
- **`--all` discovery.** Walk the vault, skip the always-excluded directories, and read
  only the frontmatter until a note proves to be an article.
- **Writes.** Write atomically with a sha256 preimage check: re-read the file, compare
  the hash, write a temp file, rename. If the file changed in the meantime, skip it with
  a reason.
- **Modes:** `--check`, `--dry-run` (colored unified diff), `--stdin` (never writes;
  requires exactly one NOTE), and `-f json` (§6).
- **Output.** Human output is silent for unchanged notes. `list` and `outline` render as
  specified.
- **Exit codes:** 0, 1, and 2, as defined in §7.

**Docs:**

- `docs/articles.md` is the full contract: the marker, the N, F, and finding rules,
  heading identity, the CLI, the JSON, write safety, the escape, and known risks.
- Add rows to `docs/README.md` and to the README command index under Vault.

**Tests:**

- CLI integration tests on temp vaults: write, check, dry-run, stdin, json, all, the
  skip reasons, preimage drift, and exit codes.
- Help tests that check option order and examples.
- A root help test showing `articles` in Vault.
- A completion provider test.

**Done when:** `just check` passes, and `bob articles --help` and the subcommand help
match the CLI rules memory.

## Phase `adopt` — `bob articles adopt` with preview (bob-cli)

**Planning the change:**

- Insert `type: "[[article]]"` after the `parent:` line, or before the closing `---`
  when there is no `parent`.
- When the note has no frontmatter, create one that holds only `type`.
- Then plan the first format in the same ArticlePlan, with the frontmatter edit included
  in `edits`, and the notice `Adopted as an article · …`.

**Outcomes:**

- A note that is already an article succeeds as a no-op: `already an article`.
- A note with another `type`, with tasks, or with anchors is refused with the §1 codes,
  for example "note already has type [[restaurant]]".

**Output:**

- **Human preview:** a header, a colored diff, a "Numbers replaced" list
  (`## 3. Reasons` → `1.2`, or `none`), findings, and a
  `Preview only · rerun with -w to write` footer.
- **`--stdin`:** the JSON plan for the plugin's preview modal.
- **`-w`:** writes atomically, with the same preimage check as `format`.

**Also:** help, completion with `VaultNote`, docs, and tests covering the frontmatter
forms, refusals, idempotency (adopting twice), the replaced list, and stdin JSON.

## Phase `inbound` — inbound heading links, nightly check, and memory (bob-cli)

**Inbound link repair:**

- **When it runs:** in write mode (`format`, `adopt -w`) whenever any heading's text
  changes in that run. With `--all`, it also runs as one shared vault scan that repairs
  links left stale by earlier runs.
- **How it finds links:**
  1. Prefilter notes outside the excluded directories by a case-insensitive substring
     match on the article's stem.
  2. Parse wikilinks, embeds, and Markdown links, skipping code.
  3. Resolve link targets with `NoteIndex`.
  4. Rewrite fragments with the §4 resolution against the article's pre-format headings.
- **Unrepairable links** become `inbound-heading-link` findings on the article, each
  with the source's `path:line`.
- **Write order:**
  1. Write the article.
  2. Write each linking note with an atomic write and a sha256 preimage check. A file
     that changed in the meantime is skipped with a reason.
- **Preview modes:** `--dry-run` shows inbound diffs, and `--stdin` fills the plan's
  `inbound` entries for the plugin.
- **Tests:** a renumber that rewrites links in two other notes; duplicate titles; an
  unrepairable link; a preimage race; `--stdin` entries.

**Nightly.** Add a report-only `articles` step to `bob nightly` after `task archive`. It
runs the `format --all --check` logic and prints which articles need formatting. It
counts as passed unless bob itself fails (exit ≥2). Update the nightly help, docs, and
tests.

> [!decision] memory_glossary
>
> Add the glossary strand `glossary:article-note` through `/sase_memory_write`. Define
> an **Article Note** (also called an _article_) as a Bob vault note with frontmatter
> `type: "[[article]]"` that holds long-form prose, not tasks. bob owns its section
> numbers and layout through `bob articles format` and the bob-articles plugin. Work
> about an article lives in its parent project note. See `docs/articles.md`. Link it to
> `glossary:project-note`, then run `sase memory init`.

> [!decision] memory_decision
>
> Add the decision strand `decisions:article-numbers-are-bob-owned` through
> `/sase_memory_write`.
>
> - **Applies to:** bob-cli, bob-plugins, and the vault.
> - **Claim:** article section numbers are written into the source in house style, `1.`
>   / `1.1` / `1.1.1`. bob strips one owned token per heading and rewrites it. Clients
>   never compute numbers.
> - **Why:** agents, git, and Obsidian surfaces all share "§1.3". The house-style counts
>   back it. Rejected alternatives: display-only widgets, CSS counters, and a `1.1.`
>   trailing dot.
> - **Cost:** `1.5 million` headings need escaping, and diffs touch heading lines.
> - **Reopens when:** Bryan wants raw Markdown and git unnumbered.
> - **Evidence:** the research artifact and `docs/articles.md`.
>
> Then run `sase memory init`.

## Phase `vault` — type note, template, base, and first adoption (Bob vault)

**Before you start:**

- Work in `~/bob` and follow its `AGENTS.md`: inspect `git status` first, and stage only
  the files this phase changes.
- Every new note needs a `parent`.
- If the installed `bob` lacks `bob articles`, run bob from the bob-cli checkout
  (`cargo run --quiet -- …`) or run `just install` first.

**Steps:**

1. Create the files in §9, and add the line to the vault's `AGENTS.md`.
2. Run `bob articles adopt sase_blog_contents` and review the preview. Expect: 10
   sections numbered, 2 promoted, no numbers replaced, and `pseudo-footnote` findings.
3. Run `bob articles adopt -w sase_blog_contents`.
4. Confirm `bob articles format --check sase_blog_contents` exits 0.
5. Confirm `bob articles` lists the note as formatted.
6. Commit the vault as its `AGENTS.md` directs.

## Phase `plugin` — bob-articles formats on save and leave (bob-plugins)

**Setup.** Open bob-plugins with `/sase_repo` and read its `AGENTS.md`. Create
`plugins/bob-articles/` with:

- `manifest.json`: id `bob-articles`, name "Bob Articles", and the siblings'
  `minAppVersion` and `isDesktopOnly: false` convention, with a guarded `child_process`;
- `src/fragments.json` and its fragments (each at most 1,000 lines);
- the generated `main.js` from `npm run build`;
- `styles.css` (minimal here; phase `look` fills it in).

**Build:**

- **Resolve bob** as bob-ledger-tools' stats path does: a configured path, then the
  known install locations, then PATH. Set `BOB_DIR` to the vault and `NO_COLOR=1`. Write
  the buffer to stdin. Keep the helpers local to this plugin.
- **Detect articles** from the metadataCache frontmatter `type`, either a string or a
  list. If bob reports `skipped`, bob's answer wins.
- **The `bob-article` class.** Toggle it on article Markdown views on file-open,
  active-leaf-change, layout-change, and metadataCache `changed`.
- **The triggers, write safety, and notices** in §8: the save wrapper, the leave path,
  the "Bob: Format article" command, one narrowed transaction, the preimage checks,
  inbound `vault.process` writes, and the undo guard.
- **"Bob: Convert note to article…"**, shown only on notes that are not articles. It
  runs `bob articles adopt --stdin NOTE -f json` and opens a preview modal containing:
  - bob's notice;
  - a compact diff with removed and added lines and muted context;
  - the "Numbers replaced" list;
  - Convert and Cancel buttons.

  A refusal shows bob's message. Convert applies the change as one transaction.

- **Settings:** Format on save (on), Format when leaving the note (on), and bob path
  (blank means auto).
- **Errors:** show a single quiet notice per session when bob is missing or times out.
  Never block a save. Show a refusal once per note per session, as §8 describes.

**Tests.** Node `--test` files with a small harness, registered in `package.json`. They
cover:

- plan decoding, including rejecting a foreign `schema_version`;
- converting hunks into minimal changes, including the cursor-mapping cases;
- dropping the apply when the buffer changed;
- the undo-guard state machine;
- article detection;
- the inbound line-preimage guard;
- notice composition.

`npm test` and `npm run validate` must pass.

**Deploy:**

1. Make sure the installed `bob` has `bob articles`; run `just install` from the bob-cli
   checkout if it does not.
2. Run `bob plugins sync`, passing `--repo <opened path>` because of the
   foreign-checkout guard.
3. Add `bob-articles` to `~/bob/.obsidian/community-plugins.json`, and commit it per the
   vault's `AGENTS.md`.
4. Tell Bryan that Obsidian needs a reload.

## Phase `look` — reading look and live section signals (bob-plugins)

**Build:**

- **`styles.css`.** All of §8's look, scoped under `.bob-article`, plus the
  `bob-article-serif` setting class: Article typeface, Serif or Theme.
- **Live Preview.** A CodeMirror 6 view plugin marks each section's number token as
  `span.bob-secno`. It takes its data from the latest plan outline (`line` plus
  `written`) and maps the marks through transactions between refreshes.
- **Reading view.** A post-processor wraps the leading number token of each heading. It
  finds the line with `ctx.getSectionInfo(el)` and matches it against a cached outline
  for the file. Numbers hang in the left margin on wide panes and stay inline on narrow
  panes and in print.
- **Stale hints.**
  - Trigger: a dry run about 400 ms after typing stops. Skip it while IME composition is
    active.
  - Allow at most one spawn in flight per file, and drop responses for an older doc
    version.
  - Mark stale numbers with `is-stale`, with a tooltip such as "Becomes 1.4 on save".
  - Desktop only.
  - Setting "Show stale-number hints" (on).
- **The `§ 1.3.1` status-bar item.** It uses the computed numbers, so it is always
  correct, and shows the section title in its tooltip. It is hidden in the preamble, in
  notes that are not articles, and when numbering is off.

**Tests:**

- decoration ranges;
- the section at a given line;
- the debounce and in-flight state machine;
- token wrapping in the post-processor.

Then rebuild, validate, and run `bob plugins sync`.

**Manual acceptance.** Check `sase_blog_contents` in both light and dark themes, in a
narrow pane, at 200% zoom, and in a PDF export. Report what you saw.

## Epic verification

- `just check` passes in bob-cli, and `npm test` plus `npm run validate` pass in
  bob-plugins.
- `bob articles format --check -a` exits 0 on `~/bob` after adoption, and a second
  `bob articles format sase_blog_contents` is silent.
- The corpus test passes against `~/bob`.
- In Obsidian, these all behave as §8 describes:
  - `:w` on `sase_blog_contents` after adding a section renumbers the note in one
    undoable step;
  - a link elsewhere to a renumbered section still resolves;
  - `@sase_blog_contents#agent` captures into `1.3 Agent CLIs`;
  - the look.
