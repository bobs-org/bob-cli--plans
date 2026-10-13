---
tier: tale
title: Rust planner emits successor offers
goal:
  Every would-be successor link becomes a keyed offer, and only a selection links,
  mints, raises to Next, and stamps freshness.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-7c.2
bead: bob-cli-7c.2
create_time: 2026-10-13 10:49:01
status: wip
---

- **PARENT:**
  [202610/successor_offers.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/successor_offers.md)
- **BEAD:** bob-cli-7c.2

# Plan: Rust planner emits successor offers

Implement phase `bob-cli-7c.2` (`rust_offers`) in bob-cli. The contract in
`docs/task-dependencies.md` §12, §11.6, §12.10–§12.12 is already written and is binding.
Epic `bob-cli-7c` stays open. Its later phase `rust_capture` (`bob-cli-7c.3`) adds
`-p, --pull-in`, the `offer` JSON object, the top-level `pull_in` object, numbered offer
rows, and batch renumbering. This tale leaves capture compiling by passing an empty
selection from the existing call sites.

Read these before editing:

- `docs/task-dependencies.md` §12.2–§12.5 and §12.10, plus the SL, PP, and SO tables
  (§11.6, §12.11, §12.12).
- `src/native/task_complete/successors.rs` (`plan_successors`, `apply_verdicts`,
  `insert_successor_links`).
- `src/native/task_complete/project_close.rs` (`recover_project_members`).
- `src/native/task_complete/project_pull_in.rs` (`classify_pull_in`, `emit_rows`).
- `src/native/capture_task_toggle/links.rs` (`select_implicit_open_entry`).
- `src/native/freshness/placement.rs` (`stamp_fresh`).

## Settled interpretations

`plan.link_unblocked: false` produces no offers of any group. The sequential ladder
lists the implicit-destination step before the kill switch, and SO19 plus the §12 shared
rule say the kill switch recovers with `not_linked: disabled` and emits no offers. Check
it before creating a `hand_off`, `project`, or `project_finished` offer. An emptied
`^prj` still loses `#hide` (PP11, SO19).

Rust does not plan cancelled closes. Leave that to the cycler.

`project_member` is no longer the resident outcome. A resident that is offered and
unpicked reports `not_selected`. A resident suppressed by the kill switch reports
`disabled`. Hidden and already-planned residents keep `hidden` and `already_planned`.
Keep the existing human-copy arm for `project_member`.

Unknown selection keys are ignored inside the planner. The teaching error that writes
nothing is `rust_capture`'s job. This tale's capture call sites pass an empty selection,
so that error is unreachable from the CLI.

## Out of scope

Leave these to `bob-cli-7c.3` and the later epic phases:

- `-p` / `--pull-in`, hidden `--require-pull-in-choice`, and any clap or help-text
  change.
- Serializing `offer` onto `TaskCompleteUnblockedJson`, the top-level `pull_in` object,
  folded-count JSON, and numbered `offer N` rows under group headers.
- Batch net-pass rules for later-item invalidation (SO12, SO13, SL20, PP21, PP35). If an
  existing net-pass test fails only because a row that used to be linked is now an
  unpicked offer, update that assertion. Do not add new net-pass rules.
- Obsidian, Bob Mac Capture, and memory. The epic's accepted memory decisions belong to
  the later `decision_record` phase. This tale edits no file under `sase/memory/`.
- Docs. The contract phase already landed them.
- Embedded-completion caps (25 levels / 250 targets).

## Types

In `src/native/task_complete/successors.rs`:

- Add an owned selection enum on `SuccessorInput`: `None`, `All`, and
  `Keys(BTreeSet<String>)`. `None` picks nothing.
- Delete `MAX_SUCCESSORS_PER_GESTURE` and the breaker block in `plan_successors`.
- Give `DependentVerdict` an optional offer draft: group (`hand_off` | `project` |
  `project_finished`), optional project path, destination, key, `block_id_needed`, and
  `selected`. `SlotAnchor` stays private to this file. `project_close` returns a
  classification and this file attaches the anchor.
- Add the same offer description to `UnblockedRow` (`key`, 1-based `n`, `group`,
  optional `project`, `destination` with `day_file`, `entry_name`, `entry_role` of
  `slot` | `closing` | `implicit` | `created`, `fallback`, `selected`,
  `block_id_needed`). Capture JSON does not grow a field in this tale; the engine row is
  what `rust_capture` will serialize.
