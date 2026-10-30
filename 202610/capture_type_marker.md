---
tier: epic
title: Capture t:<type> task-type markers with configured-type completion
goal: "Typing `t:` at either edge of any capture line opens a fast, beautiful menu of
  your configured task types in Bob Mac Capture (and zsh). The accepted `t:fix` marker
  types the task the item creates or targets, and the live preview says exactly what
  will happen. Existing Task Links stay in agreement with the new type, and adding `t:`
  never changes what the rest of the draft means.

  "
decisions:
  menu_order:
    ask: How should an empty `t:` menu order your configured types?
    choices:
      config: Config-file order; rows never move, so muscle memory works
      recent: Most recently used first (PT7 Recent); rows shift as usage changes
    default: config
    why: A small, curated vocabulary reads best in a stable order the user chose
    answer: config
  unknown_types:
    ask: What should `t:<name>` do with a name outside your configured `types:`?
    choices:
      warn: Write it, with a "not one of your types; did you mean …?" warning
      refuse: Refuse the capture until a configured type is chosen
    default: warn
    why:
      PT12 says type warnings never block, and the live preview shows the warning first
    answer: warn
  memory_record:
    ask:
      Add a decisions record for the `t:` orthogonality and named-Pomodoro agreement
      rules?
    memory:
      - decisions:type-marker-is-orthogonal
    default: false
    answer: false
phases:
  - id: grammar
    title: bob-cli t:<type> grammar, parse spans, and new-task typing
    depends_on: []
    size: large
    description: "grammar: lex `t:<type>` at line edges with one shared lexer, mask it
      out of the execution, editor, and completion parse paths without moving byte
      offsets, emit the `task_type` span, need, and diagnostics, type newly captured
      tasks (including the named-Pomodoro agreement rule), refuse non-task items with
      teaching errors, and document the marker.

      "
  - id: existing
    title: bob-cli retyping of targeted existing tasks with D8 relocation
    depends_on:
      - grammar
    size: medium
    description: "existing: let `t:` retype the existing task an item targets
      (whole-item links, Ensure Next, toggle, relationship owners), and share the D8
      relocation applier with `bob task move` so the task's live Task Link keeps
      agreeing with its type.

      "
  - id: complete
    title: bob-cli task_type completion context and zsh rows
    depends_on:
      - grammar
    size: medium
    description: "complete: add the `task_type` capture-complete context with
      configured-type candidates, current, inherited, and effect annotations, a
      typed-name row, lossless replacements, and zsh completion rows.

      "
  - id: mac
    title: Bob Mac Capture t:<type> painting, type menu, and preview
    depends_on:
      - grammar
      - existing
      - complete
    size: large
    description:
      "mac: decode the new contract, paint `t:` markers as type pills, open a compact
      task-type menu with a consequence footer and Ctrl+D hide, render the type row and
      ledger relocation in previews, and document and test it all."
proposed_by: bbugyi200.apollo.b6
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-30 04:22:31
status: wip
---

- **PROMPT:**
  [prompts/202610/capture_type_marker.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/capture_type_marker.md)

# Plan: Capture `t:<type>` task-type markers

## What the user experiences

Bryan opens Bob Mac Capture and types `Fix the parser t:`. A compact **Task type** menu
opens at the caret. Its header reads `dev suggests ■ BUILD`. Below it, every configured
type sits on one 30 pt row: a hue swatch, the name in its PT6 pill tint, and a quiet
usage caption. He types `f`, and the list narrows to `FIX`. The footer reads
`this task becomes FIX · dev suggests BUILD`. Return inserts `t:fix`, which the editor
paints as a dimmed `t:` followed by `fix` on a red pill tint. The preview adds a
`Type ■ FIX · dev suggests BUILD` row. A second Return captures:

```markdown
- [ ] #task Fix the parser [type:: FIX] [created::2026-10-30]
```

The same marker works:

- **Leading:** `t:fix Fix the parser`, `@sase t:fix Fix the parser`.
- **On a child line:** `- check the lexer t:fix`, or `- t:fix` alone.
- **On existing tasks:** `^sase:fix-x t:fix` retypes the task, links it under the FIX
  Pomodoro, and keeps any live Task Link agreeing with its new type.
- **In zsh:** `bob capture Fix the parser t:<TAB>`.

## Ground rules for every phase

- **The thin-client decision holds** (`decisions:mac-capture-is-a-thin-client`).
  - bob-cli owns the grammar, completion data, previews, and writes. Bob Mac Capture
    renders what bob returns.
  - bob-cli phases land first. The JSON contract grows additively and `schema_version`
    stays 1. Swift decodes every new field with `decodeIfPresent`, so an older `bob`
    still works; there `t:` is just prose.
- **Task-type rules are in `docs/task-types.md`, and nothing here contradicts them.**
  - Hue comes from Bob (PT5). Pills, swatches, and the dashed unconfigured mark follow
    PT6.
  - Hide affects Recent-style suggestions only (PT9,
    `decisions:theme-hide-is-recent-only`).
  - A task field stores only an exception to inheritance (PT13).
  - Every live Task Link agrees with its task's type, or sits under a container (Routing
    → Agreement).
