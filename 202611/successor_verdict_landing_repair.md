---
tier: tale
title: Finish successor-card verdicts in bob capture and close epic bob-cli-hn
goal:
  Card-resolved offer rows never trap a capture in an inbox-route prompt they cannot
  answer. Merge offers print their verdict receipt. Dry-run hints teach -x/-X. The SV13
  disk-change refusal has a test. Then epic bob-cli-hn is closed.
size: medium
bead: bob-cli-hn
proposed_by: bbugyi200.athena.bob-cli-hn.land
proposed_with: claude/opus
create_time: 2026-11-06 12:02:35
status: wip
---

- **PARENT:**
  [202611/successor_card_verdicts.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/successor_card_verdicts.md)
- **BEAD:**
  [bob-cli-hn](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-hn/README.md)

# Finish successor-card verdicts and land bob-cli-hn

## Context

Epic **bob-cli-hn** ("Successor card verdicts — mark offers Done or Cancel from the
card"; plan `plan:202611/successor_card_verdicts.md`) shipped all nine phases:

- bob-cli `723af324`, `53f9c2c7`, `44c5c8c8`, `efbe3470`, and `5c119a06`;
- bob-plugins `8a16ebe`, `996fc83`, `ef9257d`, `3af8d0c`, `1c3aaf1`, `d45d8f7`, and
  `f7ae51e`;
- Bob Mac Capture `8ab7c4e`, `6f4e279`, and `5b0dca6`.

The land agent verified the following on bob-cli `ec39e1a0`:

- `just check` passes (9135 passed, 0 failed).
- `capture::successor_verdicts` and `capture::note_pull` pass (29/29). The lib tests
  `offer_close` and `offer_verdict` pass (14/14).
- The bob-plugins verdict, successor, notice, promotion, and review-advance suites pass
  (343/343). The cycler copy of `tests/fixtures/offer_close/vectors.json` is
  byte-identical to the bob-cli original.
- The Mac exact-SHA run 42250497927 on `5b0dca6` fails only the existing
  separator/comma/word-motion cluster (bob-cli-h7). Every Pull-In verdict test passes.
- Merge-in captures (bob-cli-hl, landed concurrently) already get correct `offer.close`
  descriptors. A new merge target is refused as `new_task`, and an existing `>` target
  is eligible.

Follow-up triage is recorded on bob-cli-hn. The memory collision is filed as bob-cli-hy.
**Do not edit any memory note.** This tale needs no bob-plugins or Bob Mac Capture
change. The Mac renders whatever bob returns.

The landing found four gaps in the bob-cli capture surface. All are epic work. The
parent plan's shared design and §12.22–§12.23 of `docs/task-dependencies.md` are
binding.

## 1. Card-resolved rows are not inbox-route candidates

**Bug.** The verdict post-pass (`plan_offer_verdict_post_pass`,
`src/native/capture/plan.rs` ≈ line 851) runs before `detect_inbox_routes`. A row closed
by `-x`/`-X` that lives in an inbox note (`inbox.md` or a direct area child) therefore
becomes an inbox-route candidate with `answer: null`, `outcome: "pending"`. The epic's
binding rule makes any route answer for it `offer_verdict_conflict`
(`check_inbox_route_verdict_conflict`, `src/native/capture/offer_verdicts.rs` ≈ line
373). The result:

- `--require-inbox-route-choice` (the Mac path) refuses with
  `inbox_route_choice_required` for a row whose only accepted answer is keep.
- The Mac shows a route card for a task Bryan just closed. Choosing a destination then
  fails with the conflict.
- Human output prints `route with -i 1=<note> · -i keep` for it.

Reproduce with the `tests/fixtures/note_pull/np16` vault (copy `shared/` and then
`np16/vault/` into a temp vault; `BOB_DAY_FILE=<vault>/20261005.md`,
`BOB_NOW="2026-10-05 09:30:00"`):

```text
bob capture -b <vault> -f json -x 1 --require-inbox-route-choice -- '@mac_inbox'
→ ok:false, code inbox_route_choice_required; the only candidate is "Already Next"
  (status Next → Done), the row -x 1 just resolved
bob capture -b <vault> -f json -x 1 -i 1=health -- '@mac_inbox'
→ offer_verdict_conflict
```

**Fix.** A card-resolved row is resolved in the card; it is never an inbox-route
candidate.

- After `detect_inbox_routes`, drop every candidate that matches a verdict row. Use the
  identity rule `check_inbox_route_verdict_conflict` already uses
  (`verdict_row_identities`: same `note_path`, and the same non-empty block ID or the
  same line).
- Renumber the surviving candidates' `n` from 1 in their existing order. Update
  `candidate_count`. Set `destinations` to `None` when no candidate survives, matching
  what detection does for an empty report.
- The required-choice check (`src/native/capture/mod.rs` ≈ line 393), human output, and
  JSON then see only real candidates.
- Keep the binding conflict rule for explicit keys. An `-i KEY=…` whose `KEY` equals a
  dropped candidate's key is still a write-free `offer_verdict_conflict` (exit 2).
- Numeric `-i` selectors index the filtered list, because route numbers are "positions
  in this plan". A number that indexes nothing is the existing `inbox_route_stale`.
- Run the key conflict check against the unfiltered report, before dropping. Never
  resolve a number against the unfiltered list.

**Tests** (`tests/cli/capture/successor_verdicts.rs`):

- Rewrite `inbox_route_answer_on_resolved_task_conflicts` to answer by key. Check the
  real key in the JSON; expect something like `inbox.md#^ialpha`. Expect
  `offer_verdict_conflict`, exit 2, and no writes. Add the numeric case: `-i 1=notes`,
  when the resolved row was the only candidate, is `inbox_route_stale` (exit 2) and
  writes nothing.
- Add a regression for the np16 repro.
  `-x 1 --require-inbox-route-choice -- '@mac_inbox'` succeeds;
  `inbox_route.candidate_count` is 0; the row is written `[x]` with `[completion:: …]`;
  `offer_closes` names it.
- Add a mixed case. Another inbox task's status changes in the same capture, for example
  a second pulled-in inbox row via `-p`. It stays a candidate numbered 1 and is still
  required.

## 2. Merge offers print their verdict receipt

**Bug.** `print_human_transfer_merges` (`src/native/capture/output.rs` ≈ line 4156)
prints each `record.offers` entry as
`◌ {entry} link removed · offer {n}: link “{target}” under {entry} (-p {n})`. It does
this even when that offer was resolved by `-x`/`-X`. JSON is already correct:
`merge_offers[].offer.verdict`, `not_linked: "resolved_in_card"`, and `offer_closes`.
Every other offer printer already uses `offer.verdict_message` and `verdict_ticklers`
(for example `print_human_relation_row` ≈ line 6470).

Reproduce with the `setup_merge_vault` layout from `tests/cli/capture/merge_in.rs`. The
day file is `<vault>/day.md` with
`## Pomodoros\n\n- [ ] () — BUILD\n  - [[a#^f]]\n  - keep this slot\n  - [[b#^b]]\n`.
Run `bob capture -b <vault> -x 1 -- '@a+f @b+b > @inbox+t'`. It prints
`◌ BUILD link removed · offer 1: link “Existing target” under BUILD (-p 1)`.

**Fix.** Find the matching row in `result.relation_merge_output.offer_rows` by offer key
(or `n`). When it carries `verdict_message`, print
`  ◌ {entry} link removed · offer {n}  {verdict_message}` followed by one `  {tickler}`
line per `verdict_ticklers` entry. This is the plan's rule that a resolved row prints
its receipt in place of the link or `-p` hint. Dry runs already get the `would` forms
from `verdict_message`.

**Test.** Add the repro as a CLI test, in `merge_in.rs` or `successor_verdicts.rs`.
Assert:

- the receipt line `offer 1  ✓ completed “Existing target” · Ready → Done`;
- no `(-p 1)`;
- the dry-run `✓ would complete` form;
- with section 1 in place, no inbox-route `route with -i` line for the resolved merge
  target.

## 3. Dry-run hints teach the verdict flags

**Gap.** The parent plan says the dry-run pull-in hint becomes
`-p 1,2 · -p all · -x N done · -X N[=why] cancel`. `print_pull_in_hint` (`output.rs` ≈
line 2699) currently differs in two ways:

- It adds `-x {verdict numbers} done · -X N[=why] cancel` only when verdicts were
  already passed.
- It never adds it to the note-pull hint
  (`nothing pulled in · choose with -p 1,3 or -p all`).

**Fix.** On dry runs only, append ` · -x N done · -X N[=why] cancel` (literal `N`) to
both hint forms. Append it when at least one offer row without a verdict has a `close`
descriptor whose `complete` or `cancel` is true. This is the plan's feature-gating rule
11: no eligible row means no verdict hint. Drop the old listing of already-staged
verdict numbers. Real runs are unchanged.

**Tests and goldens.**

- Update `verdict_human_output_prints_receipt_and_hint`. A plain dry run with eligible
  rows now contains `-x N done · -X N[=why] cancel`.
- Update `verdict_dry_run_human_output_uses_would_forms` to expect `-x N done`.
- Add a case with no eligible rows, for example only new-in-capture offers. It keeps the
  bare `-p` hint.
- Regenerate the note-pull goldens with `cargo build --locked --bin bob` and then
  `python3 tests/fixtures/note_pull/generate.py`. Review the diff: only human hint lines
  of dry runs with eligible rows may change, such as `np1/human.txt`.

## 4. SV13: a disk change between planning and commit aborts a verdict batch

hn.3 found no end-to-end hook for SV13. A lib-level test is deterministic. Model it on
the IRA15 test in `src/native/capture/inbox_route.rs` (≈ line 1199):

1. Build a `CaptureRequest` whose `offer_verdicts` holds one `OfferVerdictRequest`
   (Done) for an eligible offer key.
2. Call `plan_capture_batch(&request, None)`.
3. Rewrite the verdict row's note on disk.
4. Assert that `commit_capture_batch` fails with `CaptureFailureKind::NoteChanged`, and
   that every vault file is byte-identical to its pre-commit state except for the
   concurrent edit.

Put it beside the verdict code, in `offer_verdicts.rs` tests or
`src/native/capture/tests/`.

## Docs

Update every place that describes these rules (`just docs-gen` if CLI help changes):

- `docs/capture.md`, "Routing inbox tasks out" (≈ line 3279). Card-resolved rows are not
  candidates. Add an `IR` vector row for it. An `-i` answer naming one by key is
  `offer_verdict_conflict`.
- `docs/capture.md`, the verdict paragraph (≈ lines 3218–3245) and the hint sentences (≈
  lines 1139, 7378, 7558).
- `docs/task-dependencies.md` §12.14 (≈ lines 2615–2655). Add the inbox-route sentence
  and the hint text. Also update the hint sentences at ≈ lines 1808 and 2911 if they
  describe dry-run hints.

## Gates

- Focused tests:
  `cargo test -p bob-obsidian --test cli -- capture::successor_verdicts capture::note_pull capture::merge_in capture::inbox_route`
  and `cargo test -p bob-obsidian --lib -- offer_close offer_verdict inbox_route`.
- `just check` must pass. Do not run `just check-full`.
- `just docs-check`: the only allowed failure is the known
  `tools/docs/tests/test_contracts.py::test_readme_table_matches_original_contract_index_snapshot`
  (README contract-table SHA, tracked by READY bob-cli-hs). Fix anything else it
  reports.
- Commit as `fix(capture): …` (additive behavior fix, no `!`). Never edit the
  `Cargo.toml` version or `CHANGELOG.md`
  (`decisions:bob-cli-releases-ride-a-daily-train`).

## Final step: close epic bob-cli-hn (in this same turn)

Do this after the code above is complete and the gates pass. It does not wait for this
work's commit SHA, a push, or CI.

1. Run `sase bead epic-symbols bob-cli-hn`. At landing it printed
   `No --epic-symbol entries for bob-cli-hn.`
   - If entries appear, resolve each one per the Symvision epic-whitelist policy: wire
     it up, privatize it, add a non-test pragma, or delete the entry.
   - Re-key an entry only to a still-open later bead that needs it.
2. Close the epic normally. Never use `--force`.

   ```bash
   sase bead close bob-cli-hn --note "<verification>"
   ```

   The note should state:
   - All nine phases were closed, and every child note was reviewed.
   - Epic DISCOVERED ISSUES #1–#3 are fixed: note-pull/successor_verdicts 29/29, and the
     Mac footer test is green on run 42250497927.
   - The shared vectors are byte-identical across bob-cli and the cycler, and the plugin
     verdict suites pass 343/343.
   - Mac exact-SHA run 42250497927 fails only the bob-cli-h7 cluster.
   - Merge-in integration was verified.
   - This tale's four fixes, with their test names and gate results.
   - Follow-up triage is in the epic note (bob-cli-hy created; bob-cli-hs, bob-cli-h7,
     and bob-cli-f8 corroborated; the rest declined as resolved).
   - No memory note was edited.

3. Run `just symvision` if the justfile has that recipe (`just --list`). At landing it
   did not, so say so in your final report instead.
4. Set `status: done` in the frontmatter of the epic's plan file. That is
   `plan:202611/successor_card_verdicts.md`, at the PLAN path printed by
   `sase bead read bob-cli-hn -r "Need the plan path for closeout"`. Its frontmatter
   currently says `status: wip`.
5. bob-cli-hn has no `parent_bead`, so nothing further is closed.
