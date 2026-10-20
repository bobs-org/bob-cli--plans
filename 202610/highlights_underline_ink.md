---
tier: tale
title: Draw Highlights underlines as gold ink underlines
goal:
  Words Bryan underlines in Highlights read unmistakably as underlines in the Refs
  panel, Obsidian, and bob ref show.
size: medium
decisions:
  surfaces:
    ask:
      Should Obsidian ref notes and `bob ref show` get the same underline as the Mac
      Refs panel?
    choices:
      everywhere:
        Mac panel, Obsidian ref notes, and `bob ref show` all draw your underlines as
        underlines
      mac_and_obsidian:
        The Mac panel and Obsidian change; `bob ref show` keeps plain ANSI bold
      mac_only:
        Only the Mac Refs panel changes; Obsidian and the terminal keep plain bold
    default: everywhere
    why:
      One visual language — a line you drew in Highlights reads as a line wherever Bob
      shows it
    answer: everywhere
proposed_by: bbugyi200.athena.12q
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-20 17:44:20
status: wip
---

# Draw Highlights underlines as gold ink underlines

## Why

When Bryan underlines words inside a highlight in the Highlights app, the export wraps
them in `**…**`. `bob ref sync` then merges that underline into its host highlight's
quote as a `**bold run**` (`docs/highlights-ref-sync.md`, "Underline emphasis"). Every
surface that shows the quote turns `**` into plain bold:

- the Bob Mac Capture Refs inspector and the Copy card's highlight picker
  (`RefsQuoteText` → `.stronglyEmphasized`),
- Obsidian, through the vault's `ref-highlights` CSS snippet (bold plus full ink),
- `bob ref show` in a color terminal (ANSI bold).

