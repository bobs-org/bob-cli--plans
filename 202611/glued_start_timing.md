---
tier: epic
title: Glue start and past timing to the Pomodoro name
goal: 'Capture `=` and `x=` tokens take their timing glued to a bare Pomodoro name
  (`=build-`, `=10-3fix`, `x=fix3`); the `@` and `<X>#` separators are retired with
  teaching errors, and every client, doc, and test teaches only the shorter spellings.

  '
decisions:
  hash_names:
    ask: Should `#` before an unquoted name stay valid only for names a bare spelling
      cannot express?
    choices:
      when_needed: '`=#deep-work`, `=3#sase-v18`, `=#2fa` stay valid; `=#fix` and
        `=3#fix` teach `=fix`, `=3fix`'
      literal_only: Retire every unquoted `#name`; non-bare names need `=#"DEEP WORK"`;
        shared vectors change
    default: when_needed
    why: Bare-spellable names must be typed bare; history names stay one short token
      away
    answer: when_needed
  name_rule_scope:
    ask: Should bob also refuse to create new names that a bare start cannot spell?
    choices:
      grammar: Only the `=`/`x=` grammar changes; such names can still be created
        and are reached via `#`
      refuse_new: Capture, `types:`, and `bob types rename`/`split` refuse new names
        a bare start can't spell
    default: grammar
    why: The grammar alone delivers the short spellings; Obsidian pickers would bypass
      a ban anyway
    answer: grammar
  memory_record:
    ask: Should the docs phase add a decisions record for the glued-timing spellings
      and bare-name rule?
    memory:
    - decisions:start-names-glue-to-timing
    default: false
    answer: false
phases:
- id: start-grammar
  title: '`=` starts and link `=` suffixes (bob-cli)'
  depends_on: []
  size: medium
  description: 'start-grammar: new bare-name rule, glued start lexer with retired-spelling
    teaching errors, editor spans, completion, composers, start_token JSON, grammar
    13, help goldens, and start tests.'
- id: past-grammar
  title: '`x=` past records (bob-cli)'
  depends_on:
  - start-grammar
  size: medium
  description: 'past-grammar: glued past lexer and teaching errors, spaced-literal
    merge, past spans and completion, removal of the pomodoro_start_timing need, help
    goldens, and past tests.'
- id: docs
  title: Document the glued spellings (bob-cli)
  depends_on:
  - start-grammar
  - past-grammar
  size: medium
  description: 'docs: rewrite capture, reference, task-type, and tutorial pages to
    teach only the glued spellings, then run the docs checks.'
- id: mac-agenda-hints
  title: Stop composing `=#` hints in Bob Mac Capture
  depends_on:
  - start-grammar
  size: small
  description: 'mac-agenda-hints: render bob''s start_token for Later entries, update
    the empty-state copy and dim-style keying, refresh tests, and gate on macOS CI.'
proposed_by: bbugyi200.apollo.cf
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-05 05:23:35
status: done
bead_id: bob-cli-ha
---

- **PROMPT:** [prompts/202611/glued_start_timing.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/glued_start_timing.md)
- **BEAD:** [bob-cli-ha](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ha/README.md)

# Glue Start And Past Timing To The Pomodoro Name

## Goal

Bryan wants `=` (start) and `x=` (record past) captures to drop the separator between a
Pomodoro name and its timing, and to require the shorter form:

| Today (retired)       | Required spelling   | Meaning                              |
| --------------------- | ------------------- | ------------------------------------ |
| `=build@-`            | `=build-`           | Start BUILD, 25 min, start 5 min ago |
| `=10-3#fix`           | `=10-3fix`          | Start FIX, 50 min, start 15 min ago  |
| `=fix@3` / `=3#fix`   | `=fix3` / `=3fix`   | Start FIX, 15 min                    |
| `x=fix@3` / `x=3#fix` | `x=fix3` / `x=3fix` | Record a past 15-minute FIX          |

Each character saved matters because capture has to stay as fast as possible.

### Answer to "we can support this by disallowing dashes and trailing digits, right?"

