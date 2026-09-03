---
tier: tale
title: Allow `+` in Pomodoro names
goal:
  A Pomodoro can be named with a `+` in it (for example `C++` or `BOB+SASE`) end to end
  — `bob capture-pomodoro-name` accepts it, the ledger scanner reports it as
  `selectable`, `@route:id#c++` targets and creates it, and completion offers it — while
  task-section titles and sub-bullet section selectors keep their current character set.
size: medium
proposed_by: bbugyi200.kellys_mbp.8
---

- **AGENTS:**
  - [bbugyi200.kellys_mbp.8](https://github.com/bobs-org/bob-cli--agents/blob/main/families/bbugyi200.kellys_mbp.8.md)
- **COMMITS:**
  - [338c90a](https://github.com/bobs-org/bob-cli/commit/338c90a3a4bbf0d9ac92003d0ac0bbd331932816)
    — feat(capture): allow + in Pomodoro names

# Plan: Allow `+` in Pomodoro names

## Goal

Add `+` to the Pomodoro-name character set so a name like `C++`, `BOB+SASE`, or `A+B` is
authorable, scannable, typeable, and completable everywhere Pomodoro names flow. Do
**not** change the task-section title grammar or the `@route+id#section` sub-bullet
selector grammar.

## Background: where the Pomodoro-name grammar actually lives

Today the Pomodoro name is validated by **two separate predicates**, each of which is
_shared with the task-section family_. That sharing is the whole difficulty of this
change.

1. **The written/canonical name** — `capture_pomodoros::canonicalize_pomodoro_name`
   normalizes whitespace, ASCII-uppercases, then defers to
   `capture_task_sections::is_section_title`. That function requires the first character
   to be an ASCII uppercase letter or digit, requires at least one uppercase letter
   somewhere, and allows only `A-Z 0-9` plus `` ` ` \t & ' ( ) , . / - `` in the rest
   (`is_section_title_first` / `is_section_title_rest`). It is also the predicate that
   decides whether a plain child bullet under a task _is a task section_
   (`parse_task_section_title`), so widening it in place would silently turn bullets
   like `A+B` into task sections.

2. **The typed selector component** — `capture_language::is_selector_component` (backed
   by `is_selector_byte`) is documented in its own doc comment as "the shared
   third-component grammar for `@route+id#section` and `@route:id#pomodoro`": ASCII
   alphanumerics plus `& ' ( ) , . / -`, with no first-character or
   must-contain-a-letter rule. Widening it in place would also widen sub-bullet
   task-section selectors.

Both must accept `+` for the feature to work, and both must be split so the task-section
side is unaffected.

The third moving part is `is_selector_component`'s use inside the ledger scanner:
`capture_pomodoros::scan` computes
`selectable = name.is_some() && is_selector_component(&slug)`. This is exactly why a `+`
name is broken today — the Obsidian `task-status-cycler` plugin's `N<ctrl+shift+m>`
"move into a new named Pomodoro" prompt applies **no** character grammar at all
(`POMODORO_NAME_TAIL_RE` is `/^[ \t]*—[ \t]*(.*)$/`), so a user can already write
`- [ ] () — C++` into a daily note, and bob-cli then reports it as `selectable: false` /
"nameable", i.e. an untypeable name that the picker offers to _repair_ instead of
target.

`capture_language::selector_slug` only lowercases and joins on `-`, so it passes `+`
through unchanged: `C++` → `c++`, `BOB+SASE` → `bob+sase`. No slug change is needed.

## Grammar-safety analysis (why this does not collide with `@route+id`)

`+` is already load-bearing in the capture marker language as the sub-bullet separator
(`@route+block-id`), so the collision risk must be checked before writing code. It is
safe, because family selection is decided by _separator order_, not by mere presence of
a character:

- `is_sub_bullet_marker_candidate` requires the `+` to appear **before** any of `:`,
  `#`, `^`. In `@sase:deep-fix#c++` the `:` and `#` both precede the `+`, so the token
  is not a sub-bullet candidate.
- `is_pomodoro_marker_candidate` requires the `:` to precede `#`, `^`, **and** `+`,
  which holds for the same token.
- `marker_field_at_cursor` (completion) and `classify_editor_token` (parse/spans) both
  test `is_sub_bullet_marker_candidate` first and then the Pomodoro candidates, so both
  agree with the above ordering.
- `@:#a+b` (the incomplete `@:` picker state) also stays out of the sub-bullet branch,
  because its `:` is at index 0, before the `+`.
- `@!`-prefixed legacy Pomodoro markers are explicitly excluded from the sub-bullet
  candidate check.
- Tokens are split on whitespace only (`tokenize_with_spans`), so `+` never splits a
  marker token.

The implementer should still add regression tests that pin this ordering rather than
trusting the reading above.

## Design decisions

- **Widen the Pomodoro name only.** Task-section titles (`parse_task_section_title`) and
  sub-bullet section selectors (`@route+id#section`) keep the exact character set they
  have today. The two grammars stop being literally identical, and the docs and doc
  comments that claim they are identical must be corrected rather than left stale.
- **`+` is allowed in the "rest" position only, not as the first character.** Keep
  `is_section_title_first`'s rule for Pomodoro names: a name must still start with an
  ASCII uppercase letter or digit and must still contain at least one letter. `+` alone,
  `+A`, and `++` stay invalid. This keeps the rule easy to state in help text and avoids
  a leading `+` that reads like a Markdown list marker.
- **No JSON schema bump.** No field is added, removed, or retyped in
  `capture-pomodoros`, `capture-pomodoro-name`, `capture-complete`, or `capture-parse`.
  `schema_version` stays `1` everywhere. The only observable change is that some entries
  that used to report `selectable: false` / `requires_name: true` now report
  `selectable: true` with a usable slug.

## Implementation

### 1. Split the title grammar (`src/native/capture_task_sections.rs`)

Keep `is_section_title` behaviourally identical and add a Pomodoro-name sibling beside
it so the two character sets stay adjacent and cannot drift:

- Extract the existing body of `is_section_title` into a private helper that takes an
  "allow `+`" switch (or an extra-allowed-characters parameter).
- `pub(crate) fn is_section_title(title: &str) -> bool` — unchanged behaviour, no `+`.
- `pub(crate) fn is_pomodoro_name(name: &str) -> bool` — same rules plus `+` in the
  non-leading position.
- Document on both that the Pomodoro name is the section-title grammar plus `+`, and
  that `+` is deliberately excluded from section titles because
  `parse_task_section_title` uses that predicate to decide whether an ordinary child
  bullet is a section.

### 2. Point Pomodoro canonicalization at the new predicate (`src/native/capture_pomodoros.rs`)

- `canonicalize_pomodoro_name` calls `capture_task_sections::is_pomodoro_name` instead
  of `is_section_title`.
- Update `POMODORO_NAME_USAGE` to include `+` in its enumerated character set.

### 3. Split the selector-component grammar (`src/native/capture_language.rs`)

- Keep `is_selector_component` / `is_selector_byte` as the **task-section**
  third-component grammar and fix its doc comment, which currently claims it is shared
  with `@route:id#pomodoro`.
- Add `pub(crate) fn is_pomodoro_selector_component(value: &str) -> bool`: non-empty and
  every byte is either a selector byte or `+`.
- Update the two Pomodoro-name call sites to the new predicate:
  - `parse_pomodoro_route_token`'s `Some(name) if !is_selector_component(name)` guard.
  - `classify_pomodoro_token`'s `invalid_pomodoro_name` diagnostic guard. Leave the two
    sub-bullet section call sites (`parse_sub_bullet_route_token` and
    `classify_sub_bullet_token`) on `is_selector_component`.
- Update `POMODORO_NAME_ERROR` to list `+`. Leave `SUB_BULLET_SECTION_ERROR` alone.

### 4. Make existing `+` names selectable (`src/native/capture_pomodoros.rs`)

In `scan`, change the `selectable` computation from
`capture_language::is_selector_component(&slug)` to the new
`capture_language::is_pomodoro_selector_component(&slug)`. This is the line that flips a
vault's existing `— C++` entries from untypeable/nameable to targetable.

Note the intended, correct consequence: because `capture-pomodoro-name` refuses an entry
that is already `selectable`, a `+` name that used to be "repairable" now behaves like
any other named Pomodoro and is targeted with `#c++` instead of renamed. Completion
likewise moves those entries from the nameable tier into the named tier.

### 5. Help text and docs

- `src/native/capture_pomodoro_name.rs`: `name_arg()`'s `.help(...)` string must list
  `+`.
- `docs/capture.md`:
  - The `#<pomodoro>` selector paragraph (currently "using the same character set as a
    task-section selector (`A-Z`, `a-z`, `0-9`, and `& ' ( ) , . / -`)") must state the
    Pomodoro set including `+` and stop claiming the two sets are identical.
  - The `capture-pomodoros` paragraph that says "Named Pomodoros use the same selector
    slug grammar as task sections" needs the same correction.
  - The `bob capture-pomodoro-name` section's grammar sentence ("requires the same title
    grammar as task sections (`A-Z`, `0-9`, spaces, and `& ' ( ) , . / -`, starting with
    a letter or digit)") must list `+` and note it is the task-section grammar _plus_
    `+`.
  - Leave the `@route+id#section` sub-bullet selector paragraphs and
    `src/native/capture.rs`'s sub-bullet `long_about` charset sentence unchanged — those
    describe task sections.
- `src/native/capture.rs`'s Pomodoro `long_about` does not enumerate the charset, so it
  needs no change; confirm this rather than assuming it.

Follow `sase/memory/cli_rules.md`: help output stays clear, complete, and alphabetically
ordered.

## Tests

Add coverage at every layer that has an existing test seam. Use a name with a trailing
`+` (`C++`) and an infix `+` (`BOB+SASE`), since those exercise different slug shapes.

- `src/native/capture_task_sections.rs`: `+` is accepted by `is_pomodoro_name` and still
  **rejected** by `is_section_title` / `parse_task_section_title`, so a `A+B` bullet is
  still not a task section. Extend the existing `parse_task_section_title` rejection
  table.
- `src/native/capture_pomodoro_name.rs`: extend
  `canonicalizes_names_and_rejects_invalid_ones` — `c++` → `C++`, `bob+sase` →
  `BOB+SASE`; `+`, `+A`, and `++` stay rejected. Add an end-to-end assignment test that
  writes `- [ ] () — C++` and asserts the returned slug is `c++` and `selectable` is
  true.
- `src/native/capture_pomodoros.rs`: a scanned `- [ ] () — C++` entry reports
  `slug: "c++"` and `selectable: true`; `select_named` resolves `c++` by whole slug and
  `c+` by prefix; `named_creation_name` returns `C++` for a novel `c++` selector.
- `src/native/capture_language.rs`: `@sase:deep-fix#c++` parses as a Pomodoro marker
  with `pomodoro_name` `c++` and no `invalid_pomodoro_name` diagnostic;
  `@sase+goog-exit#a+b` still reports `invalid_sub_bullet_section` (the task-section
  grammar did not move); the editor/spans path emits a `pomodoro_name` span covering
  `c++`; a cursor inside `c++` on `@sase:deep-fix#c+` resolves to
  `CompletionContext::PomodoroName` with the right replacement range, and the
  sub-bullet-vs-Pomodoro family ordering is pinned by a test.
- `src/native/capture_complete.rs`: a `c+` query offers the existing `C++` entry as a
  named (not nameable) row, and a novel valid `+` query produces a
  `creates_pomodoro: true` row whose `replacement` and `name` are the canonical selector
  and `C++`.
- `tests/cli.rs`: one integration test through the real binary —
  `bob capture-pomodoro-name -n 'c++'` writes `- [ ] () — C++` and reports `slug` `c++`
  in JSON, and a following `bob capture '@route:id#c++' '...'` links beneath that entry
  rather than creating a duplicate. Also assert `bob capture-pomodoro-name --help`
  mentions `+`.

## Verification

```bash
just all       # cargo fmt --check, cargo clippy --all-targets --all-features, cargo test
```

Run `just all` and make sure it passes before finishing. `just check-scripts` is not
required: no shell script under `scripts/` parses Pomodoro names.

## Out of scope / follow-ups

- **Task-section titles and `@route+id#section` selectors keep their current character
  set.** If they should also accept `+`, that is a separate, user-approved change:
  widening `is_section_title` changes which ordinary bullets are treated as task
  sections.
- **`bob-plugins` needs no change.** The `task-status-cycler` plugin's Pomodoro name
  handling is already unrestricted (`POMODORO_NAME_TAIL_RE` captures `(.*)`), so it can
  already author `+` names; this change makes bob-cli agree with it. Do not edit the
  plugins for this work.
- **`bob-mac-capture` could not be inspected** while planning (its primary workspace is
  not present on this machine, so `sase repo open bob-mac-capture` fails). It consumes
  the `capture-complete` / `capture-pomodoros` JSON contracts, which are unchanged here,
  and it should pick the widened grammar up for free through the `bob` subprocess. If
  the implementer can open that repo, grep it for a duplicated Pomodoro-name character
  class (a client-side validation regex) and file a task bead via `/sase_new_task` if
  one exists, rather than expanding this plan.
- **No SASE memory edits.** The `glossary:pomodoro` strand describes the `— NAME` shape
  generically and does not enumerate allowed characters, so it stays accurate.
