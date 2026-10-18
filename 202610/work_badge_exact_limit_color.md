---
tier: tale
title: Give exact-limit Work badges their own magenta tone
goal:
  TODAY, PENDING, NEXT, and READY badges in daily bob-plan blocks and dash.md turn a
  dedicated magenta when the count exactly equals the limit, a color no other Bob chip
  uses, while orange keeps 80% up to (not including) the limit.
size: small
decisions:
  color:
    ask: Which color should a Work badge show when its count exactly meets the limit?
    choices:
      magenta:
        Plum in light mode, orchid in dark; no Bob chip uses it, and it clears every
        Work tone
      pink:
        Simplest CSS, but in light mode it sits next to red (ΔE 3 at the 14% chip tint)
      purple:
        Reads as "complete", but matches the PROJECTS/BLOCKED accent and looks blue if
        colorblind
    default: magenta
    why:
      Magenta fills the widest unused hue gap and stays clear of red, the accent, and
      blue
    answer: magenta
proposed_by: bbugyi200.athena.11g.f0
proposed_with: claude/opus
decided_by: auto
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.11g.f0](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.11g.f0.md)
- **COMMITS:**
  - [44c04e2](https://github.com/bobs-org/bob-cli/commit/44c04e24e81c7e7c162e996fb2c981764f06a7ed)
    — docs(dashboard): describe exact-limit Work badge colors

# Give exact-limit Work badges their own magenta tone

## Outcome

TODAY, PENDING, NEXT, and READY share one count-to-limit color rule in daily-note
`bob-plan` blocks and on `dash.md`. Today, orange covers 80% through the limit
inclusive. Split off the exact-limit point: a badge whose count equals its limit gets a
new **magenta** tone. Orange keeps `4L/5 <= n < L`. Every other band stays the same.

For an available nonnegative integer count `n` and a positive integer limit `L`, using
exact comparisons (never rounded displayed percentages):

| Condition                    | Color       | Was                   |
| ---------------------------- | ----------- | --------------------- |
| `n === 0`                    | grey        | same                  |
| `0 < n < 2 * L / 5`          | blue        | same                  |
| `2 * L / 5 <= n < 3 * L / 5` | green       | same                  |
| `3 * L / 5 <= n < 4 * L / 5` | yellow      | same                  |
| `4 * L / 5 <= n < L`         | orange      | `4 * L / 5 <= n <= L` |
| `n === L`                    | **magenta** | orange                |
| `n > L`                      | red         | same                  |

Precedence: zero, then strict excess (red), then exact equality (magenta), then descend
through 80%, 60%, and 40%. Small caps skip bands. Orange first appears at cap 5 (`4/5`).
For example, cap 1 goes grey → magenta → red, and cap 3 goes blue → yellow → magenta.

Nothing else changes:

- the count sources (section counts for PENDING/NEXT, the gated READY count, and TODAY's
  `budget.links`)
- the TODAY theme-overflow red override. Themes exactly at cap still keep the link-based
  color, so links `10/10` with themes `3/3` is magenta.
- unavailable `–` handling
- the existing six tone tokens
- `over` and lint semantics. Magenta is a presentation state only. It is never a lint,
  an `over` condition, or a `ready_cap_exceeded` trigger.

CLI, tmux, Mac Capture, Review/Browse badges, and per-note Ready chips are out of scope.
Mac Capture uses pink for its New/Created and Pomodoro-start capsules. Accept that
cross-app overlap, because Mac Capture never renders Work badges.

### Why magenta (evidence for the default)

These measurements came from Obsidian's bundled default theme (`app.css` in the
installed app), the vault's snippets, `dash.md`, and every bob-plugins stylesheet:

- Work badges already use grey, blue, green, yellow, orange, and red.
- Purple is Obsidian's interactive accent (hsl 258 88% 66%) and `--color-purple`. It
  appears on the dash PROJECTS, REFERENCES, and BLOCKED chips, as the default chip
  accent, and in nav-hotkeys styling.
- Cyan is the claimed "identity ink" for `#task` tag marks (a styles.css contract test
  pins it), inline code, and the dash TODAY chip's base accent when its tone is
  unavailable.
