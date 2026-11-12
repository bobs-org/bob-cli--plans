---
tier: epic
title: Fix the impact-ranked open task beads
goal: 'The three red verification gates every landing re-triages (macOS CI, bob-cli
  docs-check, the bob-plugins wikilink perf budgets) are green again and the two diagnosed
  user-facing bugs are fixed, so beads bob-cli-jx, -jy, -h9, -jz, -jr, -k8, -f8, and
  -hp can be closed with verified evidence.

  '
decisions:
  docs_headroom:
    ask: Should the five docs pages be trimmed to 5800 words or fewer instead of just
      under the 6000 cap?
    choices:
      headroom: Trim each page to 5800 words or fewer so concurrent epics do not immediately
        re-trip the cap
      minimal: Trim only to just under 6000, preserving the most existing prose
    default: headroom
    why: These pages were pushed back over the cap five times in two weeks.
    answer: headroom
  split_pages:
    ask: May the docs phase split a section into a new mounted reference page when
      trimming would drop facts?
    choices:
      trim_only: Trim duplicated prose only; a page that cannot reach target keeping
        its facts is reported
      allow_split: Split one coherent section into its own mounted page, preserving
        existing published URLs
    default: trim_only
    why: Every page is only 1-4% over the cap; deduplication reaches target without
      IA/URL risk.
    answer: trim_only
  bob_now:
    ask: Should the created-offset phase also make an RFC 3339 BOB_NOW with an explicit
      offset be honored?
    choices:
      honor: Fix BOB_NOW parsing to honor an explicit offset, with its own regression
        test
      note_only: Only investigate and record findings on bob-cli-k8 for a follow-up
    default: honor
    why: The documented RFC 3339 form silently falls back to the clock; the worker
      is already in that file.
    answer: honor
phases:
- id: mac-typecheck
  title: Isolation-correct alias test teardowns
  depends_on: []
  size: small
  description: 'mac-typecheck: fix the two @MainActor test classes that mutate ledgerSuites
    from nonisolated tearDown so bob-mac-capture `just typecheck-tests` passes on
    clean master (bead bob-cli-jy).'
- id: docs-trims
  title: Mounted reference pages back under the word cap
  depends_on: []
  size: medium
  description: 'docs-trims: trim the four bob-mac-capture pages and one bob-plugins
    page that fail the 6000-word reference budget, keeping contract facts, and verify
    with each repo''s docs gate plus bob-cli just docs-check (beads bob-cli-h9, bob-cli-jz).'
- id: stats-borrow
  title: Stats RefCell double-borrow panic
  depends_on: []
  size: small
  description: 'stats-borrow: restructure HomesEngine::finish so remember/ensure_root
    never re-borrow the resolver, add a regression test for single-metric ranges covering
    today, and record bob-cli-hp close evidence from just check (beads bob-cli-jr,
    bob-cli-hp).'
- id: created-offset
  title: Configured-timezone created stamps
  depends_on: []
  size: small
  description: 'created-offset: pass the configured timezone into capture_project_note''s
    shared timestamp helpers so created/captured stamps carry the right offset on
    UTC-clock hosts, with a TZ=Etc/UTC regression test (bead bob-cli-k8).'
- id: mac-tip-tests
  title: Deterministic alias-tip CI timeouts
  depends_on:
  - mac-typecheck
  size: large
  description: 'mac-tip-tests: diagnose and fix the three CaptureAliasTipPanelTests
    waitUntil timeouts red on every macOS CI run, without weakening assertions, and
    watch the exact-SHA run fully green (bead bob-cli-jx).'
- id: wikilink-perf
  title: Task-wikilink CPU budget investigation
  depends_on: []
  size: large
  description: 'wikilink-perf: establish reproducible measurements for the block-id-prompt
    task-wikilink snapshot/order and keystroke-rank CPU budget failures, then land
    a production fix, measurement fix, or justified budget — never a silent bump (bead
    bob-cli-f8).'
proposed_by: bbugyi200.athena.1bm
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-11 17:20:01
status: done
bead_id: bob-cli-kj
---