- **Release train** (`decisions:bob-cli-releases-ride-a-daily-train`). Never edit the
  `Cargo.toml` version or `CHANGELOG.md`. Commits are `feat(capture): …`; nothing here
  breaks a contract.
- **Docs.** Before editing `docs/` or CLI help, read the docs rules with
  `sase memory read docs.md -r "<why>"`. Run `just docs-gen` when clap help changes and
  `just docs-check` before landing. bob-mac-capture docs are mounted into the Bob site
  and gate on that repo's `just docs-check`.
- **Shared vectors.** Do not edit `tests/fixtures/pomodoro_themes/vectors.json`:
  bob-plugins copies it byte for byte. New fixtures go in a new
  `tests/fixtures/capture_type_marker/` directory.
- **Mac builds.** bob-mac-capture's `AGENTS.md` compile rules are mandatory:
  `just portable-check`, a best-effort scratch compile on the mac, and an honest "macOS
  targets unverified" note when the mac is offline.

## 1. Language contract (normative; all phases implement exactly this)

### 1.1 Token

- A type marker is the lowercase sigil `t:` followed immediately by a name.
- **Slug name:** ASCII letters, digits, and `-`, starting with a letter or digit
  (`t:fix`, `t:deep-work`).
- **Literal name:** a quoted string, so names with spaces or punctuation work
  (`t:"DEEP WORK"`, `t:"Q&A"`).
  - It reuses the `#"…"` lexer in `capture_language/pomodoro_literal.rs`: straight or
    curly quotes, and the escapes `\"` and `\\`.
  - The token runs through the closing quote, so a literal may contain spaces.
- `T:` is never a marker; uppercase stays prose, like `X`. Mid-line `t:` tokens stay
  prose (see 1.2).
- **Claimed shapes** (each only at a marker position, 1.2):

  | Shape                                   | Outcome                                                                                                                                                                                        |
  | --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | `t:` with an empty name                 | Incomplete: the item needs `task_type` and execution refuses (1.9)                                                                                                                             |
  | An unterminated literal (`t:"DEEP`)     | Incomplete                                                                                                                                                                                     |
  | A name containing `+` (`t:bob+decks`)   | Invalid: a task has exactly one type                                                                                                                                                           |
  | Any other remainder (`t:fix.`, `t:[x]`) | Invalid, with a teaching error. A marker-position `t:` never silently becomes prose                                                                                                            |
  | `\t:<anything>`                         | Escape: the body keeps the literal `t:<anything>` without the backslash. This mirrors `\$` (`markers.rs` `record_priority_escape` / `unescape_priority_text`); a mid-line `\t:` stays verbatim |

### 1.2 Position: either edge of any line

A `t:` token is a marker when only **marker tokens** stand between it and the start of
its line, or between it and the end of its line.

- The line is an item's first line, a later line, or an authored child bullet's body
  after its `- ` prefix.
- **Marker tokens** are:
  - the existing trailing-transparent set (`dependencies.rs` `is_trailing_transparent`:
    `@…` routes and `@@` declarations, the bare `#`, `s:`, `s:<N>`, `$…`, `\$…`, `%…`);
  - complete or partial `&` / `*` / `~` / `~~` modifiers;
  - other `t:` tokens.
- **Examples:**
  - Markers: `t:fix Fix it`, `@sase t:fix Fix it`, `Fix it t:fix`,
    `Fix it t:fix s:2 @sase`, `Fix it @sase t:fix`, `- check t:fix`.
  - Prose: `Ask about t:fix today`.
- **Continuation lines:**
  - A later line, bulleted or not, that holds only a type marker (`t:fix` or `- t:fix`)
    types the item and contributes no bullet. This is like a child that holds only
    dependency modifiers, and it makes "start of any line" literally true.
  - Item boundaries are always computed on the unmasked draft, so such a line belongs to
    the item above it. It is never a blank-line separator.

### 1.3 One marker per item, on any line

- Like `s:<N>`, a type marker configures the whole item, whichever of its lines it sits
  on.
- A second marker anywhere in the same item fails with the existing duplicate machinery:
  `duplicate_marker_error("type marker (t:<type>)")` in execution, and the
  `duplicate_capture_marker` diagnostic in the editor (`editor_parse.rs` `SeenMarkers`).
- In bulk drafts, each item has its own marker. Template rows carry whatever their
  expansion contains. There is no draft-wide `@@t:`.

### 1.4 Orthogonality: the masking invariant

**Removing every type marker from an item never changes what the rest of the item
means.** One shared lexer, `capture_language/type_marker.rs`, finds the markers per
line. Every parse path then sees an offset-preserving masked view of the item, with each
marker's bytes replaced by ASCII spaces, before any existing claim runs. The paths are:

- execution (`item.rs` `parse_capture_item`, including the whole-item claims for `!`,
  `/`, `:`, `=`/`x`, `+query`, `^…`, and refs);
- the editor (`editor_parse.rs` `parse_editor_item_with`);
- completion field detection (`completion.rs` `completion_field_at`);
- template-row expansion.

