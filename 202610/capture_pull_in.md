---
tier: tale
title: Wire bob capture pull-in offers
goal:
  bob capture links only the successor offers named by -p/--pull-in, refuses a missing
  Mac choice without writing, and reports numbered offers on every success.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-7c.3
bead: bob-cli-7c.3
create_time: 2026-10-13 12:22:53
status: wip
---

- **PARENT:**
  [202610/successor_offers.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/successor_offers.md)
- **BEAD:**
  [bob-cli-7c.3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-7c/bob-cli-7c.3.md)

# Plan: Wire bob capture pull-in offers

Implement phase `bob-cli-7c.3` (`rust_capture`) from epic plan
`plan:202610/successor_offers.md`. The contract in `docs/task-dependencies.md` §12.5,
§12.12, and §12.14 is binding. Do not edit memory, `docs/task-dependencies.md`, or any
ancestor bead. The epic decisions `memory_offers_record` and `memory_glossary_offer`
belong to the later decision-record phase.

One coding agent can finish this. `plan_successors` already classifies offers and
applies `SuccessorSelection::{None, All, Keys}`. Capture hardcodes
`SuccessorSelection::None` in two places. This tale threads a CLI selection through
those places, numbers the draft-level offer list, and covers it with integration tests.

## Current code

- `src/native/task_complete/successors.rs` owns selection. `apply_selection` picks
  `All`, nothing, or an explicit key set. Unknown keys are ignored. `number_offers`
  assigns a 1-based `n` per `plan_successors` call using `offer_order_key` (group,
  anchor entry and bullet, note path, task line). `OfferReport` is copied onto
  `UnblockedRow.offer`. `SuccessorPlan.project_folds` holds hidden, already-planned, and
  sub-project counts. None of that reaches capture JSON.
- `plan_task_complete_item` in `src/native/capture/task_complete.rs` passes
  `SuccessorSelection::None` (the `plan_successors` call near the `link_unblocked`
  field).
- `plan_close_successors` in `src/native/capture/pomodoro_close.rs` does the same. The
  three call sites (plain close, link-form close, and the `^route:id=x` close) all go
  through that function. Thread the selection once, inside the function.
- `apply_successor_net_post_pass` drops a linked row whose bullet or task did not
  survive, and an unlinked `project_empty` row whose task is no longer open. An unlinked
  `dependency` offer is kept even when a later item completes it. §11.6 SL20 says draft
  `!P` then `!D` drops the first item's offer for D.
  `tests/cli/capture/successor_integration.rs::close_successor_then_consume_net` still
  expects that row to remain; its comment already describes the drop.
- `TaskCompleteUnblockedJson` has no `offer`. `CaptureResult` has no top-level
  `pull_in`. `print_capture_error` emits `ok`, `error`, and an optional `code`.
- `bob capture` flags are registered in alphabetical long-name order in
  `src/native/capture/cli.rs`. `-p` is free. `--no-ref` is `-R`. Public long options
  need a short alias; internal subprocess arguments do not (`sase/memory/cli_rules.md`).
  Never add a `bob capture` subcommand.
- `capture` in `src/native/capture/mod.rs` plans, optionally commits, then builds
  `CaptureResult`. A plan that is not committed writes nothing. Ref-job kick happens
  only after a real commit.
- The `!` long-help paragraph still says fully unblocked dependents are linked into the
  predecessor's slot. That is no longer true.

## CLI

In `src/native/capture/cli.rs`, add two arguments and register them in long-name order
between `--no-ref` and `--route`:

- `-p, --pull-in <PICKS>`. Help: pull in successor offers, as `all`, `none`, or a comma
  list of offer numbers and keys. Say that `p:<N>` in the capture text is still
  priority, not this flag.
- Hidden `--require-pull-in-choice`, `ArgAction::SetTrue`, no short alias, `hide(true)`.

`PICKS` parsing, rejected as `CaptureError::usage` (exit 2) before any plan:

- `all` and `none` are whole-value keywords, case-sensitive.
- Otherwise split on commas, trim, and reject an empty token.
- A token of only digits is an offer number (`0` is out of range later, not a syntax
  error). Anything else is a key (`note.md#^id`, `note.md@<8 hex>`, optional `~k`).

Store the flag and the hidden bool on `CaptureRequest`. Update the struct literal in
`src/native/capture/task_blocks.rs` (`dry_request`). Absence of `--pull-in` is different
from `--pull-in none`: only the latter reports `choice: "none"`.

