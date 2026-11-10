---
tier: tale
title: Spaced dot aliases & once the capture item is routed
goal:
  In Bob Mac Capture, a spaced `.` after an item's existing route marker (e.g.
  `@foo:bar`) becomes `&`, not `@`.
size: medium
decisions:
  routed_scope:
    ask:
      Should a bare `@route` or `@route#section` also make a spaced `.` prefer `&`, not
      just `@route:id`/`^id`/`+id`?
    choices:
      any_route:
        "`Buy milk @foo .` gives `&` too; a trailing prose `@name .` also gives `&`"
      block_id:
        Only block-ID markers count; `Buy milk @foo .` keeps `@`, which turns `@foo`
        into prose
    default: any_route
    why:
      After any route, a second `@` turns the first into prose or raises
      duplicate_capture_marker
    answer: any_route
  memory_glossary_capture_alias:
    ask:
      Should the glossary capture-alias strand be updated for the new spaced `.`
      precedence?
    memory:
      - glossary:capture-alias
    default: true
    corrects:
      "`*` opens the Depends On / Blocks picker and no `^`, `@`, or `:` alias applies"
    answer: true
proposed_by: bbugyi200.athena.1av
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-10 08:09:41
status: wip
---

# Spaced `.` prefers `&` once the item is routed

## Goal

In Bob Mac Capture, typing `.` after a space or tab should give `&` (Depends On), not
`@`, when the capture item already has a route marker such as `@foo:bar`, `@foo^bar`, or
`@foo+bar`. Bryan rarely uses multi-target bulk capture (`@a:x @b:y`), so after a route,
`&` is the character he actually wants. bob makes this choice and the Mac app only
applies bob's edit ([[decisions:mac-capture-is-a-thin-client]]), so this is a bob-cli
change. The Mac app's Swift code does not change; only its docs do.

## Root cause (confirmed with `bob capture-rewrite` / `capture-complete`)

`expand_sigil_alias` in `src/native/capture_language/rewrite.rs` tries `.` aliases in
this fixed order: `expand_dot_block_id_alias` (`^`), then `expand_dot_route_alias`
(`@`), then `expand_dot_dependency_alias` (`&`). The `@` alias fires wherever a bare `@`
would open route completion. After a trailing route marker, bob's completion grammar
always opens route completion for a new trailing `@`, either to extend a target run
(`Buy milk @foo:bar @` → `route`) or to replace a bare route (`Buy milk @foo @` →
`route`). So `@` wins and the `&` fallback never runs. Today's behavior:

| Draft (caret at end)                                                    | Today | Bare `&` opens    |
| ----------------------------------------------------------------------- | ----- | ----------------- |
| `Buy milk @foo:bar .`                                                   | `@`   | `task_dependency` |
| `Buy milk @foo^bar .` / `Buy milk @foo+bar .`                           | `@`   | `task_dependency` |
| `Buy milk @foo:bar@ .` / `Buy milk @foo:bar#DEEP .`                     | `@`   | `task_dependency` |
| `Buy milk @foo .` (the next `@` turns `@foo` into prose)                | `@`   | `task_dependency` |
| `Buy milk @foo:bar\n- detail .` (`@` raises `duplicate_capture_marker`) | `@`   | `task_dependency` |
| `@foo:bar .`, `@foo:bar Buy milk .` (leading run)                       | `&`   | `task_dependency` |

Leading routes already give `&`, because the grammar opens no route field there. Only
trailing routes and child lines get the wrong answer. That matches the original intent
of the alias, whose tip phrase is already "once routed".

## Design

Keep `expand_dot_block_id_alias` first; glued `.` → `^` and the `- .` / `- ^.` Move
forms are unchanged. For the spaced aliases, choose the order by whether the item is
already routed:

- **Item already routed** → try `expand_dot_dependency_alias` first. If `&` would not
  open Depends On, fall back to `expand_dot_route_alias`.
- **Otherwise** → keep today's order: `@`, then `&`.