Yes. The timing grammar is `X = [0-9]*(-[0-9]*)?`, so a name that never contains `-` and
never ends in a digit has an unambiguous boundary with a timing on either side. There
are three refinements:

1. **The rule only has to bind _bare_ names**, meaning names typed straight after `=` or
   `x=`. `#` survives only as the escape for names that break the rule (`hash_names`),
   so vault history such as `SASE V18`, `DEEP WORK`, `FIX-APOLLO`, and merged
   `BOB + DECKS` stays reachable. Whether bob should also refuse to _create_ such names
   is `name_rule_scope`.
2. **Digit-led names were already non-bare.** Bare names could never start with a digit,
   and now a leading digit run is always timing: `=2fa` starts FA for 10 minutes;
   before, it was a teaching error. Close-shaped names (`x`, `x2`) also keep needing
   `#`.
3. **`-` is today's spelling for a space** (`=deep-work` selects DEEP WORK). Multi-word
   names lose their bare spelling. They start by prefix (`=deep`) or through `#`
   (`=#deep-work`), and teaching errors say so.

`@` disappears from `=`/`x=` entirely, including the `=~2@3` and `=bugs~2@3` forms,
because the timing now always sits next to the name and the `~K` drop list always comes
last.

## Normative grammar

```text
start  = "=" [X] BARE [X] ["~" K]         ; X at most once (either side)
       | "=" [X] "#" HNAME ["~" K]        ; X only in front of "#"
       | "=" [X] "#" LITERAL ["~" K]      ; X only in front of "#"
       | "=" [X] ["~" K]                  ; unnamed: unchanged
past   = "x=" [N] BARE [N] ["!"] ["=" CLOSE]
       | "x=" [N] "#" (HNAME | LITERAL) ["!"] ["=" CLOSE]
       | "x=" [N] ["!"] ["=" CLOSE]       ; unnamed: unchanged
X      = the existing se<X> suffix [0-9]*(-[0-9]*)?, nonempty
N      = ASCII digits (past takes no offset)
BARE   = starts with an ASCII letter; Pomodoro selector bytes except "-";
         does not end in an ASCII digit; not close-shaped (x, x2, x!, *, !), not x#…
HNAME  = today's "#name" part (runs to whitespace or "~"); with hash_names=when_needed
         it must NOT be bare-spellable
LITERAL= today's #"…" exact-name literal, unchanged
```

**Lexing a bare name.** Scan the run of Pomodoro selector bytes, excluding `-` and
stopping at whitespace, `~`, `=`, `!`, `@`, and `#`. Then peel the maximal trailing
ASCII-digit run off the name. That digit run starts the trailing X, which may continue
with `-digits`. A leading X is read first, exactly as today. A bare name may now follow
a leading X; today it needs `#`.

**Name rule helper.** `can_spell_bare(name)` becomes: selector component, leading ASCII
letter, no `-`, no trailing ASCII digit, and not close-shaped. Every composer goes
through it: `named_start_token`, `start_name_spelling`, completion replacements, agenda
hints, suggestion builders, and teaching-error rewrites. That keeps `#` appearing only
where it is needed.

### Valid examples (new)

| Input                                                    | Result                                    |
| -------------------------------------------------------- | ----------------------------------------- |
| `=build-`, `=build-2`, `=-2build`                        | BUILD, 25 min, offset 1 / 2 / 2           |
| `=10-3fix`, `=fix10-3`                                   | FIX, 50 min, offset 3                     |
| `=3fix`, `=fix3`                                         | FIX, 15 min                               |
| `=fix3~2`, `=3fix~2`                                     | FIX, 15 min, dropping queued link 2       |
| `=2fa`                                                   | FA, 10 min                                |
| `=#deep-work`, `=3#sase-v18`, `=3#"DEEP WORK"`           | Unchanged non-bare names, timing in front |
| `@work:outline=bugs3`, `^bob:ready=3bugs`                | Link starts share the suffix lexer        |
| `x=3fix`, `x=fix3`, `x=fix3!=*3 note`, `x=build2 x=fix3` | Past records                              |
| `x =bugs3`, `x=3 =fix`                                   | Chains unchanged otherwise                |