In Bryan's screenshot of the Refs inspector (wide layout, light mode), the underlined
runs ("all of our clients were agents and jobs that are online by definition.", "always
online", "prior art, notably Google's Piper and Meta's Sapling,2") look only a little
heavier than the text around them. Bold also says "typographic emphasis", not "the mark
I made", so the reader has to remember that bold secretly means underline.

## The design

**Principle: what you drew is what you see.** An underline stays an underline: the words
keep a stronger weight and full ink, and a thick, rounded **gold** stroke runs under
them. Gold is the deeper sibling of the inspector's yellow quote rule. It stays in the
"yellow = your highlights" family but holds up as a thin stroke on the light panel,
where plain system yellow fades out. Bob keeps storing the underline as `**…**`, so only
the rendering changes.

### Mac Refs panel (inspector and Copy card picker)

| Property  | Spec                                                                                                                                                                                                                                                                            |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Words     | Semibold, `.primary` ink, including in the Copy card's secondary italic rows. Gaps between the words are semibold too, so word spacing stays even.                                                                                                                              |
| Stroke    | One continuous bar per laid-out line, starting at the left edge of the first underlined glyph and ending at the right edge of the last one. A wrapped phrase gets one bar per line. A bar never sits under the spaces at either end of a phrase and never crosses a line break. |
| Thickness | Starting value 0.15 em, at least 1.5 pt (about 2 pt at the 13 pt wide body font, about 1.8 pt at the 12 pt compact callout font). With Increase Contrast it is 0.20 em, at least 2 pt.                                                                                          |
| Position  | The bar's top edge sits about 0.08 em below the baseline, so the bar clears the baseline and passes through the descender zone. It stays inside the line's descent, so line spacing does not change.                                                                            |
| Layering  | Drawn **beneath** the glyphs. Descenders (g, j, p, q, y) paint over the bar the way ink sits on a pen stroke, with no skip-ink gaps.                                                                                                                                            |
| Ends      | Fully rounded: corner radius = thickness / 2.                                                                                                                                                                                                                                   |
| Crispness | Snap the bar's y position and height to device pixels (`displayScale`) so it never blurs at 1× or 2×.                                                                                                                                                                           |
| Color     | New token `RefsVisualTokens.inspectorUnderlineInk = Color.yellow.mix(with: .orange, by: 0.35)` ("gold"). It is built from adaptive system colors, so dark mode and Increase Contrast carry through, matching the tokens' "adaptive system colors only" rule.                    |
| Glyphs    | Every glyph draws exactly as SwiftUI would draw it. The renderer only adds fills underneath.                                                                                                                                                                                    |

Treat the thickness and offset as starting values. Tune them with the proof sheet (see
Verification), and record the final numbers as documented named constants.

Behavior that stays as it is: the tooltip shows the plain text, the clipboard and every
Copy payload keep the raw `**` Markdown, the existing `RefsQuoteExcerpt` lead-in still
keeps the first underlined run inside the line clamp, and unpaired `**` stays literal. A
standalone underline (a whole quote wrapped in `**`) gets a bar under every line, which
is correct because the whole quote is the underline. If tail truncation lands inside an
underlined phrase, the bar may continue under the ellipsis. That is acceptable: it reads
as "the underline continues".

**VoiceOver.** The inspector entry label announces the underlined words after the quote
and before `Your note`. If some words are underlined, it appends
`. Underlined: <phrase>; <phrase>`. If the whole quote is underlined, it appends
`, underlined`. If nothing is underlined, the label does not change.

### Obsidian ref notes

> [!decision] surfaces = mac_only Skip this section and leave the vault snippet and its
> docs block unchanged.

Replace the whole body of the vault's `~/bob/.obsidian/snippets/ref-highlights.css` with
the following, and use the same text in the docs block. CSS paints underlines beneath
the text, so this matches the Mac's layering:

```css
/* bob ref: Highlights exports an underline as **bold**, and bob ref merges it
   into its highlight's [!quote] callout. Draw it as the underline it was:
   full-ink semibold words over a thick gold rule that passes under
   descenders, matching the Bob Mac Capture Refs inspector. Nested Comment
   callouts are not affected. */
.callout[data-callout="quote"] > .callout-title strong,
.callout[data-callout="quote"] > .callout-content > p strong {
  color: var(--text-normal);
  font-weight: var(--font-semibold, 600);
  text-decoration-line: underline;
  text-decoration-color: color-mix(
    in srgb,
    var(--color-yellow) 65%,
    var(--color-orange)
  );
  text-decoration-thickness: 0.15em;
  text-underline-offset: 0.12em;
  text-decoration-skip-ink: none;
}
```

The selectors stay exactly as they are today, so nested Comment callouts are still left
alone. Bob's quote text sits on the `[!quote]` title line, which is why the
`.callout-title strong` selector matters.

### `bob ref show` in a color terminal

> [!decision] surfaces = mac_and_obsidian Skip this section; `bob ref show` keeps ANSI
> bold.

> [!decision] surfaces = mac_only Skip this section; `bob ref show` keeps ANSI bold.

Underlined segments render as bold plus underline (SGR `1;4`). This works in every
terminal, unlike colored-underline SGR 58, which older terminals misread as blink or
italic. Any whitespace at either end of a painted segment goes outside the escape codes,
so the underline never runs under a space at a wrap edge. Plain or `NO_COLOR` output
still shows the literal `**`, exactly as today.

### Non-goals

- No change to the stored or wire format. `**` stays what Highlights exports, what
  `bob ref sync` merges, and what `bob ref show -f json` returns. Switching to `<u>`,
  `==`, or a new JSON field would mean migrating every ref note and changing three
  parsers, with nothing to gain, because each renderer can restyle `**`.
- No change to `bob ref sync` merge rules, IDs, or the `bob-cli-9o` ordinal limitation.
- No per-color underlines. The Highlights underline color is not carried through the
  export, and the inspector's quote rule is already a fixed yellow.
- No new setting. This is the one default look.
- Bob Mac Capture remains a thin client. It already splits the `**` runs bob returns for
  display, and this changes presentation only.

## Implementation

Run `sase repo open bob-mac-capture -r "<reason>"` (the `/sase_repo` skill) and work in
the printed checkout. Read its `AGENTS.md` before editing. Match each file's existing
indentation: RefsCore uses 4 spaces and `Sources/BobMacCapture/Refs/` mostly uses 2.

### A. bob-mac-capture

**A1. Pure underline model in RefsCore (Linux-testable).** Add
`Sources/RefsCore/RefsQuoteUnderline.swift`, built on the existing
`RefsQuoteEmphasis.runs` and kept beside it in spirit. Use `Double`, not CoreGraphics
types, so it builds on Linux.

- `RefsQuotePiece { text, role }` with `role ∈ { plain, underlined, underlineGap }`.
  `RefsQuoteUnderline.pieces(_ quote:)`:
  - leading and trailing whitespace of a strong run becomes `plain`;
  - inside a strong run, non-whitespace stretches become `underlined`, and whitespace
    stretches become `underlineGap` unless they contain a newline, in which case they
    become `plain`;
  - adjacent pieces with the same role merge.
  - Invariant to test: the pieces joined back together equal
    `RefsQuoteEmphasis.plainText(quote)`.
- `phrases(_:)`: the underlined phrases in order, trimmed and non-empty.
  `isWhollyUnderlined(_:)`: true when the quote has underlined words and no
  non-whitespace plain piece. Both feed VoiceOver.
- `bars(in:)`: takes the run extents of one laid-out line (`role`, `minX`, `maxX`, in
  layout order). It merges consecutive `underlined`/`underlineGap` runs into one span,
  drops `underlineGap` runs at either end of a span, and returns the spans. A span made
  only of gaps produces no bar.
- `bar(minX:maxX:baseline:ascent:descent:increasedContrast:displayScale:)` returns
  `{x, y, width, height, cornerRadius}?`. It estimates the em size from the line metrics
  (≈ `(ascent + descent) / 1.19` for SF), applies the thickness, offset, and minimum
  sizes from the design table, snaps `y` and `height` to `1 / displayScale`, and returns
  nil for a zero or negative width. Keep these constants as named, documented statics.
- Unit tests in `Tests/RefsCoreTests/`. Cover: a plain quote; a mid-quote run; a run
  with leading and trailing spaces; a multi-word run (gaps); a run containing a newline;
  a whole-quote run; adjacent runs (`**A** **B**`); unpaired `**`; the join-back
  invariant; `bars(in:)` gap trimming and merging across font-fallback splits (two
  consecutive `underlined` runs); and `bar(...)` minimum thickness, Increase Contrast,
  pixel snapping at scale 1 and 2, and nil for zero width.

**A2. VoiceOver label.** In `RefsInspectorFeed.Entry.accessibilityLabel`
(`Sources/RefsCore/RefsInspector.swift`), add the underline suffix from the design right
after the quote text, in both highlight branches (with and without a note card). Extend
the existing label tests in `Tests/RefsCoreTests/RefsInspectorTests.swift`.

**A3. Renderer (`Sources/BobMacCapture/Refs/RefsUnderlineRenderer.swift`).**

- `struct RefsUnderlineMark: TextAttribute { let role: RefsQuotePieceRole }`.
- `struct RefsUnderlineRenderer: TextRenderer` with stored `increasedContrast: Bool`.
  Keeping it stored rather than read from the environment lets proofs force it, since
  `colorSchemeContrast` cannot be set through `.environment`. For each line in the
  `Text.Layout`:
  1. Map its runs to extents: the role is `run[RefsUnderlineMark.self]?.role ?? .plain`,
     and `minX`/`maxX` come from `run.typographicBounds.rect`.
  2. Take the baseline, ascent, and descent from the line's typographic bounds, so every
     bar on a line aligns even when fonts mix.
  3. Fill each `RefsQuoteUnderline.bars(in:)` → `bar(...)` rect as a rounded rect with
     `RefsVisualTokens.inspectorUnderlineInk`, using `ctx.environment.displayScale`.
  4. Then call `ctx.draw(line)`.

  Set `displayPadding` so a bar under the last line is never clipped.

- A small `View` helper (for example `refsUnderlineInk()`) reads
  `@Environment(\.colorSchemeContrast)` and applies
  `.textRenderer(RefsUnderlineRenderer(increasedContrast: …))`.

**A4. Builder and call sites.** Replace `RefsQuoteText.attributed(_:)` with
`RefsQuoteText.text(_ quote: String) -> Text`. Build it from
`RefsQuoteUnderline.pieces`:

- `plain` → `Text(verbatim:)`;
- `underlined` → `Text(verbatim:)` with `.fontWeight(.semibold)`,
  `.foregroundStyle(.primary)`, and
  `.customAttribute(RefsUnderlineMark(role: .underlined))`;
- `underlineGap` → the same but with `.underlineGap`.

Join the pieces with an API that is not deprecated in the macOS 26 SDK. If `Text + Text`
warns, use `Text("\(a)\(b)")` interpolation. Use the new builder plus
`refsUnderlineInk()` in:

- `RefsInspectorView.quoteView`;
- both quote lines of `RefsCopyCardView.pickerRow`. These keep their italic secondary
  style; underlined words lift to primary.

Remove `attributed(_:)` once nothing calls it, and update the doc comment of
`RefsQuoteText`.

**A5. Tokens.** Add `inspectorUnderlineInk` to `RefsVisualTokens` with a doc comment
explaining gold as the yellow rule's deeper sibling. Extend the enum's header comment
("yellow … and orange … mean your highlights and notes") to mention gold for underlines.

**A6. Design renders.** Add `testRefsInspectorUnderlinesToPNG` to
`Tests/BobMacCaptureTests/RefsPanelDesignTests.swift`, reusing its
`write(model:name:width:)` pattern. Use the screenshot's real content as fixture quotes:

- Page 2 plain;
- Page 4 with the two merged runs, one long enough to wrap;
- Page 5 with the run ending in `Sapling,2`;
- a whole-quote standalone underline;
- a short underline full of descenders (for example `**typography, quickly**`).

Render compact `880` and wide `1344`, light and dark. If the file has a Copy card picker
render helper, add one picker render with an underlined row. CI runs these, and the Mac
only type-checks them.

**A7. Docs.** In `docs/refs-inspector.md`, replace "Quotes render balanced `**bold**`
runs as strong emphasis" with the new behavior: the semibold full-ink words, the rounded
gold underline (one bar per line, never under the end spaces), the same look in the Copy
card picker, VoiceOver announcing underlined words, and raw `**` on the clipboard. Grep
`docs/` for other quote-bold wording (`strong emphasis`, `bold`) and fix it. Follow
`docs/` conventions and run `just docs-check`.

### B. bob-cli: `bob ref show`

> [!decision] surfaces = mac_and_obsidian Skip part B.

> [!decision] surfaces = mac_only Skip part B.

- `src/native/style.rs`: add a `Styler` method that paints SGR `1;4` (for example
  `underline_ink`).
- `src/native/ref_library/show.rs` `flush_quote_segment`: for an active segment, write
  its leading and trailing whitespace unpainted and paint only the inner text with the
  new method. Leave the pairing and wrap logic as it is.
- Update the show.rs tests that assert `\u{1b}[1m` (rename
  `quote_emphasis_is_bold_only_in_color_output` to match its new meaning, and fix
  `quote_emphasis_reopens_after_wrapping`). Add one test showing that a wrapped
  segment's edge whitespace sits outside the escape codes. Grep `src/` and `tests/` for
  other expectations of quote bold.

### C. Vault snippet and bob-cli docs

> [!decision] surfaces = mac_only Skip the vault and CSS edits. In
> `docs/highlights-ref-sync.md`, change only the wording that names the Mac panel's
> rendering, if any.

- Use your `/sase_memory_read` skill to read `obsidian.md` (vault sync rules). Then
  replace the body of `~/bob/.obsidian/snippets/ref-highlights.css` on athena with the
  CSS above. `ref-highlights` is already in `enabledCssSnippets`. Do not hand-commit the
  vault: the vault sync service commits and propagates the change.
- `docs/highlights-ref-sync.md` (use your `/sase_memory_read` skill for `docs.md`
  first):
  - **MacBook validation notes:** change "renders as bold … verify ANSI bold" to
    "underlined bold" plus SGR `1;4`. Check the gold underline in Obsidian light and
    dark themes.
  - **Underline emphasis:** reword the simple rule: "an underlined run inside a quote
    means an underline merged; a wholly underlined quote means it stayed standalone; the
    note stores both as `**`".
  - Replace the snippet block with the CSS above, byte for byte, and update the
    surrounding prose ("give bold quote text full ink", "applies to bold text in all
    quote callouts") to match.
  - Mention the Mac inspector's matching gold underline in one sentence.
- Do not edit `docs/reference/mac-capture/*`. Those are mounted copies owned by
  bob-mac-capture.

## Verification

1. **bob-mac-capture portable gate (athena):** `just portable-check`. The new RefsCore
   tests must pass.
2. **Mac compile (required attempt):** use your `/sase_memory_read` skill for
   `tailnet.md`. Rsync a scratch copy (without `.build`/`.git`) to a temporary directory
   on the Mac, run `Scripts/bundle.sh` and `just typecheck-tests` there, and fix every
   error and every new warning. Never use Bryan's checkout or `just install`. If the Mac
   is offline, say in the final report that the macOS targets are unverified.
3. **Visual proof sheet (best effort, while the scratch copy still exists):** in the
   scratch copy only, add a throwaway executable target that depends on RefsCore.
   - Give it copies of `RefsQuoteText.swift`, `RefsUnderlineRenderer.swift`, and the ink
     token.
   - Render the A6 fixture quotes through `ImageRenderer` at scale 2: wide body and
     compact callout widths, light and dark, with Increase Contrast forced through the
     stored flag. Add one italic secondary picker row.
   - Copy the PNGs to `~/tmp/bob-underline-proof/` on athena and inspect them with your
     image reader.
   - Tune the constants until the underlined words pop at a glance against the plain
     text in both appearances, and each bar is crisp, sits clearly below the baseline,
     runs under descenders without hiding them, never extends under end spaces, and
     breaks cleanly per wrapped line.
   - Then delete the scratch copy. List the proof paths in the final report.
4. **bob-cli:** `just check`. Also force a TTY with
   `script -qc 'bob ref show ref/blogs/redesigning_source_control.md' /dev/null` (run
   from the workspace's built binary, with `BOB_DIR` left at its default) and confirm
   that the Page 4 underlined runs carry `ESC[1;4m` and that piped output still shows
   literal `**`.
5. **Docs:** `just docs-check` in bob-cli and in bob-mac-capture.
6. **Obsidian:** agents cannot see it. In the final report, ask Bryan to open
   `ref/blogs/redesigning_source_control.md` (Page 4) in light and dark themes.

## Done when

- The Refs inspector and the Copy card picker draw `**` runs as semibold, full-ink words
  over a pixel-crisp, rounded, gold underline that follows the design table, and every
  glyph still renders unchanged.
- VoiceOver labels announce underlined words. RefsCore tests cover pieces, bars, bar
  geometry, and labels.
- Design-render tests for underlines exist. Both docs trees describe the new look and
  pass `docs-check`.
- Per the `surfaces` decision: the vault snippet and its docs block match byte for byte,
  and `bob ref show` paints underlined segments with SGR `1;4`, with edge whitespace
  outside the codes and `just check` green.
- Nothing changes in how bob stores or returns underlines (`**` everywhere, JSON
  unchanged).