- Add `project_folds` on `SuccessorPlan`: one record per project whose `^prj` entered
  project offers, with `project`, `hidden`, `already_planned`, and `sub_projects`.
  Include the record when the group exists, including when every selectable count is
  zero and the only signal is a sub-project count.

Update the module doc. Picked rows stamp freshness through `stamp_fresh`. Unpicked
recoveries still use the `[unblocked::]` WRITE rule and do not stamp fresh.

On `ProjectSurface` (`src/native/projects/surface.rs`), add `open_subprojects: usize`,
counted with the same open-marker predicate that already sets `subopen`
(`subproject_displays_in_marker_line`, ignoring a `closed_child` stem). Do not walk
child notes.

## Hand-off ladder

Keep steps 1–6 of §12.2 as they are (`waits_on`, `scheduled`, `project`, `project_task`,
`hidden`, `already_planned`). Then:

1. If `link_unblocked` is false, `Recover("disabled")` and no offer.
2. If no predecessor in C has a §12.3 anchor, resolve the implicit destination. Success
   becomes a `hand_off` offer with `entry_role: implicit` and `fallback: true`. Failure
   becomes `Recover("no_destination")` and no offer.
3. Otherwise a `hand_off` offer at `earliest_anchor`, `entry_role` `slot` or, in closing
   mode, `closing` / `created`, `fallback: false`.

Implicit destination is `capture_task_toggle::links::select_implicit_open_entry` over
`capture_pomodoros::scan` of the insertion base (`post_day`, else `pre_day`). That is
the plain `@route:id` rule: the single open timed entry, else the first open entry.
`NoPomodorosSection`, `MultipleOpenTimedEntries`, `NoEligibleOpenEntry`, or a missing
base text is `no_destination`. Do not write a second resolver and do not create a
placeholder when resolution fails.

Insert an implicit pick as an entry child: one tab, after the entry's existing children,
replacing a lone stub. Reuse the closing-target insertion path
(`SuccessorTarget::Closing` with `entry_index: Some(...)` and `created_name: None`) so
`entry_created` stays false. Store `entry_role: implicit` on the offer; do not infer it
from the target variant. Several implicit picks append in placement order.

`n` follows §12.2 ORDER: group (`hand_off`, then `project`, then `project_finished`),
then anchor ledger position (the implicit entry's position when that is the
destination), then note path, then line. Minting and insertion keep today's placement
order (anchor ledger, note path, line) among picked rows only.

## Selection and writes

Apply selection after classification, project offers, and implicit resolution, and
before minting:

- `All` picks every offer. `Keys` picks matching keys. `None` picks none.
- A picked offer keeps today's link path: `successor_new_status` (`' '` / `'?'` → `'*'`;
  `'*'` and `'/'` unchanged), mint in placement order via the existing `mint_block_id` /
  `clean_description` path, `insert_successor_links`, and strip `#prj_is_blocked`. Then
  `freshness::placement::stamp_fresh` on that line, which writes `[fresh:: today]` and
  clears `[unblocked::]` and `[keeps::]`. Do not call `apply_recovery_writes` or
  `plan_task_link` for a pick.
- An unpicked offer takes the recover path with `not_linked: not_selected` and the
  `[unblocked::]` WRITE rule. It does not mint and it inserts nothing. The offer draft
  stays on the verdict with `selected: false`.
- `^prj` is never minted.
- `block_id_needed` is true when the task has no block id before mint.

`emit_rows` must emit an `unblocked` row for every offer, picked or not, even when the
status is unchanged (SO8, a stale `[ ]` resident). The current recover loop skips
non-`'?'` rows; an unpicked offer bypasses that skip. `link` is set only for a picked
row. `not_linked` is omitted on a picked row and is `not_selected` on an unpicked offer.
Recovered non-offers keep today's "emit when the status changed" rule, and pull-ins also
emit when `#hide` was removed.

## Project offers

Replace the recover-only cascade in `recover_project_members`. When a `^prj` R recovers
and the project state has no own or inherited reason left, enumerate every open resident
(`is_resident`: `[?]`, `[ ]`, `[*]`, `[/]`, `#ref`, nested; not `^prj`; not fenced).
Drop the `task.status != '?'` filter. Pass R's predecessor indexes back so
`successors.rs` can call `earliest_anchor`. Also run this cascade when a §12.10 pull-in
R was `[?]` and leaves Blocked through steps 5–9.