- The magenta/pink family is the only hue that no Bob Obsidian surface uses.
- The bare `--color-pink` token is light-mode `#d53984`. That is too close to light-mode
  red `#e93147`: CIEDE2000 17 solid, and 3.2 at the 14% chip tint that daily chips rely
  on. The existing Work tones' closest pair is yellow/orange at 22 solid, 6 tinted.
- The recommended mix pulls pink slightly toward purple and then toward `--text-normal`.
  The `--text-normal` step is the same move the task-tag hue makes with cyan. The mix
  renders `#923675` (plum) in light mode and `#e5a9d8` (orchid) in dark.
- Its minimum distance from any Work tone, the accent, or cyan is about 24 solid in
  light mode and 22 in dark, which matches the existing tones' own floor.
- Under simulated deuteranopia and protanopia it stays 10 or more from every Work tone.
  That beats the existing tones' own floor of 5–8.
- Text contrast is 7.0:1 on white and 8.9:1 on the dark background.
- Meaning: magenta sits off the blue→red heat ramp, so it reads as a distinct "exactly
  full" marker rather than one more heat step. Some public scales (UV, AQI) use purple
  past red. Here red keeps "over", and magenta never sits above it.

> [!decision] color = pink
>
> Use tone name `pink` everywhere this plan says `magenta`. Its CSS value is
> `var(--color-pink, #d53984)` (no mix).

> [!decision] color = purple
>
> Use tone name `purple` everywhere this plan says `magenta`. Its CSS value is
> `var(--color-purple, #7852ee)` (no mix). The existing invalid-tone assertion
> `workToneClass("purple") === ""` must switch to another non-tone such as `"teal"`.

## Repository access

From the bob-cli workspace, use `/sase_repo` and only the printed paths:

```sh
sase repo open bob-plugins -r 'Add the exact-limit magenta Work badge tone to bob-ledger-tools'
sase repo open gh:bobs-org/bob -r 'Mirror the exact-limit magenta Work badge tone in the dash.md fallback'
```

Read each repository's `AGENTS.md`, check `git status` before editing, and preserve
unrelated changes. bob-ledger-tools uses `src/fragments.json`. Edit `src/` fragments and
`styles.css`, then run `npm run build`. Never hand-edit the generated `main.js`.

## The color value

Use this exact declaration value in both stylesheets, so the plugin and the dash
fallback stay byte-identical:

```css
color-mix(in srgb, color-mix(in srgb, var(--color-pink, #d53984) 80%, var(--color-purple, #7852ee)) 70%, var(--text-normal))
```

The value stays theme-native: community themes that remap pink, purple, or text remap it
too. Like the other tones, it never derives from `--task-status-*`.

## Changes

### 1. bob-plugins: the shared helper

`plugins/bob-ledger-tools/src/075-work-badge-tone.js`:

- `WORK_BADGE_TONES` becomes
  `["grey", "blue", "green", "yellow", "orange", "magenta", "red"]` (ladder order).
- In `workBadgeTone(count, cap)`, insert `if (count === cap) return "magenta";` right
  after the `count > cap` → `"red"` check and before the 80% check. Leave the
  cross-multiplied band checks unchanged.
- In the header comment, change the precedence line to `4L/5 <= n < L -> orange`,
  `n === L -> magenta`, and `n > L -> red`.
- Add `magenta` to the `withWorkToneClass` stale-tone regex. Otherwise a `15/15` →
  `16/15` refresh keeps both classes. Better still, build the alternation from
  `WORK_BADGE_TONES` so the regex can't drift again. Either way, the result must be
  exactly one tone class.
- `todayBadgeTone` and `workToneClass` need no logic change. They pick up magenta
  through `workBadgeTone` and `WORK_BADGE_TONES`.

`plugins/bob-ledger-tools/src/170-plugin-lifecycle.js`: bump the `workBadges` namespace
to `version: 2`. Update its comment to say v2 adds the `magenta` exact-cap tone, and
that the top-level api stays v3. The return vocabulary grows, and `dash.md` enumerates
tones, so the namespace version must say so.

