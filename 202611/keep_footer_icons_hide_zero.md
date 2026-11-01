---
tier: tale
title: Draw the Keep footer icons and hide zero counts
goal:
  The Keep inbox status-bar item draws its lightbulb, inbox, and chevron icons and shows
  each count only when it is above 0.
size: small
decisions:
  signin_pill:
    ask:
      Keep showing the dashed lightbulb-off pill when Keep sign-in has expired, even
      though it carries no count?
    choices:
      hide:
        The pill hides; a lost sign-in then surfaces only via the once-per-session
        Notice
      show:
        The pill stays, since Keep checks stop until bob gkeep login and the bar is
        where you'd notice
    default: hide
    why: It follows your rule that a segment shows only when its count is above 0.
    answer: show
proposed_by: bbugyi200.apollo.bs
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-01 07:34:21
status: wip
---

# Plan: Draw the Keep footer icons and hide zero counts

## Context

Bryan's Obsidian footer (desktop status bar, `bob-ledger-tools` item `bob-keep-inbox`,
shipped by epic `bob-cli-ez` / `plan:202610/gkeep_footer_indicator.md`) shows a gold
pill with `4` and a grey `0`, but no icons. There is no lightbulb, no inbox tray, and no
chevron connector. He wants:

1. The intended icons drawn: Lucide `lightbulb` in the Keep pill, `inbox` beside the
   inbox count, and `chevron-right` between the two segments.
2. Each segment shown only when its count is above 0. In his screenshot, only the Keep
   pill (`4`) should show; the `gkeep_inbox.md` segment (`0`) should not.

### Root cause of the missing icons

`keepInboxSetIcon` in `plugins/bob-ledger-tools/src/293-keep-inbox-render.js` looks for
`setIcon` as a bare global (`typeof setIcon === "function"`) and then as
`window.setIcon`. Obsidian exports `setIcon` only from the `obsidian` module. It is not
a global. Other plugins in the repo already import it that way
(`plugins/task-status-cycler/src/010-core.js`,
`plugins/bob-command-mode/src/170-sheet.js`). bob-ledger-tools never imports it:
`src/010-load-and-constants.js` destructures only `MarkdownRenderChild … parseYaml`.
Every icon call therefore falls through to the `data-icon` attribute fallback, which
draws nothing. The connector is an icon-only span, so it renders empty too.

The tests missed this for two reasons. The test file's `require("obsidian")` stub
(`scripts/test-ledger-tools-keep-inbox.cjs`) has no `setIcon`, and no test asserts that
an icon was set. The deployed vault copy (`~/bob/.obsidian/plugins/bob-ledger-tools/`)
is byte-identical to the repo build, so a stale deploy is not the cause.

### Why zeros show

`src/292-keep-inbox-model.js` paints zeros on purpose, following the original plan's
Visibility states table:

- Keep total 0 builds an outlined `zero` pill (`keepInboxZeroView`).
- A verified Keep count last checked more than 24 h ago shows `–`
  (`keepInboxUnknownView`).
- An expired sign-in shows a `lightbulb-off` pill (`keepInboxSigninView`).
- The inbox part shows a faint `0`, or `–` when the count is unknown or the note is
  missing (`keepInboxInboxPart`).

This plan replaces those rules with the >0 rule below.

## Target behavior

