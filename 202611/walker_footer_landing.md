---
tier: tale
size: medium
title: Finish landing the walker-footer epic (bob-cli-ky)
goal:
  "Close the gaps the bob-cli-ky land agent found: stale walkMode v2 test assertions,
  the missing review-attention-dot tests, a ledger-unload dot clear, the real-Obsidian
  selector-geometry e2e the visual phase skipped, and the nav-hotkeys reference page's
  word cap; then close epic bob-cli-ky."
proposed_by: bbugyi200.athena.bob-cli-ky.land
proposed_with: claude/opus
bead: bob-cli-ky
create_time: 2026-11-12 16:57:52
status: wip
---

- **BEAD:**
  [bob-cli-ky](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ky/README.md)

# Plan: Finish landing the walker-footer epic (bob-cli-ky)

## Why

Epic `bob-cli-ky` ("Walker footers follow the active bracket walk", plan
`plan:202611/walker_footers.md`) has all seven phases closed, but its land agent found
work the phases reported complete and did not finish. Every item below is caused by the
epic, so it must land before the epic closes. Read the epic plan first
(`sase artifact read plan:202611/walker_footers.md "<why>"`): its "Shared design
contract", "Phase: dot", and "Phase: visual" sections are the binding spec here. The
epic's reviewer decisions are final and allow no further memory edits; this tale edits
no memory files.

The land agent already did the follow-up triage and recorded it on the epic as a
`LANDER FOLLOW-UP TRIAGE` note: bob-cli-kw (+1), new tasks bob-cli-l7 and bob-cli-l8. It
also regenerated bob-cli's `docs/reference/glossary.md`. Do not redo that work, and do
not fix bob-cli-kw, bob-cli-l7, or bob-cli-l8 here. Their failures (task-tag-marks
wiring, album UI e2e self-test, the gcal-notes lychee link) are known and not caused by
this epic.

## Where the code lives

All code work is in the **bob-plugins** linked repo. Open it with `/sase_repo`
(`sase repo open bob-plugins -r "..."`), read its `AGENTS.md`, and follow it exactly:

- edit `src/` fragments only, never generated `main.js`;
- keep every hand-edited fragment at or below 1000 lines;
- run `npm run build` after fragment edits;
- register new test files in `package.json`'s `test:unit` list;
- run `bob plugins sync` after changes, following AGENTS.md's refusal procedure if sync
  refuses (never `-F` on your own).

Start with `git fetch` and `git log --oneline HEAD..@{upstream}`. Another agent may
already have landed one of these fixes (for example the v3 assertions). If so, skip that
item and say so in the epic close note.

Relevant fragments (as of bob-plugins `6050124`):

- nav `plugins/bob-navigation-hotkeys/src/482-walk-mode-nav-api.js`: `createWalkModeApi`
  with `version: 3`, `setAttention(type, value)` storing `plugin.bracketModeAttention`
  and calling `refreshBracketModeIndicator`.
- nav `src/547-bracket-mode-indicator.js` (911 lines): `bracketModeViewKey`,
  `buildBracketModeView(type, attention)` (`attention`, `attentionTier`,
  `attentionLabel`, aria text on the `s` segment only while mode ≠ `s`), and
  `paintBracketModeIndicator` (`has-attention` class, `data-attention-tier`, `title`).
  Exported for tests from `src/700-exports.js`.
- nav `styles.css`: the `button.bob-bracket-mode-seg.has-attention::after` dot, the
  per-tier `--bob-review-attention` rules, and the `forced-colors` block using
  `Highlight`.
- ledger `plugins/bob-ledger-tools/src/122-walker-footer-host.js`:
  `walkerFooterPushReviewAttention(plugin, route, memo)` pushes only commitment tiers
  (`FRESHNESS_FOOTER_COMMITMENT_TIERS`). It pushes `null` for legacy/`s`/unavailable/no
  commitments and skips when `api.walkMode.version < 3`. It is **not** exported for
  tests yet.
