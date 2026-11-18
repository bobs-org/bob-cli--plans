---
tier: tale
title: Label the Daily Headline in the capture panel's idle agenda
goal:
  At a glance, the capture panel's idle agenda shows which line is today's Daily
  Headline. A small-caps HEADLINE label leads the line, adds no height, and keeps
  measured and rendered heights equal. It renders cleanly in light and dark mode at
  every panel width, and the docs show the new look.
size: medium
proposed_by: bbugyi200.apollo.ew
proposed_with: claude/opus
create_time: 2026-11-18 09:17:21
status: wip
---

# Label the Daily Headline in the capture panel's idle agenda

## Problem

The idle agenda in the Bob Mac Capture panel prints today's Daily Headline straight
under the `Today · Wed 18 Nov · Nothing running` title, in semibold body text, with
nothing that says what it is:

```text
Today · Wed 18 Nov · Nothing running                 3 done · 50m  ┊15m off-clock┊
Add 30 zettel to sase_blog_zettel album!
▍TODAY 13/13
▦ Lincoln Park w/ Abby at 19:00 · in 9h 55m       1 left · 1h 00m · ↻ 08:27
```

The original design (Daily Headline epic, phase `mac`; `plan:202611/daily_headline.md`)
chose "newspaper hierarchy with no chrome, icon, or card" and relied on type size alone.
Here the headline is only one type step above task text, so it reads like the agenda's
title, a task, or the running Pomodoro. Bryan has to work out what the line is.

## Design (decided)

Lead the headline row with a run-in, small-caps **HEADLINE** label on the same line as
the headline's first line:

```text
Today · Wed 18 Nov · Nothing running                 3 done · 50m  ┊15m off-clock┊
HEADLINE  Add 30 zettel to sase_blog_zettel album!
▍TODAY 13/13
```

When the headline wraps, the second line hangs under the headline text, never under the
label:

```text
HEADLINE  Converge take-home submitted and the zettel album's first draft
          published for Friday review
```

### Spec

1. **Label.** The constant text `HEADLINE` sits at the row's leading edge, aligned to
   the headline's first baseline (`HStack(alignment: .firstTextBaseline)`).
2. **Label type.** Use the TODAY badge's recipe from `TodayBadge.swift`, unboxed:
   `.callout.weight(.semibold).smallCaps().uppercaseSmallCaps()`, `.tracking(0.5)`, and
   `.foregroundStyle(.secondary)`. It is limited to one line and uses `.fixedSize()`, so
   it never truncates, wraps, or compresses. Add no capsule, fill, stroke, color, or
   icon.
3. **Gap.** Put 8 pt between the label and the text, from a new
   `CaptureAgendaLayoutMetrics.dailyHeadlineLabelSpacing`.
4. **Headline text.** Keep it as it is now: bob's display string verbatim, body
   semibold, `.primary`, at most `dailyHeadlineLineLimit` (2) lines, tail truncation. It
   takes the width the label leaves, so a wrapped line hangs under the text.
5. **Unchanged.**
   - Row order and spacing stay the same: title → headline → TODAY badge, with 4 pt
     below the title and 4 pt below the headline.
   - Hovering still shows the full text (`.help(row.text)`).
   - VoiceOver still reads one element, "Today's headline, <text>". The label is not
     read a second time.
   - With no headline the row is still absent. Do not add an empty-state placeholder,
     because an empty headline is valid (glossary: Daily Headline).
   - No bob-cli or protocol change: the label is app UI around bob's text
     (`decisions:mac-capture-is-a-thin-client`).

### Why this design

- **A word, not an icon.** The label repeats the `**Headline:**` slot Bryan types in his
  daily note, so the note and the panel use the same word and there is nothing to learn.
  Each candidate icon already means something in Bob, or is easy to misread: ★ is the
  type highlight, flags read as "flagged", targets read as goals. An icon would also
  compete with the calendar band's `calendar` glyph and the walk marks.
