---
tier: tale
title: Mac capture TODAY badge mirrors Obsidian's TODAY chip
goal:
  "Every Bob Mac Capture surface that shows today's Task Link budget (idle agenda,
  capture preview, pull-in chooser) renders a `TODAY n/L` badge that matches Obsidian's
  daily TODAY chip: the same label, seven-tone ladder, default-theme colors, and chip
  shape."
size: medium
proposed_by: bbugyi200.athena.17m
proposed_with: claude/opus
create_time: 2026-11-03 14:40:49
status: wip
---

# Plan: Bob Mac Capture's "Links N/C" capsule becomes Obsidian's TODAY badge

## Goal

Bob Mac Capture shows today's Task Link budget as a green/red `Links 9/13` capsule.
Obsidian's daily `bob-plan` block shows the same number as a `TODAY 9/13` chip. That
chip has its own shape and a seven-step color ladder. Make the Mac badge closely mirror
the Obsidian chip: the same label, the same color ladder with the same exact-integer
thresholds and the same default-theme colors, and the same chip shape. Apply it on every
Mac surface that shows the link budget.

Bob already reports the numbers (`plan_budget.links.count` / `.cap`), and they are the
same numbers the Obsidian chip shows. The ladder is a presentation state, not a lint or
an `over` condition (bob-cli `docs/plan.md`, "Today" and the PENDING/NEXT ladder
paragraph). So this work is all presentation: **no bob-cli code changes, no wire
changes, no Bob release needed.**

## Source of truth (already verified while planning)

- **Ladder**: bob-plugins `plugins/bob-ledger-tools/src/075-work-badge-tone.js`
  (`workBadgeTone`), specified in bob-cli `docs/plan.md`. For count `n` and cap `L`,
  using exact integer comparisons and never rounded percentages:
  - `n == 0` is grey
  - `0 < n` and `5n < 2L` is blue
  - `2L ≤ 5n < 3L` is green
  - `3L ≤ 5n < 4L` is yellow
  - `4L ≤ 5n` and `n < L` is orange
  - `n == L` is magenta
  - `n > L` is red

  Invalid input (`n < 0` or `L < 1`) is _unavailable_, never zero. Test vectors:
  bob-plugins `scripts/test-ledger-tools-work-badge-tone.cjs`.

- **Label**: `TODAY n/L`. Missing data is `TODAY –` (en dash), never a fake `0/L`
  (`340-dashboard-views-and-plan-block.js`, `planText`).
- **Chip shape** (`plugins/bob-ledger-tools/styles.css`, `.bob-plan-chip`):
  - 7px corner radius (a rounded rectangle, not a capsule)
  - a 1px border at 28% accent
  - a **2px full-accent leading border**
  - a 14% accent background wash
  - ink text in `--text-normal`, not accent-colored
  - `font-variant-caps: all-small-caps`, weight 650, `letter-spacing: 0.045em`
  - padding `0.28em 0.65em 0.3em 0.6em`

  The tone only tints the border, the bar, and the wash.

- **Colors**: Bryan's vault uses Obsidian's default theme (no `cssTheme`, and no snippet
  overrides `--color-*`). These hex values were read from the installed Obsidian
  `app.css`. Magenta is the plugin's
  `color-mix(color-mix(pink 80%, purple) 70%, text-normal)`, computed per appearance.

  | Tone    | When (n of L)  | Light     | Dark      | Obsidian source      |
  | ------- | -------------- | --------- | --------- | -------------------- |
  | grey    | n = 0          | `#5c5c5c` | `#b3b3b3` | `--text-muted`       |
  | blue    | 0 < 5n < 2L    | `#086ddd` | `#027aff` | `--color-blue`       |
  | green   | 2L ≤ 5n < 3L   | `#08b94e` | `#44cf6e` | `--color-green`      |
  | yellow  | 3L ≤ 5n < 4L   | `#e0ac00` | `#e0de71` | `--color-yellow`     |
  | orange  | 4L ≤ 5n, n < L | `#ec7500` | `#e9973f` | `--color-orange`     |
  | magenta | n = L          | `#923675` | `#e5a9d8` | pink/purple/text mix |
  | red     | n > L          | `#e93147` | `#fb464c` | `--color-red`        |

  Example: the screenshot's `Links 9/13` becomes **`TODAY 9/13` in yellow**, because 45
  ≥ 39 and 45 < 52.