| Part      | Shows when                                                                                                                                              | Hidden when                                                                                                                              |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Keep pill | It displays a known pull total above 0: tone `fresh` (state `ok`, checked ≤ 24 h ago, total > 0) or tone `stale` (stale, checked ≤ 24 h ago, total > 0) | Total 0 (ok or stale); unknown (no success in 24 h); unverified; `bob gkeep status` unsupported; sign-in expired (per `signin_pill`)     |
| Inbox     | The open count is above 0 (the Tasks-cache count when Tasks is ready, else bob's `inbox.open`)                                                          | Count 0; count unknown (Tasks not ready and no bob fallback); target note missing                                                        |
| Connector | Both segments show (unchanged)                                                                                                                          | Otherwise                                                                                                                                |
| Item      | At least one segment shows                                                                                                                              | Both are hidden. The existing `bob-keep-inbox-hidden` class (`display: none`) applies, so no empty padded slot remains in the status bar |

These do not change:

- Refresh scheduling and attention gating. They key off `configured` and the session
  toggle (`keepInboxActionCanRefresh`), not item visibility. Keep checks continue while
  the item is hidden.
- The palette commands, the context menu on visible segments, and the sign-in transition
  Notice. The Notice is driven by the status report, not the view.
- Tooltip and aria copy for visible states, the click routing for visible tones, and the
  bob-cli `bob gkeep status` contract. No bob-cli code changes.

## Implementation

**Repo:** `bob-plugins`, plugin `plugins/bob-ledger-tools`. Open it with `/sase_repo`
(`sase repo open bob-plugins -r …`) and read its `AGENTS.md`. The plugin uses the
fragment build: edit `src/` fragments, run `npm run build`, never hand-edit `main.js`,
and keep each fragment at or below 1000 lines.

### 1. Draw the icons (`src/293-keep-inbox-render.js`, `styles.css`)

- At the top level of 293, add a load-time capture modeled on 295's
  `KeepInboxActionMenu` capture:

  ```js
  let KeepInboxObsidianSetIcon = null;
  try {
    const obsidianMod = require("obsidian");
    KeepInboxObsidianSetIcon =
      obsidianMod && typeof obsidianMod.setIcon === "function"
        ? obsidianMod.setIcon
        : null;
  } catch (error) {
    KeepInboxObsidianSetIcon = null;
  }
  ```

  The capture must run at load time. The tests install their `obsidian` stub only while
  `main.js` is being required, so a lazy `require` at call time would miss it.

- Make `keepInboxSetIcon` call `KeepInboxObsidianSetIcon(node, name)` and drop the
  bare-global and `window.setIcon` probes. Keep the `data-icon` marker as the fallback
  when no setter exists (tests and other harnesses).
- Leave `keepInboxApplyTooltip` alone; this is a non-goal. Obsidian's `setTooltip`
  writes the element's `aria-label`, which would clobber each segment's separate aria
  copy. The review footer beside this item also uses native `title` tooltips.
- In `styles.css` under `.bob-keep-inbox`, give `.bob-keep-pill-icon` and
  `.bob-keep-connector-icon` `display: inline-flex; align-items: center;`. Today only
  the inbox icon has this. Without it, the real SVGs (already sized at 1.05em and
  0.85em) sit on the text baseline instead of centering in the pill.

### 2. Hide zero and unknown counts (`src/292-keep-inbox-model.js`)

Keep the decision in the pure model so `keepInboxPaint` stays dumb.

- In `keepInboxViewModel`:
  - Route a fresh `ok` total of 0 to the keep-hidden path (the existing
    `keepInboxInboxOnlyView`), and delete `keepInboxZeroView`.
  - The stale branch requires `total > 0`; otherwise the Keep pill is hidden.
  - The unknown branch (no success within 24 h, or stale without a count) also hides the
    Keep pill. Delete `keepInboxUnknownView`.
- `keepInboxInboxPart` returns `visible: true` only for a positive count, and
  `visible: false` for zero, unknown, and missing-note cases. Drop the copy those cases
  no longer need, and `KEEP_INBOX_UNKNOWN_GLYPH` if it becomes unused.
  - Every builder currently spreads `{ visible: true, ...inbox }`. Make sure the part's
    own `visible` wins.
- Add one finalizer, for example `keepInboxFinalizeView(view, keySuffix)`, that every
  visible builder calls. It replaces the repeated `connector.visible = …` and
  `paintKey = …` lines:
  - Set `connector.visible = keep.visible && inbox.visible`.
  - If neither segment is visible, return `keepInboxHiddenView("hidden-empty")`.
  - Otherwise compute `paintKey`. Keep the inbox-only reason suffix so `unverified` and
    unsupported frames stay distinct.
- Update the header comment of `keepInboxViewModel`. Each segment now shows only for a
  positive count, and the whole item hides when both are hidden.
- The fragment is 962 lines today, and this change should shrink it. Confirm it stays
  ≤ 1000.

> [!decision] signin_pill = hide The `auth` branch returns the keep-hidden path. Delete
> `keepInboxSigninView`. Remove the now-unreachable `tone === "signin"` branch from the
> bulb click handler in `src/295-keep-inbox-actions.js`, and the `signin` selector from
> the dashed-pill CSS. The context menu's **Keep setup check** item stays.

> [!decision] signin_pill = show Keep `keepInboxSigninView`, its click branch (doctor
> report), and its CSS. It is the only count-less Keep state still painted. Route it
> through the new finalizer like the other builders.

### 3. CSS cleanup (`styles.css`)

Remove rules for states that can no longer paint:

- `.bob-keep-pill[data-keep-tone="zero"]`
- `unknown` (and `signin` under `signin_pill = hide`) in the dashed-pill selector
- `.bob-keep-inbox-btn[data-inbox-tone="zero"] .bob-keep-inbox-count`
- `.bob-keep-inbox-btn[data-inbox-tone="unknown"] .bob-keep-inbox-count`

Keep `fresh`, `stale`, checking-pulse, count fade-in, reduced-motion, and forced-colors
rules.

### 4. Feedback for a manual check that finds nothing (`src/295-keep-inbox-actions.js`)

Today a forced check that succeeds relies on the visible `0` pill as its feedback. In
`keepInboxCheckNow`, when `refresh.ran === true`, `keep.state === "ok"`, and the new
total is 0, show the Notice `Google Keep · nothing to pull`. The pill is hidden, so a
palette or context-menu **Check Google Keep now** would otherwise give no feedback. A
positive total keeps today's behavior: no Notice, and a fade-in when the number changes.

### 5. Tests (`scripts/test-ledger-tools-keep-inbox.cjs`)

- Add `setIcon(node, name)` to the file's `obsidian` stub. It records each call (for
  example pushes `{ node, name }` and sets `node.iconName = name`).
