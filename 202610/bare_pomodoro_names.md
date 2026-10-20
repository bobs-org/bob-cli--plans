---
tier: epic
title: Bare Pomodoro names after `=` (`=bugs`, `@route:id=bugs`)
goal: "`=bugs` starts the BUGS Pomodoro exactly like `=#bugs`, and `@route:id=bugs`
  links and starts exactly like `@route:id#bugs=`, across execution, capture-parse,
  live-editor spans, completion, and docs; every existing spelling keeps its current
  meaning.

  "
decisions:
  link_suffix:
    ask: How much of the standalone start grammar should a link's `=` suffix accept?
    choices:
      shared:
        Same start token as alone (`=bugs@3`, `=3#bugs`, `=#bugs` all work on links)
      bare_only:
        Only `=bugs`; name plus timing on a link stays `#bugs=3` (others get a hint)
    default: shared
    why: One lexer for both positions, and completion can always fall back to `=#name`
    answer: shared
phases:
  - id: standalone
    title: Bare names on whole-item and chained starts
    depends_on: []
    size: medium
    description:
      "standalone: lex `=name` as the `=#name` named start (shared bare-name predicate,
      execution, editor spans, completion, spelling-aware diagnostics, terminal hints)
      with equivalence tests against the `#` spelling."
  - id: links
    title: Bare names in `@route:id=` and `^route:id=` suffixes
    depends_on:
      - standalone
    size: medium
    description:
      "links: accept `=name` (and, under the shared answer, the full start token) in
      link suffixes for execution, editor spans, and completion, with link equivalence
      tests."
  - id: docs
    title: Contract, reference, and help text
    depends_on:
      - standalone
      - links
    size: medium
    description:
      "docs: teach `=name` as the primary spelling in the capture contract, syntax
      reference, and CLI help; regenerate references; pass the docs gate; propose the
      Mac Capture hint follow-up."
proposed_by: bbugyi200.apollo.9p
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-20 15:12:09
status: wip
---

- **PROMPT:**
  [prompts/202610/bare_pomodoro_names.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/bare_pomodoro_names.md)

# Plan: Bare Pomodoro names after `=`

## Why and the one rule

Today a named start needs a `#` even when nothing sits between `=` and the name
(`=#bugs`), and a link names its Pomodoro before the `=` (`@sase:deep-fix#bugs=`). The
`#` adds nothing when no timing comes first. The whole feature follows one rule:

> **After a lone `=`, a word is the Pomodoro.** Digits are timing (`=3`, `=-2`), `x` is
> a close (`=x`, `=x2!1`), and a word that starts with a letter names the Pomodoro to
> start (`=bugs`). Write `#` only when a number comes first (`=3#bugs`), or when the
> name itself starts with a digit or reads as a close (`=#2fa`, `=#x`, `=#x2`).

The short spelling is a pure alias. `=bugs` and `=#bugs` lex to the same token. They
produce the same `CaptureKind`, the same vault writes, and the same `capture-parse`
semantics. Only `input`/`body` echoes and span offsets differ. Every command spelling
that works today keeps its meaning. The inputs that change are listed in the tables
below, and there are only two kinds. The first is an item that _begins_ with a bare
`=word`, which today becomes a junk inbox task (`=bugs`, `=xx`, `=bugs more`). It
becomes a named start or a teaching error. The second is a link suffix that is a refused
error today (`@work:outline=bugs`, `@work:outline=3#bugs`). A mid-body `=word` stays
prose.

Bob teaches the short spelling from now on: docs, help, terminal hints, and the
suggestions Bob composes itself. The `#` spelling stays documented as the always-valid
form. Diagnostics that rewrite a token the user typed keep that token's spelling
(`=deep work` → `=deep-work`; `=#deep work` → `=#deep-work`). Diagnostics that compose a
new start (`x#bugs` → `x =bugs`) use the canonical short spelling.

### Bare-name eligibility (one shared predicate)

Add one small shared module of helpers in `src/native/capture_language/`. Every position
(lexer, link suffix, completion, hints) must call these helpers, never re-derive the
rule:

- `bare_start_name_len(after_eq) -> Option<usize>`: `Some` when the text right after a
  single `=` begins a bare name. That means its first byte is an ASCII letter,
  `close_after_eq_parts(after_eq)` (in `close_selection.rs`, the existing "text after
  `=` is close-shaped" classifier) does not claim it, and it does not start with
  `x#`/`X#`. Return the name's extent. A bare name ends exactly where a `#name` part
  ends today: at whitespace, `~`, or `@` (and, inside a link suffix, at the end of the
  token).
- `can_spell_bare(name) -> bool`: `name` is a valid `is_pomodoro_selector_component`,
  starts with an ASCII letter, and `close_after_eq_parts(name)` is `None`.
- `start_name_spelling(name) -> String` (`name` or `#name`, the text after `=`) and
  `named_start_token(name) -> String` (`=name` or `=#name`). Use these for every place
  Bob composes a start for the user.

Pomodoro names start with a letter or digit (see `bob capture-pomodoro-name`), so only
digit-led names and the reserved `x` / `x<digit>…` / `x,…` shapes ever need `#`.

## Phase `standalone`: bare names on whole-item and chained starts

### Behavior

| Typed                                                   | Same as                      | Result                                                                                                         |
| ------------------------------------------------------- | ---------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `=bugs`                                                 | `=#bugs`                     | Start BUGS (open match by whole slug, else prefix; completed match starts "again"; otherwise create and start) |
| `=bugs@3`                                               | `=#bugs@3`, `=3#bugs`        | Start BUGS for 15 minutes                                                                                      |
| `=bugs~2`, `=bugs@3~2`, `=bugs~2@3`                     | `=#bugs~2`, `=3#bugs~2`      | Drop list and timing, either order                                                                             |
| `x =bugs`                                               | `x =#bugs`                   | Close, then start BUGS (same-line chain)                                                                       |
| `=bugs +2`                                              | `=#bugs +2`                  | Chain left to right                                                                                            |
| `x wired the lexer =bugs`                               | `x wired the lexer =#bugs`   | Close with the inline Work Log entry, then start BUGS                                                          |
| `=xx`, `=xa`, `=xmas`, `=Xmas`, `=Bugs`                 | `=#xx`, …                    | Named start (`=xx`/`=xa` were documented prose; that row changes)                                              |
| `=bugs more`, `=deep work`                              | `=#bugs more`, `=#deep work` | Claimed: the existing named-start shape error with the multi-word hint, in the typed spelling (`=deep-work`)   |
| `=bugs=3`                                               | `=#bugs=3`                   | Existing order error, typed spelling: `=bugs@3` or `=3#bugs`                                                   |
| `=bugs@`                                                | `=#bugs@`                    | Incomplete `pomodoro_start_timing` (unchanged semantics)                                                       |
| `=3bugs`, `=-2bugs`                                     | —                            | **New** teaching error: the timing runs into a bare name; write `=bugs@3` or `=3#bugs`                         |
| `=2fa`                                                  | —                            | Same new error; also says a name that starts with a digit takes `#` (`=#2fa`)                                  |
| `=x`, `=X`, `=x2!1`, `=*`, `=!`, `=x wired the lexer`   | —                            | Unchanged retired-close teaching errors                                                                        |
| `=x#bugs`                                               | —                            | Unchanged retired error, now teaching `x =bugs`                                                                |
| `x#bugs`                                                | —                            | Unchanged error, now teaching `x =bugs` and "`=bugs` swaps it in"                                              |
| `=#x`, `=#x2`, `=#2fa`                                  | —                            | The `#` spelling for names that cannot be bare (unchanged)                                                     |
| `=`, `=3`, `=-2`, `=~2`, `=#`, `=#bugs`                 | —                            | Unchanged                                                                                                      |
| `= bugs`, `Plan =bugs`, `==bugs`, `==important==`, `=>` | —                            | Unchanged prose (doubled `=` keeps protecting Obsidian highlights)                                             |

To capture a task that must begin with `=word`, put any word before it; a mid-body
`=word` stays prose. A Work Log entry that must end in `=word` goes on a child bullet,
because bullet text is literal. Both escapes already exist for `=3`.

### Implementation

1. **Lexer** (`item.rs::session_equals_token`). Today any `=x…`/`=X…` returns
   `EqualsToken::Close`. Narrow that to the tokens the close family really claims:
   `retired_whole_item_close_rewrite(first_token).is_some()`, which covers `=x`, `=X`,
   selection-shaped `=x…`, `=x#…`, `=*…`, and `=!…`. Everything else falls through to
   start lexing. After the `<X>` digits, when `<X>` is empty, the token is not doubled,
   and `bare_start_name_len` matches, read a bare name exactly like the `#name` part.
   The existing `~<K>`/`@<X>` part lexing must then apply unchanged. Record the spelling
   on `EqualsToken::Start`, for example `name_spelling: NameSpelling { Hash, Bare }` or
   a token-relative `name_offset`, so no caller recomputes `1 + suffix.len() + 1` by
   hand. Audit every `EqualsToken::Start` destructure (`item.rs`, `editor_pomodoro.rs`,
   `completion.rs`, `close_log.rs`, `draft.rs`, `start_selection.rs`).
   `is_session_chain_token` already treats `name.is_some()` as a chain token, so chains
   and inline Work Log trails follow.
2. **Execution** (`item.rs::parse_pomodoro_equals_item` and the helpers
   `named_token_error`, `named_shape_error_with_drop`, `named_timing_twice_error`,
   `spaceless_start_drop_is_valid`). Thread the spelling into every message that echoes
   or suggests the token. These helpers hard-code `"{sigil}#{name}"` today. A bare token
   can never have a leading `<X>`, so the "timing twice" path stays hash-only.
   `named_nospace_error` (`=# bugs`) stays hash-only. Add the glued-timing teaching
   error (`=3bugs`, `=2fa`) on the counted unnamed path. It fires when the byte right
   after a non-empty `<X>` is an ASCII letter, and it suggests `named_start_token`-style
   rewrites. Put the message helper in `markers.rs` next to
   `pomodoro_start_shape_error`.
3. **Composed suggestions**. `close_selection.rs::retired_whole_item_close_rewrite`
   (`x =#{name}` → `x ={start_name_spelling(name)}`; keep `x =#` when the name is
   empty), `markers.rs` `pomodoro_close_hash_error` (`x#bugs`), and `start_selection.rs`
   (`=~2#bugs` keeps the typed `#`). The new-session message in
   `capture_pomodoro_start.rs` (`` `=#plan~1` names task 1 … start it with `=#plan` ``)
   should echo the typed spelling (`=plan~1` → `=plan`). The no-placeholder error in
   `capture/pomodoro_start.rs` ("start a new named session with `=#<name>`") becomes
   `=<name>`.
4. **Terminal agenda hint** (`capture_pomodoros.rs`, the `AgendaRole::Later` trail
   `=#{slug}`) → `named_start_token(&entry.slug)`. Update its tests.
5. **Editor** (`editor_pomodoro.rs::parse_editor_named_start_item`,
   `named_start_token_diagnostic` / `NamedStartOffsets`). Derive `name_start` from the
   recorded spelling. A bare token emits `pomodoro_start [0,1)` and
   `pomodoro_name [1,1+len)`, then the existing `@<X>`/`~<K>` spans. Modes, needs,
   specs, and diagnostic codes are identical to the `#` spelling. A bare token never
   emits the `interactive_placeholder` that the empty `=#` uses. Lone `=` stays a
   complete start.
6. **Completion** (`completion.rs::named_start_name_at`, `pomodoro_start_name_field`).
   Locate the name through the lexer's recorded offset. `=bu|`, `x =bu|`, and
   `x wired it =bu|` must report `pomodoro_start_name` with query `bu` and replacement
   over the name only, just like `=#bu`. Mark the field as bare-spelled (for example a
   `bare_name: bool` on `CompletionField`). The `capture_complete` engine then gives any
   candidate whose slug fails `can_spell_bare` the replacement `#<slug>`, so accepting
   it yields a valid `=#2fa`. This matters because ranking includes substring matches.
   `pomodoro_start_override_at` keeps its semantics: an empty `<X>` and no `@` part
   keeps the running ledger.
7. **Tests**.
   - An equivalence table test in `capture_language/tests/grammar.rs` or
     `capture_parse.rs`. For each pair (`=bugs`/`=#bugs`, `=bugs@3~2`/`=#bugs@3~2`,
     `x =bugs`/`x =#bugs`, `=bugs +2`/`=#bugs +2`, `x wired it =bugs`/
     `x wired it =#bugs`), assert identical `mode`, `route`, `section`, `block_id`,
     `needs`, diagnostic codes, `pomodoro_start`/`pomodoro_close`, and per-item fields
     other than `body`/`range`.
   - Span tests in `editor_spans.rs` and mode tests in `editor_modes.rs`.
   - Diagnostic tests for every row of the behavior table.
   - Completion tests in `capture_complete/tests/output.rs` (bare query, chain, inline
     trail, `#` fallback for `2fa`/`x2` slugs, override object).
   - A CLI test in `tests/cli/capture/pomodoro_start_named.rs` that runs `=bugs` and
     `=#bugs` against twin fixture vaults and asserts byte-identical daily notes and
     equivalent JSON. Do the same for `x =bugs` in `pomodoro_chain.rs`.
   - Update the existing tests that assert `=xx`/`=xa` are prose, or that `x =foo` is an
     error.
   - Run `just check`.

## Phase `links`: bare names in `@route:id=` and `^route:id=` suffixes

### Behavior

| Typed                                              | Same as                 | Result                                                                                  |
| -------------------------------------------------- | ----------------------- | --------------------------------------------------------------------------------------- |
| `@work:outline=bugs`                               | `@work:outline#bugs=`   | Link the existing `^outline` task under BUGS and start it                               |
| `^work:outline=bugs`                               | `^work:outline#bugs=`   | Active-task spelling, identical execution                                               |
| `Write outline @work:outline=bugs`                 | `… @work:outline#bugs=` | New Next task linked under BUGS, then start                                             |
| `@work:outline=xmas`                               | `@work:outline#xmas=`   | Bare names starting with `x` that are not close-shaped                                  |
| `@work:outline=3bugs`                              | —                       | The glued-timing teaching error from phase `standalone`                                 |
| `@work:outline#bugs=focus`                         | —                       | **New** error: name the Pomodoro once (`@work:outline=focus` or `@work:outline#focus=`) |
| `@work:outline=x`, `=x2!1`, `=x!`                  | —                       | Unchanged link-then-close                                                               |
| `@work:outline=`, `=3`, `=-2`, `#bugs=`, `#bugs=3` | —                       | Unchanged                                                                               |

> [!decision] link_suffix = shared The text after a link's `=` is lexed by the same
> start-token lexer as a whole-item start, so a link takes every spelling a lone start
> takes, except drop lists: `@work:outline=bugs@3`, `@work:outline=3#bugs`,
> `@work:outline=#bugs`, and `@work:outline=#bugs@3` all mean `@work:outline#bugs=<X>`.
> A `~<K>` part in a link suffix (`@work:outline=~2`, `=bugs~2`) gets a teaching error:
> a link start takes a timing and a name, not a drop list. Today it is a generic shape
> error. Naming the Pomodoro both before and after `=` is the name-once error. A timing
> in both places is the existing timing-twice error. Completion's `#<slug>` fallback
> (phase `standalone`, step 6) applies to link fields too, because `@work:outline=#2fa`
> is now valid.

> [!decision] link_suffix = bare_only A link's `=` suffix gains only the bare `=name`
> form, with no timing. `@work:outline=bugs@3`, `@work:outline=3#bugs`, and
> `@work:outline=#bugs` get one teaching error that names the link spelling:
> `@work:outline#bugs=3` (or `@work:outline=bugs` when no timing was typed). Link
> completion fields in a bare suffix omit candidates whose slug fails `can_spell_bare`;
> those Pomodoros stay reachable through `#name=`.

### Implementation

1. **Execution** (`tokens.rs::parse_colon_link_tail`, shared by `@route:`, legacy
   `@!route:`, and `^route:`). Keep the close branch first. In the start branch,
   classify the suffix with the phase-`standalone` helpers. Under `shared`, call the
   start-token lexer on `=` + suffix, then reject drop parts. Under `bare_only`, use
   `bare_start_name_len` plus `parse_pomodoro_start_suffix`. Merge a suffix name with a
   prefix `#name` (name-once error) and keep `#name=x` closes unchanged. Have
   `ColonLinkParts` return token-relative offsets for the name and every start part, so
   callers stop re-splitting on `#`/`=`.
2. **Caret editor shape** (`tokens.rs::classify_caret_token`, `CaretTokenShape`).
   `name_range` and `start_offset` must come from those offsets. `editor_parse.rs`'s
   caret span emission currently paints `pomodoro_start` from `start_offset` to the
   token end. It must stop before a suffix name and paint any `@<X>` part separately.
   Spans for `^work:outline=bugs`: `active_task_route [0,5)`,
   `active_task_block_id [6,13)`, `pomodoro_start [13,14)`, `pomodoro_name [14,18)`.
3. **Route editor classifier** (`editor_classify.rs::classify_pomodoro_token`). Same
   offsets. Spans for `@work:outline=bugs`: `pomodoro_route [0,5)`,
   `pomodoro_block_id [6,13)`, `pomodoro_start [13,14)`, `pomodoro_name [14,18)`.
   `marker_parse`'s `MarkerThird` assumes the name comes before the suffix, so emit the
   suffix-name span yourself. Keep spans disjoint and sorted, and never let
   `merge_spans` drop one. Mac Capture requests completion whenever the cursor sits in a
   `pomodoro_name` span, so these spans are what make Mac completion work with no Swift
   change.
4. **Completion** (`completion.rs`, the `is_pomodoro_marker_candidate` branch, which
   strips the `=` suffix before splitting `#` today, and `caret_link_field`). A cursor
   inside a suffix name yields `pomodoro_name` with the route and block ID, a query from
   the name start to the cursor, and a replacement over the name only. Apply the
   bare-field candidate rule from the active `link_suffix` branch. A cursor in the route
   or block part still completes those parts, and its replacement still ends before `=`.
5. **Tests**.
   - A link equivalence table (`@work:outline=bugs`/`@work:outline#bugs=`, the caret
     pair, the body-text pair, plus the active branch's extra spellings).
   - Span tests for both sigils.
   - Diagnostic tests for every row above.
   - Completion tests for `@work:outline=bu|` and `^work:outline=bu|`.
   - CLI tests in `tests/cli/capture/pomodoro_link.rs` and `task_link.rs` that execute
     both spellings on twin fixtures and assert byte-identical writes.
   - Run `just check`.

## Phase `docs`: contract, reference, and help text

Follow the docs reference memory: behavior changes go to the contract and reference;
guide steps change only where they became incorrect. The tutorial's `=#focus` stays
correct, so leave it.

- `docs/capture.md`.
  - In "Grammar at a glance", lead the named-start row with `=name` (keep `=<X>#name`
    and the `@<X>` form), add `@route:block-id=name` beside `#pomodoro=<X>`, and add the
    rows from both behavior tables to the "You typed" table.
  - Change the `=xx`/`=xa` prose rows and the retired-close paragraph.
  - In "Starting a named Pomodoro", "Starting the session atomically", "Linking and
    starting existing tasks", and "Chaining session operators on one line", use `=bugs`,
    `x =bugs`, and `@sase:deep-fix=bugs` in examples.
  - Keep one short spelling note: "`=bugs` and `=#bugs` are one token; write `#` only
    when a number comes first (`=3#bugs`) or the name starts with a digit or reads as a
    close (`=#2fa`, `=#x2`)."
  - Update the `capture-parse` and `capture-complete` sections that describe
    `pomodoro_name` / `pomodoro_start_name` fields.
- `docs/reference/capture-syntax.md`.
  - Add token-index rows for `=name` and `@route:block-id=name`, and a heading or
    equivalent-spelling line for each.
  - Update the overloaded-marker `#` row, the incomplete-input paragraph, the retired
    `x#bugs` row, and the `{#equals-named}` / `{#at-route-colon-start}` examples
    (`bob capture '=focus'`).
  - Keep the mnemonic short: "**`=` then a word starts that Pomodoro; `#` only after a
    number.**"
  - Add the zsh note: zsh's EQUALS expansion rewrites a word that starts with `=`, so
    `bob capture =focus` fails unquoted. Always single-quote (`'=focus'`, `'x =focus'`).
- CLI help: `src/native/capture/cli.rs` (the long help and the examples list: add
  `bob capture '=bugs'`, `bob capture 'x =bugs'`, `bob capture '@sase:deep-fix=bugs'`)
  and `src/native/capture_parse.rs` help. Then run `just docs-gen` to regenerate
  `docs/reference/cli/*.md` and the goldens, and update `tests/cli/help.rs` if it pins
  text.
- Run `just docs-check` and `just check`.
- Mac Capture follow-up. Bob Mac Capture is a thin client, so its agenda hint
  (`CaptureAgendaPresentation.swift` builds `=#\(slug)`, and `headerTimeView` styles
  rows by `hasPrefix("=#")`) and its empty-state copy ("`=#NAME` starts one") can only
  adopt the short spelling from a Bob-provided token. Swift cannot be built on Linux
  hosts. Record a `PROPOSED FOLLOW-UP:` on this phase's bead: Bob emits a `start_token`
  (from `named_start_token`) on agenda entries, and the Mac renders it, falling back to
  `=#slug` on older Bob. Everything else in this epic works in the Mac unchanged: it
  renders Bob's spans, candidates, and diagnostics.

## Contract notes

- No `capture-parse` / `capture-complete` schema change: same modes, needs, span kinds,
  and contexts. Only new offsets and new diagnostics.
- The decision record `mac-capture-is-a-thin-client` holds: all grammar stays in Bob.
- Memory notes and decision records that cite `=#NAME~K` stay true, so no memory edits
  are planned.
