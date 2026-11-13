---
tier: tale
size: medium
title: Finish the bank Category card and close epic bob-cli-lc
goal: The Category card renders as a card instead of a squashed four-column grid,
  learns from the row Bryan just filed, matches the approved look and accessibility
  contract, is deployed, and epic bob-cli-lc closes.
decisions:
  card_stays_open:
    ask: Keep the Category card open as it advances, so each waiting row takes one
      key?
    choices:
      keep: Card stays open and moves to the next waiting row (as shipped); one key
        per row
      close: Card closes after each filing (per the epic plan); row flashes, next
        Ctrl+Enter reopens it
    default: keep
    why: Already shipped and tested, and one key per row is the walk Bryan asked for
    answer: keep
proposed_by: bbugyi200.athena.bob-cli-lc.land
proposed_with: claude/opus
bead: bob-cli-lc
decided_by: auto
status: done
---

- **PARENT:**
  [202611/bank_weekly_category_walk.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bank_weekly_category_walk.md)
- **BEAD:**
  [bob-cli-lc](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-lc/README.md)

# Finish the bank Category card and close epic bob-cli-lc

## Why

Epic `bob-cli-lc` (plan `plan:202611/bank_weekly_category_walk.md`) made bank tracking a
weekly Monday walk. Ctrl+Enter on a waiting bank row opens a Category card, one key
files the category, and the walk advances. Its land agent verified the work and found
that most of it is real and correct:

- **bob-cli.** QFX cleaning, the default-account `[acct::]` fix, mapping-form categories
  with hotkeys, groups, and hidden flags, the shared read-time `suggestion` on
  `list`/`report`/`import` rows, and import's `Next:` line all match the contract, and
  `docs/bank.md` and the JSON docs describe them.
- **Vault.** It holds the weekly chore, the taxonomy, and the seed rows.
  `bob bank check` reports 0 errors. The glossary note is updated, and `bob-cli-kv`
  stays snoozed.
- **bob-plugins.** It has api.bank v2, the cycler delegation, and nav `recordJump`. The
  four bank unit suites pass.

The land agent also confirmed three defects in the plugin side. All three were caused by
the epic, so they are still epic work:

1. **The card renders as a squashed grid.** The card's root element is
   `<section class="bob-bank-category bob-key-card">`
   (`plugins/bob-ledger-tools/src/247-plugin-bank-category-card-ui.js`, `paint()`). The
   class `bob-bank-category` already belonged to the month card's category breakdown
   row. That row's `styles.css` rule
   (`.bob-bank-category { display: grid; grid-template-columns: minmax(6rem, 1fr) minmax(3rem, 2fr) auto 3.2em; … font-size: 0.82em }`)
   therefore also lays out the whole card: its title, payee, sections, and footer flow
   into four columns. Nobody saw it, because the Mac GUI check was skipped and the
   real-Obsidian e2e only drives `modal.onKeyDown`.
2. **The card cannot learn from the row just filed.** `bankCategoryCommit` in
   `src/248-plugin-bank-category-card-actions.js` edits the editor buffer and then calls
   `apiBankRefresh` right away, with no save. bob reads the file on disk, so the refresh
   still sees the row as waiting with no category. That breaks design call 1 (learning
   inside one session: after Bryan files the first coffee row, the next one is suggested
   as Coffee) and design call 2 (save, then refresh). The e2e's "learned" step passes
   only because it injects a fake suggestion through `modal.setReport`.
3. **The card is missing parts of the approved look and accessibility contract.**
   - The header is `Payee · $amount`, not amount first, and inflows get no `+` or green.
   - Progress is a bare `Row N of M`, with no month and no meter.
   - Hotkeys are plain text like `c · ☕ Coffee`, not keycaps, and tiles have no
     `aria-keyshortcuts`.
   - No live region announces a filing.
   - Restricted mode still shows the full `↵ accept · letter choose …` keys line, and
     while categories load it reads `Categories unavailable — categories unavailable`.

