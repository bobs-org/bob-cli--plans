---
tier: tale
title: Settle the Refs stale-refresh argv race and land bob-cli-5s.10
goal: "Make RefsLibraryTests.testRefreshIfStale wait for the git-date lane before it
  samples argv counts, then close epic bob-cli-5s.10 and, when still complete, its
  parent epic bob-cli-5s.

  "
size: small
bead: bob-cli-5s.10
proposed_by: bbugyi200.athena.bob-cli-5s.10.land
create_time: 2026-10-09 11:55:43
status: done
---

- **PARENT:**
  [202610/bob_refs_land_fixes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/bob_refs_land_fixes.md)
- **BEAD:**
  [bob-cli-5s.10](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-5s/bob-cli-5s.10.md)

# Settle the stale-refresh argv race and land bob-cli-5s.10

## 1. Context

Epic `bob-cli-5s.10` (plan `plan:202610/bob_refs_land_fixes.md`) is otherwise complete.
Its four phases are closed. The land agent read every phase note, the plan, and the epic
commits, and checked the source against the plan.

Verified already, so do not redo it:

- bob-cli `b566ba4` (phase 10.1): `/.build/` is gitignored, `git ls-files .build` is
  empty, `RefRow.blocked` is serialized directly after `reading_state_source`,
  `two_trackers_blocked.md` asserts `blocked == false`, and `docs/ref.md` states the
  Refs-panel client note and that `blocked` is additive under `schema_version` 1.
- bob-mac-capture `3a5fd4a` through `d5fcac0` (phases 10.2–10.4): unavailable rows stay
  in place, 2-character word prefixes reach T1, day counts use `signals.calendar`, Ready
  sorts by added desc, git dates are approximate, the weekday window is `2...6`,
  `frecencyHalfLifeDays` is wired, prepared items are cached per snapshot,
  `refs-rank --now` accepts a local date-time, typed search calls `setQuery`, open
  errors re-show through `panelRepresenter` without `prepareForPresentation`, wake
  observes the workspace notification center, Today refreshes on every open, `-g` runs
  after the snapshot publishes, ⌘R and Recheck re-rank after refresh, live settings
  apply the value the sink receives, the content well and Reduce Transparency base and
  per-show scale-in are in, the inspector omits unknown facts, the intrinsics timeout
  races a detached read, and ⌘K anchors from `selectedRowRect`. Default Highlights open
  key remains `cmdO`. No memory notes were edited.