Spans keep absolute draft offsets, so highlighting, `@route` completion beside a marker
(`Fix it @sa t:fix`), and every existing whole-item form keep working unchanged. For
example, `^sase:fix-x t:fix` masks to a solo `^sase:fix-x`.

The `grammar` phase plans the exact mechanics:

- Masking happens after item boundaries are computed (1.2).
- Bare-URL-list and session-operator-chain splitting must not change because a marker is
  present.

A property test pins the invariant: for a corpus of existing grammar drafts, inserting a
marker at a legal leading or trailing position yields the same parse (mode, body, kind,
route, spans, needs, diagnostics, other markers) minus the marker's own span.

### 1.5 Name resolution

The name resolves against the configured `types:` vocabulary, then the catalog
(`pomodoro_themes::Catalog`).

- **A literal** matches by key equality (`normalize_component`).
- **A slug** matches a type whose key equals the slug. Failing that, it matches a type
  whose key equals the slug with spaces in place of `-` (`deep-work` → `DEEP WORK`).
  Configured types win ties.
- **Prefixes never resolve at execution.** `t:fi` is the unknown type `FI`. The menu
  exists so nobody has to type whole names.
- **The written spelling** is the config or catalog spelling, else the canonicalized
  name (`task_fields.rs` `canonical_theme_name`).

An unknown or unconfigured name follows the `unknown_types` decision:

> [!decision] unknown_types = warn The capture proceeds and writes the canonical name,
> adding the existing structured `theme_not_configured` warning
> (`FXI isn't one of your types; did you mean FIX?`). `capture-parse` marks the ThemeRef
> `configured: false` / `known: false`, so editors draw the dashed mark.

> [!decision] unknown_types = refuse When a `types:` vocabulary is configured, an
> unconfigured name fails before any write with `type_not_configured` and the same
> did-you-mean copy. With no vocabulary configured, unknown names are still written with
> a warning.

### 1.6 What a marker types

**`t:` types the item's subject task:** the task the item creates, or the existing task
it targets. It never types a prerequisite, a blocked task, a relation target, or a
Pomodoro.

| Item (after masking)                                                                                                                                     | Subject            | Phase         |
| -------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------ | ------------- |
| New task: `text`, `text @r`, `@r text`, `text @r^id`, `-r`/`--route`, `-R` URL kept as a task, template rows                                             | the new task       | grammar       |
| New Pomodoro-linked task: `text @r:id[#N][=X]`                                                                                                           | the new task       | grammar       |
| New task with `&` / `*` / `~` / `~~` modifiers                                                                                                           | the new task       | grammar       |
| Whole-item link `@r:id[#N][=X]`, `^r:id…` (not `=x…`)                                                                                                    | the existing task  | existing      |
| Ensure Next `@r+id[#N]`; toggle `@r+id!`                                                                                                                 | the existing task  | existing      |
| Relationship-only existing owner (`^r:id &…`, `@r+id ~…`, an `@@r+id` owner)                                                                             | the existing owner | existing      |
| Bullet `@r#Sec`; sub-bullet `text @r+id`; Pomodoro note `#`; project note `@r^id+`; reference URL; `!note:id`; `@r:id=x…`; `=`, `x`, `+N`, `++N`, chains | none               | refused (1.9) |
| Only the marker (`t:fix`, or `t:fix` above child bullets)                                                                                                | none               | refused (1.9) |

Until the `existing` phase lands, the `grammar` phase refuses existing-target forms with
`type_marker_not_applicable`. The message is honest: retyping an existing task with `t:`
is not supported yet.

### 1.7 Field semantics

Terms:

- **X** is the resolved type.
- **I** is the inherited suggestion: PT13 precedence with the task-field step skipped,
  computed on the staged line in its destination note. A `[root::]` field counts.
- **F** is the subject's current valid task field (existing tasks only).

| Condition                                                            | Write                                | `task_theme.action` |
| -------------------------------------------------------------------- | ------------------------------------ | ------------------- |
| `X == I`, no field (a new task, or an existing task without a field) | nothing                              | `kept`              |
| `X == I`, existing task with a field                                 | clear the field (`clear_task_theme`) | `cleared`           |
| `X != I`, `F == X`                                                   | nothing                              | `kept`              |
| `X != I`, otherwise                                                  | `set_task_theme(line, X)`            | `set`               |

- The field is staged on the task line before any placement runs. A bare link form
  (`@r:id`, `=`) therefore places by the new type through the existing "Bare captures
  place by type" path, which already reads an authored `[type::]` field from the staged
  line.
- Type edits never stamp freshness, change lanes, consume dependency guards, or advance
  the review walk (Routing → Agreement).
- An item with a marker **always** reports `task_theme`, unlike named retypes, which
  omit a no-change result. `kept` is a new additive action value meaning "the task
  already has, or inherits, X; nothing was written".

### 1.8 Agreement with a named Pomodoro

When the item also names a Pomodoro D (`#NAME`, `#"NAME"`, `=name`), `t:` replaces the
existing named-capture retype as the type authority:

1. **D is a container** (no type components): field per 1.7, link under D. This agrees
   by definition.
2. **X is one of D's components:** consistent. Field per 1.7, link under D. In
   `BOB + FIX`, `t:fix` is the explicit component pick, like the Obsidian Tab pick.
