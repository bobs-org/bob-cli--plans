---
tier: tale
title: Retune Work badge color thresholds to 40/60/80
goal:
  TODAY, PENDING, NEXT, and READY badges in daily bob-plan blocks and dash.md turn green
  at >=40%, yellow at >=60%, and orange at >=80% of their limit, keeping grey at zero,
  red over the limit, and blue otherwise.
size: small
proposed_by: bbugyi200.athena.11g
proposed_with: claude/opus
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.11g](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.11g.md)
- **COMMITS:**
  - [1907419](https://github.com/bobs-org/bob-plugins/commit/1907419d0890a75e692fbc8141585be85a5dff5e)
    — feat(ledger-tools): retune Work badge color thresholds to 40/60/80

# Retune the Work badge color thresholds (40% / 60% / 80%)

## Outcome

The TODAY, PENDING, NEXT, and READY Work badges in daily-note `bob-plan` blocks and on
`dash.md` share one count-to-limit color rule. Lower the three intermediate bands. Green
starts at 40% (was 50%), yellow at 60% (was 75%), and orange at 80% (was exactly at the
limit). Orange now covers everything from 80% up to and including the limit. Grey at
zero, red strictly over the limit, and blue for everything else stay the same.

For an available nonnegative integer count `n` and a positive integer limit `L`, using
exact comparisons (never rounded displayed percentages):

| Condition                    | Color  | Was                      |
| ---------------------------- | ------ | ------------------------ |
| `n === 0`                    | grey   | same                     |
| `0 < n < 2 * L / 5`          | blue   | `0 < n < L / 2`          |
| `2 * L / 5 <= n < 3 * L / 5` | green  | `L / 2 <= n < 3 * L / 4` |
| `3 * L / 5 <= n < 4 * L / 5` | yellow | `3 * L / 4 <= n < L`     |
| `4 * L / 5 <= n <= L`        | orange | `n === L`                |
| `n > L`                      | red    | same                     |

Precedence: zero first, then strict excess, then descend through 80%, 60%, and 40%. The
separate `n === L` check goes away because 80% already covers it. Small and odd caps can
still skip bands (for example, cap 1 goes grey → orange → red).

Nothing else changes:

- the badges' count sources (section counts for PENDING/NEXT, the gated READY count, and
  TODAY's `budget.links`)
- the TODAY theme-overflow red override (`themes.count > themes.cap`; themes exactly at
  cap keep the link-based color)
- unavailable `–` handling
- the CSS color tokens and tone class names
- the `api.workBadges` shape (`version: 1`; same `tone` / `todayTone` signatures)
- `over` / lint semantics: orange, from 80% through the cap, stays a presentation state
  and is never a lint or an `over` condition

CLI, tmux, Mac Capture, Review/Browse badges, and per-note Ready chips are out of scope.

## Repository access

From the bob-cli workspace, use `/sase_repo` and only the printed paths:

```sh
sase repo open bob-plugins -r 'Retune Work badge tone thresholds to 40/60/80 in bob-ledger-tools'
sase repo open gh:bobs-org/bob -r 'Mirror the retuned Work badge thresholds in the dash.md fallback'
```

Read each repository's `AGENTS.md`, check `git status` before editing, and preserve
unrelated changes. bob-ledger-tools uses `src/fragments.json`. Edit `src/` fragments and
run `npm run build`. Never hand-edit the generated `main.js`.

## Changes

### 1. bob-plugins: the shared helper

`plugins/bob-ledger-tools/src/075-work-badge-tone.js`:

- Rewrite the band ladder in `workBadgeTone(count, cap)` below the existing validation,
  zero, and `count > cap` checks. Use integer cross-multiplication so exactness never
  depends on float division:

  ```js
  if (5 * count >= 4 * cap) {
    return "orange";
  }
  if (5 * count >= 3 * cap) {
    return "yellow";
  }
  if (5 * count >= 2 * cap) {
    return "green";
  }
  return "blue";
  ```

  Remove the now-redundant `count === cap` branch.

- Update the header comment's precedence line to the new bands. `todayBadgeTone`,
  `workToneClass`, `withWorkToneClass`, and `WORK_BADGE_TONES` stay unchanged.

Every surface already routes through this function: `070-ready-and-review.js` lane and
READY models, `170-plugin-lifecycle.js` `api.workBadges.tone`, and daily TODAY via
`todayBadgeTone`. Confirm this with a grep for other threshold copies (`/ 2`, `3 * cap`,
`cap / 2`) under `plugins/bob-ledger-tools/src/`, and fix any you find.

Advance `plugins/bob-ledger-tools/manifest.json` by one minor version (currently
`1.51.1` → `1.52.0`; bump from whatever is current) and the matching version cell in the
root `README.md` plugin table. Run `npm run build`.

### 2. bob-plugins: tests

Update the hard-coded expectations in `scripts/test-ledger-tools-work-badge-tone.cjs` to
these vectors:

- cap 100: `0` grey, `1` and `39` blue, `40` and `59` green, `60` and `79` yellow, `80`,
  `99`, and `100` orange, `101` and `250` red.
- cap 15 (bands at 6, 9, 12; no percentage rounding): `0` grey, `5` blue, `6` and `8`
  green, `9` and `11` yellow, `12`, `14`, and `15` orange, `16` red.
- cap 10: `0` grey, `3` blue, `4` and `5` green, `6` and `7` yellow, `8`, `9`, and `10`
  orange, `11` red.
- small caps: `1/1` orange and `2/1` red, `1/2` green and `2/2` orange, `1/3` blue,
  `2/3` yellow, and `3/3` orange. Add cap 5, which hits every band exactly: `1` blue,
  `2` green, `3` yellow, `4` and `5` orange, `6` red.
- changed cap with unchanged count: `10/15` yellow and `10/10` orange.
- TODAY with themes `3/3` and links out of 10: `0` grey, `3` blue, `4` green, `6`
  yellow, `8` and `10` orange, `11` red. In the theme-overflow test, change the last
  "exact cap does not force red" vector to links `3/10` → blue. In the
  unrelated-warnings vector, change links to `3/10` and keep blue.
- lane and READY models: rename the NEXT section `10/15` (whole lane `17/15`, TODAY 7)
  model to `yellow` and assert `"yellow"`; the text stays `NEXT 10/15`. READY `87/100`
  becomes `"orange"`. `8/100` stays blue, and `15/15` stays orange.
- dashboard integration test: NEXT `10/15` → `bob-work-tone-yellow` and PENDING `4/10` →
  `bob-work-tone-green`. TODAY stays red (theme overflow), and READY `15/100` stays
  blue.
- dashboard fallback test (old API without `workBadges`, which exercises the
  `dash.md`-local copy): change the inputs so old and new thresholds disagree, and
  assert exact tones rather than only "carries a tone". For example, NEXT `12/15` →
  orange, PENDING `4/10` → green, and READY `60/100` → yellow, plus the inline TODAY
  chip at links `7/10` with themes `2/3` → yellow, if the harness exposes that chip.
  This is the only automated guard on the vault's mirrored copy.

`scripts/test-ledger-tools-dashboard-parity.cjs`: rename the `green` `10/15` model to
`yellow` and assert `"yellow"`. `scripts/test-ledger-tools-ready-badge.cjs` and
`scripts/test-ledger-tools-plan-budget.cjs` assertions already hold (`50/100` is green,
`100/100` is orange, and a single TODAY link out of the default cap is blue). Re-run
them to confirm.

### 3. Bob vault: `dash.md` fallback mirror

In the opened vault checkout's `dash.md`, first `dataviewjs` block, the local `workTone`
(used only when the plugin lacks `api.workBadges`) mirrors the plugin. Apply the same
ladder there: remove the `count === cap` branch and use the `5 * count >= {4,3,2} * cap`
checks. Update its two-line comment to the new bands. Change nothing else in the
dashboard. Plugin sync does not deploy dashboard Markdown, so this change is delivered
through the vault repository's normal SASE finalization and the documented vault Git
sync.

### 4. bob-cli docs

- `docs/dashboard.md` "Work badge colors": replace the table with the new one above
  (without the "Was" column). Change "Check zero, strict excess, and equality before
  descending through 75% and 50%" to "Check zero and strict excess before descending
  through 80%, 60%, and 40%". The NEXT example (section `10/15`, whole lane `17/15`,
  TODAY 7) becomes **yellow**. Reword "Exact-cap orange is a presentation state ..." to
  say that orange, from 80% through the cap, is a presentation state, not a lint or an
  `over` condition.
