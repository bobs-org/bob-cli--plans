---
tier: tale
title: Capture priority marker becomes $<N>
goal: Replace the p:<N> capture priority marker with $<N>, where a bare $ means level 1.
size: medium
decisions:
  legacy_p_token:
    ask: Should a typed `p:<N>` fail with a teaching error once `$<N>` replaces it?
    choices:
      retire:
        Usage error `retired_priority_marker` teaching `$<N>`; never captured as text
      alias: "`p:<N>` keeps working as a silent synonym; docs teach only `$<N>`"
      literal: "`p:<N>` becomes ordinary task text with no diagnostic"
    default: retire
    why:
      Muscle memory and saved templates would silently become task text; matches retired
      `:` notes
    answer: retire
  dollar_escape:
    ask: Should `\$` / `\$<N>` keep a trailing dollar amount as literal task text?
    default: true
    why:
      Trailing amounts (`Venmo Alex $3`) are common; `\&foo:bar` already sets the
      precedent
    answer: true
  memory_glossary_schedule_log:
    ask: Should the glossary `schedule-log` strand name `$<N>` instead of `p:<N>`?
    memory:
      - glossary:schedule-log
    default: true
    corrects: or when `bob capture` uses `p:<N>`, we add a `SCHEDULE LOG` bullet
    answer: true
