---
tier: tale
title: PLAN review time follows the ^gtd link; exempt themes wear a gray pill
goal:
  An all-exempt (PLAN) Pomodoro counts as review only when it links that day's ^gtd row
  or has no Task Links, so ad-hoc planning lands on its project in Stats and home chips;
  exempt theme names render as a gray pill on the Obsidian Pomodoro line.
size: medium
decisions:
  link_budget:
    ask:
      Also count Task Links under all-exempt (PLAN) entries toward plan.max_links,
      except that day's ^gtd row?
    default: false
    why:
      The request is the Stats mismatch; the report calls this budget leak low-stakes
      and separable
    answer: false
  decision_record:
    ask:
      Add a decisions-web record "PLAN Is Review Only With The ^gtd Link" for the new R4
      rule?
    memory:
      - decisions
    default: false
    answer: false
proposed_by: bbugyi200.athena.15n
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-28 21:32:18
status: wip
---

# Stats: let the `^gtd` link, not the PLAN name, decide review time; gray pill for exempt themes

Two independent fixes requested together:

1. **Stats / homes mismatch.** This implements option D from the research report
   `research:202610/plan_theme_exemption.md`. Today, a session named only with
   `plan.exempt` themes (`PLAN`) always goes to the `@review:PLAN` bucket, and its Task
   Links never attribute. Ad-hoc planning such as `() — PLAN` →
   `[[sase_blog#^s0-spike]]` therefore never shows up under `sase_blog`. New rule: an
   all-exempt named session is review **only** when it links that day's `^gtd` row or
   has no Task Links. Otherwise its links decide its homes, exactly as in any work
   session. PLAN stays in `plan.exempt`, and nothing about the plan budget changes
   (unless `link_budget` is on).
2. **Gray chip for exempt themes.** On the Obsidian Pomodoro line, an exempt theme such
   as `PLAN` gets the `bob-pomo-theme-muted` class and no `data-theme-hue`, and
   `plugins/bob-ledger-tools/styles.css` paints that class with a transparent
   background. So `PLAN` reads as plain, unhighlighted text. Bob Mac Capture already
   gives the muted pill a gray fill (`PomodoroThemePillView`, `quaternaryLabelColor`).
   Obsidian should match it: a neutral gray tint behind the muted text, so `PLAN` reads
   as a configured theme. Hue resolution stays unchanged (`hue: null`,
   `hue_source: "exempt"`), so `decisions:theme-hue-is-config-pin-exempt-area-hash`
   still holds as written.

Two repositories change: bob-cli (this repo) and the linked `bob-plugins` repo. Open
`bob-plugins` only through `/sase_repo` (`sase repo open bob-plugins -r "<why>"`), read
its `AGENTS.md` first, and do every plugin edit in the path that command prints. Bob Mac
Capture needs no change: it is a thin client that renders the homes Bob reports, and it
already draws the gray muted pill.

## Part 1 — bob-cli: R4 decided by the `^gtd` link

### `src/native/pomodoro_homes/session.rs` (`attribute_session`)

- Factor the existing bridge predicate into one boolean, `links_today_gtd`: some
  resolved link has block id `gtd` (case-insensitive) and resolves to `daily_path`,
  where `daily_path` is a canonical daily note. Keep the `!exempt.is_empty()` guard on
  the bridge.
- `all_exempt_name` is today's `named_review` expression: name present, components
  non-empty, every component in `exempt_names`.
- New rule:
  `named_review = all_exempt_name && (resolved.is_empty() || links_today_gtd)`. The
  bridge stays `name.is_none() && !exempt.is_empty() && links_today_gtd`.
- When `all_exempt_name && !named_review`, fall through to the ordinary work path. Do
  not add a special branch: the existing `home_for_link(..., bridge_name)` already maps
  another day's canonical `^gtd` to a review-home task share. The
  `chip(name, &parts, false, false)` call already yields e.g. `sase_blog`. Leave
  `review_session` itself unchanged. Add a short comment stating R4: the `^gtd` link,
  not the name, marks the ritual.