For each resident:

- Own open prerequisite or future date: muted `still_blocked` row with that reason and
  `project` set to N. Do not replace an existing still-blocked or `hand_off` verdict.
- `#hide`: recover only, `hidden`, increment `hidden`. No offer.
- Live-linked on the pre-gesture day: recover to the derived rank, `already_planned`,
  increment `already_planned`. No offer.
- Otherwise: recover and offer, group `project`. Destination is
  `earliest_anchor(preds(R))`, or the implicit destination with `fallback: true` when
  that anchor is empty. If implicit resolution fails, recover with `no_destination` and
  no offer.
- A resident that already has a direct-dependent `hand_off` offer stays one `hand_off`
  row (SO7). Do not downgrade it to `project_member`.

R itself is the group header, not a choice, while N has open work. Do not put R in the
`project` offer set. Its own `project_task` (or `project_finished`) verdict stands.

`sub_projects` is `ProjectSurface.open_subprojects`. Descendants stay untouched (PDC5 /
PDC6). Store the count on the fold record. Do not invent a synthetic task row for the
muted "N sub-projects stay blocked" line.

Keep the existing `#prj_is_blocked` strip for every resident of a recovered project,
including muted and folded rows. PDC3 stays: a resident with its own open prerequisite
or future date remains blocked, now as a muted `still_blocked` row.

## Project-finished offers

In `classify_pull_in`, steps 1–5 stay (`waits_on`, `scheduled`, inherited `project`,
unconditional `#hide` removal from step 4 on, `already_planned`). Then:

- `link_unblocked == false` → `Recover("disabled")`, `#hide` still removed, no offer
  (PP11).
- No anchor → the shared implicit-destination rule, group `project_finished`. Failure →
  `Recover("no_destination")`, `#hide` removed, no offer (PP10).
- Otherwise a `project_finished` offer at `earliest_anchor(preds(R))`.

Picked pull-ins use the pick writes above and link. Unpicked pull-ins recover at the
recover rank (`'?'` → `'*'` only when a live link remains, else `' '`), write
`[unblocked::]` when the WRITE rule applies on the post-`#hide` line, and report
`not_selected`. A picked Next landing stamps fresh and does not write `[unblocked::]`
(PP27, UB4).

## Keys

Rust pins the key format:

- A row with a block id: `<note_path>#^<block_id>`.
- Otherwise: `<note_path>@<8 lowercase hex chars>` of SHA-256 over
  `note_tasks::clean_description` (global filter and block id). Use the existing `sha2`
  dependency. The k-th row in the same note with the same digest, in document order, k ≥
  2, gets a `~k` suffix. The first row has no suffix.
- `note_path` is the vault-relative path already stored on the file (`sase.md`,
  `sase_foo.md`), with `/` separators.

Only offers get keys. Muted and folded rows do not.

## Call sites and human copy

Pass `SuccessorSelection::None` from both `plan_successors` struct literals:
`src/native/capture/task_complete.rs` and `src/native/capture/pomodoro_close.rs`. Any
further literal added in the same edit does the same.

In `src/native/capture/output.rs` `not_linked_reason`, map `not_selected` to
`not pulled in` and `no_destination` to `no open Pomodoro`. Keep the existing arms,
including `breaker` and `not_planned_today`, so an old string still prints. New planner
output never emits those two reasons. Do not add numbered offer lines or group headers
here.

## Tests

Planner tests construct `SuccessorInput` in
`src/native/task_complete/tests/successor_tests.rs` and `project_pull_in_tests.rs`.
Thread `selection` through their helpers. Tests whose vector says "linked when picked"
pass `All`. Tests of an empty pick pass `None`. A subset test passes `Keys`.

Update existing vectors to the contract, citing the id in a comment:

- SL rows that expect a link pass `All` and still link.
- SL9: the current `day_with_fix_link` fixture has one open timed entry (`CAPTURE`) and
  one open untimed entry. With the predecessor's link removed, select-all links into
  `CAPTURE` as an implicit entry child (`fallback: true`). Add the two-open-timed and
  no-open-entry cases as `no_destination` with no offer and no day edit (SO15, SO16).
- Delete `sl17_breaker_links_none`. Replace it with SO2: an 11-way fan-out, `None` links
  nothing (`not_selected`, no mint, no day edit), `All` links all 11.
