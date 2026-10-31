---
tier: epic
title: Capture drawers and % directives
goal: "Bob's capture language reads as three rules: @ says where, # names a place or a
  type, and % says how. `Text @note+id#drawer [#type]` files a note in a task drawer,
  creating the drawer when it is missing and optionally retyping the task in the same
  transaction. `%c`, `%n`, `%t`, and `%l` replace trailing `#`, `%`, `@@template:`, and
  `URL %`. Typing `#`, `%`, or `@note+id#` in Bob Mac Capture opens a bob-described menu
  with nothing preselected. Every retired form is deleted outright, with no aliases.

  "
phases:
  - id: cli-directives
    title: bob-cli % directives, bare-# type menu, %t templates, and @@ cleanup
    depends_on: []
    size: large
    description:
      "cli-directives: add the `%c`/`%c<N>`/`%n`/`%t`/`%l` directive lexer and the
      `directive` completion context; make a bare edge `#` an incomplete type marker
      with `initial_selection: none`; split clipboard parsing from reading (`--no-clip`,
      `clip.status`); frame templates with `%t` and render shelf drafts that way; delete
      trailing `#`, `%`/`%N`/ `%header`, `URL %`/`URL @`, `@@template:`, `@@@`, and
      `@@route+id`; add `grammar: 2`; update help, docs, and the Capture and Bulk
      Capture glossary strands."
  - id: cli-drawers
    title: bob-cli task drawers, drawer menu, and capture-drawers
    depends_on:
      - cli-directives
    size: large
    description:
      'cli-drawers: make a `#` glued after `@route+id` always name a task drawer,
      matched exactly, created on demand, and guarded against reserved names and
      near-duplicates; allow a separate `#type` to retype the target in the same
      transaction; retire attached `#type` on `+id` and `^id`; add the cached drawer
      catalog and the `drawer` completion context; add `-D/--drawer` and `bob
      capture-drawers` (keeping `capture-task-sections` as a hidden alias); bump to
      `grammar: 3`; rename "task section" everywhere; write the Task Drawer term, the
      grammar decision record, and the Annotation Section fix.'
  - id: mac-directives
    title: Bob Mac Capture % menu, unselected menus, clip card, and grammar gate
    depends_on:
      - cli-directives
    size: large
    description:
      "mac-directives: add the no-selection menu state; add the bob-described `%`
      directive menu and the immediate bare-`#` Task type menu; style the `directive`
      span; read the clipboard in live preview only when bob reports a clip directive,
      and render a real Clip card; require `grammar >= 2`; delete `@@@`, the `@@`
      template row, and `@@route+id` handling; regenerate fixtures; update the Mac docs."
  - id: cli-project-drawers
    title: bob-cli ^prj drawers resolve to project-note H2 sections
    depends_on:
      - cli-drawers
    size: medium
    description:
      "cli-project-drawers: on a project note's own `^prj` line, resolve `#drawer`
      against the note's eligible `##` sections, appending or creating `## Title Case`
      the way promotion does; list those sections in the drawer menu; extend the Task
      Drawer glossary strand."
  - id: mac-drawers
    title: Bob Mac Capture drawer menu and drawer previews
    depends_on:
      - cli-drawers
      - mac-directives
    size: medium
    description:
      'mac-drawers: render the `drawer` context with On this task and Recent groups, the
      New drawer row, refused rows, and bob''s per-row consequence footer; style
      `drawer` spans; show the "new drawer" and retype rows in previews; require
      `grammar >= 3`; delete `task_section` rendering; regenerate fixtures; update the
      Mac docs.'
  - id: plugins-parity
    title: bob-plugins drawer promotion parity test and vocabulary
    depends_on:
      - cli-drawers
    size: small
    description:
      'plugins-parity: pin with a test that a drawer exactly as bob writes it promotes
      to the expected `## Title Case` section and reverses to the same ALL-CAPS key;
      rename "section bullet" to "drawer" in plugin docs and comments.'
decisions:
  memory_drawer_term:
    ask:
      Add the Task Drawer (aka drawer) glossary strand, extended with the ^prj clause
      once project drawers land?
    memory:
      - glossary:task-drawer
    default: true
    requested:
      As a part of this change, let's add a new "task drawer" (aka "drawer") glossary
      term.
    answer: true
  memory_grammar_record:
    ask:
      "Add a decisions record that @ says where, # names a place or type, and % says
      how?"
    memory:
      - decisions:capture-sigils-say-where-what-how
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: true
  memory_glossary_strands:
    ask:
      Update the Capture, Bulk Capture, and Annotation Section glossary strands for %c,
      %t, and drawers?
    memory:
      - glossary:capture
      - glossary:bulk-capture
      - glossary:annotation-section
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: true
proposed_by: bbugyi200.apollo.research.2x.linker.w0
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-31 12:52:00
status: wip
---

- **PROMPT:**
  [prompts/202610/capture_drawers_and_directives.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/capture_drawers_and_directives.md)

# Plan: Capture drawers and `%` directives

## Context

Bryan asked for a cleanup of the capture grammar that makes it consistent and easier to
understand:

- `@file+id#drawer` replaces `@file+id#type`. A drawer is an ALL-CAPS child bullet of a
  task, which becomes an H2 section if the task is ever promoted to a project note.
- A separate `#type` token retypes the targeted task.
- Typing `#` after `@file+id` opens a drawer menu.
- A missing drawer is created on demand.
- Trailing `#`, `%`, and `@@template:` move to `%n`, `%c`, and `%t`. Typing `%`
  describes each directive in Bob Mac Capture, and a bare `#` opens the Task type menu
  immediately.
- `@@file+id` is removed.
- Every retired form is deleted outright, with no backward compatibility.

The research report
`research:202610/capture_drawer_and_directive_grammar/capture_drawer_and_directive_grammar.md`
holds the evidence, the alternatives considered, and the rationale. Bryan agreed with
all of its requirements, R1 through R13. Read it with `sase artifact read` when you need
the "why". This plan is the normative "what": where this plan and the report differ,
follow this plan.

Related history:

- **Epic `bob-cli-ep`** (plan `plan:202610/capture_hash_type_marker.md`) shipped
  `#<type>` today. This epic deletes its attached forms (`@r+id#type`, `@r^id#type`).
  - It also adopts the `initial_selection: "none"` design that epic specified under the
    rejected `bare_hash_menu = open` branch.
- **Separator toggle.** Commit `fc1e449f` made `+`, `:`, and `^` after an ID toggle the
  separator in place. That stays. A glued `#…` suffix keeps its bytes and takes the new
  separator's meaning:
  - `:` makes it a Pomodoro name;
  - `+` makes it a drawer;
  - `^` makes it the R3 refusal.

## The language after this epic

> **`@` says where.** `@note`, `@note#Heading`, `@note+id`, `@note+id#drawer`,
> `@note:id`, `@@note`.
>
> **`#` names.** A `#` glued to a destination token names a place inside it: a heading,
> a drawer, or a Pomodoro. A `#` standing alone at an item edge names a task type.
> Anywhere else it is prose.
>
> **`%` says how.** `%c` clip, `%n` Pomodoro note, `%t` template, `%l` listen.

