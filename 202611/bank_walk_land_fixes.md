---
tier: tale
size: medium
title: Bank walk family b — fix landing-audit bugs and contract drift, then close
  bob-cli-lm
goal: 'The bank walk is correct and matches its binding contract: no extra full-vault
  scan at startup, correct planner edge cases, a reliable index, a footer meter that
  fetches at most once per generation and never shows a stale ✓, and the exact nav
  palette names, notices, and landing toast. Then epic bob-cli-lm is closed.'
proposed_by: bbugyi200.apollo.bob-cli-lm.land
proposed_with: claude/opus
bead: bob-cli-lm
status: done
---

- **PARENT:**
  [202611/bank_walk_family_b.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bank_walk_family_b.md)
- **BEAD:**
  [bob-cli-lm](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-lm/README.md)

# Plan: finish epic `bob-cli-lm` — bank walk audit fixes and closeout

## Why

Epic `bob-cli-lm` (plan `plan:202611/bank_walk_family_b.md`, the **binding design
contract**) shipped `]b` / `[b` / `]B` / `[B`, the `b` walker-footer panel, and the
Category-card and badge hand-off across five closed phases. Its land agent then audited
the code against the contract and found real defects. **This epic caused all of them**,
so they are still epic work:

- One full-suite regression: the bank-walk index adds an extra full-vault Markdown walk
  at startup.
- Planner and index bugs that the phase tests did not exercise.
- An account-meter fetch loop that can hang Obsidian.
- Nav copy, palette names, and toast layout that drifted from the contract's exact
  strings.

Follow-up triage is already done and recorded on `bob-cli-lm`, so do not re-triage. This
tale fixes the defects, redeploys, and closes the epic.

Before editing, read the epic contract with
`sase bead read bob-cli-lm -r "Need the bank walk contract"`. That command prints the
PLAN path; read the plan's "Shared design contract" section. Its feedback-copy table,
its command table, and "The landing toast (nav)" are quoted exactly below where they
matter.

## Where the code lives

Every code change is in the **bob-plugins** linked repo. Open it with
`sase repo open bob-plugins -r "Fix bank walk epic bob-cli-lm audit findings"` and
follow its `AGENTS.md` exactly:

- Edit `src/` fragments only. Keep every hand-edited fragment at or below 1000 lines.
- Run `npm run build` after fragment edits.
- Keep `npm run build:check` and `npm run validate` green.
- Register any new test file in `package.json` `test:unit`.
- Run `bob plugins sync` at the end. Never pass `-F` on your own; follow `AGENTS.md` if
  sync refuses.

The ledger plugin is `plugins/bob-ledger-tools`; the nav plugin is
`plugins/bob-navigation-hotkeys`. Fragment names below are relative to each plugin's
`src/`.

## 1. Startup regression: no full-vault walk for bank months (ledger `249a`)

`scripts/test-ledger-tools-note-ready.cjs` "the startup storm is linear" now fails with
`3 !== 2` Markdown walks. Bisected to `2562300` (bob-cli-lm.1). The cause is that
`bankWalkStartInitialScan` (`249a-plugin-bank-walk-index.js`, around line 240) calls
`vault.getMarkdownFiles()` at layout-ready.

Month notes are exactly `bank/YYYY-MM.md` (`BANK_MONTH_PATH` in
`241i-plugin-bank-source.js`). So discover them from the `bank` folder's children, which
costs O(months), not a full-vault walk:

- Use `vault.getAbstractFileByPath("bank")` (or `getFolderByPath` when present) and its
  `children`, then filter with `bankDiscoverMonthPaths`.
- When there is no `bank` folder, become ready with no months.
- Fall back to `getMarkdownFiles()` only when the vault exposes no folder API.

The note-ready test must pass **unchanged**. Add a bank-walk test asserting that the
initial build makes zero `getMarkdownFiles` calls when a folder API exists.

## 2. Index lifecycle bugs (ledger `249a`)

1. **Per-path debounce.** The debounce is one global timer, so edits to two month notes
   within 250 ms re-read only the last one. Keep a pending set (or a timer per path) so
   every changed month is re-read.
2. **Delete and rename-away.** A delete schedules a read that fails silently, so the
   deleted month's rows stay indexed. On `delete`, and on `rename` away from a month
   path, remove that path from the disk map, bump the generation, and fire
   `onBankWalkChange`. A rename onto a month path reads the new path.
