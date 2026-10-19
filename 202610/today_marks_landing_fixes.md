---
tier: tale
size: medium
title: 'Land Today marks: fix the remaining spec deviations, then close bob-cli-9r'
goal: 'Fix the defects the bob-cli-9r landing review found in the bob-ledger-tools
  Today marks: reduced motion, `::` in tooltips, the Live Preview builder gates, the
  keyboard command, and reading-view placement, code, clicks, and mapping. Add the
  missing tests, release and sync the plugin, then close epic bob-cli-9r.

  '
proposed_by: bbugyi200.athena.bob-cli-9r.land
proposed_with: claude/opus
bead: bob-cli-9r
status: done
---

- **PARENT:**
  [202610/today_marks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/today_marks.md)
- **BEAD:**
  [bob-cli-9r](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-9r/README.md)

# Plan: land Today marks (bob-cli-9r)

## Context

Epic `bob-cli-9r` ("Today marks: a 🍅 on task lines that are on today's plan") has five
closed phases:

- **bob-cli:** commit `1b158e66` (`docs/today-marks.md`, the authoritative display
  contract with vectors TM1–TM23).
- **bob-plugins:** commits `ee9d7e6`, `66a9597`, `e4c71bc`, and `035296a`, all in
  `plugins/bob-ledger-tools/`.

The landing review verified the following on bob-plugins `035296a`:

- `npm run build:check` and `npm run validate` pass.
- The Today suites pass: `today` 32/32, `today-marks` 25/25, `today-marks-reading`
  14/14.
- Full `npm test` runs 3168 tests with 6 failures. All six are pre-existing and tracked:
  `bob-cli-6v` (4 unblocked-glyph tests), `bob-cli-6w` ("Keep counts once…"), and
  `bob-cli-7i` ("forward conversion carries a Depends-On line…").
- The vault plugin is synced at 1.59.0.
- The bob-cli docs match the plan's TM table exactly.

The review also found the defects below. Each one is a deviation from the epic's own
spec, which is plan `plan:202610/today_marks.md` and bob-cli `docs/today-marks.md`. That
makes them remaining epic work. Follow-up triage is already finished and recorded on the
epic: memory task `bob-cli-9t` was created, and `bob-cli-6v`, `bob-cli-6w`, and
`bob-cli-7i` were corroborated. **Do not create any beads.**

The epic decisions still hold: `keyboard_jump = yes` (the "Go to today's Task Link"
command ships), and `memory_today_mark_record = no`, so **edit no memory files**.

## Where the work happens

- **bob-plugins**, a linked repo. Open it with `sase repo open bob-plugins -r "<why>"`
  and use the printed path. Read its `AGENTS.md` first.
  - Work only in `plugins/bob-ledger-tools/` plus
    `scripts/test-ledger-tools-today*.cjs`, the root `README.md` table row, and
    `package.json` if a new test file is added.
  - Edit `src/` fragments, never `main.js`, and run `npm run build`.
  - Every hand-edited fragment must stay at or below 1000 lines.
    `src/276-plugin-today-marks-reading.js` is already at 964 lines, so put new pure
    helpers in `src/143-today-marks.js` (544 lines). If 276 would still overflow, move
    the delegated-click methods (`todayMarksBindReadingClicks`,
    `setupTodayMarksReadingClicks`, `handleTodayMarkReadingClick`) into a new fragment,
    `src/277-plugin-today-marks-reading-clicks.js`. Register it in `fragments.json`
    after 276 and install its mixin in `310-install-methods.js` after the reading mixin.
- **bob-cli** (this repo): change `docs/today-marks.md` only if a fix below changes
  documented behaviour. The fixes are meant to make the code match the doc, so a doc
  edit should not normally be needed.

Every new or changed test injects its clock. Never mix a wall-clock daily path with
fixed dates; that is the `bob-cli-6v` anti-pattern.

## Fixes

### 1. Reduced motion must stop the entrance pop (`styles.css`)

