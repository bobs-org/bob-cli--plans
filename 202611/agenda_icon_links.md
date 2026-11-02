---
tier: tale
title: Show only the icon for icon-labeled links in the idle agenda
goal:
  Bob Mac Capture's idle agenda renders Markdown links labeled with one emoji (Calendar
  📅/📹/📍, Keep 💡) as just the icon.
size: small
decisions:
  text_links:
    ask:
      Should text-labeled Markdown links such as `[docs](url)` also show only their
      label in the idle agenda?
    choices:
      icons_only:
        Only single-emoji links collapse; text-labeled links keep their full Markdown
        and URL
      all_links:
        Every well-formed Markdown link shows just its label, link-tinted, like the task
        pickers
    default: icons_only
    why:
      Matches the request; the lightbulb plan deliberately kept prose links verbatim in
      the agenda
    answer: icons_only
proposed_by: bbugyi200.athena.16x
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-02 06:57:32
status: wip
---

# Show only the icon for icon-labeled links in the idle agenda

## Goal

With an empty draft, Bob Mac Capture shows today's Pomodoro agenda. A MEET Pomodoro's
Event Task row currently displays bob's whole Calendar badge link:

```text
Before: * Lincoln Park w/ Abby [📅](https://www.google.com/calendar/event?eid=… "Open in Go…
After:  * Lincoln Park w/ Abby 📅
```

Any Markdown link whose label is one emoji (an "icon link") should render as just that
icon, link-tinted and non-clickable, as Keep's `[💡](url)` already does. That covers
bob's Calendar badges `📅` (open event), `📹` (join call), and `📍` (location). It also
covers Keep's `💡` and any icon link Bryan types himself.

All changes land in **bob-mac-capture**. bob-cli, its JSON contract, the vault, and the
capture grammar stay unchanged.

## Findings

- bob-cli `src/native/gcal/render.rs` writes the badges onto Event Task lines:
  `calendar_badge` emits `[📅](<url> "Open in Google Calendar")`, `badges` emits
  `[📹](<url> "Join call")` and `[📍](<maps url> "<escaped place>")`. URLs are
  percent-encoded for spaces and parentheses; titles escape `"` as `&quot;`. Titles may
  contain spaces and parentheses.
- `bob capture-pomodoros --tasks` passes each task's raw description through in the
  agenda item `text`, so badges reach the app. That is correct: the badge belongs in the
  vault and payload, and display belongs to the app. This follows
  `decisions:mac-capture-is-a-thin-client` and
  `decisions:idle-capture-shows-ledger-agenda`, and matches the precedent of the
  Keep-lightbulb fix (`plan:202610/idle_agenda_lightbulb.md`, bob-mac-capture commit
  `4999fb7`).
- The Calendar band rows are already clean: bob's `event_task::clean_event_title` strips
  trailing Markdown links from event titles. Only task and detail rows are affected.