- Every caller (`bob plan` forecasts, capture agenda/pomodoros, `capture-complete`,
  Stats `HomesEngine`, existing-task-link, pomodoro start) goes through
  `attribute_session`/`attribute_ledger_entry`, so they all pick up the rule. Stats has
  no attribution cache, so past days re-attribute on read.
- Stats link health (`stats/homes.rs`) already counts only `SessionKind::Work`. A PLAN
  session that becomes work now contributes unfiled/ambiguous link health, which is
  correct.

### Other bob-cli code

- `src/native/stats/registry.rs`, `review_time` description: say that it counts sessions
  named only with `plan.exempt` themes (PLAN by default) that link that day's `^gtd` row
  or have no Task Links, unnamed sessions bridged through that row, and linked `^gtd`
  task shares inside work sessions. Keep it one sentence. Update any test or snapshot
  that pins the old string.
- `src/native/pomodoro_themes/split.rs:~1172`: the human preview line
  `exempt: these sessions count as review time in Stats` becomes false. Reword it to
  something like
  `exempt: these sessions count as review time in Stats when they link that day's ^gtd`.
  Update any test that asserts the old text.

### Shared vectors `tests/fixtures/pomodoro_homes/vectors.json`

These vectors are the contract for both Rust (indexed and disk sources) and the JS
mirror. Hand-calculate every expectation, following R5's splitting and rounding rules.

- `review-decisions`, session 1 (`— GTD` with only `~~[[bob#^foreign]]~~`): it now
  becomes a **work** session. Expected: `kind: "work"`, label/chip `bob`, hue `orange`,
  one `bob.md` project part with `tasks: 1`, `share: 100`, `minutes: 25`. Its
  `expected_lead_homes[0]` becomes `bob.md`. Sessions 2–3 keep their current
  expectations.
- Add a case `exempt-name-gtd-link` (`exempt: ["GTD"]`, same `daily_path`) that covers:
  - `— GTD` with same-day `[[#^gtd]]` plus `[[bob#^work]]`: review, `@review:GTD` 100%,
    `chip: null`. This is the morning walk with dispatches.
  - `— GTD` with no links (closed, with duration): review, `@review:GTD`.
  - `— GTD` with only `[[2026/20261017#^gtd]]` plus `[[bob#^work]]`: work, split between
    an `@review:GTD` task share and `bob`, mirroring `daily-review-link`.
  - An open `() — GTD` entry with one project link: a forecast work session
    (`state: "forecast"`, task-based shares). This is the `sase_blog` scenario.
- Rust `src/native/pomodoro_homes/tests.rs` already iterates all cases. Confirm the new
  case runs under both sources.

### bob-cli docs (hand-maintained; run `just docs-check`)