3. **Otherwise:** refuse before any write with `type_marker_conflict`:
   `` `t:fix` disagrees with Pomodoro BUGS: drop `#bugs` to link under FIX, or drop `t:fix` to type the task BUGS ``

D's resolution (open match, completed "again", or named creation) is unchanged. A newly
created D follows the same three rules: with a vocabulary configured, a new non-type
name is a container.

### 1.9 Errors, diagnostics, and needs

| Code                         | Copy (execution message; editor diagnostic message)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Where                                         |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------- |
| need `task_type`             | `` `t:` is incomplete: type a task type after `t:` (for example `t:fix`) ``                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | parse need + execution refusal                |
| `invalid_type_marker`        | `` `t:bob+decks` names two types; a task has one type: write `t:bob` or `t:decks` `` / `` `t:fix.` is not a task type: names use letters, digits, and `-`; quote other names (`t:"Q&A"`) or write `\t:fix.` to keep the text ``                                                                                                                                                                                                                                                                                                                                 | both                                          |
| `duplicate_capture_marker`   | existing duplicate copy, label `type marker (t:<type>)`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | both                                          |
| `type_marker_needs_task`     | `` `t:fix` needs a task: add the task text (`Fix the parser t:fix`) or target one (`^sase:fix-x t:fix`) ``                                                                                                                                                                                                                                                                                                                                                                                                                                                      | both                                          |
| `type_marker_not_applicable` | `` `t:fix` types a task, but <what this item does>; <hint> ``, per kind: bullet `@notes#Ideas writes a plain bullet` / drop `#Ideas` to capture a task; sub-bullet `@sase+fix-x with text adds a child bullet` / capture `@sase+fix-x t:fix` alone to retype that task; project note `@cash^goog-exit+ creates a project note` / set `task_type:` in the new note; reference `a link queues a reference typed by plan.reading_type` / add `-R` to keep it a task; completion, close, start, adjust, shift, chain, and Pomodoro note each name their own gesture | both                                          |
| `type_marker_conflict`       | 1.8 copy                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | execution (`capture --dry-run` shows it live) |
| `theme_not_configured`       | existing warning                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | capture warnings                              |

When the masked item itself fails (for example a malformed `x` selection), its own error
stands. When the masked item parses as a non-subject form, `type_marker_not_applicable`
wins.

## 2. Wire contract (additive; `schema_version` stays 1)

### 2.1 `capture-parse`

- **New span kind `task_type`.**
  - It covers the whole marker token (`t:`, the name, and any quotes).
  - It carries `themes: [ThemeRef]` with exactly one ref. The ref's `start`/`end` cover
    only the name content (excluding `t:` and quotes), plus `hue`, `hue_source`,
    `known`, and `configured`.
  - Attach it through the existing `capture_parse.rs` theme attachment. Generalize
    `is_theme_name_span` (or the equivalent) so `task_type` spans load the catalog only
    when needed.
- **New need `task_type`** for an incomplete marker; the mode is `incomplete`.
- **Diagnostics:** the codes from 1.9.
- **`body` excludes the marker.** Every consumer of the parsed body (including
  `capture-related` similar-task matching) sees text without it.

### 2.2 `capture` (dry run and real run alike)

- **`task_theme`** on the item:
  `{action: "set" | "cleared" | "kept", theme, previous, inherited}`. The shape is
  unchanged; only the `kept` value is new.
  - Note: the existing JSON key is `theme`, not `type`. Fix the drifted example in
    `docs/capture.md` (§ Task-type changes) while you are there.
- **`type_relocation`** (added by the `existing` phase, omitted when unused):
  `{outcome: "rename_entry" | "append_component" | "move_to" | "rename_placeholder" | "create_below", from: "<entry name>", to: "<entry name>"}`.
- **Human rows** (`capture/output.rs` `format_task_theme_row`):
  - `set` and `cleared` keep their existing rows.
  - `kept` renders as `type   ■ FIX · <label>`, where `<label>` is the label of the
    source that gives the effective type: `this task`, `from sase`, `default type`, or
    `reading task`.
  - `type_relocation` renders as a `ledger` row:

    | Outcome              | Row                                          |
    | -------------------- | -------------------------------------------- |
    | `rename_entry`       | `BUILD → FIX · running entry renamed`        |
    | `append_component`   | `BUILD → BUILD + FIX · running entry shared` |
    | `move_to`            | `moved under FIX`                            |
    | `rename_placeholder` | `BUILD → FIX · planned entry renamed`        |
    | `create_below`       | `moved under new FIX`                        |

- **Warnings:** `theme_not_configured` through the existing structured channel.
- Dry-run JSON equals real-run JSON except for `dry_run`.

### 2.3 `capture-complete` context `task_type` (`complete` phase)

**Trigger.** The cursor is inside, or at the end of, a marker-position `t:` token,
complete or incomplete, as decided by the 1.2 lexer.

- No context for mid-line `t:`.
- No context when the cursor is on another field (another field wins at its own token).

**Top-level fields:**

