---
tier: tale
title: Finish and land the requirement close engine
goal:
  Fix the epic-caused requirement-close defects found at landing (overlap ancestry
  across hand-ticked rows, Pomodoro-close carry placement at the closing continuation,
  permanent dead-code suppressions, a weak test), then close bob-cli-on.8.1 and its
  parent phase bob-cli-on.8.
size: medium
proposed_by: bbugyi200.athena.bob-cli-on.8.1.land
proposed_with: claude/opus
bead: bob-cli-on.8.1
status: done
---

- **PARENT:**
  [202611/requirement_close_engine.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/requirement_close_engine.md)
- **BEAD:**
  [bob-cli-on.8.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-on/bob-cli-on.8.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-on.8.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-on.8.1.land.md)
- **COMMITS:**
  - [04adcf3](https://github.com/bobs-org/bob-cli--plans/commit/04adcf3c2e5368e1cff71509080e9b98c8106d8a)
    — docs(plans): mark requirement_close_engine done after bob-cli-on.8.1 land

# Finish and land the requirement close engine (bob-cli-on.8.1)

## Context

Epic **bob-cli-on.8.1** ("Requirements on the capture successor card", plan
`plan:202611/requirement_close_engine.md`, parent phase **bob-cli-on.8**) shipped all
six phases in commits `e4ee55bd`, `0a83bb1c`, `d5e31abc`, `bddf3868`, `7073b7c8`, and
`96a23f50`. The land review reproduced three defects that this epic caused and one weak
test. Fix them, then close the epic. This tale is the epic's landing: nothing resumes
the landing after you, so you finish the closeout at the end of this plan.

Rules that still apply. Edit no memory, plugin code, Mac code, live vault, Cargo
version, or CHANGELOG. Do not create task beads for anything in this plan. Use
`/sase_new_task` only for a genuinely distinct issue the epic did not cause. Run
`just check`, not `just check-full`. The contract of record is
`docs/task-dependencies.md` §12.25 (requirements on the successor card) and §12.26 (RV
vectors). The fixture is `tests/fixtures/req_close/vectors.json` plus its `README.md`.

## 1. Overlap and fold ancestry must cross non-Pending intermediate rows

This is the `PROPOSED FOLLOW-UP` from bob-cli-on.8.1.4. Two functions walk `parent_key`
only through Pending card rows: `validate_non_overlapping` and `ReqChoiceState::fold`,
both in `src/native/task_complete/req_verdicts.rs`. A hand-ticked `[x]` intermediate is
not a card row, so the walk stops there. Reproduce with this owner block:

```text
- [ ] #task Ship parser [id:: ship] ^ship
	- [ ] Edge cases
		- [x] Empty input
			- [ ] Zero bytes
	- [ ] Handles unicode
```

On `!sase:ship`, card row 1 is "Edge cases" and row 2 is "Zero bytes":

- `-X 1 -k 2` currently **commits both** writes: it drops "Edge cases" and carries "Zero
  bytes" out from under it. It must refuse as `req_verdict_overlap` (exit 2, overlap
  copy naming “Edge cases”) and leave every file byte-identical.
- `-k 1 -X 2` currently refuses as `req_close_stale` during the write pass. It must
  refuse as `req_verdict_overlap` before any write planning.

Fix:

- Walk ancestry through the owner's full pre-close forest, `ReqCloseOwner.nodes`, which
  holds every recognized node including met, dropped, and carried ones. `req_landing.rs`
  already does this. In `validate_non_overlapping`, the owner is in the `rows` index
  entries. Build or reuse a key → node map per owner and follow `parent_key` through it.
- Give `ReqChoiceState::fold` access to the owner's nodes, for example by taking
  `&ReqCloseOwner` beside the group, so a structural verdict on "Edge cases" hides "Zero
  bytes". Its existing `pending_descendants` count already includes that row. Update its
  callers and tests.
- A descendant **jump pin** under a structural ancestor stays allowed.
- Add an RV `choices` case, next free id `RV44`, that pins both verdict orders across a
  hand-ticked intermediate and the fold of the hidden descendant. Update the fixture
  README and the §12.26 inventory table: the `choices` row's case range and pins.
- Add a CLI regression to `tests/cli/capture/req_close.rs` covering both orders. Assert
  the error code, that stdout is the JSON error envelope, and byte-identical vault files
  via the existing `vault_bytes` helper.

## 2. A carry plus pull-in on a Pomodoro close must use the closing continuation