Search the rest of `plugins/bob-ledger-tools/src/` for any other hard-coded tone list or
`bob-work-tone-(` alternation, and update it. Known call sites (`070`, `180`, `290`,
`340`) route through the helpers.

### 2. bob-plugins: styles

`plugins/bob-ledger-tools/styles.css`, Work badge tones block: add a rule between the
orange and red rules:

```css
.bob-plan-chip.bob-work-tone-magenta {
  --bob-plan-accent: <the color value above>;
}
```

Extend the block comment by one short sentence: magenta marks a count exactly at its
limit, and no other Bob chip uses that hue.

### 3. bob-plugins: version and README

- Advance `plugins/bob-ledger-tools/manifest.json` one minor version (currently `1.52.0`
  → `1.53.0`; bump from whatever is current) and the matching README version cell.
- In the README `bob-ledger-tools` row, change `grey/blue/green/yellow/orange/red` to
  `grey/blue/green/yellow/orange/magenta/red`, and `workBadges` namespace v1 to v2.
- Run `npm run build`.

### 4. bob-plugins: tests

`scripts/test-ledger-tools-work-badge-tone.cjs`:

- In the boundary vectors, the exact-cap cases become `"magenta"`: `100/100`, `15/15`,
  `10/10`, `1/1`, `2/2`, `3/3`, `5/5`, and the changed-cap `10/10`. `80/100`, `99/100`,
  `12/15`, `14/15`, `8/10`, `9/10`, and `4/5` stay orange. `101/100`, `16/15`, `11/10`,
  `2/1`, and `6/5` stay red.
- Assert `workToneClass("magenta") === "bob-work-tone-magenta"`, and that
  `withWorkToneClass("bob-plan-chip bob-work-tone-magenta", "red")` returns
  `"bob-plan-chip bob-work-tone-red"`. Do the same in the opposite direction.
- Assert that `WORK_BADGE_TONES` equals the seven-tone ladder.
- TODAY with themes `3/3`: links `8/10` stays orange and `10/10` becomes magenta.
- Lane and READY models: rename the `15/15` (whole lane `16/15`, TODAY 1) model from
  `orange` to `magenta` and assert `"magenta"`. `over` stays false, and the tooltip
  still says `1 over the whole-lane cap`. READY `87/100` stays orange, and a new READY
  `100/100` assertion expects magenta.
- Lane DOM transition test:
  - Insert a `12/15` step (orange) before the `15/15` step, and expect magenta at
    `15/15`.
  - The existing `16/15` step must still yield exactly `["bob-work-tone-red"]`. This
    guards the regex.
  - Change the READY DOM test's `100/100` expectation to magenta.
