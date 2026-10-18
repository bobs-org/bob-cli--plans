---
tier: tale
size: small
title: Repair Newest refresh fixtures and finish bob-cli-9d landing
goal:
  Align the existing Mac refresh tests with the added_at contract, verify the integrated
  Newest feature, and close bob-cli-9d in this coding turn.
proposed_by: bbugyi200.athena.bob-cli-9d.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-9d
create_time: 2026-10-18 16:53:32
status: wip
---

- **PARENT:**
  [202610/refs_newest_mode.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/refs_newest_mode.md)
- **BEAD:**
  [bob-cli-9d](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-9d/README.md)

# Remaining work

Epic `bob-cli-9d` implements Bob Refs Command-0 Newest mode. Its three phases are
closed, but the landing audit found two existing macOS refresh tests that still model
the old git response while asserting the new response is complete. This tale fixes that
bounded regression and finishes the original epic's landing. It has no separate land
agent. Do not stop after fixing the tests: the final step below is part of this coder's
scope, in the same turn as the final edits.

Read the epic and its notes with
`sase bead read bob-cli-9d -r "Need landing audit, decisions, and closeout context"`,
and read its plan with
`sase artifact read plan:202610/refs_newest_mode.md "Need approved Newest contract for the remaining landing repair"`.
Open `bob-mac-capture` with `/sase_repo`; use only the printed path and read its
`AGENTS.md`.

The approved decisions remain final: Newest resets on each presentation, and no memory
notes or decisions records may be edited. Preserve the additive schema-version-1
timestamp contract and old-bob decoding support.

## Evidence already established

- The lander reviewed all three child beads and every note, the approved plan, bob-cli
  commit `de78e98`, and bob-mac-capture commits `5a0bff5` and `7d6063a`. Source
  implements timestamp derivation, rename-following git birth, Newest
  ordering/search/grouping, keyboard routing, tokens, reset behavior, and docs.
- The only `PROPOSED FOLLOW-UP:` is `bob-cli-9d.2` note #1, about carrying `gitAddedAt`
  through refresh. It is already addressed by `7d6063a`: `RefsLibrary.runSnapshotPass`
  carries both maps after the sweep, and `runGitDatePass` calls
  `RefsSnapshot.mergingGitDates`. The lander recorded the outcome on `bob-cli-9d`; no
  separate task was warranted. There were no other proposals or unrelated issues needing
  task triage.
- Concurrent bob-cli changes `5848c39`, `4703619`, `b291f23`, `6bc432b`, and `0f47802`,
  plus Mac change `2c007c7`, were reviewed for integration. The new intake routes use
  `write_single_pdf_sync` and the existing v1/v2 executors, both of which stamp
  `created` immediately before a new note is written. Other concurrent changes do not
  conflict with this feature.
- At the audit boundary, bob-cli HEAD was `0f47802`, Mac HEAD was `2c007c7`, there were
  no `--epic-symbol` entries, and `bob-cli-9d` had no `parent_bead`.
- Prior phases reported `just check` and 1,368 portable Swift tests green. Those are
  prior reports, not fresh landing verification. The panel phase's macOS build was
  unverified. A fresh lander SSH attempt to `mac` timed out.

## 1. Repair the current-bob git response fixtures

In bob-mac-capture, `Tests/Fixtures/refs-list-git.json` has two git-sourced rows,
`ref/blogs/small_opened.md` (added 2026-09-11) and `ref/books/small_book.md` (added
2026-08-30), with no `added_at` fields. `RefsSnapshot.pathsNeedingGitDates`
intentionally refills missing git instants. After merging that fixture, both paths still
need a git pass.

An executable Swift probe using the unchanged production `RefsSnapshot` implementation
and these fixture records confirmed that a following plain refresh still requests both
paths. Consequently, these existing native tests have obsolete one-pass expectations for
the fixture they use:

- `RefsLibraryTests.testTriggersDuringRefreshRunExactlyOneFollowUp` uses the default
  fake-bob git fixture and expects exactly three list calls (split count 4).
- `RefsLibraryTests.testSnapshotPassKeepsGitDatesMergedDuringItsSweep` uses that fixture
  through `ScriptedRefsFetcher` and expects `gitListCount == 1`.

Make the normal fake-bob `-g` response represent current bob: give both git rows valid
offset-bearing RFC 3339 instants on their respective `added` days. Keep the plain
response dateless for those rows. Do not remove instant refill from production code or
relax the one-pass assertions to hide the regression.

Consolidate the new `patchedGitFixtureWithAddedAt()` helper in `RefsLibraryTests.swift`
with the corrected canonical fixture where sensible; it currently assigns a September
instant to both rows, including the August row. Preserve the new refresh regression, and
strengthen the existing overlapping-sweep regression to check that the instant survives
as well as the date and that no redundant git fetch is issued.