- New regression test. Paint a fresh view (Keep 3, inbox 1) and assert these icons:
  - `lightbulb` on `.bob-keep-pill-icon`
  - `inbox` on `.bob-keep-inbox-icon`
  - `chevron-right` on `.bob-keep-connector-icon`
- Rewrite the zero, unknown, and missing expectations:
  - The screenshot case, Keep 4 with inbox 0: Keep visible, inbox hidden, connector
    hidden, item visible.
  - Keep total 0 with inbox > 0: Keep hidden, inbox visible.
  - Both 0: `visible: false`, and painting adds `bob-keep-inbox-hidden`.
  - Stale with total 0: Keep hidden.
  - A verified `ok` count older than 24 h: Keep hidden for totals 3 and 0. This replaces
    the "become unknown" test.
  - Inbox hidden for count 0, a missing note, and Tasks-not-ready without `inbox.open`.
    Tasks-not-ready with `inbox.open: 1` still shows `1`.
  - Unsupported status with the default file and a positive count: inbox only.
    Unsupported with count 0: hidden.
  - Unverified with inbox > 0: inbox only. Unverified with 0: hidden.
- Paint transitions: visible, then all-zero (hidden class on), then visible again
  (hidden class off, counts correct).
- A forced check whose result has total 0 shows `Google Keep · nothing to pull`. A
  positive total shows no Notice.
- `signin_pill`: `hide` asserts Keep hidden under `auth`. `show` keeps the existing
  `signin` assertions.
- Drop the tooltip assertions for zero views that no longer exist.

### 6. Docs (`docs/bob-ledger-tools.md`, "Keep inbox indicator")

- Say that each segment appears only while its count is above zero, and the item
  disappears when both are zero.
- The bulb is gold when fresh and dashed when last-known. The Keep pill also hides when
  its count is unknown or Keep is unverified (and when sign-in has expired, per the
  decision).
- Click behavior: a fresh bulb opens the pull preview; a stale bulb checks now. **Check
  Google Keep now** still works from the palette while hidden, and says when there is
  nothing to pull.
- Remove "outlined when clear", "faint at zero, `–` when unknown", and "click it when
  clear … or unknown".
- Run `npm run docs:check`. The command inventory does not change.

### 7. Build, verify, deploy, commit (bob-plugins)

- Run `npm run build`, `npm run build:check`, and
  `node --test scripts/test-ledger-tools-keep-inbox.cjs`, then `npm test`.
- Before blaming your change for a failure in `npm test`, check whether it also fails on
  unchanged `master`. Known unrelated failures: Chromium-less e2e (bead `bob-cli-f1`),
  `bob-cli-et`, and `bob-cli-eh`.
- Run `bob plugins sync`.
- Commit as `fix(ledger-tools): draw Keep inbox icons and hide zero counts`.

### 8. bob-cli guide (`docs/guides/drain-google-keep.md`)

In "See the count in Obsidian", reword the first sentence. The status bar shows a bulb
with the Keep count and an inbox tray with the open-task count in `gkeep_inbox.md`, each
only while it is above zero. Run `just docs-check`. Commit as
`docs(gkeep): note the footer hides zero counts`. Do not touch `Cargo.toml` or
`CHANGELOG.md`.

## Out of scope

- No bob-cli code or JSON contract changes.
- No changes to refresh scheduling, Keep check gates, or the cache.
- No tooltip mechanism change.
- No memory changes. No memory note describes the footer's zero rendering, and the
  pending decision-record task `bob-cli-f0` is about bob owning the Keep count, which
  this plan does not change.

## Live verification (Bryan)

After the vault syncs and Obsidian reloads bob-ledger-tools on the Mac, the screenshot
state should show only a gold `💡 4` pill: no inbox segment, no chevron. Hovering keeps
the existing tooltip. Once `gkeep_inbox.md` has an open task, the pill, the chevron, and
the inbox tray with its count all appear.