3. **Overlay retention (contract Reliability invariant 3).** Today an overlay is dropped
   as soon as its editor closes. Tick a row in the editor, close the note before the
   debounced re-read, and the row is WAITING again. Fix: retain the last overlay content
   per path, with a change sequence. Retire it only when a disk read that was
   **scheduled after the overlay's last change** completes. Keep the existing rule that
   an older-generation disk read never overwrites a newer overlay.
4. **Per-path scan memo** (contract drift). Memoize `bankScanSource` per path on the
   exact content. Then an overlay keystroke rescans only that note, not every month. The
   snapshot memo keeps keying on (disk generation, overlay signature). The signature
   should use a cheap per-path change sequence or hash rather than concatenating every
   overlay's full text.

## 3. Planner bugs (ledger `249-plugin-bank-walk-model.js`, `bankWalkPlan`)

The cursor-gap and anchor-gap branches copy each other's logic and have drifted. Factor
one helper, `bankWalkStepFromTuple(entries, days, tuple, request)`, that both branches
call. The member-row branches stay as they are.

1. **Cursor above the first day heading** (around line 564). It currently gets
   `date: ""`, which sorts before the whole vault. Per the contract, it sorts **before
   that month's first day**: use a tuple date just below that month's first day, e.g.
   `<YYYY-MM>-00` from the note's month key.
   - From line 1 of `bank/2026-11.md` in the fixture, `]b` / `]B` must land the first
     November scope row, not October's.
   - `[b` / `[B` must land the last row / first row of the last day before November,
     with no wrap.
2. **Anchor-gap day jumps** (around lines 703-717). These use
   `days[i].date >= anchor.date`, so when the anchor's row is gone but its day still has
   scope rows, `]B` lands on that same day. Like cursor-gap, a day jump from inside a
   day must leave it: `]B` → next day's first row, `[B` → previous day's first row.
3. **Counted jumps from a daily note** (around lines 647-655). `N]B` moves rows instead
   of days, and `N[B` moves forward. Define the daily-note origin as "the daily day's
   first scope row is landing 1", then count from there:
   - `N]B` lands the first row of the day `N-1` days later.
   - `N[B` lands the first row of the day `N-1` days earlier.
   - `N]b` lands `N-1` rows after the day's first row.
   - `N[b` lands `N-1` rows before the day's last row.

   Set `wrapped` when a count crosses an edge. Delete the dead `wrapped` / `spilled`
   locals.

4. **The daily origin must not need an integer `line0`**: key it only on the active
   note's `YYYYMMDD` stem, so a null `locate` result still takes the daily rule.
5. Remove the leftovers: `true ||` (around line 630), unused locals (around 756-790),
   and `this.bankGeneration ||` in `249a` (around line 268).

## 4. Tests and parity fixture (ledger)

In `scripts/test-ledger-tools-bank-walk.cjs`:

- **Parity checks the index, not the scanner.** Build the snapshot with
  `bankWalkBuildSnapshotFromContents` in ALL scope from the fixture months. Assert its
  entries `(path, line, date)` and the `waiting` flags equal `bob-bank-list-all.json`.
  Then delete the unused `snap`.
- **Fix the regeneration command in the header.** The fixtures live in
  `plugins/bob-ledger-tools/scripts/fixtures/bank-walk/`. State that real path and the
  working directory the command runs from, so the command works as written.
- **Add regression tests** for every item in sections 1-3:
  - zero `getMarkdownFiles` calls at startup;
  - a two-month edit burst re-reads both months;
  - delete and rename-away drop rows;
  - a closed-editor tick stays out of WAITING until a later disk read;
  - an older read never beats a newer overlay, tested through the mixin, not only the
    pure builder;
  - cursor above the first heading, all four keys;
  - anchor-gap `]B` / `[B` when the anchor's day still has rows;
  - counted `N]B` / `N[B` / `N]b` / `N[b` from a daily note.

## 5. Footer account meter (ledger `125-bank-walker-footer.js`)

1. **Fetch loop (high severity).** `bankWalkerFooterRequestList` clears
   `bankWalkerFooterMeterPending` on completion and repaints.
   `bankWalkerFooterTakeMeter` then re-requests the same generation whenever the result
   is older, has no report, or failed. This loops without yielding and can hang
   Obsidian, or re-run `bob bank list` over and over. Fix:
   - Record the **attempted** generation and never request it again after any
     completion: success, error, older generation, or empty report.
   - Request again only once `bankVaultGeneration` advances.