- `replacement`: the whole token, sigil included.
- `query`: the typed name, unquoted.
- `subject`: `"existing"` when the masked item targets an existing task (the
  existing-task rows of 1.6), else `"new"`.
  - This includes items that are still empty, such as a bare `t:` typed at the start of
    a blank line. The menu must open there.
  - It also includes items that will refuse the marker. Their `capture-parse` diagnostic
    explains why; completion never second-guesses the item kind.
- `theme_suggestion`: the existing `ThemeSuggestion` shape for the subject, or null.
  - New task: the inherited suggestion for its destination (route, `@@` route, or
    inbox).
  - Existing task: `suggest_for_task`, including `inherited`.
  - Reuse `pomodoros.rs` `resolve_task_suggestion`.

**Candidate** (a new `TaskTypeCandidate` variant in `capture_complete/model.rs`):

```json
{
  "replacement": "t:fix",
  "cursor_after": 21,
  "name": "FIX",
  "key": "fix",
  "themes": [
    {
      "key": "fix",
      "name": "FIX",
      "hue": "red",
      "hue_source": "config",
      "configured": true
    }
  ],
  "origin": "configured",
  "last_used": "2026-10-29",
  "days_used": 7,
  "current": false,
  "inherited": false,
  "suggestion_label": null,
  "effect": "set",
  "hidden": false,
  "similar_to": []
}
```

- **`origin`:**
  - `configured` for a vocabulary type;
  - `history` for a catalog type when no vocabulary is configured;
  - `typed` for the use-this-name row.
- **`current`:** true on the row equal to the subject's effective type now. For a new
  task, that is its inherited type.
- **`inherited`** and **`suggestion_label`** (`from sase`, `default type`,
  `reading task`): set on the row equal to the inherited type.
- **`effect`** is what accepting does, per 1.7:
  - `set` writes a field;
  - `clear` removes the field because the row equals the inherited type;
  - `keep` changes nothing.
- **Optional fields** are omitted when false or empty, following existing serde
  conventions.

**Rows.** The empty-query order with a vocabulary follows the `menu_order` decision:

> [!decision] menu_order = config With a vocabulary, an empty query lists every visible
> (non-hidden) configured type in config-file order.

> [!decision] menu_order = recent With a vocabulary, an empty query lists the visible
> configured types in `Catalog::rank_for_menu` order (used types by `last_used`, then
> never-used types in config order).

- **Empty query without a vocabulary:** list catalog types in PT7 Recent order
  (`rank_for_menu`), capped at 12.
- **Non-empty query:** filter on ASCII case-insensitive slugs and keys.
  - Exact matches come first, then prefix matches, then substring matches, each in the
    empty-query order.
  - An exact match on a hidden type still shows, with `hidden: true`.
  - With a vocabulary configured, only configured types match; others come in only
    through the typed row.
- **Typed row:** appended when the query is a creatable name and no row is an exact
  match. It carries `origin: "typed"`, the canonical name, a ThemeRef with
  `configured: false` and/or `known: false`, and `similar_to` (at most two visible
  configured names, using the existing similarity rule).

> [!decision] unknown_types = refuse When a vocabulary is configured, the typed row also
> carries `refused: "type_not_configured"`. Clients render it as unavailable, and
> accepting it is a no-op.

- **Replacement** is lossless (PT8): `t:<slug>` when the slug round-trips to the type's
  key, else `t:"NAME"`.
  - At the leading position, accepting adds one space when the next byte is not
    whitespace, and `cursor_after` lands after it.
  - At a trailing position, the caret lands right after the token.
- **Fetching and filtering never write**, and never touch the journal.
- **Performance:** stay within PT3's warm budget (+5 ms p95 on apollo) and record the
  measurement in `docs/completion.md`'s performance table.

### 2.4 zsh

`capture_complete/shell.rs` maps `task_type` rows to the `task types` group:

- `full`: `t:fix`.
- `description`:
  - `your type · Oct 29 · 7 days`, or `your type · never used`;
  - `current · this task`, or `from sase` (for inherited);
  - `new · not one of your types`.

`bob capture Fix it t:<TAB>` completes the configured types.

## 3. Bob Mac Capture design (`mac` phase)

### 3.1 Editor painting

- **Highlight from the span, not the span kind.** Generalize
  `CapturePanelModel.paintHighlighting` so any span that carries `themes` gets the PT6
  treatment. Today it is gated on `pomodoro_name` and `task_toggle_pomodoro_name`. That
  treatment is:
  - the hue tint behind each theme sub-range;
  - the dashed `systemOrange` underline for `configured == false` once the caret leaves;
  - neutral while typing (`CaptureUnconfiguredThemePresentation.caretIsInName`).
- **Sigil colour.** A new `taskType` semantic category (`CompletionRowContent.swift`
  `captureSemanticCategory`, `CaptureEditorPalette.swift`) paints the `t:` sigil and
  quotes in tertiary label colour. The pill-tinted name reads as the subject and the
  sigil recedes.
- **Warnings.** `visibleThemeNotConfiguredWarnings` also reads `task_type` spans.

### 3.2 Triggering

- Add the `task_type` need and the `task_type` span kind to `shouldRequestCompletion`'s
  lists (`CapturePanelModel.swift`). Typing `t:` at either edge of any line then opens
  the menu after the normal 50 ms debounce.
