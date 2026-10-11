---
tier: tale
size: medium
title:
  Daily Stats landing — bring the review-journal writer up to its contract, then close
  bob-cli-6l
goal:
  Every counted ]s answer journals the outcome, host, and writer version the Daily Stats
  plan specified, journal failures notify once instead of every 15 seconds, and epic
  bob-cli-6l is closed with its plan marked done.
proposed_by: bbugyi200.apollo.bob-cli-6l.land
bead: bob-cli-6l
status: done
---

- **PARENT:**
  [202610/daily_stats.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/daily_stats.md)
- **BEAD:**
  [bob-cli-6l](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-6l/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-6l.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-6l.land.md)
- **COMMITS:**
  - [3888525](https://github.com/bobs-org/bob-cli--plans/commit/3888525eef3e04633b5ea76f0d0cac1d4b945431)
    — docs(daily-stats): mark parent plan done

# Daily Stats landing — review-journal writer fixes and epic closeout

## Context

This tale is the remaining work of epic **bob-cli-6l** (Daily Stats), found by its land
agent. Its plan is `plan:202610/daily_stats.md` (read it with
`sase bead read bob-cli-6l -r "<why>"`; the PLAN path it prints is the file this tale
marks done at the end). All six phases (bob-cli-6l.1–.6) and the nested epics
bob-cli-6l.2.1 and bob-cli-6l.5.1 are closed, the CLI and Obsidian Stats page work on
the live vault, and the epic DECISIONS (`ref_in_done = yes`, `stats_chip = yes`,
`memory_stats_record = no`, `memory_journal_term = no`) are honored. Do not edit any
SASE memory file.

The land review found that the **review-journal writer** shipped by phase bob-cli-6l.1
(bob-plugins commit `f4a2fe3`) misses parts of the plan's "What gets journaled" and
"Writer duties" sections. **No journal data exists yet anywhere** (no
`.bob/stats/review/` on apollo or the MacBook; the only `]s` walk on 2026-10-11 ran
05:42–06:30 EDT, before the writer shipped at 07:22), so the contract can be corrected
now with no migration. The next morning walk starts accruing permanent, non-backfillable
data, so these fixes matter before then.

### Gaps to close (all caused by this epic)

1. **Task Card answers always journal `card`.** The plan maps a Card answer by its
   after-line: `[x]`/`[X]` → `done`, `[-]` → `drop`, scheduled moved past today →
   `defer`, a new `dependsOn` → `sequence`, otherwise `card`. Nothing emits `defer` or
   `sequence` today. `continueReviewWalkAfter` in
   `plugins/bob-navigation-hotkeys/src/536-plugin-review-advance.js` (~line 425) passes
   only `input.journalOutcome || input.outcome || input.kind` (a string) to
   `recordReviewAnswer`, discarding the `beforeLine`/`afterLine` that the Card callers
   send (`350-picker-task-card.js` ~line 703, `455-picker-inbox-route-gate.js` ~line
   573, both `{kind: "card", beforeLine, afterLine, handledRefs}`).
2. **Host naming differs from the plan.** `235-plugin-stats-journal.js` keeps case and
   dots (`replace(/[^A-Za-z0-9._-]/g, "-")`) and throws when empty. The plan says:
   lowercase, cut at the first `.`, non-`[a-z0-9-]` → `-`, trimmed, fallback `host`. The
   MacBook's hostname is `Kellys-MBP`, so today it would write
   `2026-10.Kellys-MBP.jsonl` instead of `kellys-mbp` (which also matches the
   `vault(kellys-mbp)` label that `bob vault-sync` already uses). The plan's
   `api.stats.host()` was never added.
3. **`plugin` is hard-coded** to `bob-navigation-hotkeys@2.x` in ledger-tools. The plan
   has navigation add `plugin: "bob-navigation-hotkeys@<manifest version>"`. Phase
   bob-cli-6l.1 also skipped the plan's manifest bump for bob-navigation-hotkeys (still
   `2.18.0`, the pre-journal version), so a journal line cannot identify the writer
   build.