| You type                                                                 | Meaning                                                                |
| ------------------------------------------------------------------------ | ---------------------------------------------------------------------- |
| `Fix the parser #fix`, `#fix Fix the parser`                             | new task typed FIX (unchanged)                                         |
| `Fix the parser @sase^parser #fix`                                       | new `^parser` task typed FIX (separate token, R3)                      |
| `Some note goes here. @foo+bar#requirements`                             | bullet in `^bar`'s REQUIREMENTS drawer, created if missing             |
| `Some note goes here. @foo+bar#requirements #fix`                        | the same, plus retype `^bar` to FIX, atomically                        |
| `Some note. @foo+bar #fix`                                               | plain child bullet under `^bar`, plus retype `^bar`                    |
| `@foo+bar #fix`                                                          | Ensure Next + retype (unchanged)                                       |
| `@foo+bar#requirements` alone                                            | incomplete: needs note text (R2)                                       |
| `#` alone at an item edge                                                | incomplete type marker; the Task type menu opens with nothing selected |
| `%` alone at an item edge                                                | incomplete directive; the directive menu opens with nothing selected   |
| `remembered the timeout %n`                                              | Pomodoro note (was the trailing `#`)                                   |
| `Build output %c`, `%c3`                                                 | clipboard as child bullets; `%c3` = last 3 clips                       |
| `{0} @job+search#leads %t`, a blank line, then rows                      | template: the first item repeats once per row                          |
| `https://example.com/essay %l`                                           | queue the reference with companion audio                               |
| `@notes#Ideas`, `@sase:fix#bugs`, `=3#bugs`, `@x^y+#admin`, `@@route`    | unchanged                                                              |
| mid-line `#`/`%`, `#1234`, `C#`, `##`, `100%`, URL `%25`, `\#tag`, `\%c` | prose / literal                                                        |

## Ground rules for every phase

- **Thin client** (`decisions:mac-capture-is-a-thin-client`).
  - bob owns grammar, menu rows, menu copy (labels, summaries, details, consequences,
    refusal reasons), previews, and writes.
  - The Mac renders what bob returns. Never add Swift that parses capture syntax.
- **Release train** (`decisions:bob-cli-releases-ride-a-daily-train`).
  - Never edit `Cargo.toml`'s version or `CHANGELOG.md`.
  - `cli-directives` and `cli-drawers` are contract breaks. Use a `feat(capture)!:`
    subject and a `BREAKING CHANGE:` footer that lists the removed forms and the wire
    renames.
  - `cli-project-drawers` is additive: it turns a refusal into a write.
- **Coherent master.** Bryan's hosts track master. Each bob-cli phase lands as one
  coherent change, and grammar, writers, completion, help, docs, and tests must never
  disagree at any pushed commit.
- **No compatibility.**
  - Delete the retired lexing, spans, needs, contexts, codes, CLI options, Swift paths,
    fixtures, and docs. Do not add aliases or "X is retired" sections, and never write
    error copy that names a retired spelling.
  - Old habits fail through ordinary grammar errors that teach only the current grammar.
  - Narrow negative tests proving an old form no longer runs are fine. Phrase them with
    the current grammar's error codes.
  - Do not touch immutable history: `CHANGELOG.md`, research reports, plan artifacts,
    accepted decision records, and git history.
- **CLI rules** (read `sase memory read cli_rules.md`).
  - Options stay alphabetical, and every public long option gets a short alias.
  - A renamed command keeps its old spelling as a permanent hidden alias, with
    byte-identical behavior and no deprecation output. That alias,
    `capture-task-sections`, is the one place the old name may survive.
- **Docs rules** (read `sase memory read docs.md`).
  - Run `just docs-gen` after clap help or glossary memory changes, and
    `just docs-check` before landing.
  - Mac docs are owned by bob-mac-capture and only mounted into the Bob site.
- **Mac builds.** bob-mac-capture's `AGENTS.md` compile rules are mandatory. When the
  Mac cannot be reached, say plainly "macOS targets unverified".
- **Other repos.** Open bob-mac-capture and bob-plugins only through `/sase_repo`, and
  read each repo's `AGENTS.md` first.
- **Shared fixtures.** Do not edit `tests/fixtures/pomodoro_themes/vectors.json`.
- **Grammar revision (R12).** A single constant (for example
  `capture_language::GRAMMAR_REVISION`) feeds a new top-level `grammar` integer in
  `capture-parse` and `capture-complete` JSON. `schema_version` stays 1.
  - `cli-directives` introduces it as 2, and `cli-drawers` bumps it to 3.
  - Each Mac phase requires the revision its contract needs (≥ 2, then ≥ 3). See §6.5.
- **Memory.** Edit memory only as §7's accepted decisions allow, through
  `/sase_memory_write`, then run `sase memory init` and `just docs-gen`, since
  `docs/reference/glossary.md` is generated from the glossary web.

## 1. Contract: `%` directives (phase `cli-directives`)

### 1.1 Token and position

- **Position.** A directive sits at an item edge: either edge of any physical line of
  the item, with only other markers between it and that edge.
  - This is the exact position rule `#type` uses (`type_marker.rs` / `dependencies.rs`
    `is_trailing_transparent`).
  - Directives are themselves transparent markers for that rule, replacing the bare `#`
    and the clip tokens in the transparent set.
  - A line holding only directives (or `- %c`) contributes no bullet.
- **Claim.** At an edge, every whitespace-delimited token that starts with `%` is a
  directive. Mid-line `%` tokens stay prose.
- **Escape.** `\%…` at an edge captures the text without the backslash. Mid-line `\%`
  stays verbatim. This mirrors `\#`.