- Confirm that the `:` separator-toggle intent (`noteSeparatorToggleIntent` →
  `capture-rewrite`) leaves `t:` untouched and adds no visible latency. Add a test.

### 3.3 The task-type menu

The menu reuses the inline `CompletionList` plumbing (selection, Return/Tab accept,
Ctrl-N/P, Esc, `applySelectedCompletionReplacement`). It gets its own presentation for
`context == "task_type"`:

```text
┌────────────────────────────────────────────────┐
│ TASK TYPE                    dev suggests ■ BUILD │
│ ■ [BUILD]  from dev · current          32 days │ ← selection tint
│ ■ [CHORE]                               9 days │
│ ■ [FIX]                                 7 days │
│ ■ [MEET]                            never used │
│ □ [PLAN]   Exempt                      56 days │
│ ■ [READ]                               28 days │
│ ■ [WRITE]                              11 days │
│ ─────────────────────────────────────────────  │
│ stays BUILD · from dev        ⏎ choose  ⌃D hide │
└────────────────────────────────────────────────┘
```

- **Header.** An uppercase `TASK TYPE` caption. On the trailing side, the subject
  context from `theme_suggestion`, using the existing `CaptureTaskThemePresentation`
  phrases: `dev suggests ■ BUILD` with the area dot, `this task is ■ FIX`, or
  `no default here`.
- **Rows.** Single-line rows about 30 pt tall.
  - An 8 pt `PomodoroThemeSwatchView`.
  - The name in a `PomodoroThemePillView`-style tint, with the query match in semibold.
  - Badges: `from <stem>` / `default type` / `reading task` when `inherited`, `Current`
    for an existing subject's current type, then `Pinned`, `Exempt`, `Hidden`.
  - A right-aligned caption: `7 days`, `never used`, or `today`.
- **Height.** Up to 8 rows are visible for this context (other contexts keep 5×48 pt).
  The list sizes to its content, so a 7-type vocabulary shows with no scrolling.
- **Typed row.** Below a hairline divider: a dashed-outline pill, the existing
  `Not in your types` badge, and `did you mean FIX?` built from `similar_to` (decoded
  today but never rendered). See the `unknown_types` callout below the list.
- **Consequence footer.** One caption line for the selected row, mapped from `effect`
  with PT13 pinned copy:
  - `set`: `this task becomes FIX · dev suggests BUILD`;
  - `clear`: `task type cleared · dev suggests BUILD`;
  - `keep`: `stays BUILD · from dev`.

  It is followed by keycaps: `⏎ choose`, `⌃D hide` (or `⌃D show` on a hidden row).

- **Selection.** It starts on row 0, so the order stays stable and predictable. Ctrl+D
  hides or unhides the selected configured or history row through the existing
  `toggleSelectedCompletionThemeVisibility`. Extend `selectedHistoryThemeCandidate`'s
  gate to `task_type` rows. The ghost row stays until the draft changes (PT9).
- **Accessibility.** For example:
  `Task type FIX, red. This task becomes FIX; dev suggests BUILD.` The list reads
  `Task type suggestions, 7 results`. Colour is never the only signal.

> [!decision] unknown_types = refuse A `refused` typed row renders dimmed with
> `Add it under types: to use it`, and Return does nothing.

### 3.4 Previews and notifications

- `standardPreviewItem` (and the other task-creating previews) render a `Type` row from
  `task_theme`: a pill, then the copy. Today only link and toggle previews render
  `task_theme`.
- Extend `CaptureTaskThemePresentation.line` with `kept`. Render `type_relocation` as a
  ledger line using the 2.2 copy.
- The notification body appends `type FIX` when an item sets or keeps a type.

### 3.5 Docs and tests

- **Docs** (bob-mac-capture `docs/`):
  - `protocol.md`: the context, span, need, candidate fields, the `kept` action,
    `type_relocation`, and compatibility rows.
  - `capture-panel.md`: highlighting, inline completion, and the marker list.
  - `keyboard.md`: Ctrl+D in type menus. Also document the existing, undocumented Ctrl+D
    in Pomodoro-name menus.
  - `capture-previews.md`: the type row.
- **Tests:**
  - Record real fixtures with a `Scripts/generate-task-type-fixtures.sh` modelled on
    `generate-relation-label-fixtures.sh`, against a scratch vault with a `types:`
    block.
  - `CaptureCoreTests`: decoding, row content, and presentation copy.
  - Hosted `BobMacCaptureTests`: painting, the completion gate, Ctrl+D routing, and the
    footer.
  - `fake-bob`: `task_type` answers.

## Phase `grammar` — bob-cli grammar, spans, and new-task typing

1. **Lexer.** Add `src/native/capture_language/type_marker.rs` with:
   - token parsing (1.1);
   - the position predicate (1.2), built on `is_trailing_transparent` and the modifier
     scanners;
   - per-item collection with duplicate detection (1.3);
   - the masked item view (1.4).

   Export it through `capture_language/mod.rs`. Execution, editor, completion field
   detection, and template expansion all call this one lexer.