§12.25 says a carry plus pull-in links the new task "at the ordinary successor
destination". On a Pomodoro close, ordinary successors go to the **closing
continuation** (`ClosingContext` → `resolve_closing_target`, §12.3). The capture
requirement post-pass (`src/native/capture/req_choice.rs::plan_req_post_pass`, called
from `src/native/capture/plan.rs` after `plan_offer_verdict_post_pass`) always passes
`closing: None`. So `task_complete::req_placement::closing_after_successors` is never
called outside tests, and the RV "closing continuation" placement vector has no
production caller.

Reproduce in a test vault. Use the `ship_note()` task in `sase.md`, a dependent `[?]`
task in `deps.md` with `[dependsOn:: ship]`, and this day file:

```text
## Pomodoros
- [ ] (**0920-0950**) — CAPTURE
  - [[sase#^ship]]
- [ ] () — FIX
```

Use `BOB_NOW=2026-10-05 09:30:00`.

- Capture `x!` with `-p 1` (pull in requirement 1 only). The carried task is linked
  under a newly created `- [ ] () — CHORE` entry.
- Capture `x!` with `-p 3`, where offer 3 is the ordinary `deps` successor. It lands in
  a created `- [ ] () — CAPTURE` continuation.
- With `-p 1,3`, both land in the CAPTURE continuation, but only because the implicit
  fallback happens to find it.

Fix: every capture path that closes a Pomodoro must give its owners' pulled carries the
same closing context its ordinary successors used:

- Plain close, chain, and past close: callers of
  `capture/pomodoro_close.rs::plan_close_successors` in `pomodoro_close.rs` and
  `pomodoro_past.rs`.
- The dependency close paths that copy `close.req_owners`: about lines 3508–3674 in
  `capture/dependencies.rs` and about lines 4297–4353 in `capture/plan.rs`.

Apply `closing_after_successors(&closing, &successor_plan)` so a continuation that
ordinary successors already materialized is reused instead of creating a second
placeholder.

- **The early return.** `plan_close_successors` returns `empty("checked")` before it
  computes `continuation_name`, `closed_line`, and `continuation`. When the close has
  requirement owners, still derive that closing context. It is cheap: `post_day`,
  `close_entry_line`, and `plan.summary.ledger.next_pomodoro`. A requirement-only
  Pomodoro close must still create or reuse the CAPTURE continuation.
- **Carry it to the post-pass.** Add a field on `CloseSuccessorOutcome`, propagated
  through the planned item next to `req_owners`, that records:
  - the adjusted `ClosingContext`;
  - the owner keys it applies to;
  - the exact staged day text its line coordinates refer to.
- **Forward the coordinates.** Later items, the successor net post-pass, and the
  offer-verdict post-pass can edit the day file. Before placement, forward `closed_line`
  and `continuation.line` into the post-pass day text with
  `capture/block_diff.rs::line_maps`, the Myers map `PomodoroBlockTracker` already uses.
  Drop the continuation when its headline no longer maps; `resolve_closing_target` then
  falls back to the same-name reuse or creation cases.
  - If `closed_line` itself no longer maps, refuse the whole batch with
    `req_pull_in_no_destination`. Never fall back to a guessed entry.
- **Place per context.** In `plan_req_post_pass`, run placement for pulled carries whose
  owner has a closing context with that context, and the rest with `closing: None`.
  Direct task Done (`!note:id`) has no closing context today, and its behavior must not
  change.
  - Either call `plan_req_link_pass` once per distinct context, in card order, restaging
    the day text between calls, or extend `ReqLinkInput` to take per-owner closing
    contexts.
  - Keep one link per created task, Next with no `[fresh::]`/`[unblocked::]`, shared D7
    routing, and the whole-batch refusal when no destination exists.
- Keep `ReqLinkReport`/receipt shapes unchanged. If you add nothing to the wire, no JSON
  doc change is needed. Otherwise update `docs/reference/json-outputs.md` and §12.25 in
  the same change.
- Add CLI tests in `tests/cli/capture/req_close.rs`:
  - `x!` with `-p 1` links the carried task in the created CAPTURE continuation, the
    same entry an ordinary successor uses. No separate entry is created, the task is
    `[*]`, and there are no freshness stamps.
  - `x!` with `-p 1,3` puts both links in one continuation entry, with no duplicate
    headline.
  - A multi-item draft where a later item edits the day file before the post-pass still
    places the carry in the continuation. A preceding capture item in the same draft is
    an acceptable trigger, if the grammar allows it.
  - Direct `!sase:ship` with `-p 1` is unchanged.

  Add a focused unit test for the line-forwarding helper if you write one.