Rewrite the `!` long-help paragraph so recovery stays automatic and a link is created
only for a `--pull-in` pick. Add two examples: `bob capture -p 1 -- '!sase:fix'` and
`bob capture -p all -- '!sase:fix'`. Keep `-h` free of the hidden flag.

In `tests/cli/help_options.rs`, extend `capture_help_lists_options_alphabetically` so
`-p, --pull-in` appears after `-n, --no-clip` and before `-r, --route`, and assert the
help text does not contain `--require-pull-in-choice`.

## Selection and passes

Resolve the flag inside `capture()` before commit. `plan_capture_batch` reads the
selection from `CaptureRequest` (add a `selection: SuccessorSelection` field that
defaults to `None` for the task-block unit tests). Both successor call sites pass
`request.selection` into `SuccessorInput`.

`plan_successors` stays as it is. Capture rejects a pick the engine would ignore, and it
rejects that pick before commit.

Passes:

- No `--pull-in`. Plan once with `None`. If `--require-pull-in-choice` is set and the
  net offer list is non-empty, return the refusal below and do not commit. If the list
  is empty, commit that same plan (the flag is a no-op, SO29).
- `--pull-in all` or `--pull-in none`. Plan once with `All` or `None`. Zero offers is a
  success no-op. Commit.
- Key tokens only. Plan once with `Keys`. Before commit, every requested key must be on
  a surviving offer. A miss is the teaching error below; do not commit.
- Any number, alone or mixed with keys. Plan once with `None` and do not commit. Map
  each number through that probe's draft numbering. An out-of-range number is the
  teaching error. Then plan again with `Keys` of the mapped keys plus any explicit keys,
  and apply the same pre-commit key check.

A probe uses a fresh planner. Discard it by not committing. Do not kick ref jobs for it.

Draft numbering happens after `apply_successor_net_post_pass`, on surviving rows that
have an offer:

- Add the `offer_order_key` tuple to `OfferReport` as crate-private data. It is not a
  JSON field. Copy it onto a side channel on the planned item (item index, row index,
  key, order tuple).
- Sort survivors by that tuple, then by item index. Assign `n` as the 1-based position.
  A one-item capture keeps the engine's `n`.
- When a number probe ran, the committing result keeps the probe's `key → n` map, gaps
  included. `-p 1` still names the row the probe called 1. A committing-only offer,
  which should not occur, takes the next number after the probe's maximum, in the same
  order.

`choice` is JSON `null` when the flag is absent, `"all"`, `"none"`, or `"subset"` for
any explicit list (even a list that names every offer). `picked_count` is how many
surviving offers have `selected: true`. `offer_count` is the surviving offer rows.

## Guard and teaching error

`--require-pull-in-choice` fires only when the flag is set, `--pull-in` is absent, and
`offer_count > 0`. Exit 1. Write nothing (compare vault bytes in the test). JSON,
produced by extending `print_capture_error` with an optional extra object merged into
the error value:

```json
{
  "ok": false,
  "code": "pull_in_choice_required",
  "error": "this capture has successor offers; pass --pull-in",
  "pull_in": { "supported": true, "offer_count": 2, "choice": null, "picked_count": 0 },
  "offers": []
}
```

`pull_in` matches the success object, including `groups` when the success object would
include them. `offers` is the surviving offered `unblocked` rows, same schema as
success, so Bob Mac Capture can open its choice step from this reply. Do not put muted
`still_blocked` rows in `offers`. Human output prints the error to stderr and the same
numbered offer rows a success would print.

A stale, unknown, or out-of-range pick is `CaptureError::usage` (exit 2), no `code`
(only `plan_theme_cap_exceeded` and `pull_in_choice_required` carry `code`). The message
names each bad token and lists the current offers, one `N <key> <text>` line each. Write
nothing.

## Net pass

Extend `apply_successor_net_post_pass` so an **offer** row (one that will serialize
`offer`) is removed from that item's `unblocked[]` when, in the final staged text, any
of these is true:

- The task is no longer open. This is SL20 (`!P` then `!D`) and a later completion.
- A later item linked it, and the live bullet is not the link this row itself inserted.
  This row's own picked link stays.
- A later item re-blocked it: the final status is `?` while this row reported a recovery
  to Ready or Next.
- A later item dropped it with `=x~K`. Same-item `=x~K` stays the engine's
  `already_planned` result (SL27).
- It is a `project_finished` offer (`group` is `project_finished`, trigger
  `project_empty`) and a later item left N with open work again, so the emptied-project
  predicate no longer holds.

