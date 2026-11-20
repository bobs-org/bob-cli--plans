---
tier: tale
size: medium
title:
  "Land Log Recall (bob-cli-nj): fix Work Log completion, the Mac pull-in recall tests,
  and the doc budgets, then close the epic"
goal:
  "Log Recall matches its approved plan everywhere: work_log capture completion queries
  the text before the cursor, drops exact repeats, offers up to 8 rows, and keeps the
  pre-epic empty successes. Mac CI is green again, the three reference pages this epic
  grew are back under the docs budget, and epic bob-cli-nj is closed with its plan
  marked done."
proposed_by: bbugyi200.apollo.bob-cli-nj.land
proposed_with: claude/opus
bead: bob-cli-nj
status: done
---

- **PARENT:**
  [202611/log_reason_recall.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/log_reason_recall.md)
- **BEAD:**
  [bob-cli-nj](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-nj/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-nj.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-nj.land.md)
- **COMMITS:**
  - [6d54cb0](https://github.com/bobs-org/bob-mac-capture/commit/6d54cb0f42ef19cd6679a5e493671748274f28cd)
    — feat(capture): enable pull-in close recall fixture and update work-band docs

# Land Log Recall (bob-cli-nj)

## Context

The Log Recall epic `bob-cli-nj` shipped all eight phases across bob-cli, bob-plugins,
and bob-mac-capture. Its plan is `plan:202611/log_reason_recall.md`; the PLAN path is
shown by `sase bead read bob-cli-nj -r "<why>"`. Design §2 and the
`capture-log-complete` phase in that plan are the spec for item 1 below.

The land agent verified the epic on 2026-11-20 against bob-cli `f811397d`, bob-plugins
`3d1f553`, and bob-mac-capture `b6b7f6cf`. Most of the work checks out:

- All three repos vendor a byte-identical `vectors.json`.
- `bob logs list` works on the real vault.
- The log_history registry tests pass.
- The plugin build is current, and all 14 Log Recall plugin test files pass.
- All 12 plugins report `synced`.
- No `--epic-symbol` entries exist.

Three groups of epic-caused defects remain. This tale fixes them and then closes the
epic. The land agent's triage note on `bob-cli-nj` already records every child
`PROPOSED FOLLOW-UP:` outcome, so do not re-triage them.

Work in the linked repos only through `/sase_repo` (`sase repo open bob-plugins …`,
`sase repo open bob-mac-capture …`), using the path it prints. Read each repo's
`AGENTS.md` first. In bob-cli, use Conventional Commits and never edit the `Cargo.toml`
version or `CHANGELOG.md`. The epic's DECISIONS forbid any memory edits, so edit no
memory notes.

## 1. bob-cli: make `work_log` capture completion match the plan (nj.6, `ddf28f64`)

Four tests fail today on master because of this phase:

- lib
  `native::capture_complete::tests::drawers::bullet_starts_that_cannot_name_a_drawer_are_empty_successes`
- lib
  `native::capture_complete::tests::output::close_items_and_suffixes_request_no_completion`
- lib
  `native::capture_complete::tests::output::close_bullet_lines_suppress_wikilink_block_but_keep_note`
- CLI
  `capture::complete_drawers::bullet_starts_that_cannot_name_a_drawer_are_empty_successes`

They fail because the engine returns `work_log` for `"x more"` at cursor 2, `"x\n- #"`,
and `"x\n- 1 @@"`, where these tests expect the pre-epic empty success. The
implementation also deviates from the approved plan.

Fix the implementation. Keep these four tests **unchanged** and make them pass; do not
weaken the marker or block-link suppression.

In `src/native/capture_language/completion.rs` `work_log_completion_field_at`:

- Return `None` when the cursor's summary text contains `[[`. Wikilink note and heading
  completion keep owning those cursors.
- Gate on the summary text **before the cursor**. Return `None` unless
  `raw_text[span.start..cursor]` has at least 2 non-whitespace chars.
- Set `query` to the summary text up to the cursor, not the whole span. `replacement`
  stays the whole summary span `(span.start, span.end)`.

In `src/native/capture_complete/engine.rs`:

- Set `MAX_WORK_LOG_CANDIDATES` to **8**, as the plan says ("There are at most 8").
- Drop any phrase whose `key` equals the normalized **whole summary**
  (`raw_text[field.replacement.0..field.replacement.1]`). `log_history`'s `rank` module
  is private, so expose a small `pub(crate)` normalize wrapper from
  `src/native/log_history/mod.rs` and reuse it. Do not copy the normalization.
- When a Work Log field yields **zero** candidates after the drop, do not answer
  `work_log`. Fall through to exactly the pre-epic path
  (`capture_language::completion_field_at_with(…)`, or an empty success when that
  returns `None`). This keeps the existing guarantees in Work Log text: marker
  completion is suppressed, and an empty or blank catalog behaves as it does today.
- Keep `initial_selection: "none"`, the 30 s snapshot, and the candidate JSON shape (all
  additive).

In `src/native/capture_language/editor_pomodoro.rs`, skip pushing the
`pomodoro_close_log_summary` span when the entry text contains `[[`. There are three
push sites: the close lexed entries, the close inline entry, and the past entries/inline
entry. The plan specifies that the span appears "only for summaries containing no `[[`".

Tests (`src/native/capture_complete/tests/work_log.rs` and `capture_parse.rs`):

- Change `work_log_completion_waits_for_two_typed_characters`. `"x a"` and `"x a "` now
  yield an empty success (context `None`, no candidates).
- Add a test that a cursor in mid-summary queries only the text before it.
  `"x auth retry"` with the cursor after `au` gives query `au` and a whole-summary
  replacement.
- Add an exact-key drop test. `"x Auth  Retry"` at the end must not offer `auth retry`.
- Add a cap test: a vault with more than 8 matching Work phrases yields exactly 8 rows.
- Add a test that a `[[` summary gets no `work_log` context.
- Extend `json_marks_work_log_entry_summaries_with_additive_spans` with a `[[` summary
  that has no summary span.

Docs and help: update the prose to say "up to eight", "two non-space characters before
the cursor", the exact-match drop, and "no `work_log` answer when nothing matches". The
places are:

- `src/native/capture_complete/cli.rs` (long help near line 191)
- `src/native/capture_parse.rs` (the `pomodoro_close_log_summary` help sentence: spans
  only for summaries without `[[`)
- `docs/log-recall.md` (the inline Work Log paragraph, "top three")
- `docs/capture.md` (around line 8983, "up to three")

Then run `just docs-gen` and accept any regenerated reference or help snapshots.

## 2. bob-mac-capture: fix the 6 failing `CapturePullInRecallTests` (nj.5, `e58a9e27`)

Mac CI was green at `bfb21321` and has been red since nj.5 `e58a9e27`. The failures are
6 test cases in `Tests/BobMacCaptureTests/CapturePullInRecallTests.swift`, all starting
at line 33 with `activeReasonKey` nil. CI run 44613921772 logs them.

Root cause: the fixture never enables pull-in close, so `beginPullInCancelForHighlight`
refuses with "Closing offers needs a newer bob" and never opens the reason field. Fix
the fixture, not the product code:

- In `modelWithReason()`, use
  `CapturePullIn(supported: true, offerCount: 3, close: CapturePullInClose(supported: true))`.
- In `offers(count:)`, give each `TaskCompleteOffer` the argument
  `close: TaskCompleteOfferClose(complete: true, cancel: true)`.

`CapturePullInChoiceModelTests.testIneligibleCloseChordsShowAndClearRefusalWithoutVerdicts`
already uses this pattern.

If assertions after setup still fail, fix the real cause. The test may encode the wrong
expectation, or the nj.5 model code may be wrong; check against plan Design §3. Never
delete or skip a case.

Also change `docs/capture-previews.md` "up to 3 Work phrases" to "up to 8" so it matches
item 1.

## 3. Bring the three reference pages this epic grew back under the 6000-word cap

bob-cli `just docs-check` fails only on these budget errors, and the epic caused all
three:

| Page                                         | Before the epic | Now  | Commits that grew it             |
| -------------------------------------------- | --------------- | ---- | -------------------------------- |
| bob-mac-capture `docs/capture-previews.md`   | 5993            | 6171 | nj.5 `e58a9e27`, nj.7 `ddeab6c8` |
| bob-mac-capture `docs/keyboard.md`           | 5970            | 6027 | nj.5 `e58a9e27`                  |
| bob-plugins `docs/bob-navigation-hotkeys.md` | 6000            | 6042 | nj.8 `3d1f553`                   |

Trim each page to **≤ 6000** words. Tighten the epic's own added prose first, then
redundant adjacent prose. Do not drop any shipped key, gate, or failure behavior. Where
a page repeats Log Recall details, link to bob-cli `docs/log-recall.md`
(`https://bobs-org.github.io/log-recall/`). `count_words` lives in bob-cli
`tools/docs/bob_docs/markdown.py`; the budget is enforced by bob-cli `just docs-check`,
which reads the linked checkouts.

Then run:

- `npm run docs:check` in bob-plugins
- `just docs-check` in bob-mac-capture
- bob-cli `just docs-check`

When bob-cli `just docs-check` reports no budget errors, close tracker `bob-cli-jz`. Its
note from the land agent documents that this epic caused the current overage. Close it
with `sase bead close bob-cli-jz --note "<page counts after trimming>"`. After the
plugin doc edit, run `bob plugins sync` and follow the bob-plugins `AGENTS.md` refusal
protocol (never `-F` on your own).

## 4. Verification gates

- bob-cli: `just check` must pass except for three tests owned by active epic
  `bob-cli-nh`, already recorded as its DISCOVERED ISSUE and not this tale's work:
  `catalog::tests::public_leaves_are_entries_or_excluded`,
  `completion::kinds::tests::every_value_arg_has_a_decision`, and
  `periods::update::tests::planning_applies_ordered_intention_ops_and_reports_bad_indexes`.
  These come from `periods update`. Date-dependent `tick::` failures are tracked by
  `bob-cli-nf`. Do not run `just check-full`.
- bob-cli: `just docs-check` passes.
- bob-mac-capture: `just portable-check` passes; run it on athena when local `swift` is
  missing.
- bob-mac-capture: finish **every** Mac edit (tests and docs) first. Then run
  `Scripts/macos-ci-check` from the Mac checkout as the last Mac step. It uploads the
  working tree to a throwaway `ci/precommit/*` branch and runs exact-tree macOS CI. Run
  it in the foreground with an explicit timeout of at least 60 minutes. The Lint, Build,
  and Test steps must be green; if the Test step fails, fix the cause and rerun. A known
  transient `CaptureDotRouteAlias` timing case may pass on a rerun. If any Mac file
  changes afterwards, rerun the script. This pre-push gate is the Swift verification for
  this tale. Do not wait for or order anything after the post-push master CI run.
- bob-plugins: `npm run docs:check` passes and `bob plugins sync` completes.

## 5. Closeout (final step; do it in the same turn as the code)

1. Run `sase bead epic-symbols bob-cli-nj`. Resolve every listed `--epic-symbol` entry:
   wire it up, privatize it, add a non-test pragma, or delete it per the Symvision
   epic-whitelist policy. Re-key it only when a still-open later bead needs the
   exemption. None existed at landing time.
2. Close the epic with
   `sase bead close bob-cli-nj --note "<what you verified: the 4 capture-complete tests and new work_log tests green, plan-conformant work_log (cursor query, non-space gate, exact drop, max 8, no [[), Mac pre-push CI green incl. 6 CapturePullInRecallTests, three pages ≤6000 and docs-check green, plugins synced>"`.
   Never use `--force` to make the close succeed.
3. Run `just symvision` if that recipe exists; bob-cli's justfile currently has none.
4. In the epic plan file (PLAN `202611/log_reason_recall.md` in the plans repo; resolve
   the directory with `sase repo path plans`), set the frontmatter `status: wip` to
   `status: done`.
5. `bob-cli-nj` has no `parent_bead`, so no parent bead needs handling.