`.bob-today-mark.is-entering { animation: … }` has specificity (0,2,0). It beats the
reduced-motion override `.bob-today-mark { animation: none; transition: none; }`, which
has (0,1,0). So the pop still plays under `prefers-reduced-motion: reduce`.

- Inside the existing `@media (prefers-reduced-motion: reduce)` Today block, change the
  selector to `.bob-today-mark, .bob-today-mark.is-entering`.
- In `scripts/test-ledger-tools-today-marks.cjs`, extend the CSS contract test to assert
  that the reduced-motion block's rule names `.bob-today-mark.is-entering` and sets
  `animation: none`.

### 2. Tooltips never contain `::` (`src/143-today-marks.js`)

Entry names are the raw text after `—`, so a ledger entry such as
`- [ ] () — BLOG [cat:: w]` produces `Today · queued in BLOG [cat:: w]…`. That breaks
the contract, because Dataview re-scans `innerHTML`.

- In `todayMarkSessionText` and in the final string from `todayMarkTooltip`, collapse
  every run of two or more `:` into a single `:` (`.replace(/:{2,}/g, ":")`).
- Add tests: a name containing `[cat:: w]`, and a home label containing `::`. The output
  contains no `::`, and the rest of the text is unchanged.

### 3. A pure inline-code helper (`src/143-today-marks.js`)

Add `todayMarkPosInInlineCode(lineText, pos)`. It returns true when UTF-16 offset `pos`
falls inside a CommonMark inline code span of `lineText`, backticks included.

- A backtick run of length n opens a span that closes at the next run of exactly n
  backticks.
- An unmatched opener is literal text.
- It never throws.

Export it through `src/350-exports.js` and unit-test it: single and double backtick
spans, an unmatched backtick, and `pos` before, inside, and after a span.

### 4. Live Preview builder gates, prefilter, memo, and clock (`src/275-plugin-today-marks.js`)

`buildTodayMarkDecorations` skips the spec's `todayIdsForPath` gate and id prefilter. It
also rebuilds an all-lines array on every rebuild whenever any candidate is visible, so
every keystroke in any area note costs O(N), even with nothing in Today. And it reads
`new Date()` / `Date.now()` directly.

- **Clock:** add a `todayMarksNow()` method that returns `new Date()` and that tests can
  override. The builder takes `now = this.todayMarksNow()`, then uses
  `isTodayCacheReady(now)` and `nowMs = now.getTime()`. The keyboard command's readiness
  check (fix 6) also uses `this.todayMarksNow()`.
- **Gates, in spec order:** enabled; Live Preview on; file path present; not
  `PLAN_DAILY_PATH_RE`; `isTodayCacheReady(now)`; then
  `ids = this.todayIdsForPath(path)` must be non-empty. Otherwise return
  `Decoration.none`. `path` is the host path with `.md`. Apply the id gate **before**
  resolving the syntax tree and walking lines.
- **Prefilter, in the line walk:** a line is considered only when it includes `#task`
  and `todayMarkTrailingBlockId(line.text)` is in `ids`. Only then call
  `todayMarkLineCandidate`.
- **Per candidate, in order:** skip code with `freshnessMarkPosInCode`; require the key
  in `todayCache.rank` and non-empty `todayDetailsForKey(key)`; then check duplicates.
- **Duplicates, lazily:** compute them only when the first candidate has passed the
  checks above.
  - Memoize in a module-level `WeakMap` keyed on `view.state.doc`. CodeMirror `Text` is
    immutable.
  - The stored value is `{ signature, duplicates }`, where `signature` is the sorted id
    list joined with `\n` and `duplicates` is the `Set` from
    `todayMarkDuplicateIds(lines, idsArray)`.
  - Use `doc.iterLines()` when it is a function, else a line-array fallback.
  - Recompute when the signature differs, for example after a cache rebuild changes the
    note's id set.
- **Tests:**
  - A note whose path has no Today ids returns `Decoration.none` without calling
    `doc.line` or `doc.iterLines`. Use a spy.
  - Two rebuilds on the same `doc` compute duplicates once.
  - The `todayMarksNow` override drives readiness.