## 3. Remove the permanent dead-code suppressions

`src/native/task_complete/mod.rs` keeps `#[cfg_attr(not(test), allow(dead_code))]` on
`req_close`, `req_landing`, `req_placement`, `req_verdicts`, and `req_writes`, under a
comment about "plugin and later-phase consumers". The epic plan forbids permanent
dead-code suppressions, and the JS and Mac clients consume the JSON and RV fixture
bytes, not Rust items. Remove those five attributes and the comment. Then resolve every
resulting `dead_code` warning. At `96a23f50` they were:

- `req_close::copy::{DROP_PROMPT, JUMP_HINT, TEACHING, nested_go_with_it, summary}`
- `req_landing` `…::as_str`
- `req_placement::closing_after_successors` (wired by step 2)
- `req_verdicts::{ReqChoiceState, ReqRowFold}` and their methods

For each, wire it into real output where the contract already calls for it, or mark it
`#[cfg(test)]` when it exists only as the RV reference model or fixture copy check. Do
not add new `allow(dead_code)` or `--epic-symbol` exemptions. Confirm `cargo check` and
`cargo clippy` show no new warnings. Leave the pre-existing, unrelated
`#[allow(dead_code)]` on `offer_close` and `verdicts` alone.

## 4. Strengthen the weak carry-pulled test

In `tests/cli/capture/req_close.rs`, `pull_in_on_req_is_carry_pulled` asserts
`day.contains("Handles empty input") || day.contains("[[sase#^")`. The pre-existing
`[[sase#^ship]]` link makes that always true. Assert the exact new link,
`[[sase#^handles-empty-input]]` or the created block ID reported in
`req_outcomes[].carried.block_id`, and that it sits under the expected open entry.

## 5. Verify

Run the focused suites:

- `cargo test native::task_complete::tests`
- the capture `req_close` CLI tests
- the existing successor suites (`successor_offers`, `successor_verdicts`,
  `successor_jump`, relation successors, Pomodoro successor/selection, note_pull
  fixtures)

Then run `cargo fmt --all -- --check`, `just check`, `just docs-gen` (expect no diff
unless CLI help changed), and `just docs-check`. Run long commands through
`/sase_monitor` and wait for them to finish. A failure that reproduces identically on a
clean `master` is not this work: file it with `/sase_new_task` and note it on
bob-cli-on.8.1. Fix every failure this work causes.

## 6. Closeout (final step; do it in this same turn)

1. Run `sase bead epic-symbols bob-cli-on.8.1`. For each listed entry, resolve the
   symbol or, only if a still-open later bead needs the exemption, re-key the Justfile
   line to that bead. The land review found none, and steps 1–5 must not add any.
2. Close the epic. Do not use `--force`; if the close is rejected, fix the reason and
   close again.

   ```bash
   sase bead close bob-cli-on.8.1 --note "<what you verified: phases 8.1.1–8.1.6 reviewed; overlap ancestry, Pomodoro-close carry placement, dead-code suppressions, and the weak test fixed with tests; just check / docs-check results and tree revision; integration of a21fd556, b7945e75, c833be00 needed nothing; the only PROPOSED FOLLOW-UP (8.1.4 #2) was epic-caused and fixed here>"
   ```

3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the YAML frontmatter of the epic plan file
   `plan:202611/requirement_close_engine.md`. `sase bead read bob-cli-on.8.1` prints its
   local path under EPIC PLAN; it currently says `status: wip`.
5. The epic's `parent_bead` is the **phase** bob-cli-on.8, in epic `bob-cli-on`. Verify
   the closed child plan delivered that phase's description: per-owner requirement
   groups for managed Done closes; met/drop/carry/pull-in verdicts with overlap and
   staleness refusals; CLI flags and the `--req-choice` gate; JSON and receipts; RV
   vectors; and §12 docs. Then:
   - Run `sase bead epic-symbols bob-cli-on.8` and resolve any entries.
   - Close only that phase with
     `sase bead close bob-cli-on.8 --note "<what you verified, with test evidence and contract locations: docs/task-dependencies.md §§12.5, 12.13, 12.14, 12.25, 12.26; docs/reference/json-outputs.md; tests/fixtures/req_close/vectors.json schema_version 1, RV1–RV44+>"`.
   - Run `just symvision` again.

   Never close `bob-cli-on` itself. Its land agent owns client integration
   (bob-cli-on.11, bob-cli-on.15), memory (bob-cli-on.17), and live-vault acceptance.

Use `!` in the commit subject only if the CLI or JSON contract changes. The host
finalizer performs the commit.