- Dashboard integration test (with `workBadges`):
  - Assert `ledgerApi.workBadges.version === 2`.
  - Keep the current vectors.
  - Add a second `runDashBlock` pass with themes `3/3`, links `10/10`, PENDING section
    `10/10`, NEXT `12/15`, and READY `100/100`. Assert TODAY `bob-work-tone-magenta`
    (the inline dash chip, through the dash's own `workToneClass`), PENDING magenta,
    NEXT orange, and READY magenta.
- Dashboard fallback test (old API without `workBadges`, which exercises the
  `dash.md`-local copy). Change the inputs so every new boundary shows: PENDING `10/10`
  → magenta, NEXT `12/15` → orange, READY `60/100` → yellow, and TODAY links `4/10` with
  themes `2/3` → green. Keep exact `tonesIn` assertions. Update the inline comment to
  say these vectors guard the local copy's exact-cap and 80/60/40 boundaries.
- New CSS contract test:
  - Every `WORK_BADGE_TONES` entry has a `.bob-plan-chip.bob-work-tone-<tone>` rule in
    `plugins/bob-ledger-tools/styles.css`.
  - The magenta rule contains the exact color value above.
  - When `BOB_DASHBOARD_FILE` is set, the first `dataviewjs` block of `dash.md` has a
    `.task-count-widget .task-count-chip.bob-work-tone-<tone>` rule for every tone, and
    its magenta rule contains the same exact value. Skip only that part, with a log
    line, when the variable is unset.

`scripts/test-ledger-tools-dashboard-parity.cjs`: rename the `orange` `15/15` model to
`magenta` and assert `"magenta"`.

`scripts/test-ledger-tools-ready-badge.cjs`: the `100/100` assertion becomes
`"magenta"`.

`scripts/test-ledger-tools-plan-budget.cjs` compares against `workBadgeTone` and should
still hold. Re-run it to confirm.

### 5. Bob vault: `dash.md` fallback mirror

In the opened vault checkout's `dash.md`, first `dataviewjs` block:

- Local `workTone`: add `if (count === cap) return "magenta";` right after the
  `count > cap` line. In the comment above it, change `4L/5<=n<=L orange` to
  `4L/5<=n<L orange, n===L magenta`.
- Local `workToneClass`: add `"magenta"` to the allowed list (between `"orange"` and
  `"red"`). Without it, the plugin's magenta tone falls back to the TODAY chip's base
  cyan accent.
- Inline style block: add
  `.task-count-widget .task-count-chip.bob-work-tone-magenta { --task-count-accent: <the color value above>; }`
  between the orange and red tone rules.

Change nothing else in the dashboard. Plugin sync does not deploy dashboard Markdown.
This change reaches the vault through the vault repository's normal SASE finalization
and the documented vault Git sync.

### 6. bob-cli docs

- `docs/dashboard.md`, "Work badge colors":
  - Replace the table with the new one above (without the "Was" column).
  - Change the precedence sentence to: "Check zero, strict excess, and exact equality
    before descending through 80%, 60%, and 40%."
  - In the NEXT example, section `15/15` with a larger whole lane becomes **magenta**.
  - Replace "Orange, from 80% through the cap, is a presentation state ..." with: orange
    (80% up to the cap) and magenta (exactly at the cap) are presentation states, not
    lints or `over` conditions.
  - Add one sentence on why: magenta marks a lane that is exactly full, and no other Bob
    chip uses that hue.
- `docs/plan.md`:
  - TODAY paragraph: `grey/blue/green/yellow/orange/red` becomes
    `grey/blue/green/yellow/orange/magenta/red`.
  - PENDING/NEXT paragraph: `4 * L / 5 <= n < L` is orange and `n === L` is magenta. The
    `15/15` example becomes magenta. Apply the same presentation-state sentence.
  - READY paragraph: same bands. Change "From 80% through the cap is orange" to orange
    from 80% up to the cap and magenta exactly at it, both presentation states. Only a
    strict excess raises `ready_cap_exceeded`.
- Grep bob-cli `docs/` and `README.md` for leftover `n <= L`, `through the cap`, or
  `/orange/red` wording about Work badges, and fix any you find.

No SASE memory note or decision record states the Work badge colors, so no memory change
is needed.

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
and don't hard-code a home or workspace path. It must report the integration, fallback,
and dash CSS contract tests as run, not skipped.

The previous Work badge change found 6 `npm test` failures that predate it: Live Preview
unblocked-link glyphs, Keep counts, and Depends-On conversion. If those same failures
still appear, show they're unrelated by running with the changes stashed, and report
them. Don't fix them here.

Run `git diff --check` in the vault and bob-cli checkouts too. No Rust code changes, so
the Rust suite isn't required. If a live Obsidian UI is available, look at an exact-cap
badge in light and dark modes. If not, say so instead of claiming visual verification.

## Deployment

```sh
bob plugins sync --no-pull --repo <opened-bob-plugins-path> -p bob-ledger-tools --dry-run
bob plugins sync --no-pull --repo <opened-bob-plugins-path> -p bob-ledger-tools
```

Don't use `--force` over dirty deployed files.

- Report plugin installation separately from `dash.md` delivery, and note the remaining
  Obsidian plugin reload and vault-sync steps.
- Until the vault sync lands, the dash TODAY chip at exactly its cap shows the base cyan
  accent, because the old local list doesn't know magenta. Daily blocks and the dash
  PENDING/NEXT/READY chips, which the plugin renders, show magenta right after the
  reload.
- Account for every modified repository (bob-cli, bob-plugins, and the vault) in the
  SASE final declaration.