### Retired spellings (claimed, never executed, never prose)

Each gives a teaching error that names the exact rewrite. Retirements chain to the final
spelling: `==3#bugs` teaches `=3bugs`.

| Typed                                          | Teaches                                                                                   |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `=fix@3`, `=fix@-`, `=fix@10-3`, `=fix@`       | `=fix3`, `=fix-`, `=fix10-3`, `=fix3` (the timing goes right after the name)              |
| `=3#fix`, `=10-3#fix`, `=#fix`, `=#fix~2`      | `=3fix`, `=10-3fix`, `=fix`, `=fix~2` (`#` only for names a bare spelling cannot express) |
| `=~2@3`, `=fix~2@3`, `=fix@3~2`                | `=3~2`, `=fix3~2` (the timing comes before the `~` drop list)                             |
| `=#"DEEP WORK"@3`, `=#"DEEP WORK"3`            | `=3#"DEEP WORK"`                                                                          |
| `@work:outline=bugs@3`, `@work:outline=3#bugs` | `@work:outline=bugs3`, `@work:outline=3bugs`                                              |
| `x=fix@3`, `x=3#fix`, `x=#fix`                 | `x=fix3`, `x=3fix`, `x=fix`                                                               |
| `x=#"DEEP WORK"@3`                             | `x=3#"DEEP WORK"`                                                                         |

### New or changed errors

| Typed                                    | Error                                                                                                                                                         |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `=deep-work`, `=3deep-work`, `=sase-v18` | A bare name cannot contain `-` (it starts the timing); write `=#deep-work`, `=3#deep-work`, or `=#sase-v18`. Detect this as a letter right after a trailing X |
| `=3fix2`                                 | Timing twice: write `=3fix` or `=fix2`                                                                                                                        |
| `=3x`                                    | Close-shaped name: write `=3#x`                                                                                                                               |
| `=deep work`                             | The multi-word hint becomes `=#deep-work` (it used to teach `=deep-work`)                                                                                     |
| `=bugs=3`                                | The order error teaches `=bugs3` or `=3bugs`                                                                                                                  |
| `x=fix-`, `x=fix3-1`, `x=3-1fix`         | Past work has no offset: `x=fix`, `x=fix3`, `x=3fix`                                                                                                          |
| `x=3x`                                   | `x=3#x` (`x=3build` is now valid; it used to teach `x=3#build`)                                                                                               |

> [!decision] hash_names = literal_only Every unquoted `#name` after `=`/`x=` (with or
> without X) is retired. Non-bare names use `#"…"` literals: `=#"DEEP WORK"`,
> `=3#"SASE V18"`, `=#"2FA"`, `=#"X"`. Teaching errors and composers emit the literal
> (the uppercased canonical name in straight quotes). `=#` alone and `=#"…` partials
> stay incomplete editing states that need `pomodoro_name`. The `=#…` completion vectors
> change, so phase `start-grammar` must mirror them (see that phase).

### Accepted consequences

- A bare name that really ends in a digit silently becomes a count: `=sasev18` starts
  SASEV for 90 minutes. The live preview shows the duration, and the existing
  `did you mean …?` near-miss warning still fires against open entries. No new guard.
- Selection semantics do not change: an open whole-slug match first, then a prefix
  match; then completed entries; literals match by key equality. Bare selectors simply
  cannot contain `-` anymore.

### Non-goals

- `=*base#dest…` merge and `=/base#type…` split keep their `#` lists, because they take
  no timing.
- The link route-component form `@route:id#name=<X>` / `^route:id#name=<X>`, the
  Obsidian `se<X>` + Tab keymaps, and close grammar (`x…`) are unchanged.
- No automatic `capture-rewrite` rule for the retired spellings; teaching errors only.
- Vault history is never rewritten. `CHANGELOG.md` and the Cargo version stay untouched
  (release train). Commits that change the grammar use `feat(capture)!:`.