2. **Stale `✓ balanced`.** On a `bob` failure, the data service returns the last good
   report stamped with the new generation (`241j-plugin-bank-data.js` around lines
   271-279). The footer ignores `error` and `stale`. Fix:
   - Only a result with no `error`, not `stale`, and `generation >= current` may paint
     `✓ balanced`.
   - Any other result paints no meter, or the worst non-balanced status. It never paints
     `✓`.
3. **Cache key.** `bankWalkerFooterCachedList` looks up `bankCacheKey(["bank","list"])`,
   but the service stores the list under its built args. Use
   `bankCacheKey(bankBuildArgs({ command: "list" }))`.
4. **Contract cosmetics** (the panel table and Rules in the contract):
   - Drop the `]} next day` hint from the **footer's** ALL-on-a-row state, which lists
     no hint. The toast keeps its own hint.
   - Paint inflow with `--bob-bank-in-ink`, the badge's token, on the amount only, not
     the payee.
   - Put the WAITING warning ink on the scope word only, not on the `3/9` position.
5. **Tests** in `scripts/test-ledger-tools-walker-footer.cjs`:
   - an error result never paints `✓`;
   - an older cached result never paints `✓`;
   - overlapping and failed fetches make at most one `apiBankFetch` per generation, and
     a new generation fetches once more;
   - the correct cache key is used.

   Keep `scripts/preview-walker-footer.cjs` passing.

## 6. Nav copy, palette names, and toast (nav `490`, `549`, `549a`, `styles.css`)

1. **Palette names, exact** (`490-plugin-lifecycle.js`, around lines 351-379):
   - `jump-to-next-bank-row` → `Jump to next bank row (waiting rows first)`
   - `jump-to-prev-bank-row` → `Jump to previous bank row (waiting rows first)`
   - `jump-to-next-bank-day` → `Jump to the first row of the next bank day`
   - `jump-to-prev-bank-day` → `Jump to the first row of the previous bank day`
2. **Notices, exact** (`549-plugin-bank-walk.js`):
   - indexing → `Bank walk unavailable — still indexing bank notes`
   - empty → `No bank rows yet — bob bank import adds them` (also the
     `showBankWalkEmptyNotice` default in `549a`)
   - a target note that cannot be opened → `Could not open <path>`, with the entry's
     path. Keep `Bank rows changed — try again` for stale-twice.
   - The `api required` copy is already correct.
3. **Flip banner inside the toast.** `bankWalkObserveScope` currently shows a separate
   `Notice`, and its WAITING copy is wrong. Make it return the banner text and pass it
   into the toast model, where it renders as a banner line above the header:
   - flip to ALL → `Nothing waiting — ]b now walks every row`
   - flip to WAITING → `<N> waiting — ]b now walks waiting rows`, with `N` from
     `snapshot.counts.waiting`

   Show it once, from the first run that sees the new scope. If that run lands nothing,
   show the banner as a plain `Notice` so the flip is never silent. Keep the queued-run
   drop on a scope flip.

4. **Toast layout per the contract's "The landing toast (nav)" section:**

   ```
    $  WAITING  3/9                       ↻ wrapped
    ●●○●●●○   (day rail: one dot per scope day, current day lit; split bar past 30)
    Thu Nov 13 · 1 of 4
    $42.50  ShopRite
    November 2026 · Ctrl+Enter file
   ```

   - **Header:** the `$` glyph in the bank hue, the scope word, `rank/total`, and
     `↻ wrapped` when wrapped. WAITING is warning amber. ALL is **body ink**
     (`--text-normal`), not `--text-accent`.
   - **Day rail:** one dot per entry in `snapshot.days`, with the plan's `dayIndex` lit.
     Past 30 days, show a split bar instead of dots. This replaces the
     `bob-nh-bank-walk-progress` progress bar.
   - **Day line:** `entry.label · <dayRank> of <dayTotal>`.
   - **Money line:** `amountText`, green when `inflow`, then the payee.
   - **Last line:** the month label, then the hint. The hint is `Ctrl+Enter file` on a
     waiting row and `]} next day` in ALL. For the month label, add an additive
     `monthLabel` field (`November 2026`, from `bankWalkMonthLabel`) to every ledger
     snapshot entry. Nav falls back to formatting `entry.month` (`YYYY-MM`) when the
     field is absent. Adding the field keeps `api.bankWalk` at version 1.
   - **Plain-text fallback:** keep a single-line fallback that mirrors the card,
     following the lane-toast pattern in `543-lane-walk-notice.js`.
   - The toast still replaces the previous bank toast rather than stacking.