### 5. Refresh timing and daily leaves (`src/275-plugin-today-marks.js`)

- `scheduleTodayMarksRefresh` coalesces into a single **16 ms** timer, per the spec. It
  is currently 150 ms, which puts appear and vanish times near the quarter-second
  budget.
- `refreshTodayMarkEditors` must skip leaves whose `leaf.view.file.path` matches
  `PLAN_DAILY_PATH_RE`.
- **Tests:** the refresh effect is dispatched to a non-daily leaf only. Update any test
  that assumed 150 ms.

### 6. The keyboard command applies the mark's line gates (`src/275-plugin-today-marks.js`)

`todayMarksTaskAtCursor` never checks the open status or the host path. So right after
`- [x] #task Fix ^fix` is completed, while its link is still in the ledger, "Go to
today's Task Link" still jumps, even though the mark has already gone.

- **Keep the "No task on this line" results:** a line with no exact `#task` range or no
  standalone trailing `^id`, a `#task` in code, and no syntax tree.
- **Return `Not on today's plan` for each of these:**
  - The line fails `todayMarkLineCandidate`, for example a closed status (`x`, `X`, `-`)
    or any other non-open status.
  - The host file matches `PLAN_DAILY_PATH_RE`.
  - The id is duplicated in the note.
  - The cache is not ready (`todayMarksNow()`).
  - The key is not in `todayCache.rank`.
- The session toggle and Live Preview mode do not gate the command; it is an explicit
  request.
- **Tests:** a closed-status line whose key is still in `rank` shows
  `Not on today's plan` and does not jump. A line in a canonical daily note shows
  `Not on today's plan`.

### 7. Reading view: no second jump from Live Preview marks (`src/276-…`, or 277 if split)

Live Preview widgets also carry `.bob-today-mark` and `data-bob-today-key`. So after the
widget's own `mousedown` jump, the document-level capture `click` handler runs a second
jump, and in failure cases shows the Notice twice.

- In `handleTodayMarkReadingClick`, return without acting (no `preventDefault`, no jump)
  when the mark has an ancestor with class `cm-editor`.
- **Test:** Mod+click on a mark nested in a `.cm-editor` element does not call
  `goToTodayTaskLink`. The existing rendered-row Mod+click test still jumps.

### 8. Reading view: mid-body `#task` goes at the body start (`src/276-…`)

The contract (`docs/today-marks.md` §Placement) says: "When `#task` is not the body's
first token, the mark goes at the body start, right after the checkbox and its space."
Reading view currently always inserts after the `#task` anchor.

- **In `todayMarkRenderRow`:** compute
  `atBody = candidate.tagFrom !== todayMarkBodyStart(sourceLine)`. When `atBody`, stamp
  `data-bob-today-place="body"` on the row; otherwise remove that attribute.
- **In `todayMarkInsertAnchor(li)`:** when the row's place attribute is `body`, use the
  checkbox anchor and skip the `#task` and `#ref` anchors.
- The sweep inserts through the same function, so it picks up the stamped placement too.
- The existing whitespace split in `todayMarkInsertAfter` keeps the mark directly before
  the title.
- **Test:** for `- [*] Fix the build #task ^fix`, the mark sits right after the checkbox
  and its whitespace, and not after the tag. This holds both from the post-processor and
  from a sweep-added mark.

### 9. Reading view: `#task` inside inline code stays neutral, TM21 (`src/276-…`)

- In `todayMarkRenderRow`, after `todayMarkLineCandidate` succeeds, return neutral (no
  stamp, no mark) when `todayMarkPosInInlineCode(sourceLine, candidate.tagFrom)` is
  true. This mirrors Live Preview, which skips the line when the first `#task` range is
  in code.
- **Test:** ``- [*] `x #task` Fix ^fix``, with the key in Today, gets no mark and no
  `data-bob-today-key`.

### 10. Reading view: refresh the key on an existing mark (`src/276-…`)

