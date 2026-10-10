---
tier: tale
title: Show daily theme and Task Link budgets in the idle capture agenda
goal:
  Display the saved daily plan's theme and Task Link counts and configured limits in Bob
  Mac Capture's idle agenda using the existing CLI budget engine and preview capsules.
size: medium
proposed_by: bbugyi200.apollo.6g.w1
create_time: 2026-10-10 13:35:53
status: wip
---

# Show daily theme and Task Link budgets in the idle capture agenda

When Bob Mac Capture opens with an empty draft, show the current daily file's theme and
Task Link counts and limits alongside its Now / Next / Later agenda. Reuse the familiar
`Themes 3/3` and `Links 8/10` capsules already shown in capture previews such as `=`.
These describe the saved daily plan; a typed capture preview continues to describe the
proposed capture's result.

This is one medium tale spanning bob-cli and bob-mac-capture: a bounded additive JSON
change, reuse of existing presentation, integration into agenda measurement, and
contract/layout regression coverage. One coding agent can complete it in order; separate
epic phases are unnecessary.

## Findings and implementation context

Inspected bob-cli at `44d2ed6` and bob-mac-capture at `26aac7d`.

- In bob-cli, `src/native/capture/budget.rs::append_plan_budget` already uses
  `plan_budget::compute_for_daily` for preview/submit budgets. It emits a budget only
  when a capture changes today's Pomodoros section. This explains why the capsules
  appear while typing some operators but are absent at idle.
- `src/native/capture_pomodoros.rs::list_capture_pomodoros` reads the selected daily
  note once and adds agenda fields for `--tasks`, but its result has no budget.
  `src/native/capture_pomodoros_agenda.rs` resolves displayed tasks; those rows are not
  the plan-budget counting rule.
- `docs/plan.md` is the counting contract. Open entries count; completed/cancelled
  entries do not. Themes are distinct non-exempt components, including components of
  merged names. Links are distinct target/block-ID pairs in non-exempt entries,
  including accepted links in indented prose. Exempt entries, struck links, fences,
  daily-note self-links, and unnamed entries have existing rules. Counting rendered
  groups or summing `items`/`task_link_count` would give incorrect totals.
- In the Mac repo, `Sources/CaptureCore/CaptureAgendaModels.swift` decodes the snapshot;
  `CaptureAgendaPresentation.swift` creates rows; `CaptureAgendaFitPlanner.swift`
  budgets and folds them. AppKit/SwiftUI rendering and measurement live in
  `Sources/BobMacCapture/CaptureAgendaView.swift` and `CaptureAgendaRowMeasurer.swift`.
- `Sources/CaptureCore/CaptureModels.swift` already has `CapturePlanBudget` and
  `CapturePlanBudgetMeter`: `before` is optional and missing `added_themes` and
  `warnings` decode to empty lists. `CapturePlanBudgetPresentation.swift` already
  formats capsules and uses Bob's `over` flags. The current SwiftUI meter row is
  `planBudgetMeterRow` in `Sources/BobMacCapture/CapturePanelView.swift`.
- `BobProcessClient.captureAgenda` makes one `capture-pomodoros --format json --tasks`
  call and compares raw output bytes. `CaptureAgendaStore` caches that snapshot,
  preserves a last good same-day snapshot after failure, and invalidates yesterday's
  snapshot. Budget changes should participate in these existing paths.

Before implementation, read the applicable repo instructions and audited memories
`decisions:idle-capture-shows-ledger-agenda`, `decisions:mac-capture-is-a-thin-client`,
`glossary:Pomodoro`, and `glossary:Task Link`. Open bob-mac-capture with `/sase_repo`
and use its returned path. During planning, `sase repo open bob-mac-capture` failed
because the configured primary checkout was absent;
`sase repo open gh:bobs-org/bob-mac-capture -r "Implement daily budget in idle capture agenda"`
successfully opened its source. Use that supported fallback if necessary; do not search
sibling checkouts.

## Intended behavior and wire contract

Add optional top-level `plan_budget` to the existing schema-version-1
`capture-pomodoros --tasks` response:

```json
"plan_budget": {
  "status": "ok",
  "themes": {"count": 3, "cap": 3, "over": false},
  "links": {"count": 8, "cap": 10, "over": false}
}
```

The fields are a snapshot from the same daily-note contents as the agenda, computed with
the same plan configuration used by capture. No `before`, `added_themes`, capture-growth
warnings, or destination is invented for idle display. At-cap is within budget; `over`
means strictly greater than the cap. The default limits come from the existing
configuration loader (currently 3 and 10), never Swift constants.