5. **Tests:**
   - Update `scripts/test-navigation-bank-walk.cjs` to the exact copy above. Add the
     flip banner (both directions, shown once), the could-not-open path, and the toast
     model (day rail count and lit index, day line, month label, hint per scope).
   - Extend the ledger bank-walk test for the `monthLabel` field.
   - Run `npm run docs:gen` so the commands inventory picks up the new palette names.

## 7. Docs

In bob-plugins `docs/bob-navigation-hotkeys.md`:

- The regenerated command rows come from `npm run docs:gen`.
- Fix the hand-written uppercase-repeat table's `b` row to the new day-command wording
  if it quotes the palette names.
- Make sure the Bank walk section's toast sentence matches the shipped toast: scope
  word, position, day rail, day and amount lines, month and hint.

The page sat at **5993/6000** words for the reference budget, and the new names add
about 6 words. Run `npm run docs:check`. If the page goes over budget, trim prose
elsewhere in this page's Bank walk section rather than raising the cap.

No bob-cli docs change is needed: `docs/reference/plugins/` is a gitignored mount.

## 8. Verify and deploy

In bob-plugins:

1. Run `npm run build`, `npm run build:check`, `npm run validate`, and
   `npm run docs:check`.
2. Run the focused suites (`node --test scripts/<file>.cjs`):
   - `test-ledger-tools-bank-walk`
   - `test-navigation-bank-walk`
   - `test-ledger-tools-walker-footer`
   - `test-navigation-bracket-repeat`
   - `test-navigation-bracket-mode-indicator`
   - `test-ledger-tools-bank-category-card`
   - `test-ledger-tools-bank-category-card-ui`
   - `test-ledger-tools-bank-badge-hosts`
   - `test-ledger-tools-bank-period-hosts`
   - `test-ledger-tools-freshness-footer`
   - `test-navigation-lane-walk`
   - `test-ledger-tools-navigation`
   - `test-ledger-tools-note-ready`, which must now pass the startup-storm test
3. Run `node scripts/preview-walker-footer.cjs` if it runs headless here.
4. Run the full `npm test`. The only failures allowed are the four tracked timing
   flakes: bob-cli-io (frees-marks), bob-cli-km (event-link-times), bob-cli-ki
   (keep-moving-marks-surface), and bob-cli-ju (off-clock-link-chips). Any other failure
   is this tale's to fix.
5. Run `bob plugins sync`, then `bob plugins list --no-pull`. Confirm `bob-ledger-tools`
   and `bob-navigation-hotkeys` read `synced`.

Mac deployment is tracked separately in bob-cli-lg; do not attempt it here.

## 9. Close out epic bob-cli-lm (final step)

Do this in the same turn as the final code. Do not wait for this tale's own commit,
push, or CI.

1. Run `sase bead epic-symbols bob-cli-lm`. At planning time it printed "No
   --epic-symbol entries". For any entry that now appears, resolve the symbol: wire it
   up, privatize it, add a non-test pragma, or delete it per the Symvision
   epic-whitelist policy. Do not leave it for a later agent.
2. Run `sase bead close bob-cli-lm --note "<verification>"`. The note summarizes:
   - the five phases verified against the contract;
   - the audit fixes in sections 1-6, with test counts;
   - the full `npm test` result, with only the four tracked flakes failing;
   - the `bob plugins sync` result;
   - the follow-up triage already recorded on the epic: io/km/ki/ju +1, lg +1, the
     bob-cli-lj DISCOVERED ISSUE for json-outputs, and the declined resolved vault-sync
     refusal.

   Never pass `--force` to make the close succeed. If the close is rejected for leftover
   epic-symbols, finish that cleanup and close again.

3. Run `just symvision` in the bob-cli repo and confirm the whitelist is clean.
4. In the epic's plan file, the PLAN path printed by `sase bead read bob-cli-lm`
   (`plan:202611/bank_walk_family_b.md` in the plans sidecar), change the frontmatter
   `status: wip` to `status: done`.
5. `bob-cli-lm` has no `parent_bead`, so there is no ancestor to close.

## Out of scope

- Mac deployment (bob-cli-lg).
- The json-outputs docs budget (recorded on bob-cli-lj).
- The four timing flakes (their own beads).
- Any change to the settled contract decisions (A1-A8, OQ).
- New `api.bankWalk` versions; `monthLabel` is additive.
- Memory edits: the epic's only authorized memory edit, glossary:walker-footer, is
  already done and stays accurate.