`todayMarkApplyToRow` updates only `aria-label` on an existing mark. A re-processed row
under a new path keeps the old `data-bob-today-key` on its mark, so Mod+click jumps to
the old key.

- When updating an existing mark, also set its `data-bob-today-key`, both the dataset
  and the attribute, to `key`.
- **Test:** re-render the same row with a new `sourcePath`. The mark's key follows the
  new path.

### 11. Reading view: check duplicates once per render (`src/276-…`)

`todayMarkRenderRow` calls `todayMarkDuplicateIds(lines, [id])` once per row. That is
O(rows × lines), about 35 ms per render on `sase.md`.

- Restructure `renderTodayMarksInWithText` into two passes:
  1. Resolve each row's source line and candidate.
  2. Call `todayMarkDuplicateIds(lines, allCandidateIds)` once, then stamp and apply per
     row.
- Behaviour is unchanged. Duplicate ids stay neutral (TM15).
- **Test:** a spy or counter shows one duplicate scan for a section with several rows.

### 12. Reading view: the fallback mapping follows the 280/281 pattern (`src/276-…`)

The row-index fallback in `todayMarkResolveSourceLine` compares the **task** rows
(`todayMarkCandidateRows(el)`) with **all** source list-item lines. So any plain child
bullet, such as a Depends-On line, makes the counts disagree and the fallback never
fires.

- Use every list item under `el`, `Array.from(el.querySelectorAll("li"))`, exactly as
  `src/280-plugin-dependency-render.js` and `src/281-plugin-blocks-render.js` do. Take
  the row's index among them, and accept the mapping only when that count equals
  `itemLines.length` and the mapped line's status equals the row's `data-task`.
- **Tests:** a task row without `data-line` that has one plain child bullet maps to its
  own source line. Disagreeing counts stay neutral.

### 13. Revalidation: layout-ready gate and clock steps (`src/290-plugin-today-and-location.js`)

- **Layout gate:** `revalidateTodayCache` returns `false` when
  `this.app && this.app.workspace && this.app.workspace.layoutReady === false`, the same
  gate `refreshTodayCacheForVaultEvent` uses. A focus or visibility event during startup
  can otherwise build a premature, partial cache. The later `onLayoutReady` or
  `resolved` rebuild would then treat that cache as "previously ready" and stamp every
  key, so marks would pop in at load.
- **Clock steps:** use `Math.abs(nowMs - last) < 1000` for the throttle, so a backwards
  clock step does not suppress revalidation until the clock catches up.
- **Tests** (in `scripts/test-ledger-tools-today.cjs`, injected clock):
  - With `layoutReady === false`, nothing refreshes.
  - With the clock stepped back one hour from the last revalidation, revalidation runs.

### 14. Missing Live Preview tests from the `live_preview` phase spec

Add these to `scripts/test-ledger-tools-today-marks.cjs`. Seed the cache through
`rebuildTodayCache` with a fixed clock wherever the vector involves the cache:

- **TM14 rollover:** rebuild for day N. Override `todayMarksNow` to day N+1; no
  decoration.
- **TM17:** assert the widget decoration has `side: -1` at `line.from + insertCh`.
- **TM18 entering:**
  1. Make the cache ready at time t.
  2. Rebuild with a newly linked key at t+50.
  3. At t+150 the widget's `entering` is true; at t+800 it is false.
- **TM20 toggle:** the body class flips, the Notice reads `Today marks off` and then
  `Today marks on`, and the refresh is dispatched only to non-daily leaves.
- **Rebuild triggers:** each of `docChanged`, `viewportChanged`, a Live Preview field
  change, and a file switch returns true. `selectionSet` alone returns false.
- **Jump (TM23):**
  - A second `jumpTodayTaskLink` call while the first is pending returns false and opens
    nothing.
  - With no open daily leaf, the jump opens through `openTodayDailyNoteWithState`.
  - A correct cached `ledgerLine` is chosen over a later match.
- **Home labels:** one test drives `rebuildTodayCache` with an unnamed timed entry and
  checks the home label and the 1-based `entryLine` in `todayDetailsForKey`. It must
  stub `XDG_CONFIG_HOME` or the plan caps so it never reads the host's real config.