proposed_by: bbugyi200.apollo.8b
decided_by: auto
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.8b](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.8b.md)
- **COMMITS:**
  - [76191d7](https://github.com/bobs-org/bob-mac-capture/commit/76191d7ce72079632761f6f7204b23c9dede4df3)
    — feat(capture): show $\<N\> in hints, fixtures, and README

# Capture priority: migrate `p:<N>` to `$<N>`

## Goal

Replace the capture priority marker `p:<N>` with `$<N>`, where `<N>` is optional and
defaults to `1`. After this change these two captures parse identically (priority level
1, rolled scheduled date, Schedule Log entry):

```text
foo bar baz @bob $1
foo bar baz @bob $
```

and today's `foo bar baz @bob p:1` follows the `legacy_p_token` decision.

Nothing else about priority changes. The vault output (`[priority::<value>]`, the rolled
`[scheduled::]`, the `🗓️ **SCHEDULE LOG**` / `🎲 P0 → P<n> …` entry), the config file
and its levels, `BOB_PRIORITY_ROLL_SEED`, JSON result fields (`priority`,
`schedule_log`, …), and the editor span wire kind `priority` all stay exactly as they
are. Only the typed token changes.

## Grammar contract for `$<N>`

`$<N>` has every positional rule `p:<N>` has today: same terminal-region rules, same
item-wide slot, same conflicts. Only the spelling changes.

| Token                                                 | Meaning                                                                |
| ----------------------------------------------------- | ---------------------------------------------------------------------- |
| `$`                                                   | Priority level 1 (identical to `$1`)                                   |
| `$<digits>`                                           | Priority level N (the Nth configured level)                            |
| `$0`, `$9` (past the configured count)                | Same usage error as today's `p:0` / `p:9`, reworded (see Messages)     |
| `$$`, `$1.50`, `$x`, `$-1`, `$1k`, overflowing digits | Not a marker; stays literal text (same rule as a malformed `p:` today) |

- `$`/`$<N>` is recognized only in the terminal marker region, on either side of a
  trailing route token, and in any order with `s:<N>`, `%…`, `&`/`*` dependency
  modifiers, and the bare `#`. A `$3` in the middle of a line stays text:
  `Pay $3 to Bob`.
- One priority marker per item. A second one on another physical line keeps today's
  duplicate-marker error. A second one on the same line keeps today's stop-extraction
  behavior.
- All existing combination rules carry over unchanged: `s:<N>` wins the date, and
  refusals for Pomodoro start `=<X>`, close `=x`, link, adjust/shift, toggle,
  project-note `#`, bare existing-owner relationships, and `!` completion all still
  apply.
- Template rows: a placeholder can produce the token (`${0}` with field `2` expands to
  `$2`). The masked-template highlighting rule that today mentions `p:{0}` now uses
  `${0}`.
- `$` has no "still typing" partial form, because a bare `$` is already a complete
  marker. Remove the `text == "p:"` partial-token case from `is_trailing_transparent`
  (`src/native/capture_language/dependencies.rs`). Keep `"s:"`.

The single grammar entry point is `parse_priority_token` in
`src/native/capture_language/markers.rs`. It is the only parser, and every other site
(terminal extraction, `is_trailing_transparent`, `first_relationship_text_token`, and
the solo Pomodoro-link conflict checks in `tokens.rs`) calls it. Change it to accept `$`
(→ `Some(1)`) and `$<digits>` (→ parsed `u64`, `None` on overflow), and to no longer
accept `p:`. Update its doc comment.

## Legacy `p:<N>`

> [!decision] legacy_p_token = retire Add
> `parse_retired_priority_token(token) -> Option<u64>` in `markers.rs`. It matches
> exactly the old shape `p:<digits>`, the only shape that ever parsed. A malformed `p:x`
> was always literal and stays literal.
>
> Treat a retired token positionally like a priority marker everywhere
> `parse_priority_token` gates a decision: the `marker_like` region walk and
> `extract_terminal_marker`, `is_trailing_transparent`, `first_relationship_text_token`,
> and the solo-link checks in `tokens.rs`. That way `foo p:2 @bob`, `foo @bob p:2`,
> `foo p:2 s:1 %`, a marker on a child line, and a template row are all caught, and none
> of them becomes literal text.
>
> The cleanest plumbing is probably a `retired_priority` slot on `TerminalMarkers`,
> holding the typed token and its span, which `extract_terminal_marker` fills. Then
> refuse it in both parse paths:
>
> - Execution: in `resolve_line_inner` (`line.rs`), beside
>   `reject_legacy_bullet_markers`.
> - Editor: in `editor_parse.rs`, beside `legacy_bullet_marker_diagnostic`. This is an
>   error diagnostic with code `retired_priority_marker`, ranged on the `p:<N>` token,
>   with the same message as execution.
>
> Message (one shared constructor in `markers.rs`):
> `` `p:2` is retired; write `$2` for priority level 2 (a bare `$` means `$1`) ``. When
> an item-family conflict check runs first (for example a solo Pomodoro link), reporting
> that family's conflict message is acceptable, because it now names `$<N>`. Every path
> must fail, and every failure must teach the `$` spelling.
>
> The retired token sets no priority and gets no `priority` span.

> [!decision] legacy_p_token = alias `parse_priority_token` also accepts `p:<digits>`.
> Nothing is refused. All messages, help, and docs teach `$<N>`. One sentence in
> `docs/capture.md` "Scheduling and priority" says `p:<N>` is still accepted as a legacy
> spelling.

> [!decision] legacy_p_token = literal `p:<N>` simply stops parsing. Docs say once that
> the old spelling is now ordinary text.

## Literal dollar amounts

> [!decision] dollar_escape Mirror the `\&`/`\*` escape rule (`docs/capture.md`,
> "Escape"). A token `\$` or `\$<digits>` is an escaped literal only where the unescaped
> token would have been a priority marker, meaning it sits in the terminal marker
> region. There only the leading `\` is consumed:
>
> - `Pay Bob \$3` → body `Pay Bob $3`, no priority.
> - `Pay Bob \$3 $2 @cash` → body `Pay Bob $3`, level 2, routed to `cash`.
>
> Elsewhere `\$` stays verbatim (`cost \$3 today` keeps the backslash), exactly like
> `\*`. The escaped token stops terminal extraction like any non-marker token. Apply the
> unescape in the shared body-building path so execution, the dry-run preview, and
> `capture-parse`'s `body` agree. Add the case to the docs' grammar notes and to the
> reword/escape advice below.

> [!decision] dollar_escape = no No escape. The docs tell users that a trailing `$<N>`
> is always a priority marker and to move the amount (`Pay $3 to Bob`), just as with a
> trailing `%` or bare `#`.

## Messages and help text (bob-cli)

Replace every user-facing `p:<N>` spelling with `$<N>`. Known sites:

- `src/native/capture/plan.rs` `resolve_priority`: the out-of-range error becomes
  `$<N> is not a configured priority level; use $ or $1 through $<count> (<labels>)`.
  Keep `<labels>` as today. Update the `ResolvedPriority` and `resolve_priority` doc
  comments.
- `src/native/capture_language/line.rs`: duplicate label `priority marker ($<N>)` and
  the `#` Pomodoro-note conflict (`… cannot be combined with '$<N>'`).
- `src/native/capture_language/editor_parse.rs`: the `SpanKind::Priority` label
  `priority marker ($<N>)`, the `=x` conflict comment, and the `"p:2"` fallback becomes
  `"$2"`.
- `src/native/capture_language/dependencies.rs`: the `"p:2"` example fallback in
  `refuse_relationship_needs_text` becomes `"$2"`, plus the doc comments.
- `src/native/capture_language/markers.rs`: `POMODORO_START_PRIORITY_CONFLICT_ERROR`,
  `POMODORO_CLOSE_PRIORITY_CONFLICT_ERROR`, and the doc comments.
- `"Pomodoro link capture cannot be combined with p:<N>"` in `tokens.rs` (the const plus
  two inline copies), `item.rs`, and `capture/pomodoro_link.rs`.
- `src/native/capture/pomodoro_adjust.rs` (toggle, adjust, shift) and
  `src/native/capture/pomodoro_close.rs`.
- `src/native/capture/budget.rs`: `… queue it with ^ or defer with $<N>` in all four
  messages, plus the doc comments.
- `src/native/capture/cli.rs` long help: lines describing terminal markers, the priority
  paragraph (also say a bare `$` means level 1), `=<X>` / `!` / project-note / toggle
  conflict lines, the `BOB_PRIORITY_ROLL_SEED` env line, and the Examples block. **Shell
  quoting:** an unquoted or double-quoted `$2` is expanded by the shell, so examples
  must be `bob capture buy milk '$2'` and `bob capture research rust '$4' @dev`. Add one
  help sentence telling users to single-quote `$<N>` in a shell.
- `src/native/capture_parse.rs` help text (the three `p:<N>` mentions) and the
  `json("body p:2 s:1 % @groceries")` test.
- Comment-only mentions: `capture_schedule_log.rs`, `editor_classify.rs`.
- `src/native/config/mod.rs` test `missing_file_message_is_command_neutral`: assert the
  message names neither `$<N>` nor `p:<N>`.

When done, `rg -n "(^|[ \"'\`(])p:([0-9]|<N>|<count>|\{)" src tests docs
README.md`should match only the retired-marker code, its tests, and the retired-syntax docs paragraph (or nothing at all under`alias`/`literal`). Historical files under `sase/repos/plans/`and accepted`sase/memory/decisions/`
records are archives. Do not edit them.

## Documentation (bob-cli)

- `docs/capture.md`:
  - Grammar table row →
    `` `$` / `$<N>` | Write priority level N (bare `$` = 1) and roll a scheduled date in that level's window ``.
  - Rewrite "Scheduling and priority" for `$`/`$<N>`, including the `$0` note ("there is
    no `$0`"), the `$<N> s:<N>` no-entry rule, and the new out-of-range message. Add a
    shell-quoting note (single-quote `'$2'`; double quotes expand it).
  - Add the legacy paragraph per `legacy_p_token` (under `retire`, modelled on the short
    "`#now` is retired" section) and the escape/reword advice per `dollar_escape`.
  - Convert every other `p:<N>` / `p:1` / `p:2` mention: multi-item, saved and bulk
    templates (`p:{0}` → `${0}`), authored sub-bullets, clipboard, Pomodoro
    start/close/link/adjust/shift/toggle conflicts, relationship refusals and
    conformance vectors (RX9), `!` completion refusals, the plan budget JSON example,
    the project-note `^prj` priority paragraph, the sub-bullet section, the JSON result
    field notes, and the diagnostics.
  - Under `retire`, add `retired_priority_marker` to the diagnostic-code list.
  - Shell examples that carry `$<N>` must single-quote it.
- `README.md` (≈ lines 148, 415, 476, 1301, 1404), `docs/getting-started.md`
  (`$1`–`$4`), and `docs/projects.md` (≈ lines 497–507, 956).

## Bob Mac Capture (linked repo `bob-mac-capture`)

Mac Capture is a thin client. It renders bob's spans and messages, so `$` highlighting
and live preview need no Swift logic change. Open the repo with `/sase_repo`
(`sase repo open bob-mac-capture -r "…"`), read its `AGENTS.md`, and then:

- `Sources/BobMacCapture/CapturePanelView.swift`: in the hardcoded
  `plan_theme_cap_exceeded` hint, change both the visible text and the
  `accessibilityLabel` to `Queue it with ^ or defer with $<N>.`
- `README.md` (≈ lines 1074, 1175, 1934, 1950, 2257): convert `p:<N>` / `p:1`.
- Tests and fixtures that embed bob's own wording should model current bob output:
  - the plan-budget warning/refusal JSON fixtures;
  - `BobProcessClientTests` (`buy milk % $1` draft and argv assertion);
  - the template-shelf draft `Read {0} @reading^{1} $2`. Its byte-offset spans and
    `cursor` must be recomputed, or regenerated from the new bob, in both
    `CaptureTemplateShelfModelTests.swift` and `capture-templates-list-populated.json`.

  Opaque made-up strings such as the `"p:1 roll"` schedule-log reason in
  `CaptureModelTests` may stay.

- Verify per that repo's `AGENTS.md`: `just portable-check` (on athena if `swift` is
  missing). Then do the best-effort scratch compile of `Sources/BobMacCapture*` /
  `Tests/BobMacCapture*` on the mac (`bundle.sh` + `just typecheck-tests`) and delete
  the scratch copy. If the mac is offline, say so in the final report. Commit this repo
  as part of the turn's final declaration.

The Obsidian plugins (`bob-plugins`) and chezmoi have no `p:<N>` usage, so nothing
changes there.

## Memory

> [!decision] memory_glossary_schedule_log Use `/sase_memory_write`. In
> `sase/memory/glossary/schedule-log.md`, change "or when `bob capture` uses `p:<N>`" to
> "or when `bob capture` uses `$<N>`". Leave the rest of the strand unchanged, then run
> `sase memory init`. Do not touch the superseded, immutable record
> `decisions/now-tag-is-user-owned`.

## Tests

Convert existing fixtures mechanically from `p:N` to `$N`, mixing in bare `$` where a
test only needs level 1. Tests that exist to cover retirement are the exception. Rust
`.arg("$1")` needs no quoting. Any test that pipes through `sh -c`/`printf` must
single-quote. Files:
`tests/cli/capture/{priority, sub_bullet,project_note,pomodoro_*,relate_existing,parse_*,authored}.rs`,
`src/native/capture/tests/{grammar,template}.rs`, `src/native/capture/task_blocks.rs`,
and
`src/native/capture_language/tests/{grammar,draft,chain,editor_modes, editor_spans,completion,ref_grammar}.rs`.

New coverage:

1. `parse_priority_token` unit table: `$`→1, `$1`→1, `$4`→4, `$0`→0, `$12`→12; `$$`,
   `$1.5`, `$x`, `$-1`, `$1k`, `p:1`, `p:`, and an overflowing `$99999999999999999999` →
   `None`.
2. Grammar equivalence: `foo bar baz @bob $1` and `foo bar baz @bob $` parse to the same
   item (body `foo bar baz`, route `bob`, level 1). Also cover `$` before the route,
   `$ s:3`, `$2 %`, `&dep $`, a marker on a child line, and a duplicate `$` on two lines
   (duplicate error naming `$<N>`).
3. CLI (`tests/cli/capture/priority.rs`): bare `$` writes P1's value with a seeded roll
   and Schedule Log, byte-identical to `$1`. The `$9`/`$0` errors use the new message.
   Mid-line `Pay $3 to Bob` stays text.
4. Editor/`capture-parse`: `$` and `$2` produce `priority` spans with correct ranges.
5. Under `retire`: `foo p:2`, `foo p:2 @bob`, `foo @bob p:2 s:1`, a child-line `p:2`,
   and a template row expanding to `p:2` each fail. Execution prints the retired message
   (exit 2). `capture-parse` reports `retired_priority_marker` ranged on the token.
   `p:x` stays literal.
6. When `dollar_escape` is accepted: the escape cases listed in that callout, checked
   through both execution and `capture-parse`.

## Verification

- `just check` (fmt, clippy with the repo's gate, full test suite) passes in bob-cli.
- Manual smoke test with a seeded temp vault and config (`BOB_DIR`, `BOB_CONFIG_FILE`,
  `BOB_PRIORITY_ROLL_SEED`, `--dry-run`): `bob capture -f json -- 'foo bar baz @bob $'`
  and `… '$1'` print identical output. `bob capture-parse` shows a `priority` span for
  `$`.
- The bob-mac-capture verification listed above.

## Non-goals

- No rewrite of saved template shelf entries or other stored drafts. A saved template
  that still says `p:<N>` follows `legacy_p_token` when used (under `retire` the error
  tells Bryan to re-save it with `$<N>`).
- No change to vault content, the Obsidian `Ctrl+Shift+P` picker, config schema,
  `bob task reroll`, or JSON field names.
- No new completion or picker for `$`.
