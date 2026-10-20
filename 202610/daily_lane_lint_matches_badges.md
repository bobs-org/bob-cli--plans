---
tier: tale
title: Make the daily PENDING/NEXT cap warnings agree with their badges
goal:
  A daily bob-plan block shows a PENDING or NEXT cap warning only when that badge is
  red, quoting the same section count the badge shows.
size: small
proposed_by: bbugyi200.apollo.9o
proposed_with: claude/opus
create_time: 2026-10-20 14:50:52
status: wip
---

# Make the daily PENDING/NEXT cap warnings agree with their badges

## Outcome and scope

In every daily note's `bob-plan` block, the `pending_cap_exceeded` / `next_cap_exceeded`
warning lines under the badges must describe the same numbers the PENDING and NEXT
badges show. A warning appears only when its badge is red (displayed section count
strictly over the cap), and it quotes that section count. A badge under or at its cap
never has a warning line under it.

This is one small tale: a bounded change to one bob-ledger-tools source fragment, its
existing tests, and the documentation that currently promises the opposite. No new
commands, options, config keys, task states, or memory changes. The native `bob plan`
CLI, `nextBudget()` / `pendingBudget()`, navigation notices, and the dashboard badges
are untouched.

## Diagnosis (root cause)

Evidence gathered while planning (counts are a live snapshot, not fixed acceptance
values):

- The screenshot `~/tmp/screenshots/20261020_144321.png` shows `TODAY 4/5 · 6/15`,
  `PENDING 5/8`, `NEXT 7/16`, `READY 13/32`, and under them
  `PENDING whole lane has 9/8 tasks (including TODAY); release some with Alt+N pending_cap_exceeded`.
- The vault has ten `[/]` task lines. One (`job_2026_apply_29.md`, `#hide`) is hidden,
  leaving nine visible In Progress tasks. Four of them are linked under today's open
  Pomodoros (three `#ref` tasks in `sase_pypi_releases.md`, one in `dev.md`), so they
  are TODAY. The dashboard PENDING section is therefore 9 − 4 = 5. `bob plan -f json`
  agrees: whole lane `pending.count = 9` with four `[/]` rows in `today_tasks`. Every
  count is computed correctly. The two surfaces just use different populations.
- Bryan raised the caps on the Mac at 14:42 today (chezmoi commit `eafb7a20`,
  `max_pending: 8`, `max_next: 16`, `max_ready: 32`). With cap 8, the section (5) is
  under the cap but the whole lane (9) is over it. That made the contradiction visible.
  apollo's `~/.config/bob/config.yml` still has 5/15/45 because it has not pulled that
  chezmoi commit yet. That is ordinary sync lag, not part of this bug.
- Code path in bob-plugins
  `plugins/bob-ledger-tools/src/340-dashboard-views-and-plan-block.js`,
  `planBlockModel()`:
  - The badges come from `resolvePlanDashboardBudget()` → `dashboardLaneBadgeModel()`
    (in `070-ready-and-review.js`), which shows and colors the **section** count (TODAY
    excluded).
  - The lints come from `next` / `pending` = `laneBudgetFromTasks()` (in
    `060-today.js`), the **whole lane, TODAY included**. They fire on `next.over` /
    `pending.over` and print
    `… whole lane has ${count}/${cap} tasks (including TODAY) …`.
- History: the approved plan `plan:202610/dashboard_badge_warnings.md` had Bryan choose
  `lane_warning_scope = section` for badge warnings. The follow-up plan
  `plan:202610/daily_dashboard_badge_parity.md` (bob-plugins `bf1703a`, 1.42.0) moved
  the daily badges to section counts. It deliberately kept the daily lints whole-lane
  and only relabeled them, and its acceptance table even required `PENDING 4/10` +
  `PENDING whole lane has 11/10` to coexist. That leftover whole-lane lint is the root
  cause. Bryan's report shows it reads as a wrong count, not as useful extra context.

## Behavior contract

1. The daily `next_cap_exceeded` / `pending_cap_exceeded` lints are derived from the
   same per-lane badge model that paints the badge (`nextModel` / `pendingModel` in
   `planBlockModel()`). A lane lint is emitted if and only if that model is available
   (`placeholder === false`) and `sectionOver` is true (`section > cap`, strict).