| Source state                                       | JSON and idle display                                                                                   |
| -------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| Readable daily note with a Pomodoros section       | Include the budget and display both capsules, including real zero counts.                               |
| Empty section or only completed/cancelled entries  | Show `Themes 0/<cap>` and `Links 0/<cap>` above the existing empty-agenda message.                      |
| Missing daily note or missing Pomodoros section    | Omit the budget; preserve existing successful empty-list warnings and empty-state behavior.             |
| Missing config file                                | Use the loader's existing defaults.                                                                     |
| Invalid/unreadable plan config                     | Omit the budget, append one bounded `plan budget unavailable: ...` warning, and keep the agenda usable. |
| Older CLI omits `plan_budget`, or sends null       | Decode as nil and show the existing agenda without capsules.                                            |
| Failed refresh with a last good snapshot for today | Keep that snapshot's agenda and budget together under the existing stale title indicator.               |
| No current-day snapshot / day rollover             | Show the existing loading state; never show yesterday's budget.                                         |

`--all --tasks` still computes the budget from open entries only. Calls without
`--tasks` keep their output and configuration independence. Preserve existing human
output; this change adds an optional field to the JSON agenda, not a new command or
option. Invalid config does not fail the agenda or silently substitute default caps.

Place one compact meter row immediately below the Today/date/completed-summary title,
before agenda warnings, empty-state text, and Now/Next/Later groups. Reuse the preview
capsule wording, typography, colors (green within cap, red over cap), and accessible
count/limit labels. The idle row has no delta chip or capture warning captions. Keep the
existing title and its stale indicator intact.

The budget row is ordinary measured agenda content and never a foldable task or
Pomodoro. Account for it in every fit candidate so the existing farthest-first folding
absorbs its height below the fixed editor eye line. Preserve the current
expansion/overflow behavior and all operator numbers. No extra budget subprocess, Swift
vault parsing, new cache, or new settings toggle is needed.

## Implementation sequence

### 1. Supply the saved ledger budget from bob-cli

In `src/native/capture_pomodoros.rs`, extend `CapturePomodorosResult` with an optional,
omitted-when-None agenda budget. Use a small serializable snapshot type containing the
existing `plan_budget::PlanStatus` and `PlanMeter` values; do not make capture's
mutation-specific before/after type the agenda's dependency.

Only for `include_tasks` and an existing Pomodoros section, load
`config::load_plan_config(&config::config_path())` and call
`plan_budget::compute_for_daily` using the already-read daily contents. Derive the daily
key exactly as `append_plan_budget` does, including the external `BOB_DAY_FILE`
fallback, so empty-target and explicit daily links deduplicate correctly. Project only
status/themes/links from the result; keep counting rules in the shared engine. Budget
computation is read-only even with `plan.strict`.

Handle the missing-file early return and every result constructor. A missing section
omits the budget even though the shared engine internally supplies zero meters for that
case. On config failure use the existing bounded-warning machinery and return the
otherwise normal agenda. Do not add the engine's full lint report to the idle UI.

Document the additive field, omission cases, configured limits, and saved-state meaning
in `docs/capture.md` under `capture-pomodoros`; reference `docs/plan.md` for counting
semantics. Update source/API comments that otherwise claim identical vault bytes alone
determine output: budget bytes also depend on plan configuration.

### 2. Decode and present the budget in bob-mac-capture

Add `planBudget: CapturePlanBudget?` to `CaptureAgendaSnapshot`, defaulting to nil in
its initializer and decoded with `decodeIfPresent`. Retain schema version 1 and the
existing rejection of unsupported schema versions. Update budget-model comments to
acknowledge both capture previews and agenda snapshots.

Add an optional budget row to `CaptureAgendaPresentation`, built through
`CapturePlanBudgetPresentation(budget: ..., destination: nil)`. Only construct it for a
same-day snapshot with a budget; loading and date-mismatch paths omit it. Ensure the
explicit loading factory and all presentation initializers handle it. Reuse the existing
presentation's no-before/no-delta behavior.

There is one nearby edge case required by the new diagnostic: the current empty state
treats _any_ snapshot warning as a missing daily note. Make that check identify the
existing missing-note/missing-section diagnostics, so an empty readable ledger with a
budget-config warning still says no Pomodoros are planned. Add focused coverage; do not
redesign the general warning UI.

Extract the existing SwiftUI meter row into a reusable app view and call it from both
the capture preview and agenda row renderer. Preserve the preview's existing delta,
destination, and warning behavior, with the destination still in its current separate
row. Give the agenda row a typed presentation payload rather than parsing its formatted
text back into numbers. Keep defaulted initializer parameters to minimize unrelated
fixture churn.

Introduce the corresponding agenda row kind (for example `.planBudget`) and wire it
through row rendering, spacing, accessibility, and exhaustive switches. Its key must
reflect display content affecting measurement; its value/equality must include the
budget presentation, so a count/cap/over-state-only change updates the view.

### 3. Integrate measurement and snapshot refresh