2. **Model.** Add `type_marker: Option<TypeMarker { name, literal, span }>` to
   `ParsedCaptureText`. Set it at every construction site, `None` where not applicable.
   Thread it into the editor `ParsedItem` for spans, needs, and diagnostics.
3. **capture-parse.** Emit the span, need, and diagnostics from 2.1 and 1.9. Generalize
   theme-ref attachment to `task_type` spans.
4. **Execution for new tasks.** In `capture/plan.rs`, stage the field on the formatted
   task line before placement and linking.
   - The formatters are `format_task_line`, `format_task_with_block_id_line`, and
     `format_pomodoro_task_line`. Use `set_task_theme`.
   - Compute I with the batch's `CaptureTaskThemeContext` and `suggest_for_task` against
     the staged destination snapshot, as `capture/task_theme.rs` `apply` does today.
   - Apply 1.7 and report `task_theme` (including `kept`).
   - Apply the 1.8 agreement rule ahead of the existing named retype in
     `apply_new_task`.
   - Refuse non-applicable kinds, and (temporarily) existing-target kinds, with 1.9
     errors.
5. **Unknown names.** Implement 1.5, including the `unknown_types` branch chosen in
   review.
6. **Human output.** Add the `kept` row (2.2).
7. **Docs.**
   - `docs/capture.md`:
     - Grammar-at-a-glance rows for `t:<type>`, `t:"NAME"`, `t:`, and `\t:`.
     - The leading/terminal paragraph.
     - A new `### Typing tasks with t:` section after § Scheduling and priority, holding
       the 1.1–1.9 contract with examples.
     - § Task-type changes (`kept`, and the key-name drift fix).
     - § Interactive editor markers, and the capture-parse spans and needs lists.
   - `docs/reference/capture-syntax.md`: the table row and a `### t:<type> {#t-type}`
     entry.
   - `docs/task-types.md` PT13: add "Typed captures (`t:`)" beside "Named captures
     retype the task", and update the writer-boundary sentence.
   - `src/native/capture/cli.rs` marker help and `capture_parse.rs` help, then
     `just docs-gen`.
8. **Tests.**
   - `capture_language/tests/` for the lexer: positions, escapes, literals, invalid and
     incomplete shapes, duplicates across lines, and marker-only continuation lines.
   - The masking property test (1.4).
   - Editor span and diagnostic tests.
   - `tests/cli/capture/type_marker.rs`:
     - inbox, routed, and `@r^id` writes;
     - `@r:id` placement by the new type;
     - `#NAME` container, consistent, and conflict cases;
     - `kept`, set, and the unconfigured warning (or refusal);
     - every refusal;
     - dry-run vs real-run equality.
   - A `capture-rewrite` test showing `t:` tokens are untouched.
   - A fixture vector file under `tests/fixtures/capture_type_marker/`.

> [!decision] memory_record **Memory.** Use `/sase_memory_write` to add
> `sase/memory/decisions/type-marker-is-orthogonal.md`. It records the masking invariant
> (1.4), the named-Pomodoro agreement refusal (1.8), and the no-prefix-resolution rule
> (1.5), with the rejected alternatives: full grammar integration at every claim site,
> `t:` winning over `#NAME` with drift, and prefix resolution. Cite the epic's approved
> plan and `docs/capture.md` as evidence, then run `sase memory init`.

> [!decision] memory_record = no Write no memory.

Verify with `just check` (and `just all` before landing), plus `just docs-check`.

## Phase `existing` — retyping targeted existing tasks

1. **Remove the temporary refusal** for the existing-target rows of 1.6.
2. **Retype before the gesture.** For each form, read the target's staged line, apply
   1.7 (`set` / `cleared` / `kept` against F and I), and stage the field edit before the
   gesture runs, so bare placement uses the new type.
   - The forms are the whole-item `@r:id` / `^r:id` link (`capture/pomodoro_link.rs`
     `plan_existing_task_pomodoro_link_item`), Ensure Next and toggle
     (`capture_task_toggle`, `capture/ensure_next.rs`, `capture/task_toggle.rs`), and
     relationship-only owners (`capture/plan.rs` `relationship_only_existing_owner` →
     `capture/dependencies.rs` `plan_dependency_only_item`).
   - The 1.8 agreement rule applies to named forms (`@r:id#N`, `=name`, `@r+id#N`).
   - Closed targets are refused (`closed tasks keep their fields as history`).
3. **Shared D8 applier.** Extract the D8 relocation applier from
   `src/native/task_move/command.rs` into a shared module, for example
   `src/native/pomodoro_themes/relocation_apply.rs`.
   - The pieces are `find_live_link`, `entry_child_links`, `rename_entry`,
     `create_below_and_move`, and the `routing::relocate` call.
   - The extracted function takes day-note contents, the task identity, and the new
     type, and returns new contents plus a `RelocatedInfo`.
   - `bob task move` must stay byte-identical: its existing tests are the guard.
