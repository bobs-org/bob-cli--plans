---
tier: epic
title: Capture `#<type>` replaces the `t:<type>` task-type marker
goal: 'Typing `#fix` at either edge of any capture line, or right after `@route^id`
  or a marker-only `@route+id`, types the task the item creates or targets. The same
  Task type menu that `t:` opens today appears. An unconfigured type name is refused
  before any write with a clear, teaching message, and `t:` is retired.

  '
decisions:
  bare_hash_menu:
    ask: When should a free-standing `#` open the Task type menu?
    choices:
      letter: Once a letter follows (`#f`); a bare `#` stays the Pomodoro note; no
        Swift change
      open: Also on a bare `#`, with no row selected so Return still submits; needs
        Mac work
    default: letter
    why: Return always accepts row 0 today, so a menu on a bare `#` would break Pomodoro
      notes
    answer: letter
  retire_t:
    ask: What should a marker-position `t:fix` do after the migration?
    choices:
      teach: Refuse with "`t:fix` is retired; write `#fix`" (`\t:` still escapes)
      prose: Become ordinary task text; Bob forgets `t:` entirely
    default: teach
    why: Silently writing `t:fix` into task text would hide a day's muscle memory
    answer: prose
  memory_record:
    ask: Add a decisions record that `#<type>` replaced `t:` and refuses unconfigured
      types?
    memory:
    - decisions
    default: false
    answer: false
phases:
- id: cli
  title: bob-cli `#<type>` grammar, refusal, completion, and docs
  depends_on: []
  size: large
  description: 'cli: switch the shared type-marker lexer to `#`, add the attached
    `@route^id#` and marker-only `@route+id#` forms, refuse unconfigured types, retire
    `t:` and the legacy `#tag` check, move `task_type` completion to `#`, and update
    docs, help, fixtures, and tests in one coherent change.'
- id: mac
  title: Bob Mac Capture fixtures, tests, and docs for `#<type>`
  depends_on:
  - cli
  size: medium
  description: 'mac: confirm the Swift sources stay sigil-agnostic, regenerate the
    task-type fixtures with the new bob, update fake-bob, tests, and docs to `#`,
    and record a smoke on the mac.'
proposed_by: bbugyi200.apollo.be
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-30 15:04:27
status: wip
bead_id: bob-cli-ep
---

- **PROMPT:** [prompts/202610/capture_hash_type_marker.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/capture_hash_type_marker.md)
- **BEAD:** [bob-cli-ep](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ep/README.md)

# Plan: Capture `#<type>` replaces `t:<type>`

## Context

Epic `bob-cli-ee` (approved plan `plan:202610/capture_type_marker.md`, now closed)
shipped the `t:<type>` capture marker. That epic covered:

- the shared lexer `src/native/capture_language/type_marker.rs`, which masks markers so
  the rest of the grammar never sees them;
- new-task and existing-task writers, including D8 relocation;
- the `task_type` completion context and its zsh rows;
- the Bob Mac Capture Task type menu.

Bryan asked to migrate that syntax to `#<type>`:

- `#<type>` works right after the `@file^id` and `@file+id` route tokens
  (`@sase^foobar#fix`).
- It also works on its own, on any line, because many captures have no `@file`.
- Unlike the places where `#NAME` names a Pomodoro, an unconfigured type is an error
  with a good message. "This context" is a `#` that names no Pomodoro.
- The completion menu must look exactly as it does for `t:`.

Nearly everything from `bob-cli-ee` carries over unchanged:

- masking and orthogonality (old plan §1.4);
- subject rules (§1.6);
- field semantics (§1.7);
- agreement with a named Pomodoro (§1.8);
- D8 relocation;
- the `task_type` wire contract;
- the Mac menu.

This epic changes four things, and nothing else:

1. the sigil;
2. the token's attachment points;
3. the unknown-name policy (`warn` → refuse);
4. the fate of `t:`.

## What the user experiences

Bryan types `Fix the parser #f` in Bob Mac Capture.

- The same compact **Task type** menu opens. The header reads `dev suggests ■ BUILD`,
  there is one 30 pt row per type, and the footer shows the consequence.
- Return inserts `#fix`. The `#` is dimmed and `fix` sits on a red pill tint.
- A second Return writes:

```markdown
- [ ] #task Fix the parser [type:: FIX] [created::2026-10-30]
```

The same marker works in all of these places:

- **Either edge of any line:**
  - `#fix Fix the parser`;
  - `@sase #fix Fix the parser`;
  - `Fix the parser #fix @sase`;
  - a child line `- check the lexer #fix`;
  - a marker-only child line `- #fix` (or an unbulleted `#fix` continuation line).
- **After `@route^block-id`:** `Fix the parser @sase^parser#fix` creates `^parser` typed
  FIX.
  - On this form, an empty `#` (for example, typed through the Mac New-ID picker) opens
    the full menu in config order.
- **After a marker-only `@route+block-id`:** `@sase+parser#fix` retypes `^parser` to
  FIX. Ensure Next then places its Task Link by the new type.
  - This is exactly what `@sase+parser t:fix` does today.
- **Existing-task forms keep the free-standing marker:**
  - `^sase:parser #fix` and `@sase+parser! #fix`;
  - relationship-only owners: `@sase+parser#fix &notes:spec` is now valid.
- **In zsh:** `bob capture Fix the parser '#f'<TAB>` completes the configured types.

A typo is refused before anything is written, in the live preview and in the CLI:

```text
`#fxi` is not one of your task types; did you mean `#fix`? Add FXI under `types:` in <config> to use it, or write `\#fxi` to keep it as a tag
```

These stay exactly as they are:

- `remembered the timeout #`: Pomodoro note, no menu;
- `Ask about #fix today`: mid-line `#` is prose;
- `@notes#Ideas`: heading bullet;
- `Need a parser @work+prep#requirements`: child under a task section;
- `@sase:deep-fix#bugs` and `^sase:deep-fix#bugs`: Pomodoro names, unconfigured names
  warn;
- `=bugs` / `=#bugs`, `@cash^goog-exit+#bugs`, `x#bugs`.

## Ground rules for every phase

- **Thin client holds** (`decisions:mac-capture-is-a-thin-client`).
  - bob-cli owns the grammar, completion data, previews, and writes. The Mac app renders
    what bob returns.
  - The wire shapes stay additive and `schema_version` stays 1. The meaning of
    `@route+id#NAME` and of `t:` does change, so this is a contract break (next bullet).
- **Release train** (`decisions:bob-cli-releases-ride-a-daily-train`).
  - Never edit `Cargo.toml`'s version or `CHANGELOG.md`.
  - The bob-cli commit is a contract break: subject
    `feat(capture)!: type tasks with #<type> and retire t:`, with a `BREAKING CHANGE:`
    footer that names the `@route+id#NAME` meaning change and the `t:` retirement.
- **Coherent master.** Bryan's hosts track master, so the `cli` phase lands as one
  coherent change. Grammar, writers, completion, help, and docs must never disagree at
  any pushed commit.
- **Task-type rules.**
  - `docs/task-types.md` stays authoritative: hue (PT5), pills and dashed marks (PT6),
    hide is Recent-only (PT9), field-stores-only-exceptions (PT13), and Agreement.
  - This epic adds exactly one exception, stated in PT12: **the capture `#<type>` marker
    refuses unconfigured names.**
- **Docs.**
  - Before editing `docs/` or CLI help, read the docs rules with
    `sase memory read docs.md -r "<why>"`.
  - Run `just docs-gen` when clap help changes, and `just docs-check` before landing.
  - Mac docs are owned by bob-mac-capture and only mounted into the Bob site
    (`docs/reference/mac-capture/` is gitignored here).
- **Shared vectors.** Do not edit `tests/fixtures/pomodoro_themes/vectors.json`; the
  type-marker fixtures live in `tests/fixtures/capture_type_marker/`.
- **Mac builds.** bob-mac-capture's `AGENTS.md` compile rules are mandatory. When the
  mac cannot be reached, state honestly "macOS targets unverified".

## 1. Language contract (normative)

### 1.1 Token

A **type marker** is a `#` followed immediately by a name.

- **Slug:** `#` then an ASCII letter, then ASCII letters, digits, and `-` (`#fix`,
  `#deep-work`).
  - A name that starts with a digit needs the literal (`#"2FA"`). This mirrors the
    `=#2fa` rule for named starts.
- **Literal:** `#"NAME"` or `#“NAME”`. It reuses `lex_exact_name_after_hash`
  (`capture_language/pomodoro_literal.rs`): the escapes are `\"` and `\\`, and the token
  runs through the closing quote.
- **Openers.** A free-standing candidate opens only on `#` + ASCII letter or `#` +
  quote. These are never type markers:
  - a bare `#` (the Pomodoro note when trailing, otherwise unchanged);
  - `#` + digit (`Review PR #1234`);
  - `##…`, `#-…`, `#` + non-ASCII.

  These stay prose (see 1.6 for what replaces the legacy check).