- `docs/plan.md`: update the PENDING/NEXT paragraph (bands, the `10/15` example now
  yellow, and the exact-cap orange sentence) and the READY paragraph (bands, plus "at
  the cap is orange" becomes "from 80% through the cap is orange"; only a strict excess
  raises `ready_cap_exceeded`). The TODAY paragraph's generic
  "grey/blue/green/yellow/orange/red bands" wording can stay.
- Grep bob-cli `docs/` and `README.md` for leftover `L / 2`, `3 * L / 4`, `n === L`,
  `75%`, or "Exact-cap" text about Work badges, and fix any you find.

No SASE memory note or decision record states these thresholds, so no memory change is
needed.

## Verification

In the opened bob-plugins checkout:

```sh
npm run build
node --test scripts/test-ledger-tools-work-badge-tone.cjs scripts/test-ledger-tools-dashboard-parity.cjs scripts/test-ledger-tools-ready-badge.cjs scripts/test-ledger-tools-plan-budget.cjs
BOB_DASHBOARD_FILE=<opened-vault-path>/dash.md node --test scripts/test-ledger-tools-work-badge-tone.cjs
npm test
npm run validate
git diff --check
```

The `BOB_DASHBOARD_FILE` run must exercise the edited vault `dash.md`. Don't skip it,
and don't hard-code a home or workspace path. Run `git diff --check` in the vault and
bob-cli checkouts too. No Rust code changes, so the Rust suite isn't required. If a live
Obsidian UI is available, glance at the daily and dashboard Work badges. If not, say so
instead of claiming visual verification.

## Deployment

```sh
bob plugins sync --no-pull --repo <opened-bob-plugins-path> -p bob-ledger-tools --dry-run
bob plugins sync --no-pull --repo <opened-bob-plugins-path> -p bob-ledger-tools
```

Do not use `--force` over dirty deployed files. Report plugin installation separately
from `dash.md` delivery, and note any Obsidian reload or vault-sync step that remains.
Account for every modified repository (bob-cli, bob-plugins, and the vault) in the SASE
final declaration.