Rows that are not offers (`disabled`, `hidden`, `project_task`, `already_planned`,
`no_destination`, `project_member`) stay. `surviving_successor_links` already runs after
this pass; picked links that survive keep pomodoro-block `reason: "unblocked"`. Do not
invent a second task-block role path. `push_unblocked_ref` may still record a role
before the net pass; when `close_successor_then_consume_net` is updated, drop the first
item's D offer from JSON and keep D from being reported as only `["unblocked"]` if the
completed role replaces it. E remains the later offer.

## JSON

Add optional `offer` on `TaskCompleteUnblockedJson`, skipped when absent. Both builders
(`task_complete.rs` and `pomodoro_close.rs`) fill it from `OfferReport` for offered rows
only, including unpicked ones and stale `[ ]` residents whose status did not change:

- `key`, `n`, `group` (`hand_off` | `project` | `project_finished`), `selected`,
  `block_id_needed`
- `project`: vault-relative path, present only for the `project` group
- `destination`: `day_file`, `entry_name`, `entry_role` (`slot` | `closing` | `implicit`
  | `created`), `fallback`

`n` on the struct is overwritten by the draft numbering above. Serde still omits
`not_linked` when the row was linked.

Add top-level `pull_in` on `CaptureResult`, present on every success, including dry-run
and captures with zero offers:

```json
"pull_in": {
  "supported": true,
  "offer_count": 2,
  "choice": null,
  "picked_count": 0,
  "groups": [
    {"group": "project", "project": "sase_foo.md",
     "folded": {"hidden": 0, "already_planned": 0, "sub_projects": 0}}
  ]
}
```

`supported` is always true. Omit `groups` when there is no project group (an empty list
skips). Include a project group even when every folded count is zero. Build `groups`
from `SuccessorPlan.project_folds`, threaded out of both planners onto a non-serialized
item field and aggregated by project path. `choice` is null without the flag.

Carry `project_folds` by extending `CloseSuccessorOutcome` and the task-complete item
path. `CaptureResult::from_items` takes the assembled `pull_in`.

## Human output

Keep the existing `Styler` and the existing `not_linked` phrases (`not pulled in`,
`no open Pomodoro`, `project member`, `linking off`). No new color tokens.

Offered rows print under group headers, in `n` order, header once per group:
`Hand-offs`, `Project <vault-relative note>`, `Project finished`. Each offered row
starts with `offer N`. A selected row uses today's `linked` / `would link` line. An
unselected offer uses the recovered line with `· not pulled in`. Non-offer rows and
`still_blocked` rows stay in their current shape. Muted project residents stay
`still blocked` lines; they are not numbered.

After the offer list, when `offer_count > 0` and nothing was picked:

- Dry-run hint: `-p 1,2 · -p all` using the real numbers (or `-p 1` when there is one).
- Real run without a pick: the offers were not pulled in, and the way to link later is
  Obsidian, Bob Mac Capture, or Ctrl+Shift+Enter.

When at least one offer was picked, the task-complete summary and the pomodoro-close
`next:` line include `pulled in K of N`.

## Tests

Add `tests/cli/capture/successor_offers.rs` and register it in
`tests/cli/capture/mod.rs`. Use the existing vault helpers (`BOB_NOW`, `BOB_DAY_FILE`,
`-f json`). Cite the vector id in a comment on each test.

- **SO1 / `^bar` → `sase_foo`.** Project note `sase_foo.md` whose `^prj` depends only on
  `sase.md` `^bar` and is linked under an open Pomodoro. Two open residents, one `[?]`
  and one `[ ]`, one of them missing a block id, `#ref` allowed on one. Dry-run: both
  residents are offers in group `project`, `^prj` is not a numbered offer
  (`project_task` or the group header only), `pull_in.offer_count` is 2, `choice` is
  null. `-p 1` mints the missing id, sets `[*]`, writes `[fresh:: today]`, clears
  `[unblocked::]`, and inserts the link under `^bar`'s entry. The other resident
  recovers, `not_selected`, and is not minted.
- **SO2 / SO32.** 11-way and 20-way fan-out from one gesture. That many `hand_off`
  offers, `selected: false`, no breaker field, no cap.
- **SO3 / SO4 / SO5.** `-p all` links, raises, mints, and stamps every offer. `-p none`
  recovers, mints nothing, creates no successor-only placeholder, and reports
  `not_selected`. A subset by numbers and the same subset by keys links only those rows.
- **SO6.** A stale key, an unknown key, and an out-of-range number each exit 2, leave
  the vault byte-identical, and list the current offers.