- **Claimed shapes at a marker position:**

  | Shape                                                                          | Outcome                                                                                                                                           |
  | ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------- |
  | `#fix`, `#"DEEP WORK"`                                                         | Complete                                                                                                                                          |
  | `#"DEEP` (unterminated), `#""`, and an empty attached `#` (`@r^id#`, `@r+id#`) | Incomplete: need `task_type`; execution refuses                                                                                                   |
  | `#bob+decks`                                                                   | Invalid: a task has one type                                                                                                                      |
  | `#fix.`, `#fix!`, `#a[b]`                                                      | Invalid, with a teaching error                                                                                                                    |
  | `\#fix`, `\#"Q&A"`                                                             | Escape: the body keeps `#fix` (an ordinary Obsidian tag) without the backslash. Mid-line `\#` stays verbatim. This mirrors `\$` and the old `\t:` |

- **Attached forms.** The token is the `#…` suffix glued to a route token. Its span
  starts at the `#`:
  - **`@route^block-id#<type>`.** The block part must not end in `+`, so `@r^id+#NAME`
    stays the project-note Pomodoro. A misordered `@r^id#NAME+` keeps today's
    `project_note_misordered_error`.
  - **`@route+block-id#<type>`**, only when the item, with every marker masked, has no
    body text and no authored child bullets. This is exactly where today's parser reads
    that `#` as the Pomodoro selector (the `task_toggle_pomodoro_name` span):
    - With body text or children, the suffix stays the task-section selector
      (`sub_bullet_section`), unchanged.
    - Decide this with one shared predicate, used by the lexer, the classifier
      (`sub_bullet_is_toggle` in `completion.rs`), and `editor_classify.rs`, so the
      three can never disagree.
  - **`@route+block-id#<type>!`** is refused. Update `EXPLICIT_TOGGLE_NAMED_ERROR` to
    `` explicit toggle `!` cannot follow a task type; write `@<route>+<block-id>! #<type>` ``.
  - The route token itself must sit at a marker position (1.2). Mid-line route tokens
    keep today's behavior.
- **Not changed.** `#` inside other tokens keeps its meaning:
  - `@route#Section` and `@route#`: heading bullets;
  - `@route:id#NAME`, `^route:id#NAME`, `=#NAME`, `=<X>#NAME`, `@route^id+#NAME`:
    Pomodoro names;
  - `x#bugs`: error;
  - URLs, `C#`, wikilinks, and code spans, which are already protected by
    `protected_ranges`.

### 1.2 Position, ownership, and masking

These are unchanged from `bob-cli-ee` §1.2–1.4, with `#` in place of `t:`:

- **Position.** A free-standing marker sits at either edge of any line. Only marker
  tokens may stand between it and the edge: the trailing-transparent set (which includes
  the bare `#`), `&` / `*` / `~` / `~~` modifiers, and other type markers.
  - A continuation line that holds only a marker (`#fix` or `- #fix`) types the item and
    contributes no bullet.
- **One marker per item, on any line.** The duplicate label becomes
  `type marker (#<type>)`. An attached and a free-standing marker in one item are a
  duplicate.
- **Masking invariant.** Every parse path sees the offset-preserving masked view:
  execution, the editor, completion field detection, templates, and rewrite. For an
  attached marker, only the suffix bytes are blanked, so `@sase^parser#fix` masks to
  `@sase^parser`.
  - An escaped marker masks only its backslash.
  - Update the property test to cover attached forms too.

### 1.3 Subjects, fields, and agreement

These are unchanged from `bob-cli-ee` §1.6–1.8:

- **Subjects.** `@r^id#X` is the "new task" row. A marker-only `@r+id#X` is the Ensure
  Next row: it is the existing `@r+id t:X` path, including D8 relocation and
  relationship-only owners.
- **Conflict copy** for 1.8 becomes:
  ``the type `#fix` disagrees with Pomodoro BUGS (`@sase:fix-x#bugs`): drop `#bugs` to link under FIX, or drop `#fix` to type the task BUGS``
- **Teaching copy.** Every `not_applicable` and `needs_task` message uses `#` spellings.
  - An attached `@r^id#fix` with no text says
    ``add the task text (`Fix the parser @sase^parser#fix`)``.
  - The sub-bullet remedy becomes
    ``capture `@<route>+<block-id>#<type>` alone to retype that task``.

### 1.4 Name resolution and the configured-type refusal

Resolution is unchanged:

- exact names only;
- configured vocabulary first, then the catalog;
- literals by key, slugs also with spaces for hyphens;
- prefixes never resolve.

**Acceptance (new).** One shared predicate decides acceptance, and execution and
completion both call it:

- **With a `types:` vocabulary:** accept only keys that are entries of the vocabulary
  (`Catalog::configured_themes()`).
  - Inventory labels are not types. They refuse even though their ThemeRef reports
    `configured: true`.
- **Without a vocabulary:** accept keys the history catalog already knows as a type
  (ThemeRef `known: true`, inventory labels excluded).
  - This matches PT12's "without a vocabulary, every component except inventory labels
    is a type", and keeps the menu's history rows usable.

**Refusal.** Anything else fails the whole capture before any write, with code
`type_not_configured` (exit 2, like other capture usage errors). The copy is:

- **With a vocabulary:**
  `` `#fxi` is not one of your task types; did you mean `#fix`? Add FXI under `types:` in <config> to use it, or write `\#fxi` to keep it as a tag ``
  - The did-you-mean clause uses the existing similarity rule (at most two names,
    ``did you mean `#fix` or `#fax`?``), and is omitted when nothing is similar.
  - `<config>` is the config path Bob already names in PT12 messages.
- **Without a vocabulary:**
  `` `#snack` is not a task type yet: no Pomodoro has used SNACK and no `types:` vocabulary is configured; add `types:` to <config>, or write `\#snack` to keep it as a tag ``
- **Context hints**, appended in parentheses, at most one:
  - On `@r^id#NAME`:
    ``for a project note whose ` :id` tasks go under the NAME Pomodoro, write `@r^id+#name` ``.
    The escape clause is dropped for attached forms.
  - On `@r+id#NAME`: ``to link the task under the NAME Pomodoro, write `^r:id#name` ``.
    The escape clause is dropped.
  - On a free-standing marker in an item that also carries a plain `@route` (the retired
    `#Section @route` bullet shape):
    ``for a bullet under a heading, write `@route#name` ``.

**Where it surfaces.** Refusal is an execution-time judgment: it needs the config and
the catalog. It surfaces in three places:

- `bob capture` and `bob capture --dry-run`. The Mac live preview shows the message as
  its preview error.
- The completion typed row (2.3).
- The ThemeRef flags on the `task_type` span. The Mac already draws the dashed mark and
  its caret-aware `isn't one of your types` caption from them.

`capture-parse` emits **no** `type_not_configured` diagnostic. Its diagnostics stay
lexical, and a diagnostic would flash in the editor footer on every partial name (`#f`,
`#fi`) while the menu is open.

**Pomodoro names are unaffected.** Every `#NAME` that names a Pomodoro keeps today's
create-and-warn behavior (`theme_not_configured`).

**PT12 change.** The `warn` path the `t:` marker used for unknown names is deleted.
PT12's "Warnings never block anything; there is no strict mode" gains this one stated
exception.

### 1.5 `@route+block-id#…` no longer names a Pomodoro

On a marker-only `@route+block-id` (Ensure Next, or a relationship-only owner), `#` now
always means the task type.