The epic plan also said the card closes after each filing. Phase `bob-cli-lc.7` instead
kept it open and moved it to the next row (commit `b2a77a7`, "keep the Category card on
one modal through the note"). `card_stays_open` settles which behavior stays.

The Mac never received the plugins: it was unreachable for `bob-cli-lc.7` and again at
landing. Vault git ignores `.obsidian/plugins/bob-ledger-tools/`, so every host needs
its own `bob plugins sync`.

## Ground rules

- bob-plugins is a linked repo. Open it with `/sase_repo`
  (`sase repo open bob-plugins`), read its `AGENTS.md`, and edit only `src/` fragments
  plus `styles.css` and the tests. Register any new fragment in `src/fragments.json`,
  keep every hand-edited fragment at or below 1000 lines, and run `npm run build` (never
  hand-edit `main.js`). After any change, run `bob plugins sync`, following the
  AGENTS.md refusal rules (never `-F` on your own).
- **Privacy.** Use synthetic data only (`Corner Cafe`-style fixtures). Never put a real
  merchant, amount, person, place, or account digit in a commit, test, bead note, or
  screenshot you describe. Report the real vault in counts only.
- Use Conventional Commits. In bob-cli, never edit the `Cargo.toml` version or
  `CHANGELOG.md`.
- Do not add words to bob-plugins `docs/bob-ledger-tools.md` or
  `docs/bob-navigation-hotkeys.md`. Their mounted reference pages are already over the
  6000-word docs-check cap (tracked by `bob-cli-il` and `bob-cli-jz`). Any wording
  change there must be net-zero.

## Work

### 1. Give the card its own root class

- In `247-plugin-bank-category-card-ui.js` `paint()`, change the root section's class to
  `bob-bank-category-card bob-key-card`. Rename the card-only rule
  `.bob-bank-category { --bob-bank-accent: …; color: … }` in the Category card block of
  `plugins/bob-ledger-tools/styles.css` to `.bob-bank-category-card`.
- Leave the month card's `.bob-bank-category` breakdown rule untouched, along with the
  `.bob-bank-category` probe in `scripts/obsidian-bank-ui-e2e-lib.cjs`, which reads
  breakdown rows.
- Grep for any other selector that assumed the old root class.
- In `scripts/test-ledger-tools-bank-category-card-ui.cjs`, assert that the painted root
  carries `bob-bank-category-card` and not the bare `bob-bank-category` token.

### 2. Save before every refresh

- In `bankCategoryOpenAtCursor`, keep the opening `view` on the modal (`modal.view`),
  and carry it across `setNext`.
- Add a best-effort `bankCategorySaveNote(modal)` that never throws. It awaits
  `modal.view.save()` when `modal.view.file.path === modal.context.path`. Otherwise it
  saves every markdown leaf showing that path, the same way
  `src/241k-plugin-bank-actions.js` already does before reopening a month (around the
  `leafView.save()` loop).
- In `bankCategoryCommit`, after a successful `bankCategoryApplyEdits`, await that save
  before calling `apiBankRefresh`. This applies in both close and set mode, and on the
  end-of-run path (`bankCategoryCompleteWalk`'s month refresh starts only after the
  save).
- Keep the ≤ 800 ms budget for the next card covering save plus refresh, and keep the
  late-result upgrade.
- Unit test (fake view and fake `apiBankRefresh`): the save resolves before the refresh
  is requested, and a throwing save still refreshes.

### 3. Match the approved look and accessibility contract

Do this in `247-…-ui.js` (with model support in `245-…-model.js` only where a pure field
is missing) and in `styles.css`, staying in the `.bob-key-card` family with the bank's
magenta accent.

- **Header.**
  - The amount comes first, in tabular numerals. Outflows are neutral. Inflows show `+`
    and use the theme's green, so the sign carries the state and color never carries it
    alone.
  - Then the payee, then the muted day aligned right.
  - Keep the ⚑ flagged chip and the non-default account chip.
  - In set mode, the progress line is replaced by the row state (`waiting` or
    `✓ acknowledged`), with no meter.
- **Progress** (close mode). Show `N of M · <Month>`, where the month name comes from
  the note path. Add a slim meter under it. The meter is `aria-hidden`, because the text
  already carries the count.
- **Tiles.**
  - Lay tiles out in a 3-column grid (4 on wide panes).
  - Render each hotkey as a monospace uppercase keycap (`<kbd>`), followed by the emoji
    and label.
  - Set `aria-keyshortcuts` to the hotkey.
  - Give the highlighted tile a trailing `↵`, and keep `aria-pressed`.
  - A tile with no hotkey (an old bob) renders without a keycap and stays reachable by
    arrows and ↵. Keep the `update bob for one-key filing` hint.
- **Live region.** Add a polite `aria-live` region that survives repaints. After each
  close-mode filing it announces `Filed $6.45 as Coffee · next $12.90 waiting` (or
  `… · no category` for Space, and no `next` part at the end of the run).
- **Restricted mode.**
  - The keys line shows only `␣ acknowledge · esc cancel`.
  - While the first report fetch is in flight, the reason reads `loading…`.
  - When the fetch fails, the reason is bob's error text, not
    `categories unavailable — categories unavailable`. Pass a real `reason` into the
    model.
  - The card still upgrades in place when categories arrive before the first key.
- Extend `scripts/test-ledger-tools-bank-category-card-ui.cjs` (or
  `…-bank-category-card.cjs` for model fields) to cover:
  - the header order and the inflow `+`;
  - the `N of M · Month` text;
  - keycap plus `aria-keyshortcuts` on each tile;
  - the live-region text after a filing;
  - the restricted keys line and the loading reason.

> [!decision] card_stays_open = keep Keep the card open across rows (the current
> `setNext` flow). No editor flashes are needed: the open card sits over the editor. In
> the bob-cli guide `docs/guides/acknowledge-bank-transactions.md` §2, add one sentence:
> the card stays open on the next waiting row, so the next key files it, and Esc leaves
> the cursor there.

> [!decision] card_stays_open = close After a successful close-mode filing, close the
> modal instead of calling `setNext`. Land the cursor at the next row's payee start and
> call `recordJump` (both as today). Give the filed row a brief settle flash and the
> landing row a soft focus flash (the yank-flash idiom), both off under
> `prefers-reduced-motion`. Keep the ~300 ms gesture lock as plugin state, so
> `claimRowClose` swallows a Ctrl+Enter double press on the landing row: no card and no
> fallback flip. Keep the walk session across reopenings, and show the end-of-run
> summary as a Notice of about 8 s with the month link.
>
> - Update `test-ledger-tools-bank-category-card-ui.cjs` and the e2e walk to press
>   Ctrl+Enter on each row.
> - Update the `bob-cli-lc.7` wording in bob-plugins `docs/bob-ledger-tools.md` with
>   net-zero words.
> - In the bob-cli guide §2, say that each filing closes the card and the next
>   Ctrl+Enter opens it on the landed row.

### 4. Prove it in real Obsidian

Extend `scripts/obsidian-bank-ui-e2e.cjs`, which uses synthetic fixtures and runs under
xvfb.

- **Layout.** With the card open, assert that `getComputedStyle` of
  `.bob-bank-category-card` is not `display: grid`, and that the tiles section spans the
  card's width. Capture `walk-card.png`, and `walk-card-dark.png` after toggling the
  body to `theme-dark` if the harness allows it. Open both screenshots with the Read
  tool and confirm by eye that the card is a single column with a tile grid.
- **Real learning, no `seed()`.** Use a fixture month with two waiting rows of the same
  synthetic payee, plus an unrelated row after them.
  1. Open the card on the first row through the cycler command, and press `c`.
  2. Without injecting anything, wait up to about 2 s for the card's report. Then assert
     that the second row's card shows `suggestion.category === "coffee"` with source
     `learned`, taken from bob's real JSON.
  3. Accept it with ↵.

  The e2e must run the landed bob (the category-contract build). Use the same binary
  resolution the e2e already uses.

- Keep every existing `walk-*` assertion green. `chip-today` is a known failure caused
  by a hard-coded `FIXTURE_TODAY` (tracked by `bob-cli-le`, which is not this tale's
  work).

### 5. Verify and deploy

- **bob-plugins checks.**
  - Run `npm run build`, then `npm run build:check`.
  - Run `node --test` on these suites, which must pass:
    - `scripts/test-ledger-tools-bank-category-card.cjs`
    - `scripts/test-ledger-tools-bank-category-card-ui.cjs`
    - `scripts/test-task-status-cycler-bank-guard.cjs`
    - `scripts/test-navigation-jump-history.cjs`
    - `scripts/test-ledger-tools-bank-cards.cjs`
  - Run `npm test`. Known load flakes are `bob-cli-ki` (keep-moving 200-link) and
    `bob-cli-3w` (stage ranker 16 ms). Rerun any timing failure in isolation and record
    it.
  - Run `npm run docs:check`. Only the two pre-existing word-cap errors may remain.
  - Run the bank e2e.
- **bob-cli checks.** If the guide changed, run `just check` and `just docs-check` in
  bob-cli.
  - Known `just check` failure: the time-dependent gcal test
    `sync_args_read_last_n_days_aliases_and_default_to_none` (`bob-cli-l2`).
  - Known `just docs-check` failures: the two mounted plugin word caps (`bob-cli-jz`,
    `bob-cli-il`).
  - Do not run `just check-full`.
- **Athena.** Run `bob plugins sync` from the bob-plugins checkout. Confirm that the
  deployed `bob-ledger-tools` `main.js` in the vault contains `bob-bank-category-card`,
  and that task-status-cycler and bob-navigation-hotkeys are current.
- **Mac.** Best effort: `ssh -o ConnectTimeout=8 mac`, as lc.7 attempted.
  - If it is reachable, deploy the three plugins there the way that host normally
    receives them: update its bob-plugins checkout, then run `bob plugins sync`. Report
    the result.
  - If it is unreachable, use `/sase_new_task` to file one small task: "Deploy the bank
    Category card plugins (bob-ledger-tools, task-status-cycler, bob-navigation-hotkeys)
    to the Mac before Bryan's next Monday bank walk". Name the bob-plugins commit it
    needs, and record the bead ID in the epic close note.

### 6. Close epic bob-cli-lc (final step)

1. Run `sase bead epic-symbols bob-cli-lc`. At landing it reported no entries. If any
   appear, resolve each one under the Symvision epic-whitelist policy: wire it up,
   privatize it, add a non-test pragma, or delete it. Re-key a Justfile line only to a
   still-open bead that needs it.
2. Run `sase bead close bob-cli-lc --note "<verification>"`. Start from the land agent's
   verification below and append what this tale fixed and proved: the commit SHAs, the
   test and e2e results, the screenshot check, and the Athena/Mac deploy outcome.
   > Land verification: all 8 phases closed and their notes addressed; bob-cli commits
   > d6d755a1, 1501c0c3, a1dee92f, 9c53b8d8 and bob-plugins 2522e47, 0b1000b, b2a77a7
   > match the contract (QFX cleaning + default-account acct fix; mapping-form
   > categories with derived hotkeys/groups/hidden; shared rule>learned>bank suggestion
   > on list/report/import; import Next: line; api.bank v2 claimRowClose/setCategory;
   > cycler delegation with local-flip fallback on both Ctrl+Enter entry points; nav
   > recordJump; hidden categories out of [cat:: suggestions; payee-start landings).
   > Decisions honored: tech/media split, seed rows pre-filed, kv held (note added on
   > bob-cli-kv), glossary:bank-row updated, no decisions record. Real vault: bank check
   > 0 errors (1 pre-existing coverage-gap warning), 25 categories (24 visible, all
   > hotkeyed; kids hidden), weekly chore open, daily occurrence cancelled. just check:
   > only the pre-existing gcal failure (bob-cli-l2). Integration: none of the 10
   > non-epic commits since d6d755a1 (bob-cli) or 2522e47 (bob-plugins) touch bank code.
   > Follow-ups triaged in the epic's LAND TRIAGE note (bob-cli-le and bob-cli-lf
   > created; jz, il, ki, 3w, kl, l2 corroborated).
3. Run `just symvision` in bob-cli and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN path
   `sase bead read bob-cli-lc` prints, `plan:202611/bank_weekly_category_walk.md`.
   Change only that field.
5. `bob-cli-lc` has no `parent_bead`, so nothing further closes.

Never use `--force` to make the close succeed. If the close is refused, fix the cause
and close again.