- **SO13.** A later item that completes, links, re-blocks, or `=x~K`-drops an offered
  task removes that offer. A later item that adds open work to N removes N's
  `project_finished` offer.
- **SO14 / SO15 / SO16.** One open timed entry (else the first open entry): offer with
  `entry_role: implicit` and `fallback: true`; `-p all` links there. No day file, no
  `## Pomodoros`, or no open entry: recover, `no_destination`, no offer. Two open timed
  entries: the same `no_destination` result.
- **SO17 / SO18.** Unpicked emptied-project pull-in removes `#hide`, does not link,
  reports `not_selected` and `project_finished`. Unpicked parent-project pull-in
  (trigger c) does the same. Picked, it links immediately after K's `^prj` bullet.
- **SO19.** `plan.link_unblocked: false` yields `disabled`, `offer_count: 0`, and no
  links. An emptied `^prj` still loses `#hide`. `-p all` does not override the switch.
- **SO26.** Human output of a close with offers and no flag contains `offer 1` and a
  group header, exits 0 without reading stdin, and does not link.
- **SO27 / SO29.** `--require-pull-in-choice` with offers and no `--pull-in` is the
  write-free exit-1 refusal, and the body includes `pull_in` and `offers`. With zero
  offers the flag commits. `-p all` and `-p none` with zero offers succeed.
- **Dry-run parity.** On identical vault copies, dry-run JSON and real-run JSON with the
  same `--pull-in` flags match on `pull_in`, `unblocked`, and `still_blocked`.
- **No successors.** A capture that offers nothing still has `pull_in.supported: true`,
  `offer_count: 0`, and `choice: null`.

Restore picked-link expectations by passing `-p all` on the vectors that say "when
picked". Update the assertions to the picked outcome (status `*`, `link` present,
`not_linked` omitted, day-file bullet present). Do not leave `not_selected` assertions
on those picked tests.

- `tests/cli/capture/task_complete.rs`: SL1, SL2, SL9, SL10, SL14. SL9's fixture has one
  open timed entry, so the pick links at the implicit destination (`fallback: true`), it
  does not "link nothing". SL10 mints and sets `block_id_created`. SL14 expects the
  ambiguous `[[dir/note#^id]]` form the engine emits. SL16, SL18, and SL19 also pass
  `-p all` and still do not link (empty gate, `disabled`, already Done).
- `tests/cli/capture/project_pull_in.rs` and the PP29 case in
  `pomodoro_close_successors.rs`: every PP vector whose outcome is a link when picked.
  Leave already-planned, disabled, still-blocked, and net-drop cases on their current
  assertions. `capture_json` in these files takes the extra `-p` `all` args before the
  capture text.
- `tests/cli/capture/project_close.rs` PDC2 (`pitch` is a project-resident offer). With
  `-p all`, pitch is linked, raised, stamped `[fresh:: today]`, and loses
  `[unblocked::]`. `^prj` stays `project_task`. Other PDC tests stay unpicked unless
  their vector says the row is linked when picked.

Update `close_successor_then_consume_net` so the first item no longer reports D's offer.
Fix any full-document JSON equality that rejects the new top-level `pull_in`. Field
asserts on unrelated captures should keep passing because `offer` is omitted when
absent.

Grep tests for `--no-ref` and capture help snapshots, and update any completion or help
fixture that lists public capture options so `-p` is present and the hidden flag is
absent.

## Verification and close

Run `just check` (fmt, clippy, full `cargo test`) until green.

Then:

1. `sase bead epic-symbols bob-cli-7c.3`. There is no `--epic-symbol` reference today.
   If any remain, re-key each Justfile line to a still-open bead (the parent epic
   `bob-cli-7c` or a later phase). `sase bead close` refuses while leftovers remain.
2. Close only `bob-cli-7c.3` with
   `sase bead close bob-cli-7c.3 --note "<what you verified>"`. Do not close
   `bob-cli-7c` or any ancestor. Do not create beads. Record extra work as
   `sase bead note bob-cli-7c.3 'PROPOSED FOLLOW-UP: <one-line summary — detail>'`.
3. A `just check` failure that reproduces on the clean base tree does not keep the phase
   open. Note it as a `PROPOSED FOLLOW-UP` (cite any task bead that already tracks it)
   and close this phase anyway.

## Out of scope

Obsidian picker, Bob Mac Capture UI, the decision-record memory edits, an interactive
TTY picker, a per-row destination override, and immediate sub-project recovery.