- ledger `src/240-plugin-freshness-status.js`: `updateFreshnessStatusBar` calls the push
  on every refresh and doorbell event. `setupFreshnessStatusBar` already registers an
  unload callback for `linksWalkerFooterInvalidate`.
- ledger `src/354-exports-walker-footer.js`: walker-footer test exports.
- tests: `scripts/test-navigation-bracket-repeat.cjs`,
  `scripts/test-navigation-period-links.cjs`,
  `scripts/test-navigation-bracket-mode-indicator.cjs`,
  `scripts/test-ledger-tools-walker-footer.cjs`.
- e2e pattern to copy: `scripts/obsidian-command-mode-tab-e2e.cjs` plus its offline
  `scripts/test-obsidian-command-mode-tab-e2e.cjs`, a `--self-test` mode, and a graceful
  `{"status":"skipped"}` when Obsidian is missing. `/snap/bin/obsidian` exists on
  athena.

## Work

### 1. Stale walkMode version assertions

The dot phase bumped `walkMode.version` to 3, but three assertions still expect 2 and
fail on master:

- `test-navigation-bracket-repeat.cjs`: "walkMode api is versioned and the top-level api
  version is unchanged" and "walkMode degrades with a null plugin";
- `test-navigation-period-links.cjs`: "walkMode.links matches the first destination [n
  and ]n would open".

Assert `3`. Keep the top-level `api.version === 3` assertion.

### 2. Review-attention dot tests (the dot phase shipped none)

Extend `scripts/test-navigation-bracket-mode-indicator.cjs`:

- `buildBracketModeView(type, attention)`: the `s` segment carries attention only while
  the mode is `w`, `n`, or none; never while mode is `s`; never on `w`/`n` segments.
  Tier and label come through. The `aria-label` appends the label.
- paint: `has-attention` and `data-attention-tier` are set and the `title` carries the
  label. All three are removed when attention clears. An attention-only change (same
  mode) repaints. An unchanged view key skips DOM writes. Attention never toggles
  `is-instant` or any animation class.
- a static `styles.css` check: the `has-attention::after` rule has no
  `animation`/`transition`, and a `forced-colors` rule paints the dot with `Highlight`.

Extend `scripts/test-navigation-bracket-repeat.cjs` with `walkMode.setAttention`:

- non-`s` type returns `invalid-type`;
- malformed values (non-object, blank tier or label) return `invalid-value`;
- `null`/`undefined` clears;
- a valid value is frozen and normalized (lowercased tier, trimmed label) and calls
  `refreshBracketModeIndicator`;
- a throwing refresh still returns `{ok: true}`;
- a null plugin returns `unavailable`.

Export `walkerFooterPushReviewAttention` through `src/354-exports-walker-footer.js`.
Extend `scripts/test-ledger-tools-walker-footer.cjs`:

- commitments due off-`s` push `{tier, label}` with the first nonzero commitment tier;
- ROTTEN-only and POST-only queues push `null`;
- mode `s`, a legacy route, and `tasksAvailable: false` push `null`;
- nav `walkMode.version < 3` (or no `setAttention`) makes no call;
- a throwing `setAttention` never throws out of the push;
- through `updateFreshnessStatusBar` with a fake nav, s → w → s transitions push
  non-null then null.

### 3. Clear the dot when ledger unloads

Today, disabling bob-ledger-tools leaves nav's last attention in place, so the `s`
letter keeps a stale dot until nav reloads. In `setupFreshnessStatusBar`, register an
unload callback, next to the existing `linksWalkerFooterInvalidate` one. It calls
`api.walkMode.setAttention("s", null)` when nav exposes `walkMode.version >= 3`. Wrap it
so it never throws. Add a test: the registered callback clears attention and tolerates a
missing or old nav api.

### 4. Real-Obsidian e2e for selector geometry and the panel swap