- **Named Ensure Next is gone.** The old meaning ("Ensure Next and move the Task Link
  under the named Pomodoro, retyping by the named-capture rule") is removed.
  - A container or non-type Pomodoro is reached with the whole-item link
    `^route:id#NAME`, which the refusal hint teaches.
- **Remove the dead path.**
  - `CaptureKind::TaskToggle { pomodoro_name, pomodoro_name_is_literal }` and the named
    branches in `src/native/capture/task_toggle.rs` and `capture/ensure_next.rs` become
    unreachable, so remove them.
  - Also remove the `#"NAME"` literal-to-toggle shortcut in `tokens.rs`
    `parse_sub_bullet_route_token`.
  - Remove the `task_toggle_pomodoro_name` span kind if nothing else emits it. Today
    only `@r+id#NAME` does; `^r:id#NAME` emits `pomodoro_name`.
- **What stays.**
  - The task-section form with body text is untouched. That includes its strict section
    charset, which still rejects literals.
  - `docs/task-types.md` "Named captures retype the task" drops Ensure Next from its
    Forms list.

### 1.6 Retiring `t:` and the legacy `#tag` check

**`t:` retirement** follows the `retire_t` decision.

> [!decision] retire_t = teach A marker-position `t:` token is refused, in every shape
> the old lexer claimed: complete, incomplete, invalid, or literal.
>
> - The code is `retired_type_marker`, in execution and as an editor diagnostic on the
>   token. The copy:
>   - `` `t:fix` is retired: write `#fix` ``;
>   - `` `t:"DEEP WORK"` is retired: write `#"DEEP WORK"` ``;
>   - `` `t:` is retired: type `#` and a task type (for example `#fix`) ``.
> - The token is masked so the rest of the item still parses.
> - It gets no `task_type` span, need, or completion.
> - `\t:fix` still writes `t:fix`, and mid-line `t:` stays prose.
> - Keep the old position scan only for this detection.

> [!decision] retire_t = prose `t:` is ordinary text everywhere. Delete the `t:`
> scanning, escape, copy, and tests: `Fix it t:fix` captures the body `Fix it t:fix`.

**The legacy `#tag` check is retired.**

- What goes: `reject_legacy_bullet_markers` (`markers.rs`, called from `line.rs`), the
  editor's `legacy_bullet_marker_diagnostic`, and its `legacy_bullet_marker` code.
- Why it goes:
  - The type marker now claims every letter- or quote-led edge `#` token. Unknown names
    refuse, with the heading hint (1.4).
  - The check would also wrongly reject escaped `\#tag` tokens after masking.
- Effect:
  - Remaining edge shapes become prose: digit-led `#1234`, `##…`, `#-…`.
  - `Review PR #1234` now captures instead of erroring.
  - A retired `#now` becomes an ordinary unknown-type refusal, unless `NOW` is
    configured.

### 1.7 Errors, diagnostics, and needs

| Code                                                                                                     | Where                             | Change                                                                                                                                                                                                                                                           |
| -------------------------------------------------------------------------------------------------------- | --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| need `task_type`                                                                                         | parse need + execution refusal    | `` `#` is incomplete: type a task type after `#` (for example `#fix`) `` (attached and literal forms only)                                                                                                                                                       |
| `invalid_type_marker`                                                                                    | both                              | `#` spellings: `` `#bob+decks` names two types; a task has one type: write `#bob` or `#decks` `` / `` `#fix.` is not a task type: names start with a letter and use letters, digits, and `-`; quote other names (`#"Q&A"`) or write `\#fix.` to keep the text `` |
| `duplicate_capture_marker`                                                                               | both                              | label `type marker (#<type>)`                                                                                                                                                                                                                                    |
| `type_marker_needs_task`, `type_marker_not_applicable`                                                   | both                              | `#` spellings (1.3)                                                                                                                                                                                                                                              |
| `type_marker_conflict`                                                                                   | execution (dry-run shows it live) | 1.3 copy                                                                                                                                                                                                                                                         |
| `type_not_configured`                                                                                    | execution (dry-run shows it live) | **new refusal** (1.4)                                                                                                                                                                                                                                            |
| `retired_type_marker`                                                                                    | both (`retire_t = teach`)         | **new** (1.6)                                                                                                                                                                                                                                                    |
| `legacy_bullet_marker`, `invalid_project_note_marker` for `@r^id#NAME`, `pomodoro_name` need on `@r+id#` | —                                 | **removed**; `@r+id#` now needs `task_type`                                                                                                                                                                                                                      |

## 2. Wire contract (shapes additive; `schema_version` stays 1)

### 2.1 `capture-parse`

- **`task_type` span.**
  - It covers the whole free-standing token (`#fix`, `#"DEEP WORK"`), or for attached
    forms only the `#…` suffix. It touches but never overlaps `task_block_id` or
    `task_toggle_block_id`.
  - It carries one ThemeRef on the name bytes (no `#`, no quotes).
  - Spans stay sorted and non-overlapping. Bob Mac Capture discards **all** highlighting
    when spans overlap (`CaptureTextRanges.validatedSpanRanges`), so add a test for
    every attached form.
- **Need** `task_type` for incomplete shapes (1.1), with mode `incomplete`.
- **`body`** excludes markers, so `capture-related` similarity matching is unchanged.
- **Help.** Update the `capture_parse.rs` help, including its incomplete-marker list
  (`'@route+id#'` now needs `task_type`), then run `just docs-gen`.

### 2.2 `capture`

These are unchanged: `task_theme` (`set` / `cleared` / `kept`), `type_relocation`, the
human `type` and `ledger` rows, and dry-run equals real-run apart from `dry_run`.

- The `theme_not_configured` warning no longer comes from markers (1.4).
- Ensure Next no longer reports a named `pomodoro_name` from `#`.

### 2.3 `capture-complete` context `task_type`

**Trigger.** The caret is inside, or at the end of:

- a free-standing marker at a marker position, once at least one name character or an
  opening quote is typed (`#f`, `#"`);
- an attached suffix, including the empty `#` (`@sase^parser#`, a marker-only
  `@sase+parser#`).

**Never.**

- A bare `#` that is not attached (`jot idea #`) returns **no** `task_type` context and
  no candidates. The Mac's Return always accepts row 0 of an open menu, so this keeps
  Pomodoro notes working.
- Mid-line `#`, other `#` contexts (`@r:id#` Pomodoro names, `@r#` sections, `@r+id#`
  with body text: task sections), and retired `t:` tokens get no `task_type` context.

> [!decision] bare_hash_menu = letter That is the whole rule; there is no Mac source
> change.

> [!decision] bare_hash_menu = open A bare `#` at a marker position also returns
> `task_type` with an empty query, plus a new additive top-level
> `initial_selection: "none"`.
>
> - **Mac:** decode it, keep `selectedCompletionIndex` unset so Return submits the draft
>   unchanged (the Pomodoro note or text), and let Ctrl-N/↓ or typing a letter select a
>   row. Also request completion when the caret sits right after a `pomodoro_note` span.
> - **Tests:** cover it in hosted tests. `jot idea #` + Return must still capture the
>   note.

**Fields.** These are unchanged: `replacement` covers the `#…` token or suffix, plus
`query`, `subject`, `theme_suggestion`, and every `TaskTypeCandidate` field.

- **`subject` must be `existing`** for a marker-only `@r+id#…` (Ensure Next / toggle /
  relationship-only owner).
  - Today `capture_complete/task_types.rs` recognizes only PomodoroLink/Incomplete
    whole-item links. Installed bob reports `subject: "new"` for `@sase+foobar t:fi`.
    Fix this and test it.

**Replacements.**

- `#<slug>` when the slug starts with a letter and round-trips to the key. Otherwise
  `#"NAME"`, so a digit-led `2FA` becomes `#"2FA"`.
- A free-standing leading marker still adds one space when the next byte is not
  whitespace.
- An attached suffix never adds a space.

**Refused typed row.** When the typed name fails the 1.4 predicate, the typed row
carries `refused: "type_not_configured"`, with `similar_to` as today.

- Bob Mac Capture already renders this: the row is dimmed, the footer reads
  `Add it under types: to use it`, and Return is a no-op.
- Configured, prefix, and substring rows are unchanged. `#fi` still lists FIX first.

**`@r+id#` context switch.** In `capture_language/completion.rs`, the no-body
`@route+id#` third component switches from `CompletionContext::PomodoroName` to
`TaskType` (the suffix lexer owns it). `@route^id#` gains `TaskType`. Update the
`capture_complete/cli.rs` "Contexts" help.

**Performance.** Stay within PT3. Re-measure `Fix the parser #f` on apollo and update
`docs/completion.md`'s performance row.

### 2.4 zsh

- `capture_complete/shell.rs` keeps the `task types` group and descriptions.
- `full` becomes `#fix`.
- `_describe` quotes it on insertion (`\#fix`), so `interactive_comments` cannot swallow
  the word. Pin the quoting with an adapter test, or an install-smoke assertion, in
  `tests/cli/completion/`.

## 3. Bob Mac Capture (phase `mac`)

The app is sigil-agnostic: an audit of `Sources/` found no `t:` literal and no sigil
length.

- Painting dims the whole `task_type` span, then tints only the ThemeRef range.
- The completion trigger keys only on bob's `task_type` span or need.
- The menu renders `row.name`, never the replacement.

With `bare_hash_menu = letter`, no Swift source change is expected. If one turns out to
be needed, keep it minimal and explain it in the commit.

What changes is the `t:`-shaped scaffolding:

- **`Scripts/generate-task-type-fixtures.sh`.** Re-record with the `cli` phase's bob.
  Point `BOB_CLI_DIR` at that checkout.
  - **Parse drafts:**
    - edges: `#f`, `#fix Fix it`, `Fix it #fix`;
    - child: `- check the lexer #fi`;
    - literals: `#"DEEP WORK"` and its unicode/curly variant;
    - refused: `#nope`;
    - prose: midline `Ask about #fix today`;
    - routed: `@dev #f`, `@dev:current #f`;
    - attached: `@dev^parser#` and `@dev^parser#fix`, `@dev+current#` and
      `@dev+current#fix`;
    - Pomodoro note: `jot idea #`;
    - wikilink coexistence: `Follow up ((foo #f`;
    - retired: `t:fix` (`retire_t = teach`).
  - **Complete drafts:** the same contexts, including the refused typed row (`#nope`),
    the attached empty-query lists, and an empty answer for `jot idea #`.
  - **Capture drafts:** set / kept / cleared / relocate via `#`.
- **`Tests/Fixtures/fake-bob`.**
  - Convert the `FAKE_BOB_TASK_TYPE_CHOOSER` parse answers and the `capture-complete`
    globs. The sigil length goes from 2 to 1: replace `end=2`, `${#draft} - 2`, and
    `cursor_after` arithmetic.
  - Convert the `@route+id#` Ensure Next answers (around lines 458–520) from
    Pomodoro-name completion to `task_type`.
- **Tests.**
  - `CaptureEditorHighlightTests.swift` (the five `t:fix` tests).
  - `CaptureTaskTypeHostedTests.swift`. Convert the `t:` `:` separator-rewrite test to
    `@dev^parser#fix`, or delete it with a reason.
  - `CaptureTaskTypeTests.swift`.
  - `CapturePanelModelTests.swift` `@route+id#` Pomodoro-name tests (~4431–4503) → task
    type.
  - Keep `testBareTrailingHashDoesNotOpenTaskSectionCompletion`, and add a real-bob
    fixture assertion that `jot idea #` has no completion and Return submits.
  - Wire the currently unconsumed child, second-line, leading, and attached fixtures
    into `CaptureTaskTypeTests`.
- **Docs.**
  - `docs/capture-panel.md` (~72–76, and the `#` meaning rows at ~151, 196, 203–214,
    254–259, 347–351).
  - `docs/task-pickers.md` (~26, 206–208, 315–323: `@route+id#` with no body is now the
    type menu).
  - `docs/protocol.md` (~43, 54–57, 346, and the old-bob fallback row at ~358).
  - `docs/keyboard.md` and `docs/capture-previews.md` wherever they show `t:`.

## Phase `cli` — bob-cli

1. **Lexer** (`src/native/capture_language/type_marker.rs`, exported through `mod.rs`).
   - Switch the sigil, openers, slug and literal shapes, escape, and query decoding to
     1.1.
   - Add the attached-suffix recognition and the shared marker-only `+id` predicate.
   - Update `invalid_marker_error`, `needs_task_error`,
     `type_marker_not_applicable_error`, and the module docs.
   - Add the `retire_t` handling.
2. **Token parsers and classifiers.**
   - Files: `tokens.rs`, `markers.rs`, `line.rs`, `editor_classify.rs`,
     `editor_parse.rs`, `item.rs`, `draft.rs`, `completion.rs`, `rewrite.rs`,
     `template.rs`.
   - Delete the legacy `#tag` check.
   - Stop the `@r^id#NAME` "needs the project-note `+`" error at marker positions.
   - Make the attached suffixes flow through masking.
   - Apply the 1.5 removals and the toggle error copy.
3. **Execution** (`src/native/capture/plan.rs`, `capture/task_theme.rs`).
   - Add the shared acceptance predicate and the `type_not_configured` refusal with its
     copy and hints (1.4).
   - Delete the marker warn path.
   - Update the conflict copy.
   - Refuse `retired_type_marker` under `teach`.
4. **`capture-parse`** (`src/native/capture_parse.rs`): spans, needs, and diagnostics
   per 2.1 and 1.7; help text.
5. **Completion** (`capture_language/completion.rs`, `capture_complete/task_types.rs`,
   `engine.rs`, `shell.rs`, `model.rs`): 2.3 and 2.4, including the `subject` fix, the
   replacement spelling, `refused`, and no context on a bare `#`.
   - Under `bare_hash_menu = open`, also `initial_selection`.
6. **Help.** Update `src/native/capture/cli.rs` (marker help, including the
   `@<route>+<block-id>#<pomodoro>` sentence near line 433), `capture_parse.rs`, and
   `capture_complete/cli.rs`, then run `just docs-gen`.
7. **Docs.**
   - **`docs/capture.md`:**
     - grammar-at-a-glance rows (`#fix`, `#"NAME"`, `#"`, `\#fix`, `t:` retired);
     - the "`#` is not one marker" table: replace the `@cash+id#coding` row, add the
       free-standing, `@r^id#` and `@r+id#` rows, and turn the `#now` / free-`#tag`
       error rows into refusal rows;
     - the `@route+id` `#` paragraph;
     - the `#now` section;
     - rename `### Typing tasks with t:` to `### Typing tasks with #` with the 1.x
       contract;
     - § Task-type changes; the retired forms;
     - the editor markers, `capture-parse` spans and needs, and the `capture-complete`
       context list.
   - **`docs/reference/capture-syntax.md`:**
     - the token-index row;
     - the Overloaded-markers `#` rows (free-standing, `@route^id#type`,
       `@route+id#type` vs `#section`);
     - the retired-forms rows;
     - the `{#plus-child-section}` and `{#plus-ensure-next}` entries;
     - replace `### t:<type> {#t-type}` with `### #<type> {#hash-type}`, and fix every
       inbound link.
   - **`docs/task-types.md`:** the PT12 exception, `### Typed captures (#)`, the Named
     captures Forms list, and the Routing table row.
   - **Elsewhere:** `docs/completion.md`, and the `docs/_meta/search_smoke.yml` query
     `t:<type>` → `#<type>`.
8. **Tests.**
   - **Convert every `t:` test** to `#`. Offsets shift by one byte.
     - `capture_language/tests/{type_marker,type_marker_editor,completion,rewrite}.rs`;
     - `capture/tests/type_marker.rs`;
     - `tests/cli/capture/{capture_type_marker,type_marker_existing,complete_task_types}.rs`;
     - `tests/cli/completion/capture_text.rs`;
     - `tests/fixtures/capture_type_marker/`.
   - **New coverage:**
     - attached forms: spans touching but not overlapping, incomplete empty `#`, literal
       `@r+id#"DEEP WORK"`, misordered `@r^id#x+`, `@r+id#x!`;
     - marker-only `@r+id#fix` (retype, Ensure Next placement, D8);
     - `@r+id#fix &n:id` relationship-only owner;
     - `@r+id#fix` with body text staying a section;
     - every refusal and hint (vocabulary, no vocabulary, inventory label, did-you-mean,
       attached hints, routed heading hint);
     - `\#tag` escape;
     - digit-led and `##` prose (`Review PR #1234`);
     - bare `#` Pomodoro note, with no completion and `Fix it #fix #` not applicable;
     - retired `t:`;
     - Pomodoro `#NAME` forms still warning, not refusing;
     - `capture-complete` `subject` for `@r+id#`;
     - the masking property test with attached forms;
     - `capture-rewrite` leaving `#fix` alone;
     - dry-run equals real-run.
   - **Existing suites stay green:** `tests/cli/task_move.rs` and the Pomodoro-name
     completion tests for `@r:id#` / `=#`.
9. **Verify.** Run `just check`, `just docs-check`, and `just all` before pushing. Check
   that `git grep -n 't:'` over `src/native/capture*`, `docs/`, and `tests/` finds only
   intentional retirement copy, escapes, and unrelated `t:` text such as the `[t:: 30m]`
   timespans.

> [!decision] memory_record Use `/sase_memory_write` to add
> `sase/memory/decisions/hash-type-marker-requires-configured-types.md`.
>
> - **Claim:** capture types tasks with `#<type>`, which replaced `t:`. Unconfigured
>   names refuse, unlike Pomodoro `#NAME`s, which warn.
> - **Rejected alternatives:** keeping `t:`; warn-and-write.
> - **Evidence:** this epic's approved plan and `docs/capture.md`.
> - Then run `sase memory init`.

## Phase `mac` — Bob Mac Capture

Implement section 3 against the landed `cli` contract. Verify with:

- `just portable-check`;
- `just docs-check`, with `BOB_CLI_DIR` set to a bob-cli checkout at the `cli` commit;
- the scratch mac compile (`bundle.sh` plus `just typecheck-tests`), per `AGENTS.md`;
- CI.

Run a manual smoke on the mac with the installed `bob`, and record screenshots of the
menu and the refused row for the final report:

- `Fix the parser #f` → FIX → capture;
- `#fix` at the start of a line;
- `- #fix` on a child line;
- `@<route>^<id>#` through the New-ID picker → full menu;
- a marker-only `@<route>+<id>#fix`;
- `#nope` → refused row and preview error;
- `jot idea #` + Return → Pomodoro note;
- Ctrl+D hide, then show.

## Non-goals

- No change to Pomodoro-name `#NAME` resolution, its warnings, or its completion.
- No draft-wide `@@#type`; no `#type` on project-note child bullets (they still refuse
  with the `task_type:` pointer).
- The Obsidian plugins are unaffected. They build no capture drafts and never call
  `capture-parse` or `capture-complete`; this was checked in bob-plugins.