4. **Notice policy.** In `545-plugin-review-journal.js`, every failed flush shows
   "Review answer not recorded" and reschedules a 15 s retry, so a missing `api.stats`
   (ledger-tools disabled or old) or a persistent write failure shows a Notice **every
   15 seconds** indefinitely. A ledger-side validation `TypeError` also pins the queue
   head forever, since a retry can never succeed. The plan requires one Notice per
   failure burst ("Review answer not recorded — will retry"), a recovery Notice ("Review
   journal recovered: N answers written"), a 500-event retry cap that drops the oldest
   with a counted warning, and one Notice per session ("Update Bob Ledger Tools to
   record review stats") when `api.stats` is missing or older than v1.

The bob-cli reader (`src/native/stats/`) already handles every outcome in the
vocabulary, reads `host` from each event, and unions every `*.jsonl` file, so **no Rust
engine change is needed**. Only the fixture and docs change on the bob-cli side.

## Repositories

- **bob-plugins** (linked repo): open it with `sase repo open bob-plugins -r "<why>"`
  and use only the printed path. Read its `AGENTS.md`: edit `src/` fragments only, never
  the generated `main.js`; run `npm run build`; keep every fragment ≤ 1000 lines
  (`536-plugin-review-advance.js` is at 966, so keep its net change near zero and put
  new logic in `545-plugin-review-journal.js`).
- **bob-cli** (this workspace): fixture, docs, `just check`.

## Changes

### 1. Card after-line outcomes (bob-navigation-hotkeys)

- In `545-plugin-review-journal.js`, add a pure helper (for example
  `reviewCardJournalOutcome(beforeLine, afterLine, dayText)`). It applies the plan
  mapping in this order:
  1. after-line checkbox `[x]`/`[X]` → `done`;
  2. `[-]` → `drop`;
  3. after-line scheduled date strictly later than `dayText`, and different from the
     before-line's → `defer`;
  4. after-line `dependsOn` ids include one absent from the before-line → `sequence`;
  5. otherwise `card`.

  Reuse the existing navigation parsers for checkbox status, `[scheduled:: …]`, and
  `[dependsOn:: …]` instead of new regexes. `reviewOutcomeResolves` in 536 and
  `538-review-walk-identity.js` (~line 81, `kind !== "card"`) already inspect Card
  after-lines, so start there.

- Make `reviewJournalOutcome` accept the continue input object. When `kind === "card"`
  and no explicit `journalOutcome` is set, it should return the after-line mapping using
  the walk day (`origin.day`). Change the 536 call to pass the input object rather than
  the reduced string (net zero lines in 536).
- Export the helper through `NavigationHotkeysPlugin.helpers` (`700-exports.js`) beside
  `buildReviewAnswerEvents` and `reviewJournalOutcome`.

### 2. Host naming and `api.stats.host()` (bob-ledger-tools)

- In `235-plugin-stats-journal.js`, add a pure `statsReviewHost(raw)`: lowercase → cut
  at the first `.` → replace every character outside `[a-z0-9-]` with `-` → trim leading
  and trailing `-` → `host` if empty. It never throws. Vectors: `Kellys-MBP` →
  `kellys-mbp`; `apollo` → `apollo`; `athena.tail297af1.ts.net` → `athena`;
  `My Mac_2.local` → `my-mac-2`; `.local` → `host`; empty → `host`.
- `apiStatsAppend` uses it for both the file name and the `host` field. Add
  `host: () => …` to the frozen `stats` namespace in `170-plugin-lifecycle.js`. Keep
  `version: 2`: the addition is additive, and v1 already promised `host()`. Export the
  sanitizer through `350-exports.js`.

### 3. `plugin` field

- Navigation adds `plugin: "bob-navigation-hotkeys@" + this.manifest.version` to each
  event, falling back to `@2.x` when `this.manifest` is unavailable (as in tests).
- Ledger-tools keeps a caller `event.plugin` matching `/^[a-z0-9-]+@\d+\.\d+\.\d+$/`;
  otherwise it writes the existing `bob-navigation-hotkeys@2.x` fallback.

### 4. Notice, retry, and cap policy (bob-navigation-hotkeys `545`)

- **Missing or old API** (no `bob-ledger-tools`, no `api.stats.append`, or
  `api.stats.version < 1`): show "Update Bob Ledger Tools to record review stats" once
  per plugin session. Keep events pending and keep retrying quietly on the existing 15 s
  timer and on the next answer, so a late-loading ledger-tools still receives them. Show
  no "not recorded" Notice in this state.
- **Write failure burst** (`append` rejects for a non-validation reason): one Notice,
  "Review answer not recorded — will retry", when the burst starts. No more Notices
  until a drain fully succeeds.
- **Recovery:** when a drain succeeds after a burst, show one Notice, "Review journal
  recovered: N answers written", where N is the number of events that drain wrote. Then
  reset the burst state.
- **Validation rejects:** have ledger-tools mark validation errors (for example
  `error.code = "invalid_review_event"` on the `TypeError`s it throws). Navigation then
  drops such an event with a `console.warn` instead of retrying it forever, and
  continues the drain.
- **Cap:** at most 500 pending events. On overflow, drop the oldest, count the drops,
  and `console.warn` the running count once per burst.
- **Unchanged:** an answer never fails, throws, or waits because of the journal. Leave
  walk behavior, advance rules, toasts, and locks alone. Cleanup on unload already
  exists; extend it to any new state.

### 5. Manifests, build, tests, deploy (bob-plugins)

- Bump `bob-navigation-hotkeys` `2.18.0` → `2.19.0`. Append to its description one
  sentence saying it journals each counted `]s` answer, with its outcome, through the
  bob-ledger-tools `api.stats` namespace. Bump `bob-ledger-tools` `1.47.0` → `1.48.0`,
  updating its description if it enumerates the stats namespace members.
- Tests:
  - Extend `scripts/test-navigation-review-journal.cjs` with: each Card mapping
    (done/drop/defer/sequence/card, including scheduled-but-not-past-today → `card`);
    the `plugin` field; one Notice per burst plus the recovery Notice; one missing-API
    Notice per session with events retained and later delivered; the validation-reject
    drop; and the 500 cap.
  - Extend `scripts/test-ledger-tools-stats-journal.cjs` with the host vectors,
    `api.stats.host()`, and `plugin` passthrough/fallback.
  - Copy fixture cases inline and cite `tests/fixtures/stats/review_answers.json`, as
    the existing tests do.
- Run `npm run build`, `npm run build:check`, `npm run validate`, the targeted suites
  (`node --test scripts/test-navigation-review-journal.cjs scripts/test-ledger-tools-stats-journal.cjs scripts/test-ledger-tools-stats-data.cjs scripts/test-ledger-tools-stats-block.cjs scripts/test-navigation-review-advance.cjs scripts/test-navigation-review-advance-gestures.cjs scripts/test-navigation-checklist.cjs scripts/test-navigation-checklist-ctrl-enter.cjs`),
  then the full `npm test`.
  - Known clean-base failures, not this tale's work: successor/unblocked-glyph
    date-pinned fixtures (DISCOVERED ISSUE on bob-cli-5w), picker-single clock
    (bob-cli-5r), and stage-ranker timing (bob-cli-3w).
  - Confirm any other failure against a clean-base stash before calling it pre-existing.
- Deploy to this machine's vault with
  `bob plugins sync --repo <the bob-plugins path printed by sase repo open>`, then
  confirm the vault's `.obsidian/plugins/bob-navigation-hotkeys/manifest.json` reads
  `2.19.0`.
- Do **not** run `just install-all` or `bob plugins sync` on the MacBook over SSH: it
  quits and relaunches Bryan's Obsidian. Instead, state in the final response that Bryan
  should run `just install-all` on the MacBook before the next morning walk. Until then
  the Mac writes the old host spelling, which is harmless because the reader unions
  every file.

### 6. bob-cli fixture and docs

- `tests/fixtures/stats/review_answers.json`: append R20–R24 for the Card mappings
  (`done`, `drop`, `defer`, `sequence`, plain `card`) with additive `before`/`after`
  line fields. Add an additive top-level `hosts` array of `{raw, host}` vectors from §2.
  Then update the Rust loader test in `src/native/stats/journal/tests.rs` (~line 292) if
  it pins the count or the R1–R19 ids, and make sure the fixture still deserializes.
- `docs/stats.md` "Review journal" section:
  - document the host rule with the `kellys-mbp` example;
  - document `plugin` as `<writer>@<manifest version>`, noting that earlier lines may
    carry `@2.x`;
  - document the Card after-line outcome mapping;
  - document `api.stats.host()`;
  - replace "the plugin shows “Review answer not recorded.”" with the §4 Notice, retry,
    and cap policy;
  - update the conformance text, including the later "R1–R19" mention (~line 535), to
    R1–R24 plus the host vectors.
- Run `just check`. Expected: fmt and clippy pass, and the only failures are the two
  `capture::r#ref` percent-listen CLI tests tracked by bob-cli-6m. Do not run
  `just check-full`.

## Final step — close epic bob-cli-6l

1. Run `sase bead epic-symbols bob-cli-6l`. At landing review it reported "No
   --epic-symbol entries for bob-cli-6l". If any entry appears, resolve it (wire it up,
   privatize it, add a non-test pragma, or delete it per the Symvision epic-whitelist
   policy). Re-key it to an open bead only if a still-open later bead needs it.
2. Close the epic. Use this note, adjusted to your actual results:

   ```bash
   sase bead close bob-cli-6l --note "Verified phases 6l.1-6l.6 and nested epics 6l.2.1 (6 phases) and 6l.5.1 (4 phases) closed with every child note addressed; reviewed the epic's 13 bob-cli commits (1023e6c..f11ca41) and bob-plugins f4a2fe3/db91d68/f3bc5e2/2e7f3b0/fa7db2f. DECISIONS honored: ref_in_done=yes (done/created/dropped include #ref/^ref as the ref kind part; live bob stats show done parts task/ref/prj), stats_chip=yes (label-only STATS chip in dash.md Browse row and docs/dashboard.md), memory_stats_record=no and memory_journal_term=no (no sase/memory changes). Integration: no non-epic commits landed in bob-cli or bob-plugins after the epic began. Live vault: bob stats installed on apollo (0.34 s, json ok); Mac bob serves bob stats; the Mac has journal-capable plugins and no journal lines exist yet because the only walk predated the writer. Landing tale fixed the writer contract: Card after-line outcomes, plan host naming plus api.stats.host(), plugin@manifest version, one-Notice-per-burst/recovery/missing-API/500-cap policy, R20-R24 and host vectors, docs; navigation 2.19.0, ledger-tools 1.48.0 synced to apollo's vault. just check: only the two bob-cli-6m percent-listen failures. Follow-ups per landing triage note: +1 bob-cli-6m; new bob-cli-6n (perf), 6o (snapshot kind), 6p (closeout toast), 6q (four-week revisit, snoozed to 2026-11-09); npm clean-base failures already routed to 5w/5r/3w."
   ```

   Never use `--force` merely to make the close succeed. If the close is rejected for
   leftover `--epic-symbol` entries, finish that cleanup and close again.

3. Run `just symvision` if the justfile has that recipe. At landing review it had none,
   so skip it and say so.
4. Set `status: done` in the frontmatter of the epic's plan file (the PLAN path printed
   by `sase bead read bob-cli-6l -r "<why>"`, i.e. `202610/daily_stats.md` in the plans
   repo). It currently reads `status: wip`.
5. bob-cli-6l has no `parent_bead`, so the landing ends there.