2. Message text mirrors the badge numbers:
   `NEXT has ${section}/${cap} tasks; release some with Alt+N` and
   `PENDING has ${section}/${cap} tasks; release some with Alt+N`. Keep the lint codes,
   the NEXT → PENDING → READY order, and the existing `  <code>` suffix formatting.
3. An unavailable section (`–`) never produces a lane lint, and the lint never falls
   back to whole-lane numbers. Whole-lane pressure stays only in the badge tooltip and
   aria text (unchanged `dashboardLaneBadgeModel()` output).
4. Unchanged: `planBlockModel()`'s returned whole-lane `next` / `pending` objects (their
   `count` / `over` meanings), `laneBudgetFromTasks()`, `dashboardLaneBudget()`
   (including its raw `.over`), `nextBudget()` / `pendingBudget()`, navigation notices,
   the native `bob plan` CLI and its JSON warnings, the TODAY ledger budget and its
   lints, the READY lint, badge colors, and `dash.md`.

For the screenshot case (section 5, TODAY 4, whole lane 9, cap 8), the block must show
`PENDING 5/8` with no PENDING warning line. With section 9 and cap 8, it shows a red
`PENDING 9/8` and
`PENDING has 9/8 tasks; release some with Alt+N  pending_cap_exceeded`.

## Implementation

1. **Open the plugin repo.** Use `sase repo open bob-plugins` with a task-specific
   reason, read its `AGENTS.md`, and check `git status`. Use only the printed path. The
   source may have advanced since planning, so re-locate the code by name rather than by
   line number.

2. **Re-source the lane lints.** In
   `plugins/bob-ledger-tools/src/340-dashboard-views-and-plan-block.js`,
   `planBlockModel()`: replace the `hasTasks && next.over` / `hasTasks && pending.over`
   conditions and messages with conditions on `nextModel` / `pendingModel` as specified
   above. Keep the whole-lane `next` / `pending` computation because the model still
   returns it. Update the function's header comment, which currently says whole-lane
   `next`/`pending` "remain for cap lints", to say they are retained for callers while
   lints follow the badges. Keep the fragment at or below the repo's 1000-line limit (it
   is about 971 lines now). Do not hand-edit `main.js`. Run `npm run build` to
   regenerate it.

3. **Update the tests that pin the old contract, and add the regression.**
   - `scripts/test-ledger-tools-plan-budget.cjs`:
     - In "planBlockModel adds lane cap lints when a lane is over" (16 Next tasks, no
       Today), expect `NEXT has 16/15 tasks; release some with Alt+N`.
     - In "planBlockModel daily badges follow dashboard section counts": the section-10
       / whole-17 NEXT case and the section-4 / whole-11 PENDING case must now have
       **no** lane lint, while `model.next.count` / `model.pending.count` still report
       the whole lane. The section-16 NEXT case must carry the new-format lint. Add the
       screenshot regression: PENDING section 5 + TODAY 4 with caps
       `{ maxPending: 8, maxNext: 16 }` → `pendingText === "PENDING 5/8"`,
       `pending.count === 9`, `pending.over === true`, and no `pending_cap_exceeded`
       lint. Also add an at-cap section (`8/8`) with no lint, and an unavailable
       supplied dashboard budget (nulls) with no lane lint even when the whole lane is
       over.
   - `scripts/test-ledger-tools-ready-badge.cjs` (the NEXT/PENDING/READY lint-order
     test, `isToday: () => false`): update the two lane strings to the new format,
     keeping the order.
   - `scripts/test-ledger-tools-dashboard-parity.cjs` (the real daily paint vs dashboard
     test with NEXT section 10 / whole 17): assert the painted block has no
     `next_cap_exceeded` `bob-plan-lint` child while the badge tooltip still mentions
     `whole lane 17/15`. Add a real-paint case with a section over the cap. It must
     assert the red badge's visible fraction and the lint text carry the same
     `section/cap`.
   - Run `scripts/test-ledger-tools-work-badge-tone.cjs` unchanged; its "cap equality
     still emits no cap lint" test must keep passing.