Because the fallback still runs, places where `&` never opens keep giving `@`. Today `&`
opens nothing on `>` transfer lines (`@a+x > .`, `@a+x > @c+tid .`) or in `<` merge-in
source slots (`Create task < @cash:line .`), so those stay `@`. All the existing guards
stay as they are: position guard, whole-token check, wikilink claim, and the completion
field matching exactly `(start, cursor)`. Both helpers already enforce them.

"Item already routed" means the following:

1. Make the pre-key draft by removing the typed `.` (`start..cursor`) from `raw_text`.
   Parse this pre-key draft, not the post-key one: with the `.` still present, a
   trailing `@foo:bar` is no longer the line's last token and parses as prose.
2. Run `split_capture_draft(&pre_key)` and find the item one of whose `lines` contains
   `start` (`line.raw.start <= start <= line.raw.end`). Use the same lookup pattern as
   `expand_dot_block_id_alias`. If no item matches (for example a list header line), the
   item is not routed.
3. Check `parse_editor_item(item).item.local_destination_markers`. That field already
   lists exactly the complete, item-owned markers on any line of the item. It excludes
   `@@route` declarations, list-header routes, incomplete markers, and record or
   notebook views.

> [!decision] routed_scope = any_route The item is routed when
> `local_destination_markers` is non-empty.

> [!decision] routed_scope = block_id The item is routed only when some entry in
> `local_destination_markers` has `block_id.is_some()`. A bare `@route` or
> `@route#section` alone keeps today's order (`@` first).

The check is per item, not per line. On a child line, `@` raises
`duplicate_capture_marker` when the parent line is routed, but `&` is valid there. A
different blank-line item's route has no effect. For example, in
`Buy milk @foo:bar\n\nFix it .` the second item still gets `@`.

Some cases do not change. `@@sase Buy .` is a draft-wide declaration and has no local
marker, so it stays `@`. In `Buy milk @foo:bar some text .`, the mid-line `@foo:bar` is
prose by grammar (`capture-parse` reports `route: null`), so it stays `@`. Anyone who
really wants a second target can still type `@` directly. Undo right after the alias
restores the literal `.`, as it does today.

The typing-tip reverse check (`alias_tip.rs`) proves tips by running the forward
rewrite, so it follows automatically. Typing `&` by hand after a routed item now earns
the `dependency` / "once routed" tip. Typing `@` there earns no `route_after_text` tip,
because `.` no longer produces `@` at that position. The JSON shape, alias ids,
summaries (`Typed & for .`), and the Mac protocol all stay the same. This is not a
contract break, so the commit should be `feat(capture): ...` without `!`. Do not edit
`Cargo.toml`'s version or `CHANGELOG.md`
([[decisions:bob-cli-releases-ride-a-daily-train]]).

## Implementation steps

### 1. Rewrite logic (`src/native/capture_language/rewrite.rs`)

- Add a small helper, e.g.
  `fn alias_item_is_routed(raw_text: &str, start: usize, cursor: usize) -> bool`, next
  to `expand_dot_route_alias`. It implements the three steps above under the chosen
  `routed_scope`. Give it a doc comment that states the rule and why the pre-key draft
  is parsed.
- In `expand_sigil_alias`'s `alias == "."` branch, keep `expand_dot_block_id_alias`
  first. Then, if `alias_item_is_routed(...)`, return
  `expand_dot_dependency_alias(..).or_else(|| expand_dot_route_alias(..))`. Otherwise
  keep `expand_dot_route_alias(..).or_else(|| expand_dot_dependency_alias(..))`. Only
  compute the helper after the block-ID alias has declined.