- Final macOS CI on `d5fcac0` is green:
  https://github.com/bobs-org/bob-mac-capture/actions/runs/37949647297 That success is
  the `--failed` rerun. The first attempts failed `RefsLibraryTests.testRefreshIfStale`
  with a 4-vs-3 argv count (phase 10.4 note #1).
- `sase bead epic-symbols bob-cli-5s.10` and `sase bead epic-symbols bob-cli-5s` both
  printed no entries at landing. Re-check before each close.
- No non-epic commit landed in either repo after this epic started. Do not hunt for
  integration work.
- Follow-up triage is epic note #2 (written by this land agent). Do not re-triage it and
  do not create task beads.

DECISIONS from parent epic `bob-cli-5s` still hold: `highlights_open_key = cmd_o`, and
do not edit any memory note.

The only remaining epic work is the test race below. Do not change production refresh
behavior.

## 2. Wait for the git lane in `testRefreshIfStale`

Open bob-mac-capture with
`sase repo open bob-mac-capture -r "Settle the stale-refresh argv race"` and edit only
the checkout that command prints.

File: `Tests/BobMacCaptureTests/RefsLibraryTests.swift`, function `testRefreshIfStale`.

The git-date lane starts only after the snapshot publishes
(`RefsLibrary.runSnapshotPass` sets `lastSuccessAt`, then calls `refreshGitDates`). The
test waits until `lastSuccessAt` is set and until Today has 4 entries, then samples
`argv=` counts. A late `argv=ref list … -g` line can still land inside the following 1 s
quiet window, so the quiet count is 4 and the baseline count is 3.
`components(separatedBy:)` counts separators plus one, which is that 4-vs-3 failure.

`testWakeNotificationTriggersSnapshotRefresh` in the same file already waits for this
lane. After the Today wait, and before reading `staleRecord`, add the same wait:

```swift
// Settle the `-g` lane before the baseline. It starts after the
// snapshot publishes, so a late git argv line otherwise lands inside
// the quiet window and the counts disagree (4 vs 3).
await waitUntil(timeout: 15) {
    context.library.items.contains {
        $0.id == "ref/blogs/small_opened.md" && $0.added != nil
    }
}
```

Keep the Today wait, the 1 s quiet sleep, and both assertions. Match the file's 4-space
indent and stay within 100 columns. Do not change `RefsLibrary.swift`.

If `swift` is on the host and this package's tests can build here, run
`swift test --filter RefsLibraryTests.testRefreshIfStale`. Bob Mac Capture tests need
macOS. When they cannot run on this host, do not block. Do not run `just check-full`. Do
not run `just check` unless you also edit bob-cli; this tale does not.

Do not commit this change yourself and do not wait for its CI run. The turn's finalizer
commits after the turn ends. Close the epic in this same turn, while the fix is still
uncommitted. Do not invoke `/sase_git_commit`.

## 3. Close epic bob-cli-5s.10

1. Run `sase bead epic-symbols bob-cli-5s.10`. There were none at landing. If any
   `--epic-symbol` entry is listed, resolve it (wire it, privatize it, add a non-test
   pragma, or delete it) or, only when a still-open later bead needs the exemption,
   re-key that Justfile line to that open bead. Do not leave any entry keyed to this
   epic. `sase bead close` refuses while one remains. Never use `--force` to get past
   that refusal or to close an unfinished descendant.
2. Close with this note (one shell argument, or `@` a file):

```text
LAND bob-cli-5s.10. Verified all four closed phases against the plan and the epic commits (bob-cli b566ba4; bob-mac-capture 3a5fd4a..d5fcac0). Blocked sits after reading_state_source, .build is untracked, the two-tracker [?] fixture asserts blocked == false, and docs/ref.md records the client note and schema_version 1 additivity. Refs behavior matches the land audit: setQuery, represent-without-reset through BobPanelCoordinator, unavailable rows kept and dimmed, workspace wake, Today on every open, git lane after publish, calendar days, Ready added-desc, approximate git dates, weekday 2...6, frecencyHalfLifeDays, prepared-item cache, refs-rank --now, content well, Reduce Transparency, inspector honesty, 3 s intrinsics timeout, and row-anchored Cmd-K. highlights_open_key stays cmdO. No memory edits. Final CI https://github.com/bobs-org/bob-mac-capture/actions/runs/37949647297 is green on d5fcac0 after a --failed rerun. No non-epic commits landed in either repo after the epic started. Follow-ups, also in epic note #2: completion CLI flakes recorded on bob-cli-3j (not this epic); return_links declined as duplicate of bob-cli-5t; the just-install RefsCaption parse break was fixed by 9c46702; panelPresenter left unused and declined because represent is the spec path. Remaining race fixed here: testRefreshIfStale now waits for the git-date backfill of ref/blogs/small_opened.md before sampling argv counts, the same settle the wake test uses, so a late -g line cannot fail the quiet-window compare. No --epic-symbol entries.
```

Command: `sase bead close bob-cli-5s.10 --note "<note>"`. 3. Run `just symvision` from
the bob-cli workspace. It must pass. If it is missing, say so in the final response and
continue. 4. Set `status: done` in the frontmatter of
`plan:202610/bob_refs_land_fixes.md` (the PLAN path from
`sase bead read bob-cli-5s.10 -r "Need the plan path"`). Change only that field.

## 4. Parent epic bob-cli-5s

`sase bead read bob-cli-5s.10 -r "Need the parent link"` shows parent `bob-cli-5s`, a
plan bead (tier epic), not a phase. It has no parent of its own. After `bob-cli-5s.10`
is closed, every descendant of `bob-cli-5s` is closed: phases 5s.1–5s.9 were already
closed, and this child was the only open descendant.

Recheck before closing the parent:

- Read `bob-cli-5s` again, including notes #1 (follow-up triage) and #2 (the land audit
  whose ten gaps this child fixed).
- Run `sase bead epic-symbols bob-cli-5s`. None were listed at landing. Resolve any that
  appear, the same way as in section 3. Do not re-key an entry onto a closed bead.
- Confirm you did not leave bob-cli or bob-mac-capture with an unresolved spec gap from
  the audit in note #2. The only code change this tale makes is the test wait in
  section 2.

When that recheck still shows the parent complete, close it normally:

```text
sase bead close bob-cli-5s --note "RECHECK after child bob-cli-5s.10. Phases 5s.1-5s.9 were already closed; their follow-ups were triaged in note #1 (bob-cli-5u, bob-cli-5t, bob-cli-4k +1, and the declined items). Note #2's ten spec gaps are the child epic, now closed, including the stale-refresh argv race its landing tale fixed by waiting for the git-date lane in testRefreshIfStale. highlights_open_key remains cmdO. refs_decision_memory=no stands; the memory record is task bob-cli-5u, not an edit made here. No --epic-symbol entries. No non-epic drift to integrate. Parent plan marked done."
```

Then run `just symvision` again, and set `status: done` on the `status:` line in
`plan:202610/bob_refs_panel.md`.

Stop without closing `bob-cli-5s` if the recheck finds an incomplete or ambiguous gap.
Record that blocker with `sase bead note bob-cli-5s "<what is unfinished>"` and report
it. Do not use `--force`. There is no further ancestor to close: `bob-cli-5s` has no
parent bead.