- SL18 / SO19: `link_unblocked: false` recovers `disabled`, no offers, no mints. A
  pull-in still removes `#hide`.
- Rewrite `pdc2_prj_recovery_recovers_residents_outside_breaker`. The six residents are
  `project` offers. `All` links them at `earliest_anchor(preds(R))` (that fixture links
  `done-p`), stamps `[fresh:: 2026-10-05]`, and does not write `[unblocked::]` on those
  picked lines. `^prj` stays `project_task`, recovers to Ready, and keeps the
  `[unblocked::]` WRITE. The outside dependent stays one `hand_off` row. There is no
  breaker.
- Rewrite `pdc3_resident_with_own_prerequisite_stays_blocked`. The sequenced resident is
  a muted `still_blocked` row (`waits_on`, `project: proj.md`) and loses
  `#prj_is_blocked`. The `[ ]` ready leftover is offered even though its status is
  unchanged until picked (SO8). That fixture's day links `fix-apollo`, not `done-p`, so
  the resident destination is implicit. Read the fixture before asserting a slot.

`src/native/task_complete/tests/recovery_tests.rs` does not call `plan_successors`.
Leave it unless a failure shows it observes this planner.

Add planner-level SO tests, each citing its id. Skip SO6, SO12, SO13, SO21, SO22, SO23,
SO25, SO26, SO27, and SO29; those are CLI, batch, walk, or picker behavior owned
elsewhere.

- SO1: close `sase.md` `^bar`; `sase_foo.md` `^prj` depended only on `^bar` and has two
  open residents. Both are `project` offers, `^prj` is not an offer, and picking one
  mints a missing id, sets `[*]`, writes `[fresh:: today]`, clears `[unblocked::]`, and
  links under `^bar`'s entry. The other recovers `not_selected` with no mint.
- SO3, SO4, SO5: all / none / a key subset.
- SO7: a resident that is also a direct dependent is one `hand_off` row.
- SO10, SO11, SO24: hidden and already-planned counts, and `open_subprojects` from the
  marker, with descendants unedited.
- SO14–SO16: implicit destination and both `no_destination` failures.
- SO17, SO18: emptied-project and parent-project (trigger c) pull-ins. Unpicked: `#hide`
  removed, no link, `not_selected`, `project_finished`. Picked: linked. PP29's fixture
  is the parent case.
- SO20: pick freshness bytes versus an unpicked Ready recovery's WRITE rule.

Retire the PP15 breaker test the same way as SL17: both the dependent and R are offers,
and `All` links both.

CLI integration tests that asserted an automatic successor link now see
`SuccessorSelection::None`. Update them so each such row is `not_linked: not_selected`,
has no `link`, does not mint, and does not gain a successor bullet. Known files:
`tests/cli/capture/task_complete.rs`, `tests/cli/capture/project_pull_in.rs`,
`tests/cli/capture/pomodoro_close_successors.rs`, and
`tests/cli/capture/successor_integration.rs`. Search capture CLI tests for
`not_planned_today`, `breaker`, and successor `link` assertions.

Re-read each fixture's day text before changing a string. A fixture with one open timed
entry, or one open entry and no timed entry, is an unpicked implicit offer
(`not_selected`, no link). Two open timed entries, no `## Pomodoros`, or no open entry
is `no_destination`. Do not rewrite assertions about the completed task's own Task Link
or a manual capture link.

Do not pass `--pull-in` to restore the old link assertions. `rust_capture` does that.

## Verification and finish

Format, then run the repo gate and wait for it:

```bash
cargo fmt
just check
```

`just check` is `cargo fmt --check`, `cargo clippy --all-targets --all-features` with
the justfile's rustc gate, and `cargo test --no-fail-fast`.

A failure that reproduces identically on the clean base tree does not keep the phase
open. Record it on the phase bead as `PROPOSED FOLLOW-UP:` and continue. Record any
other discovered follow-up the same way. Do not create beads.

Before closing, run `sase bead epic-symbols bob-cli-7c.2`. There were none when this
plan was written. If any `--epic-symbol` entries exist, re-key each Justfile line to a
still-open bead (the parent epic or a later phase). Then close only this phase:

```bash
sase bead close bob-cli-7c.2 --note "<what just check showed>"
```

Do not close `bob-cli-7c` or any ancestor.