Preserve explicit old-bob coverage. `refs-list-golden.json` intentionally lacks
`added_at`; `RefsNewestTests.testAnOlderBobDecodesEveryGoldenRowWithoutAddedAt` and
`testMergingGitDatesFromAnOlderBobKeepsTheDateOnly` must keep testing that
compatibility. Add a focused portable regression using the normal plain and git fixtures
to assert that merging the current git response supplies both instants and leaves no
paths needing another git request. This catches the fixture/contract mismatch on Linux
without requiring AppKit.

## 2. Verify the repaired integration

Review changes since the audit boundary in both repositories and integrate any new
overlap before closing. Re-read all child notes and the epic notes for new items.
Epic-caused defects remain in scope. For genuinely unrelated discoveries, use
`/sase_new_task`, identify the proposing bead if applicable, and record every triage
outcome on `bob-cli-9d` before closeout.

Run `just check` in bob-cli and `just portable-check` in bob-mac-capture. Do not run
`just check-full`. A green `just check` with a failing `check-full` would be
infrastructure work rather than unfinished epic work. Use `/sase_monitor` for long
verification commands, with continuation instructions that preserve this tale's final
epic closeout.

For macOS verification, follow the linked repo's `AGENTS.md`: best-effort SSH to `mac`,
rsync a disposable scratch copy excluding `.build` and `.git`, run `Scripts/bundle.sh`
and `just typecheck-tests`, and delete the scratch copy. Never use Bryan's checkout or
install the app. If the Mac remains offline, record that the macOS targets and native
XCTest execution are unverified; the approved original plan permits this best-effort
limitation. If accessible, compilation/typechecking is required; CI is the native XCTest
execution path. Do not wait for this turn's future commit, push, or CI run to close the
epic.

Complete the cross-repository check that the independent core phase could not run
against the new bob. From the bob-cli checkout run the read-only
`cargo run --release -- ref list -R all -A -f json -g`, save its JSON to a temporary
file, then feed that exact output into
`swift run refs-rank --newest --tz America/New_York` in the opened Mac repo. Report the
total rows, null `added` count, non-null `added_at` count, `added_source` distribution,
and group headers. The prior phase saw 1,075 rows, zero null `added`, 471 instants, and
captured=26/created=165/git=282/zorg_block=602; live counts may legitimately change.
Confirm there is no UNDATED group for the currently fully dated library; investigate any
new null date instead of assuming it is an epic defect. Do not mutate the vault or
backfill frontmatter.

## 3. Finish bob-cli-9d closeout in this turn

After the fixes and verification are complete, resolve every entry from
`sase bead epic-symbols bob-cli-9d`: wire it up, privatize it, add a legitimate non-test
pragma, or delete it per Symvision policy. Re-key an exemption only when a still-open
later bead actually needs it. The audit found none, but recheck immediately before
closing.

Close normally with:

```sh
sase bead close bob-cli-9d --note "<summarize all-child/source/commit audit, decisions, integration review, fixture repair, actual verification results, live date/group counts, Mac limitation if applicable, and follow-up disposition>"
```

The note must include that `bob-cli-9d.2` note #1 was already resolved by `7d6063a` and
therefore did not become a separate task. Include any later triage outcomes. If close
rejects leftover symbols, finish cleanup and retry. If it names unfinished phases,
finish or reopen them; never force a successful landing.
`--force --reason ... --resolution canceled|superseded` is only for a deliberately
canceled or superseded outcome, never to bypass readiness.

After close, run `just symvision` if the recipe is available in the relevant repository;
record when unavailable (neither justfile had the recipe at audit). Open the plans
repository with `/sase_repo`, resolve `plan:202610/refs_newest_mode.md` through
`sase artifact path`, and change only its frontmatter `status` from `wip` to `done`.
This is the epic's linked plan, under the plans repository at
`202610/refs_newest_mode.md`; use the resolved path rather than assuming a particular
checkout.

Then run `sase bead read bob-cli-9d -r "Need the parent link"`. There was no parent at
audit, so no ancestor closure is expected. If a parent link has been added, inspect it:
close only a fully satisfied parent phase normally, leaving its containing epic to its
lander; for a parent plan, re-audit its previous landing note, descendants and notes,
linked plan readiness, and post-child drift before normal symbol
cleanup/close/symvision/plan-status update. Repeat only through fully complete directly
parented plans. Stop at the first incomplete or ambiguous parent, note the blocker
there, and report it.

Use `/sase_final` after all work, with commit decisions for every changed repository.
The host commits after the coding turn ends. Closing the epic in the same turn as these
final edits is expected; do not make closeout depend on this turn's own commit SHA,
push, or CI result.