- Update the doc comments that describe the old fixed order so they state the new
  precedence:
  - the comment block above `expand_sigil_alias` ("A spaced `.` that did not become `^`
    or `@` aliases `&`...")
  - `expand_dot_route_alias`
  - `expand_dot_dependency_alias` ("where neither `^` nor `@` applies")
  - the summary comment above `collapse_global_destination` that mentions "a spaced `.`
    aliases `@` only where..."

### 2. Unit tests (`src/native/capture_language/tests/rewrite.rs`)

Use the existing `assert_modifier_alias` helper, which checks rule, text, cursor, the
single one-byte edit, the summary, and idempotence.

- New `rewrite_draft_dot_prefers_dependency_once_item_is_routed`, where `.` → `&` for:
  - `Buy milk @foo:bar .`, `Buy milk @foo^bar .`, `Buy milk @foo+bar .`
  - `Buy milk @foo:bar\t.`
  - `Buy milk @foo:bar@ .`, `Buy milk @foo:bar#DEEP .`
  - `Buy milk @work+a @dev:b .` (existing run)
  - `Buy milk @foo:bar &x:y .`, `Buy milk @foo:bar #type .`
  - `- [ ] Task @work:a .`
  - `Buy milk @foo:bar\n- detail .`, `Buy milk @foo:bar\n- detail\n  - nested .`
  - `First\n\nBuy milk @foo:bar .`
  - `café 🚀 @Cash:Goog-Exit .`
  - Under `any_route` also: `Buy milk @foo .`, `Buy milk @foo#sec .`.
- New `rewrite_draft_dot_keeps_route_alias_when_item_is_not_routed_or_and_cannot_open`,
  where `.` → `@` still:
  - `Buy milk .`, `@@sase Buy .`, `Buy milk @foo:bar some text .`
  - `Buy milk @foo:bar\n\nFix it .` (a different item is routed)
  - `@a+x > .`, `@a+x > @c+tid .`, `Create task < @cash:line .`, `Create task < .`
  - Under `block_id` also: `Buy milk @foo .`, `Buy milk @foo#sec .`.
- The existing `rewrite_draft_expands_dot_route_alias`,
  `rewrite_draft_expands_dot_dependency_alias`,
  `rewrite_draft_modifier_aliases_keep_existing_alias_precedence`, and the
  literal-position tests must keep passing unchanged. If one of them fails, stop and
  find out why; do not edit its expectations to match. Glued `@errands.` → `^` and
  `@cash^goog-exit@\n- .` → `- ^` must not change.

### 3. Tip tests (`src/native/capture_language/tests/alias_tip.rs`)

- Add a positive row to `sigil_alias_tip_proves_each_positive_row`:
  `("Buy milk @foo:bar &", AliasId::Dependency, &["."], '&', Some("once routed"))`.
- Add `Buy milk @foo:bar @` to the negative corpus in
  `sigil_alias_tip_stays_silent_for_the_negative_corpus`. This is what proves that `@`
  is no longer taught there. If that corpus has no slot for it, write a focused
  assertion that `tip_at_end` gives `None`.
- `sigil_alias_tip_agrees_with_the_forward_rewrite_for_every_prefix` must stay green.

### 4. CLI tests (`tests/cli/capture/rewrite.rs`)

- Add `capture_rewrite_json_dot_prefers_dependency_once_routed`, modeled on
  `capture_rewrite_json_expands_a_dot_dependency_alias`, for `Buy milk @foo:bar .`.
  Assert:
  - text `Buy milk @foo:bar &`
  - the single edit at the `.` byte
  - summary `Typed & for .`
  - `alias.id == "dependency"`
- Add `("Buy milk @foo:bar &", "dependency", vec!["."], Some("once routed"))` to the
  JSON tip table near the existing `("@home &", "dependency", ...)` row.

### 5. CLI help and docs (bob-cli)

- `src/native/capture_rewrite.rs`: in the "Sigil alias" help paragraph, after "A spaced
  '.' that does not become '@' becomes '&' ...", add that once the item already owns a
  route marker (per `routed_scope`), a spaced `.` tries `&` first and falls back to `@`
  only where `&` cannot open. Add the example
  `'Buy milk @foo:bar .' becomes 'Buy milk @foo:bar &'`. Keep the existing examples.
- `docs/capture.md`, "**Sigil alias.**" paragraph (around the "A spaced `.` that does
  not become `@` becomes `&`" sentence): make the same change in Markdown style, with
  the `Buy milk @foo:bar .` → `Buy milk @foo:bar &` example. Also say that a later
  blank-line item, `@@route` declarations, and transfer/merge-in slots keep `@`.
- `docs/reference/capture-syntax.md` (~line 381) only says "spaced `.` / `;` → route or
  dependency pickers". That stays true, so leave it.
- Run `just docs-gen`. It regenerates `docs/reference/cli/capture-rewrite.md` from the
  clap help and, if memory changed in step 6, `docs/reference/glossary.md` from the
  memory web. Never hand-edit either generated page. Then run `just docs-check`. Read
  the `docs` reference memory (`sase memory read docs.md`) before touching docs.

### 6. Memory

> [!decision] memory_glossary_capture_alias

Use `/sase_memory_write`. In the `glossary:capture-alias` strand, rewrite the two
spaced-`.` bullets so they agree with the new rule. Keep each bullet short.

- The `.` → `@` bullet: "wherever a bare `@` opens route completion" must gain "and the
  item is not already routed".
- The `.` → `&` / `;` → `*` bullet: replace "and no `^`, `@`, or `:` alias applies" with
  the new precedence. Once the item is routed, `.` gives `&` wherever Depends On opens.
  Examples: `Buy milk @foo:bar .` → `Buy milk @foo:bar &`, while `Buy milk .` still
  gives `@`.

Then run `sase memory init` and `just docs-gen`. If the decision is declined and no
human reviewed the plan, file a `memory` task bead through `/sase_new_task` instead.

### 7. Bob Mac Capture docs (linked repo `bob-mac-capture`, docs only)

Open the repo with `/sase_repo` (`sase repo open bob-mac-capture -r "..."`), use only
the path it prints, and read its `AGENTS.md`. These three passages describe the old
fixed `@`-then-`&` order and become false:

- `docs/relationship-pickers.md` (~lines 53–56): "`.` aliases `&` only where Bob opens
  Depends On and no earlier `^`/`@` alias applies (`@home .` → `@home &`; `Buy milk .`
  stays `@`)". Add the routed-item precedence and the `Buy milk @foo:bar .` example.
- `docs/keyboard.md` (~line 99, the `.` row): "`.` → `@` for route completion or falls
  through to `&` where Bob opens Depends On". Add "and goes to `&` first once the item
  is routed".
- `docs/capture-panel.md` (~lines 233–236): "`.` falls from `@` to `&` where Bob opens
  Depends On". Same correction.

Leave `docs/typing-tips.md` ("once routed") and `docs/protocol.md` as they are; they
stay accurate. Make no Swift, fixture, or `fake-bob` changes: the app applies bob's edit
verbatim and its tests only cover `@home .`. Run `just docs-check` in that repo. That
repo's commit becomes part of the final declaration. After it is pushed, follow its
`AGENTS.md`: watch the exact-SHA macOS CI run to green with `/sase_monitor`.

## Verification

- `just check` (or at least `just fmt`, `just lint`, `just test`) in bob-cli: all green.
- Manual probes against the freshly built binary
  (`cargo run -q -- capture-rewrite -c <len> -f json -- '<draft>'`):
  - `Buy milk @foo:bar .` → `Buy milk @foo:bar &`, rule `expand_sigil_alias`, alias
    `dependency`
  - `Buy milk .` → `@`
  - `@a+x > .` → `@`
  - `Create task < @cash:line .` → `@`
  - `@errands.` → `@errands^`
  - `Buy milk @foo:bar\n\nFix it .` → second item `@`
  - `Buy milk @foo .` → `&` under `any_route`, `@` under `block_id`
- `just docs-check` passes in both repos, and the generated
  `docs/reference/cli/capture-rewrite.md` diff is only the new sentence.

## Out of scope

- The `;` key already gives `*` after routed items, and `,` already gives `+`; neither
  changes.
- No new rewrite rule, alias id, tip phrase, or JSON field.
- No change to the order of glued-separator toggles or `@@route` collapse.