## Surfaces in bob-mac-capture (open it with `/sase_repo`, `bob-mac-capture`)

1. **Idle agenda ("brief")**: `Sources/CaptureCore/CaptureAgendaPresentation.swift`
   `makePlanBudgetRow` sets the row's text and accessibility label.
   `Sources/BobMacCapture/CaptureAgendaView.swift` renders `CapturePlanBudgetMeterRow`.
2. **Capture preview**: `Sources/BobMacCapture/CapturePanelView.swift` (around the
   "plan-budget meter is batch-level" comment) renders the same
   `CapturePlanBudgetMeterRow`.
3. **Pull-in chooser header**: `Sources/BobMacCapture/CapturePullInChoiceView.swift`
   (around line 355) has its own copy of the green/red capsule, built from
   `CapturePullInChoiceState.projectedLinksCapsuleText` / `projectedLinksOverCap` in
   `Sources/BobMacCapture/CapturePullInChoice.swift`. Next to it is a `+N` delta chip.

All three switch to one shared badge view. No surface keeps a private capsule.

## Design

### 1. Pure model (CaptureCore)

Add `Sources/CaptureCore/WorkBadgeTone.swift`:

- `public enum WorkBadgeTone: String, CaseIterable, Sendable` with the cases
  `grey, blue, green, yellow, orange, magenta, red`.
- `public init?(count: Int, cap: Int)`. It returns nil when `count < 0 || cap < 1`.
  Otherwise it applies the ladder above in the JS precedence order: zero, then over,
  then equal, then orange, then yellow, then green, then blue. Use the integer
  comparisons `5 * count >= 4 * cap` and so on. Never use floating point.
- `public var lightHex: String` and `darkHex: String`, from the table above.
- `public var spokenName: String`, which is the raw value, for tests and diagnostics.
- A doc comment saying this mirrors bob-cli `docs/plan.md` and bob-ledger-tools
  `workBadgeTone`, and that a ladder change must land in both.

Add `TodayBadgePresentation` in the same file, or in
`Sources/CaptureCore/TodayBadgePresentation.swift`. It is a public, Equatable, Sendable
struct:

- `init(count: Int, cap: Int)`.
- `text`: `"TODAY \(count)/\(cap)"`, or `"TODAY –"` (U+2013) when the tone is
  unavailable (`count < 0` or `cap < 1`). This is the case for the decoder's `cap: 0`
  fallback when Bob omits `links`.
- `tone: WorkBadgeTone?`, which is nil when unavailable.
- `spokenText`, used for VoiceOver:
  - available: `"TODAY \(count) of \(cap) links"`
  - magenta: append `", at cap"`
  - red: append `", over cap"`
  - unavailable: `"TODAY unavailable"`

  Color is never the only signal.

- `helpText`, used for the hover tooltip:
  - available: `"\(count) of \(cap) Task Links in today's open Pomodoros"`
  - unavailable: `"Today's Task Link budget is unavailable"`

Rework `Sources/CaptureCore/CapturePlanBudgetPresentation.swift`:

- Replace `linksCapsuleText` and `linksOverCap` with
  `public let today: TodayBadgePresentation`, built from `budget.links.count` /
  `budget.links.cap`.
- `accessibilitySummary` joins the destination row, `today.spokenText`, and the
  warnings.
- Update the type's doc comment, which describes "the links capsule".
- Keep everything else as it is: warnings, destination rows, swatches, and the retired
  type-cap filtering.

In `CaptureAgendaPresentation.makePlanBudgetRow`:

- `text` becomes `presentation.today.text`. The row key changes along with it, which is
  fine.
- `accessibilityLabel` becomes `"Plan budget: \(presentation.today.spokenText)"`.

### 2. The shared SwiftUI badge (BobMacCapture)

Add `Sources/BobMacCapture/TodayBadge.swift`. It holds an
`@available(macOS 26.0, *) struct TodayBadge: View` that takes a
`TodayBadgePresentation` and an optional `accessibilityLabel` override. Build it to
these measurements. They are the CSS scaled to the panel's 12 pt chip size, and SF Pro
is the same face Obsidian uses on macOS:

- **Type**:
  `.font(.callout.weight(.semibold).smallCaps().uppercaseSmallCaps().monospacedDigit())`.
  Callout is 12 pt. Together, the two small-caps modifiers equal CSS `all-small-caps`.
  Monospaced digits keep the width steady while live preview counts change. Add
  `.tracking(0.5)`, which is 0.045 em at 12 pt. Use `.foregroundStyle(.primary)` for
  ink, the counterpart of `--text-normal`. Use `.lineLimit(1)` and `.fixedSize()`.
- **Padding**: leading 9 pt (the 2 pt bar plus 0.6 em), trailing 8.5 pt, vertical 3.5
  pt.
- **Shape**: `RoundedRectangle(cornerRadius: 6, style: .continuous)`.
- **Layers**, in this order:
  1. a background fill of `accent.opacity(0.14)`
  2. a leading overlay `Rectangle().fill(accent).frame(width: 2)`
  3. `.clipShape(shape)`, so the bar curves into the corners the way the CSS
     `border-inline-start` does on a rounded box
  4. an overlay of `shape.strokeBorder(accent.opacity(0.28), lineWidth: 1)`
- **Accent**: from the tone's light or dark hex. Pick it with
  `@Environment(\.colorScheme)` and parse it with the existing `Color(pomodoroHomeHex:)`
  from `PomodoroHomeColor.swift`. That is the same pattern `PomodoroHomeColor` uses. An
  unavailable tone (nil) uses the grey tone's colors, so `TODAY –` reads as quiet and
  neutral.
- **Motion**: `.contentTransition(.numericText())` on the text, and
  `.animation(.snappy(duration: 0.15), value:)` keyed on the text and tone. Both are
  disabled under `accessibilityReduceMotion`, the same way the pull-in view's
  `snappy(_:)` helper does it.
- **Accessibility and help**: `.accessibilityElement(children: .ignore)`, with
  `.accessibilityLabel(override ?? presentation.spokenText)` and
  `.help(presentation.helpText)`.

Also add a small neutral sibling style for the pull-in `+N` delta chip, so the pair
reads as one family. It uses:

- the same font, padding, and shape
- `.quaternary` fill, `.secondary` ink, and no leading bar

This can be a `TodayBadge` variant or a `BadgeChipStyle` helper. Keep it in the same
file.

### 3. Wire the surfaces

- `CapturePlanBudgetMeterRow`: replace the private `capsule(_:overCap:)` with
  `TodayBadge(presentation: budget.today)`. Keep the orange warning captions, the
  `.combine` accessibility, and the `"Plan budget: …"` label. This one change covers
  both the idle agenda and the capture preview.
- `CapturePullInChoiceState`:
  - Replace `projectedLinksCapsuleText` with
    `projectedTodayBadge: TodayBadgePresentation?`, built from `projectedLinkCount` and
    `planBudget.links.cap`. It is nil without a budget.
  - Keep `projectedLinksOverCap`, `projectedLinksText`, and `projectedLinksDeltaText` as
    they are, unless one becomes unused. Delete an unused one.
- `CapturePullInChoiceView`: render
  `TodayBadge(presentation: accessibilityLabel: projectedLinksAccessibilityLabel)` and
  the restyled `+N` chip. Keep the existing projected wording
  (`"Projected links N of C"`, with a `"Warning: "` prefix when over). Append
  `", at cap"` when the projected tone is magenta.
- Leave every other capsule (`18m over`, template chips, and so on) untouched.

### 4. Tests (bob-mac-capture)

- New `Tests/CaptureCoreTests/WorkBadgeToneTests.swift`:
  - Port **every** vector from bob-plugins
    `scripts/test-ledger-tools-work-badge-tone.cjs` (the L = 100, 15, and 10 tables).
  - Add 9/13 → yellow and 13/13 → magenta.
  - Add the invalid inputs `(-1, 10)`, `(3, 0)`, and `(3, -1)`, which all give nil.
  - Pin all 14 hex values, so a palette edit is a deliberate test change.
- `TodayBadgePresentation` tests:
  - `TODAY 9/13`, `TODAY 0/7` (grey), and `TODAY –` for `cap: 0`
  - each `spokenText` branch
  - `helpText`
- Update the existing assertions that expect `Links N/C`, `linksCapsuleText`,
  `linksOverCap`, or `projectedLinksCapsuleText`:
  - `Tests/CaptureCoreTests/CapturePlanBudgetPresentationTests.swift`
  - `Tests/CaptureCoreTests/CaptureAgendaPresentationTests.swift`
  - `Tests/BobMacCaptureTests/CapturePullInChoiceTests.swift`

  The new values are `TODAY 8/10`, `"Plan budget: TODAY 8 of 10 links"`,
  `"TODAY 11 of 10 links, over cap, today's plan now has 11/10 links"`, and so on. Leave
  `CaptureProjectNotePresentation`'s unrelated `"Links 2 tasks into …"` header alone.