## Release and verification (bob-plugins)

1. Bump bob-ledger-tools to `1.59.1`: the `version` in
   `plugins/bob-ledger-tools/manifest.json`, and the version cell of the Bob Ledger
   Tools row in the root `README.md` table. Change no description text.
2. Run `npm run build`, then `npm run build:check`.
3. Run each Today suite and confirm all pass:
   `node scripts/test-ledger-tools-today.cjs`,
   `node scripts/test-ledger-tools-today-marks.cjs`, and
   `node scripts/test-ledger-tools-today-marks-reading.cjs`.
4. Run `npm test`. Only the 6 known failures may remain (`bob-cli-6v` ×4, `bob-cli-6w`,
   `bob-cli-7i`). Confirm that any other failure also fails on a clean checkout of
   `035296a` before treating it as pre-existing. If it does not, fix it.
5. Run `npm run validate`, and confirm with `wc -l plugins/bob-ledger-tools/src/*.js`
   that no fragment exceeds 1000 lines.
6. Run `bob plugins sync`, so the vault gets 1.59.1. Then confirm that the vault's
   `bob-ledger-tools/manifest.json` reads `1.59.1`.

## Final step: close epic bob-cli-9r

Do this in the same turn as the code. The bob-plugins and bob-cli changes are committed
by the turn's finalizer, so nothing here waits on a commit, push, or CI.

1. Run `sase bead epic-symbols bob-cli-9r`. At planning time it printed
   `No --epic-symbol entries for bob-cli-9r.` If entries now appear, resolve each one:
   wire it up, privatize it, add a non-test pragma, or delete it per the Symvision
   epic-whitelist policy. Only when a still-open later bead still needs the exemption,
   re-key the Justfile line to that bead.
2. Close the epic. Fill in the bracketed parts with what you actually ran and saw:

   ```bash
   sase bead close bob-cli-9r --note "Landed. Verified phases 9r.1–9r.5 against plan:202610/today_marks.md and docs/today-marks.md (TM table matches); bob-cli 1b158e66 + bob-plugins ee9d7e6/66a9597/e4c71bc/035296a. Landing fixes in bob-ledger-tools 1.59.1: reduced-motion beats .is-entering, tooltips never contain '::', LP builder todayIdsForPath gate + id prefilter + WeakMap duplicate memo + todayMarksNow clock, 16 ms refresh and daily-leaf skip, keyboard command open-status/daily gates, reading: no double jump from .cm-editor marks, mid-body #task at body start, inline-code neutral (TM21), mark key refresh, one duplicate scan per render, 280/281 fallback mapping; revalidation layout gate and clock-step throttle; missing TM14/17/18/20/23, rebuild-trigger, and home-label tests. [Today suites N/N pass; npm test 3168+ tests with only bob-cli-6v x4, bob-cli-6w, bob-cli-7i failing; build:check/validate clean; fragments <=1000; vault synced to 1.59.1]. Integration: only bob-cli 8c10a820 (capture status-group headings) landed since the epic started; it does not touch Today marks; no bob-plugins commits after 035296a. Follow-ups: memory task bob-cli-9t created; bob-cli-6v/6w/7i corroborated; declined items recorded in the LAND TRIAGE note."
   ```

   Never pass `--force`. If the close is rejected for leftover `--epic-symbol` entries,
   finish that cleanup and close again. If it is rejected for unclosed phases, stop and
   report it; all five phases were closed at planning time.

3. Run `just symvision` in bob-cli if the recipe exists. At planning time the bob-cli
   `justfile` had no `symvision` recipe; in that case, say it was skipped because the
   recipe is unavailable.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN path
   shown by `sase bead read bob-cli-9r -r "Need the plan path for closeout"`,
   `202610/today_marks.md` in the plans repo, which currently reads `status: wip`.
   Change nothing else in that file.
5. `bob-cli-9r` has no `parent_bead`, so there is no parent to close. Finish normally.
