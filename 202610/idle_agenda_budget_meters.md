---
tier: tale
title: Show saved theme and Task Link budgets on the idle agenda
goal:
  Paint bob's saved daily theme and Task Link counts and caps on Bob Mac Capture's
  empty-draft agenda, using the plan_budget object capture-pomodoros --tasks already
  returns.
size: medium
proposed_by: bbugyi200.apollo.6i
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.6i](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.6i.md)
- **COMMITS:**
  - [daf70c2](https://github.com/bobs-org/bob-mac-capture/commit/daf70c26edfa0569101abce1fe7449bb647eec30)
    — feat(agenda): show saved theme and Task Link budgets on idle agenda

# Show saved theme and Task Link budgets on the idle agenda

An empty capture draft shows today's Pomodoro agenda and does not show the daily plan
meters. In the 2026-10-10 capture screenshot the title row reads `Today · Sat 10 Oct`
with `6 done · 3h 20m` on the right. That accessory is `completed_summary`. It is not
the theme cap or the Task Link cap. The `Themes N/C` and `Links N/C` capsules exist only
on a typed capture preview, and only when that preview changes today's Pomodoros
section.

bob-cli already returns the saved budget.
`stitch:bob-cli@60297c2621478beca362bc0e8b5abe32d92a9147`
(`feat(capture): add idle agenda budget output`) adds optional top-level `plan_budget`
to `bob capture-pomodoros --tasks`:

```json
"plan_budget": {
  "status": "ok",
  "themes": {"count": 3, "cap": 3, "over": false},
  "links": {"count": 8, "cap": 10, "over": false}
}
```

`docs/capture.md` and `tests/cli/capture/pomodoros_agenda.rs` cover omission, real
zeros, custom caps, invalid config, and parity with `bob plan`. Leave bob-cli, its
goldens, and `docs/capture.md` unchanged.

`plan:202610/idle_agenda_plan_budget.md` specified both halves and is still `wip`. Its
bob-cli section is the commit above. Its Mac section did not land: bob-mac-capture
`master` at `787569d`
(`fix(capture): keep editor and controls visible when idle agenda returns`) still drops
`plan_budget` on decode. Implement against that master. A prior attempt described Mac
edits in an external checkout; those edits are not on `master` and are not a base to
recover.

All implementation work is in bob-mac-capture. Open it with `/sase_repo` and use the
printed path. If the configured primary checkout is missing,
`sase repo open gh:bobs-org/bob-mac-capture` is the supported fallback. Before editing,
read `decisions:idle-capture-shows-ledger-agenda`,
`decisions:mac-capture-is-a-thin-client`, `glossary:Pomodoro`, and `glossary:Task Link`.

## Intended behavior

bob owns the counts. The app decodes `plan_budget` and paints it. Do not sum agenda
`items`, do not read `task_link_count` as the link meter, and do not reimplement
`docs/plan.md`. `task_link_count` stays the current entry's `=x` lineup size for the
close-comma assist.

Place one meter row directly under the Today title and above the multi-open warning, the
empty-state line, and the Now / Next / Later groups. Reuse the preview capsules
`Themes 3/3` and `Links 8/10`: caption, semibold, green while `over` is false (including
exactly at the cap), red when `over` is true. The idle row has no `+N` delta chip and no
capture warning captions. The title row and its `6 done · 3h 20m` summary and
`Couldn't refresh` marker stay as they are.

| Snapshot                                           | Idle display                                                                                                        |
| -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| Same-day agenda with `plan_budget`                 | Both capsules, including real `0/<cap>` on an empty or completed-only ledger, above the existing empty-agenda line. |
| Missing `plan_budget`, JSON null, or an older bob  | Existing agenda, no capsules.                                                                                       |
| Date mismatch or the loading factory               | Existing `Loading today…` row, no capsules.                                                                         |
| Failed refresh with a last good snapshot for today | That snapshot's agenda and budget together, under the existing stale title marker.                                  |
| Day rollover                                       | Existing loading state. Yesterday's budget is never shown.                                                          |

The row is ordinary measured agenda content. It is not a foldable task, Pomodoro, or
chip target, and it does not increment the hidden-task count. Its height is included
once in every fit candidate so the existing farthest-first fold absorbs it below the
fixed eye line. Operator numbers, expansions, and overflow scrolling stay as they are.

`CaptureAgendaPresentation` currently treats any snapshot warning as a missing daily
note. Identify the existing missing-note and missing-section diagnostics, so an empty
readable ledger whose only warning is `plan budget unavailable: …` still says no
Pomodoros are planned and shows no capsules. Do not add a general warning renderer.

Plan config is outside the vault Markdown watcher. New limits appear on the next
ordinary refresh, including the next panel show. Document that. Do not add a config
watcher.

## Implementation

### 1. Decode the saved budget

Add `planBudget: CapturePlanBudget?` to `CaptureAgendaSnapshot` in
`Sources/CaptureCore/CaptureAgendaModels.swift`. Default it to nil and decode it with
`decodeIfPresent`. Keep schema version 1. `CapturePlanBudget` already tolerates the
agenda shape: `before` is optional, and missing `added_themes` and `warnings` become
empty lists. Update the type's comment so it covers both capture previews and agenda
snapshots.

An absent field stays nil. Do not substitute `0/0` or the default caps of 3 and 10.

### 2. Present one budget row

On `CaptureAgendaPresentation`, build an optional budget row only for a same-day
snapshot that has `plan_budget`. Use
`CapturePlanBudgetPresentation(budget:destination:)` with a nil destination. That
initializer already yields `Themes N/C` and `Links N/C`, Bob's `over` flags, and no
delta when `before` is absent. The explicit loading factory and every presentation
initializer take the row, defaulting to absent.

Add `CaptureAgendaRowKind.planBudget`. Give `CaptureAgendaRow` an optional typed budget
payload so the view colors capsules from `over` rather than parsing the label. Default
the payload to nil so existing row constructors stay valid. Include the capsule labels
in the row key's text, and include the payload in row equality, so a count, cap, or
over-state change republishes and can remeasure.

`CaptureAgendaFitPlanner.render` inserts the row once, immediately after
`presentation.titleRow`, and adds its measured height once. `finish` treats
`.planBudget` like `.title`: it does not count as shown content. Audit every
`CaptureAgendaRowKind` switch, including `CaptureAgendaView`,
`CaptureAgendaRowMeasurer`, and `CaptureAgendaSections`. The new kind is a flat row. It
must not start a group, take group insets, or become a chip target.

### 3. Draw the shared capsules

Extract `planBudgetMeterRow` from `Sources/BobMacCapture/CapturePanelView.swift` into a
reusable view that both the capture preview and the agenda row call. The preview keeps
its delta chip, warning captions, and separate destination row. The agenda call passes a
presentation whose delta and warnings are empty.

Match the preview's capsule padding, green/red fills, and VoiceOver label
(`Plan budget: Themes N/C, Links N/C`). Measure and render with that same view, and
charge the row spacing once, using the title row's bottom spacing.

### 4. Document the idle row

In the Mac README's Idle agenda section, state that the row under the title shows the
saved daily meters (`Themes N/C`, `Links N/C`), that the caps come from bob's plan
config, that an older bob simply omits the row, and that a typed capture preview still
shows the proposed budget rather than this saved one. Note that a plan-config edit shows
up on the next ordinary agenda refresh.

No SASE memory change is part of this tale.

## Verification

Use synthetic snapshots and the CLI fixtures already produced for `plan_budget`. Do not
mutate the real vault.

1. **Decode and presentation.** Extend `CaptureAgendaModelsTests` and
   `CaptureAgendaPresentationTests`. A legacy fixture without `plan_budget` decodes to
   nil and renders the agenda it renders today. A budget-bearing fixture preserves
   counts, caps, and `over`. The loading factory and a date mismatch omit the row. An
   empty ledger with a budget shows `Themes 0/<cap>` and `Links 0/<cap>` above the
   existing empty line. An empty ledger whose only warning is
   `plan budget unavailable: …` keeps the planned-Pomodoro empty state and shows no
   capsules. Idle presentation has no delta chip. Extend
   `CapturePlanBudgetPresentationTests` only if sharing the view requires it. Existing
   preview tests must still pass.
2. **Fit.** Extend `CaptureAgendaFitPlannerTests`. The meter's height is included
   exactly once, the row survives every fold step, operator numbers are unchanged, and
   hidden-task counts ignore the meter.
3. **Rendered layout.** Extend `CaptureAgendaHeightConsistencyTests` with budget-bearing
   current, folded, and empty fixtures at the existing 724- and 584-point content
   widths, keeping the 2-point tolerance. Extend `CaptureAgendaDesignTests` to render
   within-cap and over-cap capsules in light and dark. Inspect those images for
   clipping, spacing, contrast, and coexistence with the title and the stale marker.
4. **Manual Mac check.** On macOS, open an empty draft and confirm the saved capsules
   sit under the Today title. Type `=` and confirm the preview meter is the proposed
   budget. Clear the draft and confirm the saved meter returns without moving the caret.

Run focused tests while implementing. This planning host has no Swift toolchain. On
macOS run `just all` (format lint, build, tests, bundle) with
`BOB_MAC_CAPTURE_RENDER_DIR` set, and inspect the render images. Linux can run
CaptureCore tests only when `swift` is installed; that does not validate AppKit layout.
Use `/sase_monitor` for long test and CI waits.

Done means current master of bob-mac-capture paints bob's saved meters on a current
snapshot, omits them when the field is absent, keeps the typed preview's proposed meter,
and passes the fit and render checks above. Do not reinstall the live app and do not
edit the user's daily note.