- `docs/pomodoro-homes.md` R4: rewrite the first paragraph so that an all-exempt name
  makes a session review only when it links that day's `^gtd` row or has no dedicated
  Task Links. Otherwise its links decide (another day's `^gtd` is still a review-home
  share), and its name still feeds the Stats theme dimension. Replace the final
  paragraph ("The named-session rule remains based on the session name: an all-PLAN
  named session is review even when it links project tasks…") with the new rationale:
  the `^gtd` link marks the ritual, and a morning entry that links `^gtd` plus
  dispatched tasks stays whole in review. In the vectors table, update the
  `review-decisions` row and add an `exempt-name-gtd-link` row.
- `docs/stats.md:199`: make the `review_time` row match the new registry description
  word for word.
- `docs/plan.md` "Exempt entries": add one sentence. Exemption is a budget rule; Stats
  charges an all-exempt session to review only when it links that day's `^gtd` row or
  has no Task Links ([R4](pomodoro-homes.md#r4-review-sessions); verify the anchor). The
  `bob plan -f json` example's PLAN entry keeps `kind: "review"` because only `^gtd`
  sits under it. Check the ledger above the example, and change the example only if that
  is false.

## Part 2 — bob-plugins: mirror the rule and paint the gray pill

Work in the `bob-plugins` checkout printed by `sase repo open bob-plugins`. Edit `src/`
fragments only, never generated `main.js`, and keep each fragment ≤ 1000 lines.

### Homes mirror `plugins/bob-ledger-tools/src/142-pomodoro-home-chips.js`

In `pomodoroHomeShareParts`, compute `nameAllExempt` (today's `namedReview` expression).
Then pre-scan `entry.links` for a same-day `^gtd`, using the same test the loop uses:
`pomodoroHomeResolveFile`, `PLAN_DAILY_PATH_RE`, block id `gtd`, and
`pomodoroHomePathEqual(path, dailyPath)`. This pre-scan must record **no**
warnings/health. Set
`namedReview = nameAllExempt && (links.length === 0 || sameDayGtd)`. Keep skipping the
resolution loop when `namedReview` is true, so review sessions add no link warnings, as
today. Otherwise run the loop unchanged, so an all-exempt name without `^gtd` becomes a
work session. Leave the bridge and the chip logic in `pomodoroHomeSessionModel`
unchanged: its `namedReview` there derives from `aggregate.review`.

### Fixture copy

Copy `tests/fixtures/pomodoro_homes/vectors.json` byte-for-byte to
`plugins/bob-ledger-tools/scripts/fixtures/pomodoro_homes/vectors.json`. Check with
`diff -q`. `scripts/test-ledger-tools-pomodoro-home-chips.cjs` runs every
non-`rust_only` case.

### Gray exempt pill `plugins/bob-ledger-tools/styles.css` (~line 3515)

Replace the transparent muted rule with a neutral gray tint that mirrors the hue-pill
percentages, using `--text-muted` as the "hue":

- `.bob-pomo-theme-colors .bob-pomo-theme-muted` keeps `color: var(--text-muted)`.
- For `.bob-pomo-theme-colors .bob-pomo-theme-muted:not(.bob-pomo-theme-unconfigured)`:
  `background-color: color-mix(in srgb, var(--text-muted) 16%, transparent)`; dark mode
  24%, using both existing dark selector forms (`.theme-dark.bob-pomo-theme-colors …`
  and `body.theme-dark .bob-pomo-theme-colors …`); hover 24% / dark hover 32%.
- Unconfigured pills (`.bob-pomo-theme-muted.bob-pomo-theme-unconfigured`) keep their
  current look: transparent, dashed warning outline, 0.7 opacity. The gray fill then
  signals "configured exempt theme" and the dashed outline signals "not in your themes".
  The `:not(...)` scoping avoids the specificity fight with the higher-specificity
  `body.theme-dark …` selectors.
- Update the rule's comment ("Muted pill for exempt themes such as GTD: hover
  background, muted text") to describe PLAN with the gray tint.

All surfaces that use the class get the pill: the Live Preview editor marks on the
Pomodoro line (`279-plugin-pomodoro-theme-chips.js`), the Reading view
(`279-plugin-pomodoro-theme-chips-reading.js`), the `bob-plan` ★ theme line
(`290-plugin-today-and-location.js`), and `[theme:: PLAN]` task pills
(`279-plugin-task-theme-marks.js` / `-reading.js`). No JS change is needed. Add a small
assertion to an existing CSS-reading test, such as
`scripts/test-ledger-tools-pomodoro-themes.cjs`, that reads `styles.css` and checks two
things: the muted, non-unconfigured rule sets a non-transparent `color-mix(...)`
background, and the unconfigured combination stays transparent. If `npm test` runs an
explicit file list, wire any new test file into it.

### Plugin docs

In `docs/bob-ledger-tools.md` (~line 150), change "with a muted pill for exempt themes
such as PLAN" to describe a gray pill. Then, back in bob-cli, run `just docs-gen` so the
mounted copy `docs/reference/plugins/bob-ledger-tools.md` refreshes. Commit that refresh
with the bob-cli change.

In bob-cli `docs/pomodoro-themes.md` PT6, change **Muted pill** to "exempt themes such
as PLAN: a neutral gray tint (no hue) behind muted text, rising on hover, so a
configured exempt name still reads as a theme". Change **Unconfigured pill** to say that
Obsidian draws the dashed outline over muted text without the gray fill, and that the
Mac keeps its muted fill.

### Build, test, deploy

In `bob-plugins`: run `npm run build`, `npm test`, `npm run validate`, and
`npm run docs:check`, then `bob plugins sync`, as `bob-plugins/AGENTS.md` requires.

> [!decision] link_budget Also implement report option D2 in both budget engines. In
> Rust `src/native/plan_budget/mod.rs`, `collect_entry_links` stops returning an empty
> list for exempt entries. Under an all-exempt entry, drop links whose block id is `gtd`
> and whose target canonicalizes to the daily file (`block_links` already canonicalizes
> empty/daily targets). Feed the rest into `distinct_links`/`seen_links` like any entry.
> Exempt entries still never count as themes, the ★ highlight, or the duplicate lint.
> Mirror this exactly in `plugins/bob-ledger-tools/src/050-plan-budget.js`. Update the
> `docs/plan.md` "Exempt entries" and "Links" text ("below an open, non-exempt entry"),
> confirm that the Today conformance examples and the `bob plan -f json` example still
> hold, and add Rust and `scripts/test-ledger-tools-plan-budget.cjs` cases: PLAN with
> `^gtd` plus one task counts 1 link; PLAN with only `^gtd` counts 0.

## Out of scope

- `plan.exempt` membership, `max_themes`, the highlight, the duplicate lint, PT13
  task-theme stamping, and the hue order all stay unchanged. No config edits.
- Bob Mac Capture changes or fixture regeneration. Unconfigured-pill styling stays as
  is.
- Do not edit `CHANGELOG.md` or the Cargo version (release train).

> [!decision] decision_record Use `/sase_memory_write` to add the strand
> `sase/memory/decisions/plan-is-review-only-with-gtd-link.md`, keyword "PLAN Is Review
> Only With The ^gtd Link". Give it `metadata.status: accepted` and
> `metadata.decided: <today>`, and follow the shape of
> `decisions:theme-hue-is-config-pin-exempt-area-hash`. Then run `sase memory init`.
>
> - **Applies to:** the bob-cli homes resolver (R4), Stats `review_time`, and the
>   bob-ledger-tools homes mirror.
> - **Claim:** the new R4 rule. `plan.exempt` stays a budget rule.
> - **Why:** cite `research:202610/plan_theme_exemption.md` (option D over A/B/C),
>   `research:202610/pomodoro_area_project_attribution/pomodoro_area_project_attribution.md`,
>   `research:202610/work_kind_pomodoro_theme_set/work_kind_pomodoro_theme_set.md`, and
>   commit `f6287847`, which this rule replaces.
> - **Cost:** about 475 historical minutes re-attribute, and a morning entry linking
>   `^gtd` plus a project task still counts entirely as review.
> - **Reopens when:** the `^gtd` ritual row stops being the morning-review marker, or
>   PLAN leaves `plan.exempt`.

## Validation

- bob-cli: `just fmt`, `just lint`, `just test` (or `just all`), and `just docs-check`.
  Before committing, check `just check` if it is the documented gate.
- Read-only spot check against the live vault, using the freshly built binary:
  - `bob plan -f json` should forecast the open `() — PLAN` entry in `2026/20261028.md`
    as `kind: "work"`, with a `sase_blog` part.
  - The morning PLAN sessions that link `~~[[#^gtd]]~~` should stay `@review:PLAN`.
  - `bob stats show worked review_time -r 2026-10-01..2026-10-28 -f json` should show
    `review_time` dropping by about the report's 300 October minutes.

  Never write to the vault.

- bob-plugins: the commands listed above. After `bob plugins sync`, the Obsidian
  Pomodoro line should show `PLAN` in a gray pill and the open PLAN line a `○ sase_blog`
  home chip. Bryan verifies this visually on the Mac.

## Commits

- bob-cli: `fix(homes): let the ^gtd link, not the PLAN name, decide review time`, plus
  the PT6/docs-gen refresh. This is a Stats behavior change with no JSON shape change,
  so no `!`.
- bob-plugins: `fix(ledger-tools): mirror ^gtd-decided review sessions` and
  `feat(ledger-tools): gray pill for exempt Pomodoro themes`. One commit is fine if that
  is the repo's convention.