- New design render test `Tests/BobMacCaptureTests/TodayBadgeDesignTests.swift`. Follow
  `TaskBlockDesignTests` / `CaptureAgendaDesignTests`: skip unless
  `BOB_MAC_CAPTURE_RENDER_DIR` is set, then render at scale 2.
  - Write a contact sheet with one row per ladder state at cap 10:
    `0, 3, 4, 6, 8, 10, 11`, plus `TODAY –`.
  - Add a pull-in pair, `TODAY 6/10` with `+2`.
  - Render both appearances, as `today-badge-ladder-light.png` and
    `today-badge-ladder-dark.png`.
  - CI already uploads that directory as the `render-fixtures` artifact.

### 5. Docs

- **bob-mac-capture**:
  - `docs/capture-previews.md` (around line 167): replace "a `Links 8/10` capsule —
    green within the cap, red over —" with a `TODAY 8/10` badge. Say that it mirrors
    Obsidian's daily TODAY chip: grey at 0, then blue, green, yellow, and orange by
    fifths of the cap, magenta exactly at the cap, and red over. Link the bob-cli plan
    contract the way the page already links other bob docs.
  - `docs/idle-agenda.md` (around lines 30–31): the stale "saved daily meters
    (`Themes N/C` and `Links N/C`)" becomes the single saved `TODAY N/C` badge.
  - `docs/protocol.md` (lines ~203 and ~343): "links capsule" and "links meter" become
    "TODAY badge".
  - Run `just docs-check`.
- **bob-cli** (this repo), `docs/plan.md`: at the end of the paragraph that starts "The
  daily and dashboard TODAY badge shares the Work badge colors", add one sentence. It
  says Bob Mac Capture's TODAY badge (idle agenda, capture preview, and pull-in chooser)
  applies the same label, ladder, and `–` rule to Bob's reported `plan_budget.links`, so
  a ladder change must land in bob-ledger-tools `workBadgeTone` and the Mac
  `WorkBadgeTone` together. Run `just docs-check`.
- Do **not** edit bob-cli `docs/reference/mac-capture/*`. It is a gitignored mount of
  the Mac repo's docs.

## Non-goals

- No change to Bob's `plan_budget` JSON, and no `tone` field on the wire.
  - The tone is a presentation state over two integers Bob already reports.
  - The Obsidian plugin computes it client-side too.
  - Adding a wire field would gate a pixel change on the bob-cli release train.
  - This stays inside the thin-client rule, because the app owns presentation and Bob
    owns the facts.
- No PENDING/NEXT/READY badges on the Mac, and no per-type tooltip breakdown, because
  Bob's capture budget carries no per-entry counts.
- No change to the orange link-cap warning captions or to the bob-plugins repo.

## Verification

1. Run `just portable-check` in bob-mac-capture on athena. It compiles CaptureCore and
   runs its tests.
2. Run the Mac scratch compile from bob-mac-capture's `AGENTS.md` (best effort):
   - rsync a scratch copy without `.build` and `.git`
   - run `Scripts/bundle.sh` and then `just typecheck-tests`
   - delete the copy

   Never touch Bryan's checkout, and never run `just install`. If the Mac is offline,
   say in the final report that the macOS targets are unverified.

3. Run `Scripts/macos-ci-check` from the bob-mac-capture checkout. It can run up to 90
   minutes, so hand it to `/sase_monitor` or run it in the foreground with a large
   explicit timeout. Then fetch that run's `render-fixtures` artifact
   (`gh run download <run-id> -n render-fixtures`).
   - Look at `today-badge-ladder-{light,dark}.png`, and at the agenda and pull-in
     renders, with an image reader.
   - Check that the seven tones are distinct and that the ink text stays legible on
     every wash, including dark yellow and light magenta.
   - Check that the leading bar curves cleanly into the corners and that the badge sits
     on the row baseline without clipping.
   - Tune spacing or radius if a render looks off, and say what you changed.
4. Run `just docs-check` in both repos. Use `just format-lint`, or the repo's
   swift-format pin, on the changed lines only.