| Token                                                   | Meaning                                                                                                                                                                                           |
| ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `%c`                                                    | read the live clipboard into child bullets of this item (today's `%`)                                                                                                                             |
| `%c<N>`, N ≥ 2                                          | live clipboard plus history, N values in all (today's `%N`). `%c1` ≡ `%c`. `%c0`, a leading zero, and N above today's history limit are invalid                                                   |
| `%n`                                                    | Pomodoro note: a plain bullet under today's open timed Pomodoro, else the last completed, else the first open (today's trailing `#` path, `PomodoroSelection::CurrentOrLastCompleted`, unchanged) |
| `%t`                                                    | this item is a template (§1.4)                                                                                                                                                                    |
| `%l`                                                    | listen: queue this bare-URL reference with companion audio (today's `URL %`)                                                                                                                      |
| `%` alone                                               | incomplete. Need `directive`; execution refuses                                                                                                                                                   |
| anything else (`%x`, `%C`, `%clip`, `%3`, `%build_log`) | `unknown_directive` error                                                                                                                                                                         |

Letters are lowercase only, so there is one spelling of each.

### 1.2 Composition and refusals

- **Repeats.** Each directive appears at most once per item: `duplicate_capture_marker`,
  with label `directive (%c)`.
- **`%c`** composes with routes, drawers, `s:`, `$`, `#type`, and `%n`. Clip onto a
  Pomodoro note works today and must keep working. It keeps today's refusals: Pomodoro
  link, adjust, close, and start; task complete; project-note creation. Its copy now
  says `%c`.
- **`%n`** refuses routes, a forced `--route`, `s:`, `$`, and `#type`, using today's
  Pomodoro-note conflict codes with `%n` copy. A draft-wide `@@route` does not apply to
  a `%n` item, matching today's trailing-`#` behavior.
- **`%l`** applies only to an item the reference claim takes (a bare URL, or one line of
  a bare-URL list). Anywhere else: `directive_not_applicable`
  (`` `%l` queues companion audio for a bare URL ``).
- **`%t`** refuses `%c` (`template_clipboard`), session operators, and `@@`
  declarations, with today's `template_*` codes. `%n` and `%l` inside a template apply
  per expanded row.

### 1.3 Clipboard reading (R5, R8)

- **Parsing is separate from reading.**
  - `%c` is always parsed and always stripped; it never lands in a body.
  - Delete `CaptureParseOptions.parse_clip_markers` and the template force-on path.
  - The clip header is gone: `%header`, `is_valid_header`, and `ClipOutput.header`.
- **`-n, --no-clip`** means "never read the clipboard".
  - A dry run under `--no-clip` reports `clip: {"status": "not_read", "count": N}` for
    an item with `%c`/`%c<N>`.
  - A real capture with `--no-clip` and a clip directive refuses with code
    `clip_not_read`:
    `` `%c` reads the clipboard, but --no-clip forbids reading it: drop --no-clip, or write `\%c` to keep the text ``.
- **`-c, --clip`** becomes a plain flag with no value. It forces a clipboard read as if
  `%c` were written.
- **Read results.** A clip that was read reports `"status": "read"` plus today's fields
  (`mode`, `lines`, `attachments`, `snippet`, `entries`), without `header`.
- **Dry runs never write files.** That includes snippets and attachments. Add a test.
- **Same path for preview and capture.** Preview and capture read through the same
  `pbpaste` / Clipy path (`capture_clip/clipboard.rs`, `BOB_CLIPBOARD_CMD`,
  `BOB_CLIPBOARD_HISTORY_CMD`).

### 1.4 `%t` templates (R6)

- **The definition** is the draft's first item: a parent line plus optional authored
  children.
  - `%t` may sit at either edge of any line of that item, and it stays where it was
    typed. Nothing is ever rewritten.
- **Rows** are the nonblank physical lines after the first blank line that ends the
  definition. Today's row rules are unchanged:
  - dense `{N}` placeholders and `{{`/`}}` escapes;
  - quoting, and `{0}`-only splitting;
  - the 1,000-row and 1 MiB limits;
  - source-mapped errors;
  - `-E/--expand`;
  - capture(D) = capture(expand(D)).
- **Masking.** Framing masks the `%t` token with offsets preserved, so editor spans stay
  exact. The definition handed to the existing engine (`capture_language/template.rs`)
  is the item with the token and one adjacent separator space removed.
- **A misplaced `%t`.** A `%t` in any later item is `template_directive_misplaced`
  (`` `%t` marks the first item as the template ``). It replaces
  `template_header_misplaced` and `template_header_missing`.
  - The headerless-template hint (`capture_parse.rs` `headerless_template_hint`) now
    teaches `%t`.
- **Row values.** A row value that injects `%c` or `%t` at an expanded item's edge is
  refused. This also rules out nested templates.
- **Shelf IDs** (`capture_templates/`) must not change.
  - Canonical text already excludes the header. `save` extracts the definition without
    `%t` and computes the same canonical text as before.
  - Test: `{0} @x %t` + rows yields the same ID as the pre-change stored definition
    `{0} @x`.
  - `show` and `draft` render the definition with ` %t` appended to its first line, then
    a blank line.
  - `save` refuses non-template drafts with copy that teaches `%t`.
- **`@@` now only means a draft-wide route.** Delete all of the following:
  - the `@@template:` header (`template.rs` `HEADER`);
  - the `template_header` span;
  - `RewriteRule::StartTemplate` / `start_template` (`@@@`) in `capture_rewrite.rs` and
    `capture_language/rewrite.rs`;
  - the `@@` template completion candidate (`capture_complete/engine.rs`
    `template_completion_gate`, `RouteCandidateKind::Template`, and the `shell.rs`
    template branches).

### 1.5 `@@route+block-id` removal

`@@route` (a draft-wide route) is unchanged. Delete the `+id` form end to end:

- `ParsedGlobalDestination.block_id` and `mode`;
- the `draft.rs` `+` branch;
- `GlobalSubBulletRoute` / `GlobalSubBulletBlockId` spans and `Need::Task` from
  `classify_global_token`;
- `global_destination.mode` / `block_id` in parse and capture JSON (`capture_parse.rs`
  `GlobalDestinationParse`, `capture/output.rs` `GlobalDestinationSummary`);
- the "under ^id" human output;
- `capture_language/tests/globals.rs` coverage of `+`;
- `capture_related/draft.rs` handling.

`@@route+id` now gets the generic `@@` error:
``invalid `@@` declaration: `@@` takes a route (`@@foo`)``.

### 1.6 Bare `#` type marker (R7)

- **The marker.** A free-standing bare `#` at an item edge is an incomplete `task_type`
  marker. It gets a `task_type` span on the `#` and the need `task_type` (mode
  `incomplete`).
  - Execution refuses with copy like
    `` `#` is incomplete: add a task type (`#fix`), or write `\#` to keep a literal `#` ``.
  - A leading `# Heading text` is therefore incomplete, and `\# Heading text` is the
    literal.
  - Delete the "bare hash stays the Pomodoro note" branch in `type_marker.rs`
    (~151–159).
- **`capture-complete`** returns the `task_type` context with an empty query, every row
  in config order, and the new top-level `initial_selection: "none"`.
  - Subjects that cannot take a type (a `%n` item, session operators) report the refusal
    through the existing refused-subject machinery, never through rows that would fail.
- **`initial_selection: "none"`** is emitted only for an empty query opened by a bare
  sigil: `#`, `%`, or `@note+id#` (§3.6). It is omitted otherwise, and the client's
  default stays "first row".
- **Unchanged:** `#` + digit, `##…`, `#-…`, mid-line `#`, and every glued `#`.

### 1.7 Wire changes in this phase

- **`capture-parse`.**
  - Add a `directive` span: the whole token, with `directive: clip|note|template|listen`
    and optional `count`. A bare `%` gets a `directive` span without `directive`, plus
    need `directive`. An unknown token gets the `unknown_directive` diagnostic.
  - Remove the `clipboard`, `pomodoro_note`, `template_header`, `ref_listen`, and
    `global_sub_bullet_*` spans.
  - Keep the semantic `ref_listen` boolean field, updating its doc comment to `%l`.
  - Keep the `pomodoro_note` editor mode and capture kind; they name the feature, not
    the syntax.
  - Add top-level `grammar: 2`.
- **`capture-complete`.**
  - Add context `directive`. Rows have `token` (`%c`…), `name` ("Clip", "Pomodoro note",
    "Template", "Listen"), `summary`, optional `detail`, optional `disabled_reason`,
    `replacement`, and `cursor_after`.
  - Rows are always in the order c, n, t, with `l` only on bare-URL items. A typed
    letter filters.
  - Details come from bob and are cheap:
    - `%c` explains `%c3`;
    - `%n` names the Pomodoro that would receive the note (for example
      `→ 14:00–14:25 BUILD · running`), or why there is none;
    - `%t` explains rows;
    - `%l` says whether a listen command is configured.
  - The directive menu never reads the clipboard.
  - Disabled rows explain the composition rules (for example `%n` next to a route, `%t`
    on a later item, `%c` in a template).
  - Add top-level `grammar` and `initial_selection`.
- **`capture`.** `clip` gains `status`. Delete `--clip=HEADER`.
- **`capture-rewrite`.** Delete `start_template`. The separator toggles stay.
- **zsh.** `%<TAB>` completes c, n, t, l with descriptions, and a bare edge `#<TAB>`
  completes types.

## 2. Contract: `#` after `@route+id` always names a drawer (phase `cli-drawers`)

- **One meaning.** `@route+block-id#…` (glued) always names a drawer of `^block-id`,
  with or without body text. There is no type reading and no toggle fallback.
  - Delete the following:
    - `TypeMarkerAttachment::ExistingTask` and its candidate path;
    - the `marker_only_task_toggle` use for attachment;
    - the `type_marker::completion_at` precedence for `+id#`;
    - the `tokens.rs` `#"NAME"`-to-toggle shortcut, if any survived `bob-cli-ep`.
- **R3.** Retire attached `@route^block-id#…`.
  - A `#` glued after `^id` (not `^id+#`) is refused:
    ``a new task has no drawers: type it with a separate `#fix` (`Fix the parser @sase^parser #fix`)``.
  - Delete `TypeMarkerAttachment::ProjectTask` and, with it, the attachment enum if only
    `Free` remains.
  - `@r^id+#NAME` (a project-note Pomodoro) and the misordered `@r^id#NAME+` error are
    unchanged.
- **A separate `#type` on a body-bearing `@route+id` item retypes `^block-id`.** This
  covers both a plain child and a drawer entry.
  - It replaces today's `type_marker_not_applicable` refusal for sub-bullets in
    `task_theme.rs` `type_marker_not_applicable_error`.
  - No Ensure Next. Lanes are untouched, and only a marker-only `@route+id #type`
    changes lanes, as today.
  - The `[type::]` edit is staged before the write. The existing D8 applier
    (`task_theme.rs` `relocate_for_retype`) moves a live Task Link.
  - One transaction: an invalid type, a closed target, or a strict-cap refusal leaves
    nothing written.
  - Completion reports the subject as `existing`, with the target's current type, as for
    Ensure Next retypes.
- **`!` toggle.** `@r+id#drawer!` is refused, with copy that says the `!` toggle takes
  no drawer.

## 3. Contract: task drawers (phase `cli-drawers`)

A **task drawer** is a direct child bullet of a `#task` line that has no checkbox and
whose whole text is an ALL-CAPS title (`REQUIREMENTS`, `FUTURE WORK`), with its notes
nested beneath it.

- Managed lines and logs are never drawers (Depends-On, BLOCKS, LINKS, LINKED FROM, and
  the Schedule, Work, Rotten, and Cancel logs).
- Rename `capture_task_sections.rs` to a drawer module, and `is_section_title` to
  `is_drawer_title`. It keeps today's shared title grammar, which bob-plugins
  `PROJECT_SECTION_TITLE_RE` mirrors.
  - `is_pomodoro_name` keeps its own `+` variant.

### 3.1 Names

One `#name` lexer serves types and drawers (`pomodoro_literal.rs`
`lex_exact_name_after_hash`).

- **Slug.** `#` + ASCII letter, then ASCII letters, digits, and `-` (`#requirements`,
  `#future-work`).
- **Literal.** `#"…"` / `#“…”`. Once uppercased and whitespace-collapsed, it must pass
  `is_drawer_title` (`#"NON-GOALS"`, `#"Q&A"`, `#"API (V2)"`).
  - The old unquoted `& ' ( ) , . /` now need the literal form, so trailing punctuation
    is never swallowed.
- **Empty `#`.** This is incomplete and needs `drawer`, so the menu opens. Anything else
  (`#req.`, `#a+b`) is `invalid_drawer`, with teaching copy.
- **Keys.** These follow the rule task types use (`type_marker.rs` ~1077):
  - A title's key is uppercase with whitespace collapsed.
  - A literal matches by key.
  - A slug matches by key either as written or with hyphens as spaces.
- **Marker-only.** An item whose text is empty once every marker is masked needs
  `drawer_text`, even if it has authored children.
  - Execution refuses:
    ``add the note text (`Postgres 17 minimum @foo+bar#requirements`)``.
  - It never creates an empty drawer and never toggles the task.

### 3.2 Resolution at commit (R1)

Matching is exact. Menu state never decides.

1. **Scan.** List the target's drawers: direct children only, using the existing scan
   with the new predicate.
2. **One match.** Append the entry at the end of that drawer's own block, before any
   managed log nested under it. This is today's `section_insertion`.
3. **Two or more matches** (for example `FUTURE WORK` and `FUTURE-WORK` against
   `#future-work`). Refuse with `ambiguous_drawer`, naming both line numbers. bob never
   picks one.
4. **No match, reserved key.** Refuse with `reserved_drawer`, for example
   `LINKS is managed by Bob; name the drawer something else`.
   - Reserved keys are every managed line and log label, plus `TASKS`.
   - Derive them from the existing label constants (`MANAGED_TASK_LOG_LABELS`, relation
     and dependency labels), never a new hand-written list.
5. **No match, near-duplicate slug.** A slug whose key is a strict prefix of a drawer
   key on this task, or within edit distance 1 of one, is refused with
   `drawer_near_duplicate`:
   ``did you mean `#requirements`? write `#"REQ"` to create REQ``.
   - Quoted literals and `--drawer` skip this guard.
6. **Otherwise, create** the drawer and nest the first entry under it.
   - **Spelling.** For a literal, its uppercased text. For a slug, the catalog's
     most-used spelling among matching keys, else the slug uppercased with hyphens as
     spaces.
   - **Placement.** Use the anchor plain sub-bullets already use: the task's last direct
     child, before the first direct Schedule, Work, or Rotten log, and after a
     Depends-On first child. Indentation and line endings come from the existing
     helpers.
   - **Near-miss warning.** If the new key is within edit distance 2 of a catalog key
     used in at least two notes, also emit the warning `drawer_near_catalog`:
     `new drawer REQUIRMENTS · did you mean REQUIREMENTS?`.
7. **Batch.** Re-resolve from staged text between items, so two rows into one new drawer
   create it once. Any later failure rolls back the whole draft.
8. **`^prj` targets.** If the target line is a project note's own `^prj` task,
   `cli-drawers` refuses: ``use `@<note>#<Heading>` for a project section``. Phase
   `cli-project-drawers` replaces this refusal (§4).

Drawer entries are plain bullets, with no `#task` and no `[created::]`, matching what
promotion moves. Authored children and `%c` lines nest under the entry.

### 3.3 Before and after

`Postgres 17 minimum @foo+bar#requirements #fix`:

```markdown
- [ ] #task Upgrade Postgres [created::2026-07-31] ^bar
  - 🗓️ **SCHEDULE LOG**
    - _2026-08-02_ — 🎲 P0 → P2 · in **12** (8–30) days
```

```markdown
- [ ] #task Upgrade Postgres [type:: FIX] [created::2026-07-31] ^bar
  - REQUIREMENTS
    - Postgres 17 minimum
  - 🗓️ **SCHEDULE LOG**
    - _2026-08-02_ — 🎲 P0 → P2 · in **12** (8–30) days
```

The human output and dry-run preview state both effects in one block:

```text
foo › ^bar Upgrade Postgres › REQUIREMENTS        new drawer
  - Postgres 17 minimum
type  BUILD → FIX · Task Link moved under FIX
```

### 3.4 Catalog

Add a drawer catalog that reuses the `relation_labels/scan.rs` pattern:

- a per-note mtime and size cache under
  `bob_cli_cache_dir()/drawers/catalog-v<N>-<sha256(vault)>.json`;
- a 60-second snapshot TTL and atomic writes;
- a sweep of old versions;
- invalidation after a capture commit, next to the relation-label invalidation in
  `capture/commit.rs`.

What it scans:

- **Sources.**
  - Drawer children of `#task` lines, using the predicate and skipping fenced text.
  - `##` headings in project notes, keyed uppercase. Exclude the structural headings:
    Tasks, the status-group titles, Pomodoros, Events, and Highlights, taken from the
    existing constants.
- **Excluded.** Reserved keys, dot-directories, and `_templates/`. Notes under `done/`
  count toward frequency but not recency.
- **Ranking.** Distinct notes using the key that were modified in the last 90 days, then
  all-time note count, then most recent mtime.
  - On an empty query, show only keys used in at least two notes, and at most eight.
    This keeps one-off report H2s off the default list.
  - A typed query matches the whole catalog by key or slug prefix, then substring.
- **Not in v1:** usage boosts and Ctrl+D hide. Both can be added later without a break.

### 3.5 CLI

- **`bob capture -D, --drawer TITLE`** replaces `-S, --task-section`. Delete `-S`; it is
  an option, not a command, so it gets no alias.
  - It requires `--route` and `--task`/`--task-ref`, as `-S` did.
  - It matches by key, creates the drawer when missing, and applies the reserved and
    ambiguous refusals. Being explicit, it skips the near-duplicate guard.
  - Update every "cannot be combined with … --task-section …" string in `markers.rs`,
    `item.rs`, and `tokens.rs`.
- **`bob capture-drawers`** is the canonical command (`Section::CaptureProtocol`,
  alphabetical), with the same arguments `capture-task-sections` takes.
  - Its JSON:
    `{ok, schema_version: 1, route, block_id, ref, count, drawers: [{title, key, slug, line, entry_count}], recent: [{title, key, note_count, last_seen}]}`.
  - `capture-task-sections` becomes a permanent hidden alias with byte-identical
    behavior.
  - Update the registrations: `src/native.rs`, `src/runner.rs`, `completion/tree.rs`,
    and `completion/kinds.rs`/`providers.rs`, which hold the `--drawer` value
    completion.
  - Update the `justfile` smoke line and the help fixtures (`tests/fixtures/help/`).
- **Docs.** The generated `docs/reference/cli/capture-task-sections.md` becomes
  `capture-drawers.md`.
  - Update the `mkdocs.yml` nav, and add a `redirect_maps` entry from the old page so
    the published URL survives.

### 3.6 `drawer` completion context

- **Trigger.** The caret is inside, or at the end of, a glued `#…` after `@route+id`,
  including an empty `#`, with or without body text. This replaces `task_section`
  everywhere: `CompletionContext`, `Candidates`, the engine dispatch, `render.rs`
  `context_label`, and `shell.rs`.
- **Top level.**
  - `target: {route, block_id, text}`, which the Mac header uses.
  - `initial_selection: "none"` when the query is empty.
  - `grammar`.
- **Rows, in order:**
  - group `task`: the target's drawers, in document order;
  - group `recent`: catalog keys not on this task;
  - group `new`: one row, only for a non-empty typed query with no exact key among the
    other rows.
- **Row fields:**
  - `title`, `key`, `group`;
  - `entry_count` (task rows);
  - `note_count` and `last_seen` (recent rows);
  - `detail`: bob-authored copy, for example `3 notes` or
    `used in 12 notes · 3 days ago`, with correct singular and plural;
  - `creates` (true for recent and new rows);
  - `refused`: an optional reason when the commit would refuse, such as a reserved name
    or a near-duplicate;
  - `consequence`: bob-authored, for example `Adds after "2 GB RAM" in REQUIREMENTS` or
    `Creates FUTURE WORK under ^bar`;
  - `replacement`: `#slug` when the slug resolves back to exactly this title, else
    `#"TITLE"`. It covers the `#` and the name, with no trailing space.
  - `cursor_after`.
- **A missing or ambiguous target** returns a diagnostic and no rows, never a plausible
  create row.
- **Latency.** The task's own rows read only the target note.
  - Time `capture-complete` warm and cold on an apollo-sized vault, and record the
    numbers in the phase bead note.
  - Target: warm drawer menu within about 20 ms of the `task_type` menu on the same
    draft.

### 3.7 Wire changes in this phase

- **`capture-parse`.**
  - Add a `drawer` span: from the `#` through the name, carrying `title` and `key`. It
    touches but never overlaps the block-ID span, because the Mac drops all highlighting
    on overlap.
  - Add the needs `drawer` and `drawer_text`.
  - Remove the `sub_bullet_section` span and the `task_section` need.
  - Remove the `task_type` span on `+id` and `^id` suffixes.
  - Bump `grammar` to 3.
  - Keep `capture-parse` lexical. A drawer's existence is reported by the dry-run
    preview, not by the span.
- **`capture --format json`.** Each item gains `drawer: {title, key, created}`,
  replacing `parent_section`. Drawer and plain-child items now carry the type row.

### 3.8 Vocabulary sweep

Rename "task section" to "drawer" in every Rust identifier, JSON name, CLI help string,
error, test name, and doc:

- `docs/capture.md`, `docs/reference/capture-syntax.md`, `docs/completion.md`,
  `docs/task-types.md`, `docs/projects.md`, `docs/reference/json-outputs.md`, and
  `docs/_meta/search_smoke.yml`;
- the generated CLI pages.

`docs/projects.md` should call qualifying ALL-CAPS children "drawers" where it describes
promotion and reversal.

Do not rename the unrelated `task_status_hooks` `grouped_task_sections`.

## 4. Contract: `^prj` drawers (phase `cli-project-drawers`, R10)

- **When it applies.** The `+id` target line is the project note's own `^prj` task
  (`projects/tags.rs` `has_trailing_prj_anchor`, `projects/scan.rs`
  `is_valid_prj_task_line`). `#drawer` then resolves against the note's `##` headings by
  the same exact key rules.
  - Exclude the structural headings: Tasks, the status-group titles, Pomodoros, Events,
    and Highlights.
- **Match.** Append `- <text>` (with its authored children and clip lines) after the
  section's last nonblank line, before the next heading of level 2 or higher.
- **Missing.** Append `## Title Case` at the end of the note, then the entry. Use the
  promotion casing (`capture_project_note.rs` `format_project_section_title`), matching
  `docs/projects.md` ("a new `## Title` section is appended at the end of the note").
  - The reserved, ambiguous, and near-duplicate rules apply against the note's eligible
    H2 keys.
- **Menu.** The drawer menu's `task` group lists those H2s, with `entry_count` set to
  their top-level list items. The consequence copy says "section", for example
  `Adds to ## Requirements`.
- **Unchanged.** Plain `@proj+prj` (no drawer) is unchanged. A separate `#type` on a
  `^prj` line follows the existing project typing rules.
- **Additive.** This phase makes no `grammar` bump and no `!`.

## 5. Unchanged on purpose (R13)

These keep their current behavior:

- `@@route`;
- `@note#Heading`, including its prefix match and pre-heading fallback. The asymmetry
  with drawers is deliberate: a heading capture never creates anything.
- `@note:id#NAME`, `^route:id#NAME`, `=#NAME`, `=<X>#NAME`, `@r^id+#NAME`, and `x#bugs`;
- the `pomodoro_name` menus, the `#type` lexer and its refusals for unconfigured types
  (PT12), and D8 relocation;
- the separator toggles.

## 6. Bob Mac Capture experience

### 6.1 Unselected menus (phase `mac-directives`, R7)

- **Decoding.** Decode `initial_selection`. When it is `"none"`,
  `selectedCompletionIndex` stays unset.
- **Keys:**
  - **Return** submits the draft unchanged, and bob refuses the incomplete marker in the
    preview.
  - **Tab, ↓, and Ctrl-N** select the first row.
  - **Typing** gets a fresh response that preselects the best row.
  - **Escape** closes the menu.
- **Scope.** This applies to every context. It is how `#`, `%`, and `@note+id#` open.
- **Precedent.** `decisions:relation-card-tab-accepts-ctrl-d-hides`.

### 6.2 `%` directive menu (phase `mac-directives`)

```text
╭─ CAPTURE AS ───────────────────────────────────────────────╮
│ %c  Clip            Paste the clipboard under this item     │
│                     %c3 takes your last 3 clips             │
│ %n  Pomodoro note   Add this line to the current Pomodoro   │
│                     → 14:00–14:25 BUILD · running           │
│ %t  Template        Repeat this item once per row below     │
├─────────────────────────────────────────────────────────────┤
│ ⇥ take the first · type a letter · ⏎ capture as typed       │
╰─────────────────────────────────────────────────────────────╯
```

- **Copy.** Every string comes from bob.
- **Disabled rows** are dimmed with bob's reason, and Return on them is a no-op.
- **Footer.** It shows the selected row's summary, or the static key hint when nothing
  is selected.
- **Directive spans** share one new tint family, so every `%` token reads as one family.
- **Bare `#`.** The Task type menu (`CaptureTaskTypeMenuView`) now opens on a bare `#`
  as well, through bob's `task_type` need, with nothing selected.
- **Completion triggers.** Add `directive` and the bare-`#` case to the needs and span
  kinds that request completion (`CapturePanelModel.swift` `shouldRequestCompletion`).

### 6.3 Clip card (phase `mac-directives`, R8)

- **When to read.** When the latest parse reports a `directive` span with
  `directive: "clip"`, the debounced live preview drops `--no-clip`, and bob reads
  through the same path the capture will use.
  - `BobProcessClient.preconditionLivePreviewArguments` becomes "omit `--no-clip` only
    when the latest parse reported a clip directive".
  - Opening the `%` menu never reads the clipboard.
- **The Clip card** renders bob's `clip` object:
  - a kind header ("Clipboard · 3 lines", "2 files · 1 image", "Snippet · 42 lines →
    file/clip-…md", "Last 3 clips");
  - the child lines exactly as they will be written;
  - image thumbnails from `attachments[].source` when the file is readable;
  - entry dividers for `%c<N>`;
  - a quiet placeholder for `status: "not_read"`.
- **Retired caption.** Delete "Clipboard markers stay literal in live preview" and
  `livePreviewUsesLiteralClipboard`.
- **Refresh.** Re-run the preview when the panel activates. While the panel is key and
  the draft has a clip directive, poll `NSPasteboard.general.changeCount` at no more
  than 1 Hz. The counter reads no content.

### 6.4 Drawer menu (phase `mac-drawers`)

```text
╭─ DRAWER · ^bar Upgrade Postgres (foo) ─────────────────────╮
│ ON THIS TASK                                                │
│   REQUIREMENTS                                    3 notes   │
│   FUTURE WORK                                     1 note    │
│ RECENT                                                      │
│   MOTIVATION                  used in 12 notes · 3 days ago │
│   ACCEPTANCE                  used in 4 notes · last week   │
├─────────────────────────────────────────────────────────────┤
│ Adds after "2 GB RAM" in REQUIREMENTS                       │
╰─────────────────────────────────────────────────────────────╯
```

- **Groups.** The group headers reuse the `pomodoro_name` header style.
- **New row.** It reuses the create-row style, with a `＋ New drawer PERF BUDGET` label
  and dashed treatment. A refused row is dimmed with bob's reason, and Return on it is a
  no-op.
- **Footer.** It shows bob's `consequence` for the selected row, or the key hint when
  nothing is selected.
- **Counts** are bob's `detail` strings. Delete the Swift `"\(childCount) items"`.
- **Drawer spans** use the section color family.
- **Previews** render the `drawer` object ("new drawer" capsule when `created`) and the
  retype row together, as in §3.3.

### 6.5 Grammar gate (R12)

- **The check.** The Mac reads `grammar` from `capture-parse`.
  - After `mac-directives` it requires ≥ 2; after `mac-drawers`, ≥ 3.
- **When bob is too old.** If the field is missing or lower, show "Update bob to use
  drawers and % directives", disable Capture, and keep the draft. Never translate to old
  syntax.
- **Docs.** Update the `docs/protocol.md` Compatibility table.
- **Old app with new bob.** This degrades briefly:
  - unknown contexts decode as generic rows;
  - `@@@` does nothing;
  - a bare `#` preselects row 0 until `mac-directives` lands.

  So land `mac-directives` promptly after `cli-directives`.

## 7. Memory

Phase workers edit memory only through `/sase_memory_write`, only as allowed below.
After each edit they run `sase memory init` and `just docs-gen`.

> [!decision] memory_drawer_term Phase `cli-drawers` adds strand `glossary:task-drawer`
> (aliases: drawer):
>
> - **What it is.** A named container for one kind of note inside a task: a direct child
>   bullet with no checkbox whose whole text is an ALL-CAPS title (`REQUIREMENTS`,
>   `FUTURE WORK`, `MOTIVATION`), with its notes nested beneath it.
> - **Promotion.** Promoting the task to a [[project-note]] turns each non-empty drawer
>   into a `## Title Case` section, and restoring the project turns them back.
> - **Capture.** `<text> @note+id#drawer` adds to a drawer, matching its title exactly
>   and creating a missing one, before any managed log, with its first entry. Typing `#`
>   after `@note+id` lists the task's drawers first, then drawers used elsewhere.
> - **Exclusions.** Managed lines and logs, and `## Tasks`, are never drawers.
> - **Not to be confused with** a note heading (`@note#Heading`), an
>   [[annotation-section]], a [[pomodoro]], or a [[task-type]].
>
> Phase `cli-project-drawers` adds one sentence: on a project's `^prj` task, the note's
> eligible `##` sections are its drawers.

> [!decision] memory_grammar_record Phase `cli-drawers` adds the accepted record
> `decisions:capture-sigils-say-where-what-how`.
>
> - **Applies to** bob-cli, Bob Mac Capture, and bob-plugins promotion parity.
> - **Claim.**
>   - `@` chooses where.
>   - A `#` glued to a destination names a place inside it, and a free-standing `#` at
>     an item edge names a task type.
>   - `%` letters at an item edge are directives for how an item is captured.
>   - Drawers match exactly at commit and are created on demand. Prefix help lives only
>     in menus.
>   - Retired forms fail through ordinary grammar errors, with no aliases.
> - **Why it won.** Cite the research report and this plan. It beat:
>   - attached `#type` on `+id`/`^id`;
>   - a non-`#` drawer sigil;
>   - word directives;
>   - a fixed `@@template:` header;
>   - a configured drawer vocabulary.
> - **Cost.** A leading `# text` or `%x` needs `\`; old habits fail loudly.
> - **Reopen if** a glued `#` needs a third meaning.
> - **Older records.** Records that name `@@template:` drafts (for example
>   `decisions:similar-tasks-are-a-settled-heads-up`) now mean `%t` template drafts.
>   Those immutable records stay unedited.

> [!decision] memory_glossary_strands Three existing strands are updated, each by the
> phase that makes it true:
>
> - **Capture** (phase `cli-directives`). The live-preview clause becomes: the app's
>   live preview is `bob capture --dry-run`, with `--no-clip` unless bob reports a `%c`
>   clip directive.
> - **Bulk Capture** (phase `cli-directives`). Both `@@template:` mentions are replaced.
>   The item list says "one item per row from a `%t` template draft". The draft-wide
>   sentence says that the `%t` directive marks the draft's first item as a template
>   that declares and expands before parsing.
> - **Annotation Section** (phase `cli-drawers`). "Not a Markdown `##` heading or a
>   `bob capture` target section." becomes "Not a Markdown `##` heading or a
>   [[task-drawer]]."

## 8. Phases

### Phase `cli-directives` (bob-cli, large, `feat(capture)!`)

**Build:**

- A new `src/native/capture_language/directive.rs`. It is an offset-preserving masking
  lexer in the style of `type_marker.rs`. It owns:
  - the position rule;
  - complete, incomplete, unknown, and escaped tokens;
  - spans, needs, and errors;
  - the item-level `Directives { clip, note, template, listen }` result.
- Execution, the editor, completion, templates, rewrite, and `capture-related` all see
  the masked view.

**Delete the old lexing:**

- trailing `#`: `markers.rs` `is_pomodoro_note_marker`, the `line.rs` 62–99 branch, and
  the `editor_classify.rs` `pomodoro_note_marker_parse` slot;
- `%`/`%N`/`%header`: `markers.rs` `parse_clip_token`, the clip part of
  `extract_terminal_markers`, and `ClipRequest::Current{header}`;
- `URL %`/`URL @`: `url_routing/intent.rs` `is_standalone_listen_marker`, and the
  `completion.rs` `cursor_in_listen_marker`.

**Route the new results** into the existing executors: Pomodoro note, clip plan, ref
listen, and template.

**Implement the rest of the contract:**

- §1.3 clipboard reading, and its CLI flags (`capture/cli.rs`, `capture_flag_guard.rs`,
  `completion/kinds.rs`);
- §1.4 `%t` framing, the shelf changes (`capture_templates/command.rs` and `store.rs`),
  and the `@@`-template deletions;
- §1.5;
- §1.6 bare `#`;
- §1.7 wire changes;
- the new `capture_complete/directives.rs`;
- `scripts/dev/capture_related_calibrate` and the `src/runner.rs` template example.

**Help and docs:**

- Update the help in `capture/cli.rs`, `capture_parse.rs` (including the Modes/Needs
  list), `capture_complete/cli.rs`, `capture_rewrite.rs`, and `capture-templates`. Then
  run `just docs-gen`.
- Rewrite the affected docs in place:
  - `docs/capture.md`: Grammar at a glance gets the three rules, plus Global
    destination, Bulk templates, Saved templates, Clipboard, the reading-queue listen
    section, Pomodoro notes, options, JSON, capture-parse, capture-rewrite, and
    capture-templates;
  - `docs/reference/capture-syntax.md`;
  - `docs/completion.md`;
  - `docs/reference/{json-outputs,compatibility,configuration}.md`;
  - `docs/getting-started.md`, `docs/guides/recover-mac-capture.md`, and
    `docs/_meta/search_smoke.yml`, which needs queries for directives and `%t`.
  - Introduce a short "Directives (`%`)" section that the clip, Pomodoro-note, template,
    and listen sections link to.

**Memory and beads:**

- Memory: `memory_glossary_strands` (the Capture and Bulk Capture strands).
- Add a note to each open bead that still names `@@template:`, saying that `%t` replaced
  it and that shelf IDs are unchanged. At planning time those were `bob-cli-82`,
  `bob-cli-83`, `bob-cli-8h`, and `bob-cli-8i`; check `bob-cli-8b`'s land as well.

**Tests:**

- Unit and CLI coverage for every §1 row, including the cases below. Rewrite the
  existing suites
  (`tests/cli/capture/{bare,clip,ref,template,parse_template,template_expand,rewrite,batch,parse}.rs`,
  `capture_language/tests/*`, `capture_complete/tests/*`,
  `tests/cli/capture_templates.rs`) rather than keep parallel old and new cases.
- Directive tokens: a bare `%`; unknown letters; `%c3`, `%c1`, `%c0`; duplicates; `\%`;
  prose `50%`; URL `%25`; `%l` only on URLs; `%n` fallback order and the no-Pomodoro
  message; `%n %c`.
- Templates: `%t` leading, trailing, and on a child line; a misplaced `%t`; `%c`/`%t`
  injected through a row; limits; `-E` parity with the old engine output for the same
  definition; shelf IDs unchanged.
- Clipboard: a `not_read` dry run; a resolved dry run through `BOB_CLIPBOARD_CMD` and
  `BOB_CLIPBOARD_HISTORY_CMD` that writes no files; the `--no-clip` + `%c` refusal.
- Bare `#`: `initial_selection` on `#` and `%` only when the query is empty;
  non-overlapping spans for every directive placement.

**Done when** `just check` and `just docs-check` pass, and
`git grep -n -e '@@template' -e '@@@' -e start_template -e template_header -e TemplateHeader -e global_sub_bullet -e GlobalSubBullet -e parse_clip_token -e is_pomodoro_note_marker -e is_standalone_listen_marker -e 'clip=HEADER'`
outside `CHANGELOG.md` and `sase/` finds only narrow negative tests.

### Phase `cli-drawers` (bob-cli, large, `feat(capture)!`)

**Implement** §2 and §3:

- §2 deletions and the child-capture retype (`capture/plan.rs`, `capture/task_theme.rs`,
  `capture_language/type_marker.rs`, `item.rs` `resolve_sub_bullet_kind`, `tokens.rs`
  `parse_sub_bullet_route_token`, `editor_classify.rs` `classify_sub_bullet_token`);
- the drawer module, with resolution, the guard, reserved keys, and creation in
  `capture/sub_bullet.rs`;
- the `^prj` refusal;
- the catalog;
- the `drawer` context;
- `-D/--drawer` and `capture-drawers` with its hidden alias;
- the wire changes and `grammar: 3`;
- the §3.8 sweep, `docs/task-types.md` (its "Named captures retype the task" forms and
  attached examples), and `docs/projects.md` vocabulary.

**Memory:** `memory_drawer_term` (without the `^prj` sentence), `memory_grammar_record`,
and `memory_glossary_strands` (the Annotation Section strand).

**Tests:**

- **The user example:** REQUIREMENTS existing versus missing; FIX inherited versus
  explicit; a linked parent whose Task Link moves; one atomic result; a closed target
  and an unconfigured `#type` leaving nothing written.
- **Stable meaning:** adding or removing body text around `@foo+bar#` always yields
  drawer completion. A marker-only drawer reports `drawer_text`.
- **Matching:** an exact slug; no prefix match at commit; the guard; the quoted
  override; duplicate titles; `FUTURE WORK` versus `FUTURE-WORK`; reserved keys; the
  catalog near-miss warning; catalog spelling on create.
- **Structure:** tabs and spaces; CRLF; end of file; nested tasks; fenced text; empty
  existing drawers; managed logs; a Depends-On first child.
- **Batch:** two rows create one drawer; a later failure rolls back.
- **Separator toggle** with a glued suffix.
- **Catalog:** cache warm, cold, invalidation, and TTL.
- **`capture-drawers`:** the alias produces byte-identical output.
- **Spans:** non-overlapping drawer spans.
- **Timings:** record the §3.6 latency numbers.

**Done when** `just check` and `just docs-check` pass, and
`git grep -n -i -e task_section -e task-section -e TaskSection -e 'task section' -e sub_bullet_section -e SubBulletSection -e ExistingTask`
outside `CHANGELOG.md` and `sase/` finds only the hidden-alias registration, its alias
test, and `grouped_task_sections`.

### Phase `mac-directives` (bob-mac-capture, large)

**Implement:**

- §6.1;
- §6.2: a new `directive` context row view in `CompletionRowContent.swift` and
  `CapturePanelView.swift`, plus the `directive` semantic category in
  `CaptureEditorPalette.swift`;
- §6.3;
- §6.5 with ≥ 2.

**Delete:**

- `CaptureTemplateStart.swift`, its tests, and the `@@@` keystroke and retry paths in
  `CapturePanelModel.swift`;
- the `@@` template hint and the hard-coded `"@@template:"` row. The Command-S save
  message shows bob's error copy instead of a hard-coded string.
- `CaptureGlobalDestination` sub-bullet mode and `scopeSummary` "under ^id", and the
  `global_sub_bullet_*` span mappings;
- the `clipboard`, `template_header`, and `ref_listen` span mappings.

**Fixtures and docs:**

- Regenerate fixtures from a bob built from bob-cli master that includes
  `cli-directives` (`BOB_BIN`; extend `Scripts/generate-*-fixtures.sh` with a directive
  and clip generator, so these are no longer hand-maintained).
- Update `Tests/Fixtures/fake-bob`.
- Update the docs: capture-panel, capture-previews, keyboard, saved-templates,
  task-pickers, and protocol.

**Tests** (hosted or portable, as the target allows):

- `jot idea #` and `jot idea %`: Return submits, and the preview shows bob's refusal.
  Tab takes row 0. Typing filters. Escape closes.
- A disabled directive row is a no-op.
- Live-preview argv: `--no-clip` only without a clip directive. The precondition holds.
- Clip card rendering for each `mode`, plus `not_read`.
- The grammar gate with a missing or lower `grammar`.
- Stale-response generation checks and Unicode ranges.

**Build** per `AGENTS.md`: `just portable-check` on a host with Swift, and the Mac
scratch-copy typecheck. Otherwise report "macOS targets unverified".

**Done when**
`git grep -n -e '@@template' -e TemplateStart -e start_template -e global_sub_bullet -e 'stay literal'`
finds nothing outside the changelog or history.

### Phase `cli-project-drawers` (bob-cli, medium, `feat(capture)`)

**Implement** §4: replace the `cli-drawers` `^prj` refusal, extend the drawer module and
menu rows, and update `docs/capture.md`, `docs/projects.md`, and the `capture-drawers`
JSON doc.

**Memory:** `memory_drawer_term` (the `^prj` sentence).

**Tests:**

- An existing H2;
- a missing H2 appended at the end, with promotion casing;
- excluded structural headings;
- reserved, ambiguous, and guard refusals against H2 keys;
- authored children and `%c` lines under the entry;
- a separate `#type` on `^prj`;
- menu rows on a `^prj` target;
- a promotion round trip: capture into a task drawer, promote the bytes with a fixture
  equal to the plugin output, then capture `@proj+prj#same-key` into the H2.

**Done when** `just check` and `just docs-check` pass.

### Phase `mac-drawers` (bob-mac-capture, medium)

**Implement** §6.4, and raise the §6.5 gate to ≥ 3.

**Delete** `task_section` row rendering and its "items" count,
`testBareTrailingHashDoesNotOpen…` style tests, and any `task_section` fixtures.

**Fixtures and docs:**

- Regenerate fixtures with a drawer generator against a bob that includes `cli-drawers`,
  covering an existing, a recent, a new, and a refused row, plus a `^prj` target if
  `cli-project-drawers` has landed.
- Update `fake-bob`.
- Update the docs: task-pickers, capture-panel, capture-previews, protocol,
  wikilink-completion, and permissions, wherever they mention task sections.

**Tests:**

- empty-query open with nothing selected;
- typed-query best match;
- Return on a refused row is a no-op;
- the footer shows the consequence;
- the preview shows the new-drawer capsule and the retype row together.

**Build** per `AGENTS.md`, as in `mac-directives`.

**Done when** `git grep -n -i -e task_section -e 'task section' -e childCount` finds
nothing stale.

### Phase `plugins-parity` (bob-plugins, small)

**Test.** In `scripts/test-navigation-hotkeys-project-from-task.cjs` and
`scripts/test-navigation-hotkeys-project-reversal.cjs`, add cases that feed a task block
exactly as bob writes it (§3.3 bytes: tab indentation, the drawer before
`🗓️ **SCHEDULE LOG**`, and a literal title such as `Q&A`). Assert:

- the `## Requirements` / `## Q&A` sections;
- the managed log moving to `^prj`;
- reversal producing the same ALL-CAPS key.

**Vocabulary.** Rename "section bullet(s)" to "drawer(s)" in plugin docs
(`docs/bob-navigation-hotkeys.md`) and code comments. Code identifiers may stay.
`bob-project-tasks` "project task sections" is a different concept; leave it.

**Done when** the plugin test suite passes. Do not run `bob plugins sync`: no plugin
behavior changes.

## 9. Landing checks (lander)

- **Retirement sweep.** Run each phase's `git grep` across bob-cli, bob-mac-capture, and
  bob-plugins. Only the hidden `capture-task-sections` alias, narrow negative tests,
  changelogs, and history may remain.
- **Smoke tests.** Run each against a scratch vault with `bob capture --dry-run` and
  `capture-parse` / `capture-complete`:
  - `Some note goes here. @foo+bar#requirements #fix`
  - `jot idea %n`
  - `Build output %c3`
  - `{0} @job+search#leads %t` with rows
  - `https://example.com/essay %l`
  - a bare `#`
  - a bare `%`
  - `@foo+bar#`
- **Release train.** Confirm no `Cargo.toml` version or `CHANGELOG.md` edits, and
  confirm the `!` subjects.

## 10. Risks

| Risk                                               | Mitigation                                                                                |
| -------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| New app with old bob misreads `%t`/`%n`            | the `grammar` gate (§6.5); land bob phases first                                          |
| Typos create junk drawers                          | exact match, the same-task guard, the catalog warning, the preview's "new drawer" capsule |
| Catalog latency on every `#`                       | cached scan; the task's own rows never wait on it; measured in `cli-drawers`              |
| Report-style H2s pollute Recent                    | frequency-first ranking; ≥ 2 notes on an empty query                                      |
| Pasted prose with `%s`/`# Heading` at an edge      | mid-line stays prose; `\%`/`\#` escape; the preview makes it visible                      |
| Same-day reversal of `bob-cli-ep`'s attached forms | delete the branches outright; the decision record makes the reversal intentional          |
| Template shelf beads in flight                     | `cli-directives` notes them; shelf IDs are proven unchanged                               |
| `docs/capture.md` is ~7,400 lines                  | docs ship inside each phase; `just docs-check` and search-smoke gate them                 |
| Future macOS pasteboard prompts                    | reads happen only after an explicit `%c`; the `%` menu never reads                        |