Include the budget row exactly once in `CaptureAgendaFitPlanner.render`'s row list and
height total, and in `CaptureAgendaHeightResolver.rowsWithRoles` for offscreen
measurement at the full agenda row width. Use the same shared meter view for measuring
and rendering, with spacing charged once. It stays outside group insets, fold levels,
chip targets, and hidden-task counts. Audit any row-kind switches and section grouping
affected by the new kind.

Use the existing store/process-client pipeline unchanged unless a regression test
identifies a necessary integration fix. The snapshot's synthesized equality must include
the budget. A response where only a count or cap changes publishes and replans;
byte-identical responses remain no-ops. When a successful response omits a previous
budget, remove the capsules rather than retaining a separate cached value.

Existing launch/show/submit/vault/wake/unlock/day-change/recheck triggers refresh both
agenda and budget. Plan config is outside the vault's Markdown watcher; its new limits
appear on the next ordinary refresh, including the next panel show. Keep that freshness
contract and document it; do not add a config watcher or poller.

Update the Mac README's Idle agenda section with capsule meaning, configured limits,
older-CLI fallback, and the saved-budget versus typed-preview distinction. No SASE
memory changes are part of this tale.

## Verification and acceptance

Use synthetic vaults and pinned clock/configuration values. No real vault mutation is
required to verify this feature.

1. **CLI contract and semantics.** Extend `tests/cli/capture/pomodoros_agenda.rs` and
   its `tests/fixtures/capture_pomodoros/` goldens. Cover defaults and custom caps,
   at-cap versus over-cap, real empty budgets, missing file/section, bad config, and
   `--all --tasks` retaining open-only counts. Keep the no-`--tasks` contract test,
   including no budget or new config warning when that option is absent. Verify repeated
   calls with the same daily bytes and config return identical output and do not write
   the vault.
2. **Parity where naive UI counts diverge.** Build one compact fixture with merged
   themes, exempt GTD, duplicate links across entries, daily self-link aliases, indented
   prose/deferred/embedded links, struck/fenced links, closed history, and unnamed
   entries. Compare agenda status/meters with `bob plan` for the same inputs. Also check
   a known capture preview against the saved budget: its `before` counts equal the
   agenda counts, and an ordinary `=` fixture with no count change yields matching final
   meters. Reuse existing plan-budget tests; avoid duplicating the entire engine
   conformance suite.
3. **Swift contract and presentation.** Add budget-bearing agenda fixtures to
   `Tests/Fixtures` using the CLI outputs, keeping at least one legacy fixture without
   the field. Extend `CaptureAgendaModelsTests`, `CaptureAgendaPresentationTests`, and
   `CapturePlanBudgetPresentationTests`: missing/null fields decode to nil,
   counts/caps/over flags are preserved, today-only and empty-state behavior is correct,
   and idle has no capture delta. Existing preview tests must continue to pass after
   sharing the view.
4. **Fit and refresh regressions.** Extend `CaptureAgendaFitPlannerTests` to prove the
   meter's height is included exactly once, survives the fold ladder, and does not
   affect operator numbering or hidden-task counts. Extend process/store coverage
   (`BobProcessClientTests`, `CaptureAgendaStoreTests`, existing fake-bob support) for
   budget-only/cap-only changes, identical output, budget omission, stale refresh
   retention, and day rollover. Reuse existing failure/rollover tests by adding budget
   assertions where possible.
5. **Rendered Mac behavior.** Extend `CaptureAgendaHeightConsistencyTests` with
   budget-bearing current, heavy/folded, and empty fixtures at 724- and 584-point
   content widths, retaining the existing 2-point height tolerance. Extend
   `CaptureAgendaDesignTests` to render within-cap and over-cap capsules in light and
   dark appearances. Inspect those render fixtures for capsule clipping, spacing,
   contrast, and title/stale-marker coexistence. Verify on macOS that opening idle,
   typing `=`, clearing the draft, and refreshing after a ledger change preserves editor
   position and switches correctly between saved and proposed meters.

Run focused tests while implementing, then bob-cli's canonical `just check`. For the Mac
repo, run `just all` on macOS (format lint, build, tests, bundle), using
`BOB_MAC_CAPTURE_RENDER_DIR` for rendered tests and inspecting the images. Its existing
`.github/workflows/ci.yml` supplies the macOS 26 checks. Linux with a Swift toolchain
can run CaptureCore tests, but that does not validate AppKit layout. The planning host
has no `swift` executable: report any unavailable Mac validation honestly and arrange
the macOS gate during implementation. Use `/sase_monitor` for long commands/CI waits
under SASE rather than ending a turn with work running.

Done means both repos implement the additive contract, idle shows the actual configured
daily counts on a current snapshot, prior CLI/app combinations degrade gracefully,
layout and refresh coverage pass, and the existing typed preview retains its behavior.
Deliver the coordinated changes and validation results through the normal SASE workflow;
this plan does not require installing or restarting the live app or changing the user's
daily note.