4. **Release and docs.**
   - Bump `plugins/bob-ledger-tools/manifest.json` per the repo's convention for a fix
     (patch bump from the current version), and update any version references that the
     repo's validation requires.
   - bob-plugins `docs/bob-ledger-tools.md`, the `bob-plan` paragraph: replace
     "whole-lane pressure stays in the tooltip and in `next_cap_exceeded` /
     `pending_cap_exceeded` lints that name the whole lane, including TODAY". The new
     text says whole-lane pressure stays in the tooltip, and the cap lints appear only
     for a red badge and quote its section count. Run `npm run docs:gen` if the
     inventory needs it, then `npm run docs:check`. bob-cli's `docs/reference/plugins/`
     copy is mounted from this file. Do not edit it in bob-cli.
   - bob-cli `docs/plan.md`, "Lanes (NEXT and PENDING)": rewrite the sentences saying
     daily `next_cap_exceeded` / `pending_cap_exceeded` lints keep whole-lane counts.
     The new text says daily lints follow the badge's section count and fire only for a
     strict section excess, while `bob plan`, its JSON warnings, and navigation notices
     still check the whole lane including TODAY. Replace "Daily cap lints name the whole
     lane, including TODAY." in the output-surface table row for the daily `bob-plan`
     block to match. Leave the CLI warning-code table (`next_cap_exceeded` /
     `pending_cap_exceeded` rows) accurate for `bob plan`.
   - bob-cli `docs/dashboard.md`: the paragraph saying "CLI warnings and navigation
     notices continue to check the whole lane, including TODAY" stays true. Add that the
     daily block's cap lint lines follow the displayed section counts.
   - Run bob-cli `just docs-check`.

5. **Deploy.** After committing, run `bob plugins sync` so the vault picks up the
   rebuilt plugin, as the bob-plugins `AGENTS.md` requires.

## Validation and acceptance

- In bob-plugins:
  - `npm run build:check`
  - `node --test scripts/test-ledger-tools-plan-budget.cjs scripts/test-ledger-tools-ready-badge.cjs scripts/test-ledger-tools-dashboard-parity.cjs scripts/test-ledger-tools-work-badge-tone.cjs scripts/test-ledger-tools-today.cjs`
  - `npm run validate`
  - one full `npm test` on the final tree. Hand it to `/sase_monitor` if it may outlive
    the turn. If an unrelated failure appears, check it against the unchanged baseline
    before attributing it, and report it with evidence.
- In bob-cli: `just docs-check`.
- Acceptance matrix (synthetic fixtures, controlled day, explicit caps):

  | Fixture (section / TODAY / whole lane / cap) | Badge                   | Lane lint                                                              |
  | -------------------------------------------- | ----------------------- | ---------------------------------------------------------------------- |
  | PENDING 5 / 4 / 9 / 8 (screenshot)           | `PENDING 5/8`, non-red  | none                                                                   |
  | NEXT 10 / 7 / 17 / 15                        | `NEXT 10/15`, non-red   | none                                                                   |
  | PENDING 4 / 7 / 11 / 10                      | `PENDING 4/10`, non-red | none                                                                   |
  | NEXT 15 / 1 / 16 / 15 (at cap)               | `NEXT 15/15`, non-red   | none                                                                   |
  | NEXT 16 / 0 / 16 / 15                        | `NEXT 16/15`, red       | `NEXT has 16/15 tasks; release some with Alt+N  next_cap_exceeded`     |
  | PENDING 9 / 0 / 9 / 8                        | `PENDING 9/8`, red      | `PENDING has 9/8 tasks; release some with Alt+N  pending_cap_exceeded` |
  | Section unavailable, whole lane over         | `–`, non-red            | none                                                                   |

- The whole-lane APIs still report the excess. In the 17/15 and 11/10 fixtures,
  `planBlockModel().next/pending.over` and `dashboardLaneBudget().over` stay `true`.
- After `bob plugins sync`, re-run `bob plan -f json` and count the dashboard PENDING
  section to confirm against the live vault. A daily block whose PENDING badge is at or
  under its cap shows no PENDING warning line. Bryan can confirm visually on the Mac
  once the vault syncs.

## Non-goals

- Do not change the native `bob plan` CLI, its JSON `warnings`, tmux or capture
  surfaces, or navigation notices. They present the whole lane (with the TODAY rows
  listed beside it), so their warnings match their own numbers.
- Do not touch Bryan's `~/.config/bob/config.yml` or the chezmoi source. The 5/15/45 vs
  8/16/32 difference between apollo and the Mac is pending chezmoi sync, not a defect.
- No vault Markdown edits, no task mutations, no memory changes.