- **Run-in, not a label line above.** It adds no height. The agenda fits below a fixed
  eye line and folds farthest-first, so a separate label line would cost space on every
  panel show.
- **Unboxed and grey.** In this agenda, grey capsules are buttons (`+1`, `Later · MEET`,
  `4 tasks`). Tinted capsules are type pills and the budget badge. Colors mean themes,
  budget tone, warnings, or links. An unboxed small-caps label in secondary grey can't
  be mistaken for any of those. Being smaller than the text and letter-spaced, it also
  differs from one-row lines like `WRITE · Review …`, whose type names are full-size
  capitals after a hue square.
- **Matches the badge below.** It uses the same small caps as the TODAY badge directly
  underneath, so the leading edge reads as one column of labels: `Today` / `HEADLINE` /
  `TODAY 13/13`.
- **Hanging indent.** A wrapped headline keeps a clean left edge and still reads as one
  statement.

## Implementation (bob-mac-capture only)

Open bob-mac-capture with `/sase_repo` and read its `AGENTS.md` first. Its rules apply:
never land Swift you have not compiled; format only the lines you change.

1. **CaptureCore constants.**
   - In `Sources/CaptureCore/CaptureAgendaPresentation.swift`, next to `staleSuffix`,
     add `public static let headlineLabel = "HEADLINE"`. Its doc comment says it is the
     run-in label the view draws before the Daily Headline, and that it echoes the daily
     note's `**Headline:**` slot.
   - In `Sources/CaptureCore/CaptureAgendaLayout.swift`, next to
     `dailyHeadlineLineLimit`, add
     `public static let dailyHeadlineLabelSpacing: Double = 8`.
2. **View.** In `Sources/BobMacCapture/CaptureAgendaView.swift`, rewrite
   `dailyHeadlineView`. Replace its doc comment, which still says "no chrome, icon, or
   card", with one that describes the run-in label and the hanging indent:

   ```swift
   private var dailyHeadlineView: some View {
       HStack(
           alignment: .firstTextBaseline,
           spacing: CGFloat(CaptureAgendaLayoutMetrics.dailyHeadlineLabelSpacing)
       ) {
           // The TODAY badge's small-caps recipe, unboxed and secondary.
           Text(CaptureAgendaPresentation.headlineLabel)
               .font(.callout.weight(.semibold).smallCaps().uppercaseSmallCaps())
               .tracking(0.5)
               .foregroundStyle(.secondary)
               .lineLimit(1)
               .fixedSize()
           Text(row.text)
               .font(.body.weight(.semibold))
               .foregroundStyle(.primary)
               .lineLimit(row.lineLimit)
               .truncationMode(.tail)
       }
       .help(row.text)
       .accessibilityElement(children: .ignore)
       .accessibilityLabel(row.accessibilityLabel ?? row.text)
   }
   ```

   `CaptureAgendaRowMeasurer` measures this same `CaptureAgendaRowView`, so heights
   update on their own. Leave `bottomPad`, the fit planner, the measurer, and the
   presentation row (`makeHeadlineRow`) alone. `row.text` stays bob's text without the
   label.