The visual phase proved invariant 7 only in a Chromium preview. It recorded that
live-vault confirmation was left for the ship phase, and the ship phase never did it.
Add `scripts/obsidian-walker-footer-e2e.cjs`, with a `-lib.cjs` if the acceptance logic
warrants one, following the command-mode-tab harness. It should:

- launch Obsidian against a scratch vault with the built bob-ledger-tools and
  bob-navigation-hotkeys plugins, Tasks available, and a fixture with due review rows,
  lane tasks, and a note with prev/next links;
- select `s`, `w`, and `n` via `api.walkMode.select(...)` and via the `]w` family
  command;
- assert the `[[ s w n ]]` selector's bounding rect is identical across s → w → n, both
  immediately and after two animation frames;
- assert the walker footer host's `data-walker-panel` (and visible panel text) swaps
  within one frame of `]w`;
- assert exactly one walker-footer status item;
- assert the `s` dot appears while commitments are due off-`s` and clears on `s`.

Add the offline `scripts/test-obsidian-walker-footer-e2e.cjs` (`--self-test` plus the
missing-binary skip) to `test:unit`. Run the real e2e once on athena and record the JSON
result, including the measured rects, in the epic close note. If the real run cannot
launch Obsidian in this environment, record the exact failure. Do not claim it passed.

### 5. Nav-hotkeys reference page over the word cap

bob-cli's docs gate fails:
`reference/plugins/bob-navigation-hotkeys.md: over budget at 6039 words (cap 6000 for reference/)`.
The page is mounted from bob-plugins `docs/bob-navigation-hotkeys.md`. The epic's ship
phase plus the later albums commit `6050124` pushed it over. Trim at least ~60 words
there, preferring redundancy in the walker/bracket passages the epic added, without
dropping facts. Then run `npm run docs:check` in bob-plugins. In the bob-cli checkout,
run `uv run --project tools/docs bob-docs lint` to confirm the cap passes. A remaining
offline-lychee error on `gcal-notes` is bob-cli-l7: expected, not this tale's.

### 6. Gates and deploy (bob-plugins)

Run all of these:

- `npm run build` and `npm run build:check`;
- `npm run validate`;
- every walker test file:
  - `test-navigation-bracket-repeat.cjs`
  - `test-navigation-period-links.cjs`
  - `test-navigation-bracket-mode-indicator.cjs`
  - `test-ledger-tools-freshness-footer.cjs`
  - `test-ledger-tools-walker-footer.cjs`
  - `test-ledger-tools-lane-walk.cjs`
  - the new e2e offline test
- `npm test`.

The known non-epic `npm test` failures may remain: bob-cli-kw (task-tag-marks wiring),
bob-cli-l8 (album UI e2e self-test), and timing-budget flakes that pass in isolation.
Report each by name. Any other failure is yours to fix. Finish with `bob plugins sync`
and confirm the synced versions.

### 7. Close the epic (final step)

1. `sase bead epic-symbols bob-cli-ky`: the land agent saw none. If entries appear,
   resolve each one per the Symvision epic-whitelist policy: wire it up, privatize it,
   add a non-test pragma, or delete it. Re-key one only if a still-open later bead needs
   it.
2. `sase bead close bob-cli-ky --note "<verification>"`. The note should cover:
   - the land agent's findings (phases 1–4 and 7 verified in source; dot phase lacked
     tests; visual phase lacked the real-Obsidian e2e; nav page over the word cap; the
     generated glossary regenerated);
   - what this tale fixed, with test counts and the e2e JSON result;
   - the follow-up triage already on the epic (bob-cli-kw +1, bob-cli-l7, bob-cli-l8,
     v2-assertion proposal declined as epic work).

   Never pass `--force` to make the close succeed.

3. Run `just symvision` in the bob-cli checkout and confirm the whitelist is clean.
4. Set `status: done` in the epic plan's frontmatter. That is the `PLAN` path shown by
   `sase bead read bob-cli-ky -r "<why>"`, plan `plan:202611/walker_footers.md`.
5. The epic has no `parent_bead`, so there is nothing further to close.