- `Sources/CaptureCore/CaptureAgendaInlineText.swift` is the agenda-only inline parser.
  `parseKeepSourceLinkOrField` and `keepSourceLinkEnd` recognize only the literal
  `[💡](`. Every other `[label](url)` falls through to `parseField` and stays verbatim.
  That narrowing was deliberate in the lightbulb plan ("do not generalize the change to
  other link labels"); this request reverses it for icon labels.
- `projectingKeepSourceLinks` applies the same 💡-only projection to accessibility
  labels and one-row summaries. It is called through
  `CaptureAgendaPresentation.spokenKeepText` (about 7 call sites), so a long Calendar
  URL can still leak into a truncated one-row summary or a VoiceOver label today.
- `Sources/BobMacCapture/CaptureAgendaView.swift::CaptureAgendaRowInlineText` renders
  the parsed segments for every task, detail, duplicate, struck, warning, and calendar
  row. `.link` segments get the link tint. No view change is needed beyond comments.
- MEET event rows pass through `CaptureAgendaPresentation.stripLeadingBoldSpan`, so the
  displayed source is e.g. `Standup [📅](…) [📹](…) [📍](…)`.
- `TaskDisplayText` (pickers, start/close cards) already shows every Markdown link by
  its label, so those surfaces show `📅` today. Leave it unchanged.
- `Tests/Fixtures/agenda-events.json` already contains real badge-bearing Event Task
  texts (Standup with `📅 📹 📍`, Lunch with Kelly with `📅` plus `#untracked`).
  `CaptureAgendaDesignTests` renders it as `agenda-events`, but
  `CaptureAgendaHeightConsistencyTests` does not include it yet. No Swift test asserts
  the raw badge text.

## Implementation

1. Open the app with
   `sase repo open bob-mac-capture -r "Show only the icon for icon-labeled links in the idle agenda"`.
   Use only the printed path. Read its `AGENTS.md` and follow it (format only changed
   lines; 4-space indent; 100 columns).

2. In `Sources/CaptureCore/CaptureAgendaInlineText.swift`, generalize the Keep-only
   recognizer into an icon-link recognizer:
   - Rename `keepSourceLinkEnd` to
     `iconLinkEnd(chars:from:) -> (icon: String, end: Int)?`. It matches `[`, exactly
     one icon `Character`, `](`, then the existing destination and optional
     double-quoted title rules unchanged. The destination must be nonempty and stops at
     `)`, `(`, `"`, `<`, `>`, or whitespace. The title and closing `)` are consumed
     together. Unclosed or empty forms return nil.
   - Add `static func isIconLabel(_ char: Character) -> Bool`. It is true when the
     grapheme is an emoji: its first scalar has `properties.isEmojiPresentation`, or its
     first scalar has `properties.isEmoji` and either the grapheme has more than one
     scalar (e.g. `🗓️` with U+FE0F, keycaps) or the scalar value is above U+238C. ASCII
     `#`, `*`, digits, `©`, `®`, `™`, and letters must stay non-icons. Multi-character
     labels such as `[📅 Calendar]` or `[📅📹]` are not icon links.
   - Rename `parseKeepSourceLinkOrField` to `parseIconLinkOrField`. It appends the
     matched icon as one `.link` segment, otherwise it delegates to `parseField` as now.
   - Rename `projectingKeepSourceLinks` to `projectingIconLinks`. It replaces each
     complete icon link with its icon and keeps code spans and wikilinks opaque, as now.
   - Update the type and method doc comments (they currently say "Keep `[💡](url)`
     source links") to describe icon links, naming Keep's `💡` and Calendar's
     `📅`/`📹`/`📍` as examples. Keep the "tiles `text` exactly" guarantee and
     Character-offset ranges.

   > [!decision] text_links = all_links Instead of the icon-only predicate, parse every
   > well-formed `[label](destination)` or `[label](destination "title")` with a
   > nonempty label and no nested `[`/`]`. Append the label as one `.link` segment. Name
   > the helpers `markdownLinkEnd` / `parseMarkdownLinkOrField` /
   > `projectingMarkdownLinks`, and drop `isIconLabel`. Keep the `[[` wikilink,
   > code-span, and `[key:: value]` field precedence unchanged: a bracket span not
   > followed by `(` must still reach `parseField`. Projection replaces each link with
   > its label.

3. In `Sources/CaptureCore/CaptureAgendaPresentation.swift`, rename `spokenKeepText` to
   `spokenIconText` (or `spokenLinkText` under `all_links`). Point it at the renamed
   projection, update every call site, and update its doc comment. Projection must still
   run on each title before `makeOneRowRow` joins and truncates to `maxOneRowLength`, so
   a long Calendar URL can never be cut mid-link. Leave the snapshot payload and
   ordinary row `text` raw so the inline renderer still sees complete link spans.

4. In `Sources/BobMacCapture/CaptureAgendaView.swift`, update the
   `CaptureAgendaRowInlineText` doc comment ("Keep source `💡` glyphs") to say icon
   links. No rendering logic changes.

5. In `docs/idle-agenda.md`, replace "Keep source links in those task titles and details
   display as a lightbulb." with a sentence that icon links display as just their icon.
   Give Keep's `💡` and Calendar's `📅`, `📹`, and `📍` badges as examples. Under
   `all_links`, say every Markdown link shows only its label instead. Check `README.md`
   for any similar sentence and keep it consistent.

## Tests

Use a synthetic event id such as `eid=example` in every test string. Never copy the real
`eid` from a vault or screenshot, because it encodes an account address.

- `Tests/CaptureCoreTests/CaptureAgendaInlineTextTests.swift`. Rename the "Keep source
  links" section to "Icon links". Keep every existing `💡` case passing, and add:
  - `Lincoln Park w/ Abby [📅](https://www.google.com/calendar/event?eid=example "Open in Google Calendar")`
    → text `Lincoln Park w/ Abby 📅`, kinds `[.plain, .link]`.
  - The Standup text from `agenda-events.json` with its `**0930-1000**` prefix →
    `0930-1000 Standup 📅 📹 📍`, with one `.link` per icon.
  - `Lunch with Kelly [📅](…) #untracked` → `Lunch with Kelly 📅 #untracked`, kinds
    `[.plain, .link, .plain, .tag]`.
  - A `📍` link whose title has spaces, parentheses, and `&quot;`, e.g.
    `[📍](https://www.google.com/maps/search/?api=1&query=Lincoln%20Park "Lincoln Park (north) &quot;gate&quot;")`
    → `📍`.
  - `[🗓️](https://example.test/cal)` (with U+FE0F) → `🗓️`, and `[🗓](…)` without U+FE0F
    → `🗓`.
  - Assert `assertTilesExactly` for each case.
- Extend `testMalformedAndForeignLinksStayLiteral` with `[📅](`, `[📅]()`, and an
  unclosed-title `📅` link. Under `icons_only`, also add `[x](https://example.com)`,
  `[1](https://example.com)`, `[#](https://example.com)`, `[©](https://example.com)`,
  `[📅 Calendar](https://example.com)`, and `[📅📹](https://example.com)`. Every one
  stays verbatim with no `.link` segment. Keep the existing `[docs](…)` and `[note](…)`
  cases. Under `all_links`, move the text-labeled cases into a "renders as label" test
  and keep only the malformed ones literal.
- A code-span test: `` `[📅](https://example.test/e "Open in Google Calendar")` `` stays
  verbatim as one `.code` segment, and the projection leaves it alone.
- A projection test:
  `projectingIconLinks("Standup [📅](…) [📹](…) and [[tasks|tasks file]] [docs](https://example.com)")`
  returns `"Standup 📅 📹 and [[tasks|tasks file]] [docs](https://example.com)"`. Under
  `all_links`, `[docs](…)` becomes `docs`.
- Update the comment in `testKeepSourceLinkRendersAsLabelInTaskDisplayText` that says
  the agenda parser "keeps its own Keep-only contract".
- `Tests/CaptureCoreTests/CaptureAgendaPresentationTests.swift`. Rename the Keep helpers
  and section only where the names now mislead. Add:
  - A fixture test on `agenda-events.json`. The MEET Event Task rows' display text
    (through `CaptureAgendaInlineText`) and accessibility labels read `Standup 📅 📹 📍`
    and `Lunch with Kelly 📅 …`. They contain no `](`, `https://`,
    `Open in Google Calendar`, or `Join call`, and they keep their task numbers and
    statuses.
  - A one-row regression mirroring `testOneRowProjectsKeepLinksBeforeTruncation`. A
    Later entry's task carries a `📅` link with a destination long enough that the raw
    joined summary exceeds `maxOneRowLength`. The one-row text is the projected `… 📅`
    form, within the limit, and contains no `https://`.
- `Tests/BobMacCaptureTests/CaptureAgendaHeightConsistencyTests.swift`. Add
  `assertConsistent(fixture: "agenda-events.json", contentWidth: width, budget: 2_000, expanded: [])`
  next to the `agenda-lightbulb.json` case. The shorter icon rows must still measure
  within the existing tolerance. `CaptureAgendaDesignTests` already renders
  `agenda-events`, so CI's uploaded PNGs show the change.

## Verification

- On a host with `swift` (athena has it), run `just portable-check` in the opened
  bob-mac-capture checkout. It format-lints, builds with tests, and runs the CaptureCore
  tests; all must pass. Run `just docs-check` for the docs edit.
- `portable-check` cannot type-check `Sources/BobMacCapture*` or `Tests/BobMacCapture*`.
  Follow `AGENTS.md` to compile a scratch copy on the Mac (use the `tailnet.md`
  reference memory for SSH access). Rsync the checkout without `.build`/`.git` to a
  scratch directory, run `./Scripts/bundle.sh` and `just typecheck-tests` there, then
  delete the copy. Never use Bryan's checkout or `just install`. If the Mac is offline,
  say in the final report that the macOS targets are unverified. Never land Swift that
  has not compiled.
- Hand any long-running command to `/sase_monitor` instead of waiting on it.

## Acceptance

In the idle agenda, every complete icon-labeled Markdown link in task titles, details,
duplicate and struck rows, and folded one-row summaries renders as exactly its icon,
link-tinted and not clickable. That includes bob's Calendar `📅`/`📹`/`📍` badges and
Keep's `💡`. Accessibility labels and one-row summaries never expose the URL, tooltip,
or brackets. Malformed links, code spans, wikilinks, fields, tags, and (under
`icons_only`) text-labeled links keep their current display. bob's payload and the vault
still carry the real URLs. Portable checks pass, and the macOS targets are type-checked
or explicitly reported as unverified.