4. **Restore agreement.** After a capture retype, the task's live Task Link must agree
   with the new type.
   - Ensure Next already moves the link to the placed entry.
   - Every other form whose gesture does not move the live link applies the shared D8
     applier to the staged daily-note snapshot in the same transaction. Those forms are
     an already-queued whole-item link, relationship-only owners, and a toggle's link
     direction when it is already linked.
   - Report `type_relocation` (2.2).
   - A strict-cap refusal (`RelocateOutcome::Refused`) fails the capture before any
     write with code `strict_cap`.
5. **Human output.** Add the `ledger` relocation rows (2.2).
6. **Docs.**
   - `docs/capture.md`: the existing-task rows and examples in the t: section, and
     `type_relocation` in the JSON docs.
   - `docs/task-types.md` Routing table: a row for "Capture `t:` on an existing task" →
     the type → relocating the live link (D8), and the fact that capture now calls the
     shared applier.
7. **Tests.**
   - `tests/cli/capture/type_marker_existing.rs`: set, clear, and kept for each form;
     every D8 outcome; strict cap; conflict on named forms; dry-run equality.
   - Unit tests for the extracted applier.
   - The existing `tests/cli/task_move.rs` stays green, unchanged.

Verify with `just check`, `just all`, and `just docs-check`.

## Phase `complete` — the task_type completion context and zsh

1. **Field detection.** Add `CompletionContext::TaskType` in
   `capture_language/completion.rs` and detect it with the shared lexer (2.3 trigger).
   Detection runs before marker-index field detection, so `@route` completion beside a
   marker still wins at its own token.
2. **Candidates.**
   - `capture_complete/engine.rs`: a `task_type` arm.
   - A new `capture_complete/task_types.rs`: candidate building, ordering per the
     `menu_order` branch, filtering, the typed row, lossless replacements,
     `cursor_after`, and the `current` / `inherited` / `effect` annotations against
     `subject` and `theme_suggestion`.
   - Reuse `Catalog::load_for_completion`, `config::themes` (`configured_themes`),
     `Catalog::similar_to`, `theme_ref`, and `resolve_task_suggestion`.
3. **Model.** Add `TaskTypeCandidate` to the untagged `Candidates` enum, and the
   top-level `subject` field.
4. **zsh.** Add the `shell.rs` rows (2.4) and an adapter test in
   `tests/cli/completion/`.
5. **Docs.**
   - `docs/capture.md` § `bob capture-complete`: the context, fields, and example JSON.
     Also refresh the stale context list (it omits `pomodoro_start_name` and the
     `relation_*` contexts).
   - `docs/completion.md`: the capture-marker group and the performance row.
   - The `capture_complete/cli.rs` "Contexts:" help, then `just docs-gen`.
6. **Tests.** `tests/cli/capture/complete_task_types.rs`, against a temp config with a
   `types:` block:
   - the empty query in the chosen order;
   - the prefix, exact, and substring tiers;
   - hidden types (excluded, and shown on an exact match);
   - the typed row with `similar_to` (and `refused` under `unknown_types = refuse`);
   - `current`, `inherited`, and `effect` for new, existing-with-field, and
     existing-inherited subjects;
   - the literal replacement for `DEEP WORK`;
   - leading `cursor_after` spacing;
   - child-line and second-line triggers;
   - no context mid-line;
   - `@route` completion beside a marker;
   - no vocabulary → history rows.

Verify with `just check`, `just all`, and `just docs-check`. Record the warm p95 on
apollo.

## Phase `mac` — Bob Mac Capture

Implement section 3 against the landed bob-cli contract (sections 2.1–2.3):

1. **Decoding** (`Sources/CaptureCore/CaptureModels.swift`,
   `CompletionRowContent.swift`):
   - the `task_type` context case;
   - candidate fields `key`, `current`, `inherited`, `effect`, and `refused`;
   - the response field `subject`;
   - the `kept` action;
   - `type_relocation`;
   - the `task_type` span kind and need.

   All are additive with `decodeIfPresent`.

2. **Painting and the completion gate** (3.1, 3.2).
3. **The menu** (3.3) in `CapturePanelView.swift`'s `CompletionList`. Keep row content
   in `CompletionRowContent.swift` and copy in a `CaptureTaskTypeMenuPresentation` in
   CaptureCore, so it is testable on Linux.
4. **Previews and notifications** (3.4).
5. **Docs and tests** (3.5).

Verify with `just portable-check`, `just docs-check`, the scratch mac compile
(`bundle.sh` + `just typecheck-tests`) per `AGENTS.md`, and CI.

Then run a manual smoke on the mac with the installed `bob`. Record a screenshot of the
menu and the preview for the final report:

- `Fix the parser t:` → choose FIX → capture;
- `t:` at the start of a line;
- `- t:fix` on a child line;
- `^<route>:<id> t:fix` on a linked task;
- an unconfigured name;
- Ctrl+D hide, then show.

## Non-goals and follow-ups

- **No draft-wide `@@t:`.** Each item types its own subject.
- **No `t:` on project-note captures.** They refuse with a pointer to `task_type:`. A
  per-bullet `t:` for ` :id` children could follow if wanted.
- **The Obsidian plugins are unaffected.** They do not call `capture-parse` or
  `capture-complete`, and `tests/fixtures/pomodoro_themes/vectors.json` is not touched.
- **No prefix resolution at execution** (1.5).