3. **Tests.**
   - `Tests/CaptureCoreTests/CaptureAgendaPresentationTests.swift`, in
     `testHeadlineRowBuildsOnlyForSameDaySnapshot`:
     - assert `CaptureAgendaPresentation.headlineLabel == "HEADLINE"`;
     - assert the row's text does not contain the label;
     - keep the existing accessibility assertion ("Today's headline, …").
   - `Tests/BobMacCaptureTests/CaptureAgendaHeightConsistencyTests.swift`: add
     `testHeadlineLabelKeepsShortHeadlinesOnOneLineAndClampsLongOnesToTwo`. Build
     snapshots by copying `agenda-headline.json` with a replaced `headline`, using the
     same pattern as `testBusiestTitleWithWalkMarksStaysOnOneLineAtBothWidths`. Measure
     each `presentation.headlineRow` with `CaptureAgendaRowMeasurer` at the rows widths
     for content widths 760 and 620 (the panel's minimum). Use exact or ±1 pt
     comparisons. Assert at both widths:
     - a short headline (`"Ship it"`) measures one line, the same height as `"A"`;
     - a ~400-character headline measures more than the one-line height, and the same
       height as the fixture's headline at 620, which wraps to two lines there;
     - so the label never pushes the text to a third line.

     Also call `assertConsistent(snapshot:fixture:…)` with the long-headline snapshot at
     both widths.

   - `Tests/BobMacCaptureTests/CaptureAgendaDesignTests.swift`: pass `walkFixture(...)`
     for `agenda-headline` as well as `agenda-current`, so the headline render looks
     like the real panel and can be the docs hero (step 4).

4. **Docs.**
   - In `docs/idle-agenda.md`, rewrite the headline paragraph that starts "Directly
     under `Today` sits today's Daily Headline…" to say:
     - a small-caps `HEADLINE` label leads the headline;
     - the label is the same word as the daily note's `**Headline:**` slot;
     - the text is bob's `headline` display text from `capture-pomodoros --tasks`,
       verbatim and semibold;
     - it wraps under itself, never under the label, to at most two lines;
     - hovering shows the full text.

     Keep the existing same-day, stale, `.noOpen`, and older-bob sentences.

   - Show the headline in the hero image. In `Scripts/docs_media.py` `CURATED`, change
     the agenda entry to
     `("agenda-agenda-headline-760", "capture-agenda", "CaptureAgendaDesignTests / Tests/Fixtures/agenda-headline.json")`.
     Update the two hero alt texts at the top of `docs/idle-agenda.md` to "Idle agenda
     showing today's labeled headline, the running FIX Pomodoro, linked tasks, and
     upcoming sessions". First check that the fixture really has a running FIX Pomodoro,
     and adjust the wording if it does not.
   - Run `just docs-test` and `just docs-check`.

## Verification

1. Run `just portable-check`. This host may lack `swift`; if so, run it on athena (see
   the tailnet reference memory). Then do the best-effort Mac scratch compile from
   `AGENTS.md`: rsync without `.build` and `.git`, run `Scripts/bundle.sh` and
   `just typecheck-tests`, then delete the copy. Never use Bryan's checkout. If the Mac
   is offline, say in the final report that the macOS targets are unverified.
2. Run `Scripts/macos-ci-check` through `/sase_monitor`; it can take tens of minutes.
3. Look at the renders. Download `render-fixtures` from the run it reports:
   `gh run download <run-id> --repo bobs-org/bob-mac-capture --name render-fixtures`,
   into a fresh empty temporary directory. If that run has no artifact, use the
   exact-SHA master run after landing.

   Open `agenda-agenda-headline-760-{light,dark}.png` and
   `agenda-agenda-headline-620-{light,dark}.png` with the Read tool, and check:
   - the label sits on the headline's first baseline and is never cut off;
   - a wrapped line starts under the headline text, not under the label;
   - the label is clearly quieter than the headline but still readable in dark mode;
   - the title row, TODAY badge, calendar band, and groups look as they did before.

   If something looks off, adjust only within the spec (gap 6–10 pt, label weight
   `.medium` or `.semibold`) and re-render.

4. Land the code, tests, and docs text as one commit, for example
   `feat(agenda): label the Daily Headline row`. After the push, use `/sase_monitor` to
   watch the exact-SHA macOS run until it is green.
5. On green, run `just docs-media` to import the hero renders from that master run, and
   land them as a second commit, for example
   `docs(agenda): show the labeled headline in idle agenda renders`. Watch that push's
   exact-SHA run to green as well.

## Out of scope

- An icon or color for the headline.
- An empty-headline placeholder or prompt.
- Clicking the headline to open the daily note.
- Any bob-cli, bob-plugins, or vault change.
- Memory edits. The Daily Headline and Capture Panel glossary entries stay accurate.