- **PROMPT:** [prompts/202611/impact_ranked_bead_fixes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/impact_ranked_bead_fixes.md)
- **BEAD:** [bob-cli-kj](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kj/README.md)

# Fix the impact-ranked open task beads

## Goal

Fix the open task beads recommended by the research report
`research:202611/open_task_bead_impact_ranking.md` (2026-11-11). All eight beads were
re-verified as still READY and still reproducing on 2026-11-11 evening:

| Bead                  | What                                                                        | Where                         | Status today                                                                |
| --------------------- | --------------------------------------------------------------------------- | ----------------------------- | --------------------------------------------------------------------------- |
| bob-cli-jx (⚙ L, +2)  | 3 CaptureAliasTipPanelTests time out on every macOS CI run                  | bob-mac-capture               | Red on every run incl. today's                                              |
| bob-cli-jy (⚙ S, +1)  | `just typecheck-tests` fails on clean master                                | bob-mac-capture               | Reproduced today on HEAD ee26941                                            |
| bob-cli-h9 (⚙ S, +15) | protocol.md over the 6000-word mounted reference budget                     | bob-mac-capture source        | 6224 words at current mounts                                                |
| bob-cli-jz (⚙ M, +7)  | capture-panel/capture-previews/keyboard/bob-navigation-hotkeys over budget  | bob-mac-capture + bob-plugins | 6227/6222/6083/6200 words                                                   |
| bob-cli-jr (⨯ S)      | `bob stats show worked` panics "RefCell already borrowed"                   | bob-cli                       | Double-borrow confirmed in current src/native/stats/homes.rs                |
| bob-cli-k8 (⨯ S, +1)  | `created:` stamps label configured-TZ wall time with chrono::Local's offset | bob-cli                       | `format_timestamp_with_format` still uses `Local`                           |
| bob-cli-f8 (⚙ L, +14) | task-wikilink CPU budgets fail repeatedly on athena                         | bob-plugins                   | Corroborated today by two landers                                           |
| bob-cli-hp (⚙ S, +7)  | duplicate `bob clipboard` registration fails two lib tests                  | bob-cli                       | Fixed on master (one entry each in kinds.rs/table.rs); needs verified close |

These are the three red verification gates every epic landing keeps re-triaging (macOS
CI, bob-cli `just docs-check`, bob-plugins full `npm test`) plus two small,
fully-diagnosed user-facing bugs. Every phase below must read its bead(s) with
`sase bead read <id> -r "<why>"` before starting and record verification evidence in its
phase notes.

## Repository access and ground rules

- bob-mac-capture and bob-plugins are linked repos: open them with
  `sase repo open <name> -r "<why>"` and work only in the printed path. Read each repo's
  `AGENTS.md` before editing it.