## Phases

### Phase `start-grammar`: `=` starts and link `=` suffixes (bob-cli)

Implement the start half of the normative grammar in bob-cli and keep `just check`
green.

1. `src/native/capture_language/bare_name.rs`: apply the new name rule in
   `bare_start_name_len` (it returns the name length without the peeled digits and stops
   at `-`) and in `can_spell_bare`. Update `start_name_spelling`, `named_start_token`,
   and `echo_named_start` for leading and trailing X. Add unit tests for the rule's
   edges: `fix3`, `deep-work`, `x`, `xmas3`, `b2b`, `2fa`, `bob+decks2`.
2. `src/native/capture_language/item.rs`:
   - `session_equals_token`: lex a leading X, then BARE (now allowed after a leading X),
     then a trailing X, then `~K`.
   - Keep claiming the retired `#name`, `@<X>`, and `~K@<X>` shapes, flagged so they
     error.
   - Represent the trailing X so that `effective_start_timing`,
     `named_start_name_offset`, the span builders, and the link-suffix range code get
     correct byte offsets. The old `timing` offset pointed at the `@` sigil.
   - Rework `named_token_error`, `named_timing_twice_error`,
     `named_shape_error_with_drop` (multi-word hint), `spaceless_start_drop_is_valid`,
     and `spaceless_extra_is_timing` (`=bugs @3` hints `=bugs3`).
   - Replace `glued_timing_diagnostic` with the dash-in-name diagnostic.
3. `src/native/capture_language/markers.rs`: rewrite the start messages
   (`start_timing_twice_error`, `start_timing_incomplete_error`,
   `pomodoro_glued_timing_error`, named-start shape and name errors). Add one
   retired-separator message helper so every retired form above reads the same way.
4. `src/native/capture_language/tokens.rs` (`parse_link_start_suffix`): link starts
   inherit the lexer. Fix the name and timing span ranges now that no `@` sits between
   them, and keep the "no drop list in a link start" error.
   `src/native/capture_language/start_selection.rs`: the drop-list teaching rewrites
   become `=3bugs~2`-style.
5. Live editor:
   - `src/native/capture_language/editor_pomodoro.rs` and `editor_classify.rs` keep
     execution and `capture-parse` in agreement.
   - Spans: `pomodoro_start` covers `=` plus a leading X and, separately, a trailing X;
     `pomodoro_name` covers the name (a literal includes its quotes);
     `pomodoro_start_drop` is unchanged.
   - The start path no longer emits the `pomodoro_start_timing` need. Leave the variant
     in place if the past path still uses it; phase `past-grammar` removes it.
6. Completion:
   - `src/native/capture_language/completion.rs` and `src/native/capture_complete/`:
     `=3bu|`, `=-2bu|`, and `^route:id=3bu|` are name fields.
   - Accepting a candidate never leaves a retired spelling. In a `=#bo|` field, a
     bare-spellable row replaces the `#` too (`=robot`), and a non-bare row keeps its
     `#` or literal.
   - Prefer keeping `tests/fixtures/pomodoro_themes/vectors.json` byte-identical. If a
     vector must change, mirror it into bob-plugins (open it with `/sase_repo`) in both
     copies: `scripts/fixtures/…` and `plugins/bob-ledger-tools/scripts/fixtures/…`. Run
     its pomodoro-theme vector scripts there; they compute the query with
     `draft.slice(2)`, so do not change drafts' `=#` prefixes casually.
7. Composers:
   - `src/native/capture/pomodoro_start.rs` `named_start_suggestion` emits
     `={raw}{slug}` when bare, else `={raw}#{slug}`.
   - `src/native/capture/event_conflict.rs` fit suggestion: `=2bugs`.
   - `src/native/capture_pomodoro_start.rs` messages: `=plan`, not `=#plan`.
   - `close_selection.rs` and `markers.rs` close-hash advice reaches the new forms
     through the helpers.
   - `src/native/capture_pomodoros.rs`: the human agenda trail already uses
     `named_start_token`. Add an additive `start_token` string to each
     `capture-pomodoros` JSON entry (present when the entry is selectable with a
     non-empty slug, built by `named_start_token`) so Bob Mac Capture stops composing
     tokens.
8. Bump `GRAMMAR_REVISION` to 13 in `src/native/capture_language/mod.rs` and update its
   assertions.
9. Help text: update the start parts of `src/native/capture/cli.rs` (prose and examples)
   and `src/native/capture_parse.rs`. Regenerate the CLI reference goldens with
   `just docs-gen`. Edit no hand-written `docs/` pages in this phase.
10. Tests: convert every start, link-start, chain, completion, and editor test and
    fixture to the new spellings. Find them with
    `rg "[\"' ]=[0-9-]*#[a-zA-Z\"]|[\"' ]=#?[a-zA-Z][a-zA-Z0-9.+-]*@[0-9-]" src tests`.
    Key files:
    - `src/native/capture_language/tests/{grammar,chain,exact_name,retired_close,bare_close,editor_spans,editor_modes,completion,rewrite}.rs`
    - `src/native/capture_complete/tests/`
    - `tests/cli/capture/{pomodoro_start_named,pomodoro_start_drop,parse_pomodoro,parse_pomodoro_start_drop,pomodoro_override,pomodoro_chain,plan_budget,theme_warnings,meet_event_run,event_start_warnings,complete_themes,complete_editor}.rs`
    - `tests/cli/completion/`
    - `tests/fixtures/meet_event_run/vectors.json`

    Add tests for every row of the start examples, retired-spelling, and error tables.

> [!decision] name_rule_scope = refuse_new No start-side code in this phase; phase
> `past-grammar` adds the guard for both grammars.

### Phase `past-grammar`: `x=` past records (bob-cli)

Implement the past half of the normative grammar on top of the shared helpers from
`start-grammar`.

1. `src/native/capture_language/past.rs` (`split_past_token`, `validate_past_e`, and its
   teaching errors):
   - E becomes `[N] BARE [N]` or `[N] "#" (HNAME|LITERAL)`.
   - Retire `@D`, and `#` before bare-spellable names.
   - Rework the offset errors (`x=fix-`, `x=fix3-1`, `x=3-1fix`).
   - Rework count-twice (`x=3fix2`). `x=3x` teaches `x=3#x`, and `x=3build` is valid.
   - Dangling `x=build@` becomes a retired-spelling error.
   - `x=#`, `x=3#`, and `x=#"DEEP` stay incomplete and need `pomodoro_name`.
   - The repair `!`, the second `=`, and the close lexer are unchanged.
2. `src/native/capture_language/draft.rs` (`merge_past_literal_tokens`): spaced literals
   are `x=3#"DEEP WORK"` with glued `!` and `=X`; drop the `@D` merge.
3. Editor and completion: past spans give the trailing count `pomodoro_past` as well (no
   `@D` span). `pomodoro_past_name_field` treats `x=3bu|` as a name field. Remove the
   `pomodoro_start_timing` need from `editor_model.rs` and every emitter, since nothing
   produces it anymore. Bob Mac Capture and bob-plugins do not read it.
4. Help text: update the past parts of `src/native/capture_parse.rs` (the long `x=`
   paragraph) and `src/native/capture/cli.rs` (`x=build2 x=fix3`). Run `just docs-gen`.
5. Tests: convert and extend
   - `src/native/capture_language/tests/past.rs`
   - `tests/cli/capture/{pomodoro_past,parse_pomodoro_past}.rs`
   - every chain or idiom test that contains `x=`

   Cover every past row of the tables above.

> [!decision] name_rule_scope = refuse_new Add one shared guard ("a new Pomodoro or type
> name must be spellable as a bare name": no space or `-`, no trailing digit, leading
> letter). Apply it to:
>
> - capture paths that _create_ an entry from a typed name: `=`, `x=`, link `=name`, and
>   `#name` suffixes when nothing matches today;
> - `types:`/`themes:` vocabulary validation, which keeps its strict drop-the-block
>   semantics;
> - `bob types rename` and `bob types split` targets.
>
> Selecting existing open or completed entries is unaffected, history is never
> rewritten, and the Obsidian plugins are out of scope. Teaching copy suggests a
> one-word name.