- bob-mac-capture pushes: GitHub Actions is the only gate for macOS-only targets. Gate
  first with `Scripts/macos-ci-check`, and after every push watch the exact-SHA macOS
  run to green with `/sase_monitor` (see that repo's AGENTS.md). While bob-cli-jx is
  unfixed the Test step is red on every run; phases other than mac-tip-tests must triage
  that known-red as pre-existing by citing bob-cli-jx rather than treating it as their
  own failure. If master fails at **Build** when a phase starts (at the time of
  planning, commit 4a69ed2 from active epic bob-cli-ka broke the build with an
  `accessibilityHelp` compile error in CapturePanelView.swift), that break is owned by
  its active epic: rebase once it is fixed; do not fix or work around another epic's
  in-flight break silently — record what you observed in your phase notes.
- bob-cli release-train rules apply (decision `bob-cli-releases-ride-a-daily-train`):
  never edit the Cargo.toml version or CHANGELOG.md; mark CLI/JSON contract breaks with
  `!` (none are expected from this epic).
- Phase workers never create beads: record discovered work as `PROPOSED FOLLOW-UP:`
  notes on their own phase bead.
- Do not close the task beads from a phase. The land agent closes each resolved bead
  (see Landing).

## Phase: mac-typecheck (bob-cli-jy)

**Repo:** bob-mac-capture.

`just typecheck-tests` fails on clean master because two `@MainActor` XCTestCase
subclasses read and mutate their main-actor-isolated
`private var ledgerSuites: [String]` from the nonisolated `override func tearDown()`:

- `Tests/BobMacCaptureTests/CaptureRelationEqualsAliasTests.swift` (~lines 7-15)
- `Tests/BobMacCaptureTests/CaptureModifierAliasTests.swift` (~lines 9-17)

Both came from commit `542a37d` (closed epic bob-cli-j9). Real XCTest on CI still
compiles and runs them; only the local typecheck stub gate is broken.

**Fix.** Make the teardown isolation-correct without weakening the tests. Candidate
shapes (pick the one that type-checks cleanly and reads best):

- register cleanup with `addTeardownBlock { @MainActor in ... }` where suites are
  appended, or
- use `override func tearDown() async throws` and hop to the main actor, or
- make the `ledgerSuites` bookkeeping nonisolated-safe.

**Verify.** `just typecheck-tests` passes on the phase tree (and confirm the same
command fails on clean master before the change, so the fix is provably yours). Run
`just portable-check` if Swift tooling is unavailable for the full suite locally, then
push and watch the exact-SHA macOS run with `/sase_monitor` — the Test step will still
be red from bob-cli-jx; confirm the failures are only the known jx trio (plus any
already-tracked intermittents) and cite bob-cli-jx in your notes.

## Phase: docs-trims (bob-cli-h9 + bob-cli-jz)

**Repos:** bob-mac-capture (4 pages) and bob-plugins (1 page). The pages are mounted
into bob-cli `docs/reference/` and the fix belongs in the source repos.

Current counts (bob-docs lint, 2026-11-11, cap 6000 for reference/):

| Source page                                  | Words |
| -------------------------------------------- | ----- |
| bob-mac-capture `docs/protocol.md`           | 6224  |
| bob-mac-capture `docs/capture-panel.md`      | 6227  |
| bob-mac-capture `docs/capture-previews.md`   | 6222  |
| bob-mac-capture `docs/keyboard.md`           | 6083  |
| bob-plugins `docs/bob-navigation-hotkeys.md` | 6200  |

Counts move daily as active epics land; re-measure with the canonical counter
(`bob_docs.markdown.count_words`, exercised via each repo's docs gate) when the phase
starts and trim from what you find.

**Fix.** Bring each page under the cap — to ≤5800 words per the `docs_headroom` decision
default — **without dropping contract facts**. Trim duplicated prose first (the beads
name the protocol compatibility table and capability rows as the densest duplication).
Per the `split_pages` decision default, do not move sections to new pages; if a page
cannot reach the target without dropping contract facts, trim as far as facts allow and
record the blocker in your phase notes.

> [!decision] split_pages = allow_split You may instead split one coherent section into
> its own mounted reference page, keeping the existing page's published URL and adding
> the new page to the repo's `docs/nav.yml` / inventory so the mount and docs-check stay
> consistent.

**Scope boundary.** Only these five pages. Other reference pages currently over budget
are owned elsewhere: bob-cli `docs/reference/json-outputs.md` and
`docs/reference/capture-syntax.md` belong to the active k3/ka epics' own landings, and
`docs/reference/plugins/bob-ledger-tools.md` is bead bob-cli-il. If they are still red
when you run the mounted check, cite those owners in your notes.

**Verify.** bob-mac-capture `just docs-check`, bob-plugins `npm run docs:check`, and
bob-cli `just docs-check` with the siblings mounted: the five owned pages must no longer
appear in the lint output. Watch the exact-SHA macOS run after the bob-mac-capture push
per the ground rules (Test-step red is pre-existing bob-cli-jx until the mac-tip-tests
phase lands).

## Phase: stats-borrow (bob-cli-jr, plus bob-cli-hp evidence)

**Repo:** bob-cli.

`bob stats show worked -f json -r <range covering today>` exits 101 with
`RefCell already borrowed` at `src/native/stats/homes.rs:412`. Root cause (confirmed at
current master): `HomesEngine::finish` opens
`let mut resolver = self.resolver.borrow_mut();` (line ~127) and calls
`self.remember(&home)` inside that scope (lines ~139/143); `remember → ensure_root`
calls `self.resolver.borrow_mut().resolve_note(...)` (lines ~410-413) whenever the
home's `root_parent` is not yet in `seen` — a double borrow. It fires only when such a
note's root parent was not already remembered, which is why it depends on the range and
on whether `-a` is used.

**Fix.** Either collect resolved homes inside the borrow scope and call `remember` after
the borrow drops, or thread the already-borrowed resolver into `remember`/`ensure_root`.
Add a regression test with a project note whose `root_parent` is first seen through the
global-inspect pass, covering a single-metric (`worked`) request whose range includes
the note (the bead records `bob stats show worked -f json -r <today>..<today>` as the
crashing shape).

**Also:** while running `just check`, confirm the two bob-cli-hp lib tests pass
(`native::completion::kinds::tests::every_table_entry_matches_the_tree` and
`native::catalog::tests::public_leaves_are_entries_or_excluded`) and record that in your
phase notes — the duplicate `bob clipboard` registrations were already removed by Gmail
compose commit 2a37f8ce, and this evidence lets the land agent close bob-cli-hp.

**Verify.** The new regression test fails before the fix and passes after; `just check`
green.

## Phase: created-offset (bob-cli-k8)

**Repo:** bob-cli.

`capture_project_note::format_created_timestamp` / `format_captured_timestamp` (via
`format_timestamp_with_format`, src/native/capture_project_note.rs ~lines 129-180)
resolve the naive `now` with `chrono::Local` — the process timezone. Since bob-cli-2q,
`now` is wall-clock time in Bob's **configured** timezone (`env::configured_timezone()`,
America/New_York), so on a UTC-clock host (apollo) every notebook, project-note, and ref
`created:` stamp labels New York wall time with `+0000` — a silently wrong instant that
compounds daily. `records/plan.rs::format_created_timestamp` already takes `Option<Tz>`
and gets this right, including the DST-gap fallback.

**Fix.** Pass `env::configured_timezone()` into the shared helper, mirroring the records
implementation (keep the DST gap and ambiguous-time handling). Audit the callers the
bead lists: `src/native/notebooks/create.rs:287`, the project-note capture path,
`src/native/highlights_ref/note.rs:65`, and `src/native/records/cli.rs:837` (check
whether it already passes a timezone). Add a regression test with `TZ=Etc/UTC` plus a
configured America/New_York zone expecting `-0500`/`-0400`.

Per the `bob_now` decision default, also make an RFC 3339 `BOB_NOW` with an explicit
offset honored (`BOB_NOW=2026-11-11T10:00:00-05:00` is currently ignored and falls back
to the clock, while `BOB_NOW="2026-11-11 10:00"` works), with its own test.

> [!decision] bob_now = note_only Skip the BOB_NOW parsing change; investigate just far
> enough to record accurate findings on bob-cli-k8 in your phase notes for a follow-up.

**Verify.** New tests fail before/pass after; `just check` green. This fixes the stamp
for notes created from now on; backfilling already-written wrong stamps is out of scope
(propose a follow-up note if the audit shows it is worth doing).

## Phase: mac-tip-tests (bob-cli-jx) — depends on mac-typecheck

**Repo:** bob-mac-capture.

Three `Tests/BobMacCaptureTests/CaptureAliasTipPanelTests.swift` tests fail with
`waitUntil` "Condition not met before timeout" (~8 s each) on every exact-SHA macOS CI
run since commit `aafed83` ("fix(capture): anchor alias tip fixtures and tests"):

- `testFilteringKeepsTheShownTipAnchoredToTheSigil`
- `testOptionTypedHashOpensTheHashTip`
- `testTypingQueryBeforeTheRouteListOpensKeepsTheTip`

At least ten runs across unrelated SHAs show the identical failures (run IDs are in the
bead). A red Test step skips Bundle, Launch smoke, and Install/reinstall, so every Mac
change currently ships unverified.

**Fix.** Find why the tip never appears in these three hosted panel tests on CI —
candidate areas named by the bead: fixture anchoring after `aafed83`, the fake-bob tip
reply, or the panel wait condition. Fix the product if the tests reflect the intended
alias-tip contract, otherwise fix the tests. **Do not delete or weaken the assertions.**
This is macOS-CI-only work: expect to iterate via `Scripts/macos-ci-check` snapshot
runs; budget for that loop.

While in the suite, check whether the same root cause explains the tracked intermittents
(`testEditBeforeTheReplyDropsTheTip`,
`testChangedSeparatorReplyDoesNotDisableTheNextTipRequest`, and bead bob-cli-jk's
`testSecondEligibleSigilInASessionShowsNoTip`); record findings in your phase notes
either way — fixing them is in scope only if it is the same cause.

**Verify.** A green exact-SHA macOS CI run on the phase's final SHA — Test, Bundle,
Launch smoke, and Install/reinstall all executed and green — watched to completion with
`/sase_monitor`.

## Phase: wikilink-perf (bob-cli-f8)

**Repo:** bob-plugins.

`scripts/test-block-id-prompt-task-wikilink-index-performance.cjs` repeatedly fails two
CPU-timed gates on athena, including in isolated runs: warm snapshot plus order ~72-128
ms against a 50 ms budget, and one keystroke rank of 5,000 rows ~12-23 ms against 12 ms.
It has also failed on apollo, and it blocks a clean full `npm test`. Fresh
corroborations from 2026-11-11 show it is intermittent (a full `npm test` passed
6109/6109 the same day two clean-base runs failed). The tests already measure CPU time,
warm the rank path, and take best-of 5/7 samples.

**Investigate, then fix.** The bead is explicit: do not misdiagnose this as wall-clock
scheduling and do not blindly raise budgets — these gates guard Obsidian typing latency
Bryan feels directly. Establish reproducible host measurements (athena and apollo,
loaded and unloaded), profile the production snapshot/rank path in `block-id-prompt`'s
task-wikilink index, and land one of, in order of preference:

1. a production fix restoring the budgets,
2. a measurement fix (if the harness is charging the index for work it does not do at
   runtime), or
3. a justified budget change backed by written measurements showing the production
   latency target is still met — never a silent bump.

Keep a test that detects real slowdowns. Also note the secondary cold-build chunk
observation (16.1-19.4 ms vs 16 ms) recorded on the bead; resolve or explain it only if
it shares the root cause.

**Verify.** The performance file passes in isolation and inside full `npm test` on
athena repeatedly (not best-of-one), plus `npm run build:check` and the repo's standard
gates. Record the measurement evidence in your phase notes.

## Landing

- Re-run the full gates: bob-cli `just check` and `just docs-check` (with siblings
  mounted), bob-plugins `npm test` and `npm run docs:check`, and a final exact-SHA macOS
  CI run watched to green.
- Close each resolved task bead with
  `sase bead close <id> --note "<what was verified, citing the landed commit and verification runs>"`:
  bob-cli-jx, bob-cli-jy, bob-cli-h9, bob-cli-jz, bob-cli-jr, bob-cli-k8, bob-cli-f8,
  and bob-cli-hp (hp closes on the stats-borrow phase's recorded evidence plus the
  landing `just check`; its fix already landed in 2a37f8ce). If a phase resolved its
  defect only partially, note what remains on the bead instead of closing it.
- Any gate still red from work owned elsewhere (active epics k3/ka for capture-syntax.md
  and json-outputs.md, bead bob-cli-il for bob-ledger-tools.md, bead bob-cli-jk for its
  alias-tip intermittent if it proves to be a different cause) is cited, not adopted.

## Out of scope

- The bob-mac-capture master Build break from active epic bob-cli-ka (4a69ed2,
  `accessibilityHelp`) — owned by that epic's landing.
- bob-cli `docs/reference/capture-syntax.md` and `docs/reference/json-outputs.md` budget
  overages (active epics k3/ka) and `docs/reference/plugins/bob-ledger-tools.md` (bead
  bob-cli-il).
- The broader flaky-test family (ej, fz, f2, g8, gd, io, hf, hr, ju, hu, hv, jb, jk, jm)
  except where mac-tip-tests proves a shared root cause.
- Backfilling historical wrong `created:` stamps (k8 fixes future writes).
- The SASE handoff/stranding class (e0, ei, gy) — lives in the sase project's codebase,
  not bob-cli.