### Phase `docs`: document the glued spellings (bob-cli)

First read the `docs` reference memory with `/sase_memory_read`. Then rewrite every
hand-written page so it teaches only the new spellings. Keep the retired forms only in
the teaching-error tables.

- `docs/capture.md`:
  - "Starting the next Pomodoro" (lifecycle table, recognition paragraph)
  - "Dropping queued Task Links"
  - "Starting a named Pomodoro" (spelling paragraph, worked table, `=bugs@3` rows)
  - the start and past quick-reference tables near the top
  - "Recording a past Pomodoro" (shape, examples, teaching table, idioms)
  - "Chaining session operators"
  - every other `=3#…` or `name@N` example
- `docs/reference/capture-syntax.md` (sigil table: `@` no longer marks start or past
  timing; `#` marks only non-bare names), `docs/reference/json-outputs.md`
  (`grammar: 13`, `start_token`, needs list without `pomodoro_start_timing`),
  `docs/task-types.md` PT8 (a bare start emits `#` only for non-bare names),
  `docs/getting-started.md`, `docs/start/first-bob-day.md`, `docs/completion.md`.
- Final sweep: running `rg "[\"'\` (]x?=[0-9-]_#[a-zA-Z\"]|[\"'\`
  (]x?=#?[a-zA-Z][a-zA-Z0-9.+-]_@[0-9-]" docs
  src`must show only teaching-error rows and the`=\*`/`=/`/`#name=` forms named in
  Non-goals.
- Run `just docs-check` and `just docs-build`.

> [!decision] memory_record Through `/sase_memory_write`, add the
> `decisions:start-names-glue-to-timing` record (status accepted, decided on the landing
> date). It states:
>
> - **Applies to:** bob-cli capture and Bob Mac Capture.
> - **Claim:** `=`/`x=` timing glues to a bare name; `@` and `<X>#` are retired; bare
>   names have no `-` and no trailing digit; `#` only for non-bare names.
> - **Why:** keystrokes. Rejected alternatives: optional separators, which allow two
>   spellings; a global name ban.
> - **Cost:** multi-word and digit-ending names need `#`, and `=sasev18` reads as a
>   count.
> - **Reopens when:** Bryan wants dashes back in bare names.
> - **Evidence:** this epic.
>
> Then run `sase memory init`.

> [!decision] name_rule_scope = refuse_new Document the creation guard in
> `docs/capture.md` and in `docs/task-types.md` PT12 (vocabulary validation) and the
> rename/split sections.

### Phase `mac-agenda-hints`: stop composing `=#` hints in Bob Mac Capture

Open `bob-mac-capture` with `/sase_repo` and read its `AGENTS.md`.

- `Sources/CaptureCore/CaptureAgendaPresentation.swift`:
  - The Later-entry accessory renders bob's new `start_token` from `capture-pomodoros`
    (decode it as optional). Fall back to the old `=#<slug>` only when an older bob
    omits it.
  - The empty-state copy becomes `No Pomodoros planned · =NAME starts one`.
- `Sources/BobMacCapture/CaptureAgendaView.swift` `headerTimeView`: stop keying the dim
  start-hint style on a `=#` prefix. Carry an explicit flag from the presentation model
  instead.
- Update `CaptureAgendaPresentationTests` and any test strings that model bob input or
  output (`=3#bugs`, `x=fix@3`, `x =#bugs`, `=#fo`) to the new spellings where they
  stand for current bob behavior.
- The grammar gate (`minimumRevision = 5`) stays put.
- macOS targets are gated only by GitHub Actions. Run `scripts/dev/macos-ci-check`
  before pushing, or watch the exact-SHA macOS run to green with `/sase_monitor`.
