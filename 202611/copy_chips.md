---
tier: epic
title: "Copy chips: click-to-copy text authored in Obsidian notes"
goal: "Typing an inline code span that starts with `copy:` (or a `bob-copy` fence) gives
  a quiet, theme-native chip that copies exactly the authored text with one click or one
  key, in every Obsidian view. Mod+Shift+Y makes or edits a chip without hand-escaping.
  The GTD agent prompts in gtd_daily.md are the first chips.

  "
decisions:
  memory_decision_record:
    ask:
      Record a decision that copy chips are code spans whose payloads are literal text?
    default: false
    memory:
      - decisions:copy-chips-are-code-spans
    answer: false
  memory_glossary_copy_chip:
    ask: Add a Copy Chip glossary strand?
    default: false
    memory:
      - glossary:copy-chip
    answer: true
phases:
  - id: grammar
    title: Plugin scaffold, grammar module, and conformance vectors
    depends_on: []
    size: medium
    description:
      "grammar: create and register the bob-copy-chips plugin, then build the pure
      parser, scanner, and serializer every surface shares, pinned by JSON vectors and a
      round-trip property test."
  - id: inline
    title: Inline chips in Live Preview and Reading view
    depends_on:
      - grammar
    size: large
    description:
      "inline: render copy chips as quiet pills in Live Preview and Reading view, copy
      on click with in-place feedback, add the cursor-line copy command, and publish
      api.copyChips v1."
  - id: fence
    title: The bob-copy fence card
    depends_on:
      - inline
    size: medium
    description:
      "fence: render bob-copy fences as titled cards with a header Copy button and
      byte-exact payloads, and teach chipAt and the cursor-line command about fences."
  - id: copycard
    title: Copy card C row
    depends_on:
      - inline
    size: medium
    description:
      "copycard: add a C row to the Mod+Y Copy card in Bob Navigation Hotkeys that
      copies the chip at the cursor through api.copyChips."
  - id: author
    title: Make-or-edit card and Alt+Click edit
    depends_on:
      - fence
    size: large
    description:
      "author: add the Mod+Shift+Y card that makes, converts, or edits a chip with a
      live preview, writing the result in one undo step, plus Alt+Click to edit."
  - id: adopt
    title: Vault adoption, GTD prompt chips, and memory
    depends_on:
      - author
      - copycard
    size: medium
    description:
      "adopt: add the gy Vim mapping, turn the GTD agent prompts into chips, reconcile
      with bob-cli-nh's new Summarize yesterday row, apply accepted memory decisions,
      and record follow-ups."
proposed_by: bbugyi200.apollo.research.5g.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-19 10:05:30
status: wip
---

- **PROMPT:**
  [prompts/202611/copy_chips.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/copy_chips.md)

# Plan: Copy chips

## Context

Bryan asked for a button or link in Obsidian that copies author-chosen text to the
clipboard and is easy to create by typing. The consolidated research report
(`sase artifact read research:202611/obsidian_copy_chips/obsidian_copy_chips.md "<why>"`)
designed a **copy chip**. Bryan accepted every requirement in it: the adjustments
A1–A10, the reliability rules R1–R14, the code-span syntax, the `bob-copy` fence, and
Mod+Shift+Y. Every phase worker should read that report first. This plan carries out the
report's phases 1 and 2 and their adoption in the vault. Its phase 3 (snippet picker,
masked chips, rich text) is deferred, and the adopt phase records it as proposed
follow-ups.

Where the work lands:

- **bob-plugins** (open with `/sase_repo`):
  - a new plugin, `plugins/bob-copy-chips/`, built from committed source fragments;
  - a small addition to `bob-navigation-hotkeys` (the Copy card).
- **chezmoi** (open with `/sase_repo`): add the plugin to the `plugins.sync` allowlist
  in `home/dot_config/bob/config.yml`. `bob plugins sync` deploys only the ids listed
  there.
- **The Bob vault `~/bob`**:
  - `.gitignore`, `.obsidian/community-plugins.json`, and `obsidian_vimrc.md`;
  - the open rows of `gtd_daily.md`.

  Follow the vault `AGENTS.md`: inspect `git status` first, and commit only your own
  files with `git add <files> && git commit`, never `/sase_git_commit`.

Out of scope:

- no bob-cli code change;
- no capture grammar change;
- no Bob Mac Capture work (`decisions:mac-capture-is-a-thin-client`).

The bob-cli docs site mounts the bob-plugins `docs/` tree, so no bob-cli docs edit is
needed either.

## The product

### Syntax

| You type                                                   | You see                          | One click copies               |
| ---------------------------------------------------------- | -------------------------------- | ------------------------------ |
| `` `copy: ssh apollo` ``                                   | ⧉ `ssh apollo` (monospace pill)  | `ssh apollo`                   |
| `` `copy[Zoom]: https://zoom.us/j/123` ``                  | ⧉ Zoom (text-font pill)          | the URL                        |
| text containing backticks, in a longer run (example below) | ⧉ last-week prompt               | the prompt, backticks included |
| a ` ```bob-copy Weekly status ` fence                      | a titled card with a Copy button | the fence body, byte-exact     |

```markdown
launch an agent with
``copy[last-week prompt]: Update the last-week period note (`bob periods show last-week` prints its path)``
```

- An unlabeled chip shows its own text, in monospace: it _is_ the literal text.
- A labeled chip shows the label in the text font: it names a thing, like a link.
- With the plugin off, every chip still reads as legible inline code or a fence (A9).

### Look

The chip should feel quiet. Take the look from Bob's dependency and relation chips
(`.bob-dep-chip` in bob-ledger-tools `styles.css`), not from button kits.

- **Colors and shape.**
  - Use theme variables only: `--background-secondary`, `--background-modifier-border`,
    `--interactive-accent`, `--radius-s`, `color-mix`.
  - No hex values and no per-chip colors.
  - Lucide `copy` glyph (via `setIcon`), never 📋.
- **At rest.**
  - A hairline-bordered pill: padding `0 .45em`, font size `.92em`, inherited line
    height, and a faint glyph in `--text-faint`.
  - The pill is `display: inline` with `box-decoration-break: clone`, so a long
    unlabeled chip wraps like inline code and is never truncated.
- **Hover and focus.**
  - The border tints toward the accent, the glyph takes the accent color, and the cursor
    is `copy`.
  - Keyboard focus shows a `:focus-visible` outline.
  - Tooltip:
    - unlabeled chip: "Click to copy";
    - labeled chip: "Copy: " plus a preview of the payload, with `⏎` for newlines,
      truncated at 120 characters in the preview only (A7).

    The author phase appends "· Alt+Click to edit" in Live Preview.

- **Success.** The glyph swaps in place to a green `check` for about 1.2 s.
  - The chip pulses once in `--bob-yank-flash-bg` (fallback `rgba(80, 200, 120, .35)`),
    so a click-copy and a Vim yank look alike.
  - The glyph slot has a fixed width, so the chip never reflows.
  - No toast appears. A shared, visually hidden `aria-live="polite"` region announces
    "Copied".
  - Under `prefers-reduced-motion`, the pulse is dropped and the glyph swap kept.
- **Failure.** An `alert-triangle` glyph in `--text-error` for about 2 s, plus a Notice:
  "Couldn't copy: <reason>". A failure is never silent: it would leave stale clipboard
  contents behind.
- **Print and PDF export.** The chip prints as its face, styled like inline code, with
  no interactive styling.

Live Preview, a GTD row with the cursor elsewhere:

```text
☐ Start last week's review draft: launch an agent with ⧉ last-week prompt · done once …
                                                    ╰ hover: "Copy: Update the last-week period note (`bob …"
```

Block card (fence phase):

```text
╭──────────────────────────────────────────────────────────────╮
│ ⧉  Weekly status email               4 lines · 212 chars  [ Copy ] │
├──────────────────────────────────────────────────────────────┤
│ Hi team —                                                    │
│ Here is where things stand this week:                        │
╰──────────────────────────────────────────────────────────────╯
```

Make-or-edit card (author phase):

```text
╭─ ⧉ COPY CHIP ──────────────────────────────────────────────────╮
│ Label     [ last-week prompt                                 ]  │
│ Text      [ Update the last-week period note (`bob periods … ]  │
│ Preview   ⧉ last-week prompt                    inline chip     │
│ Markdown  ``copy[last-week prompt]: Update the last-week …``    │
╰─ ⏎ save · ⇧⏎ newline · Tab next field · Esc cancel ────────────╯
```

### Every way in

| Gesture                                                                      | Where                                                                   | Effect                                               |
| ---------------------------------------------------------------------------- | ----------------------------------------------------------------------- | ---------------------------------------------------- |
| Click / tap a chip                                                           | Live Preview, Reading view, embeds, hover previews, Tasks query results | Copy                                                 |
| `gy` (normal mode; vimrc, adopt phase) or palette "Copy chip on cursor line" | Editor                                                                  | Copy the chip at the cursor and flash it like a yank |
| `Mod+Y` then `C`                                                             | Copy card                                                               | Copy the chip at the cursor, previewed first         |
| `Mod+Shift+Y`                                                                | Editor, any Vim mode                                                    | Make, convert, or edit a chip                        |
| Alt+Click a chip                                                             | Live Preview                                                            | Edit that chip in the card                           |
| Move the cursor into the span                                                | Live Preview                                                            | Reveal the Markdown source, like a link              |

**The cursor rule, shared by `gy`, `Mod+Y C`, and the card through `chipAt`:**

1. A chip under the cursor wins.
2. Otherwise, the only chip on the cursor line.
3. Otherwise, the `bob-copy` fence the cursor is inside.
4. Two or more chips on the line with the cursor on none is **ambiguous**: refuse with
   "N chips on this line · move onto one". Never guess.

## Design choices beyond the research

1. **Converting a plain code span is one gesture.** `Mod+Shift+Y` with the cursor in an
   ordinary code span opens the card with that span's text, so an existing command
   becomes a chip with Enter. The vault holds about 8k `sase` spans and about 4k `bob`
   spans.
2. **`gy` yanks a chip in normal mode**, and the chip flashes like a Vim yank. During
   the `]s` walk, `]s` lands on "Start last week's review draft" and `gy` copies its
   prompt. No vimrc mapping is needed for making a chip, because `Mod+Shift+Y` already
   works in every Vim mode.
3. **One cursor rule** (above), published through `api.copyChips.chipAt`, so `gy` and
   the Copy card can never disagree.
4. **Some code spans never become chips.**
   - Spans inside link text, wikilinks, fenced code, math, comments, or frontmatter stay
     plain code. A chip inside a link would fight the link's own click.
   - Code spans that span several lines stay plain code in Live Preview, because CM6
     forbids an inline replace decoration across a line break. The serializer never
     writes them.
5. **The payload never lives in DOM attributes or innerHTML.** It is set through
   `textContent` and kept in JS closures or a `WeakMap`, which satisfies R2, R11, and
   R13. Tooltips show a truncated preview only.
6. **The card refuses stale writes.** At commit it re-reads the target range. If the
   note changed under the open card, it writes nothing and says "The note changed —
   reopen the card".
7. **Alt+Click edits; there is no hover pencil.** A pencil appearing on hover would add
   noise and reflow.
8. **The GTD prompt chips are labeled** (`last-week prompt`, `last-quarter prompt`,
   `last-year prompt`, `summary prompt`), so the walk rows stay short. The bold task
   title already says what the prompt is for, and hovering shows the prompt itself.
9. **Registration happens in the first phase.** The plugin joins the chezmoi allowlist,
   the vault ignore list, and the enabled list in `grammar`. That way every later phase
   can follow bob-plugins' rule to run `bob plugins sync` after changes.

## Rules for every bob-plugins phase

- **Build.**
  - Edit only `src/` fragments; never hand-edit a generated `main.js`.
  - Keep each fragment at or under 1000 lines.
  - Run `npm run build`, `npm test`, `npm run validate`, `npm run docs:gen`, and
    `npm run docs:check`.
- **Versions.** Bump the touched plugin's manifest version: a minor bump per feature
  phase, starting `bob-copy-chips` at `0.1.0`.
- **Tests.** Add every new `node --test` script to `test:unit` or `test:perf` in
  `package.json`.
- **Deploy.** Commit in bob-plugins, then run `bob plugins sync`. On a refusal, follow
  bob-plugins `AGENTS.md`; never pass `-F` on your own. Tell Bryan to reload Obsidian.
- **Real-Obsidian e2e.** Use the existing CDP harness pattern:
  - see `scripts/obsidian-copy-card-e2e.cjs`, its `-lib.cjs`, and the offline
    `test-obsidian-copy-card-e2e.cjs` self-test wrapper;
  - run in an isolated scratch vault under `xvfb-run -a`, never against the live vault;
  - read the clipboard over CDP.
- **Decision records to honor.**
  - `decisions:q-and-ctrl-bracket-mirror-esc`: Esc, `q`, and `Ctrl+[` close the card;
    `q` types inside text fields.
  - `decisions:code-copy-reads-render-time-catalog`: payloads come from source, never
    from rendered layout text.
  - `decisions:mac-capture-is-a-thin-client`.
- **The exactness contract (A6).** The clipboard gets exactly the declared text, byte
  for byte. Preview and clipboard never disagree. Rendering and clicking never write the
  vault (R8); only the author card writes, in one undo step.

## Phase: grammar — scaffold, grammar, vectors

**Scaffold** `plugins/bob-copy-chips/`:

- **`manifest.json`.**
  - id `bob-copy-chips`, name "Bob Copy Chips", version `0.1.0`;
  - `minAppVersion` `1.8.7`, `isDesktopOnly: false`, author "Bryan";
  - a description naming copy chips and `bob-copy` fences.
- **Fragments** (listed in `src/fragments.json`):
  - `010-load.js`: guarded requires, modeled on bob-emoji's;
  - `020-grammar.js`: pure and free of Obsidian imports;
  - `090-plugin.js`: the plugin class; it builds a frozen `api` that never throws;
  - `099-exports.js`: `module.exports.helpers`.
- **`styles.css`**: a stub.
- **Docs.** Start `docs/bob-copy-chips.md` with the syntax section. Add it to
  `docs/nav.yml`, then run `npm run docs:gen`.

**Register so the plugin deploys:**

1. **chezmoi.** Open the chezmoi repo with `/sase_repo`. Add `bob-copy-chips` to
   `plugins.sync` in `home/dot_config/bob/config.yml`, in alphabetical order. Commit
   there, then run `chezmoi apply ~/.config/bob/config.yml` on this host.
2. **Vault.**
   - Add `.obsidian/plugins/bob-copy-chips/` to `~/bob/.gitignore`, beside the other
     `bob-*` plugin entries.
   - Add `"bob-copy-chips"` to `.obsidian/community-plugins.json`.
   - Commit only those two files.
3. **Deploy.** Run `bob plugins sync`, then confirm the plugin's row in
   `bob plugins list`.

**Grammar** (`020-grammar.js`; all functions pure, all results frozen):

- **`copyChipParseContent(content) → {label, text} | null`.** `content` is code-span
  content after CommonMark normalization. The grammar is
  `copy [ "[" LABEL "]" ] ":" [ " " ] TEXT`:
  - `copy` must be lowercase and first.
  - LABEL is one line, with `\]` → `]` and `\\` → `\`; any other backslash is literal.
    The label is trimmed. A blank label means "no label".
  - TEXT is everything after the colon and one optional space, verbatim. It must contain
    a non-whitespace character, so `` `copy:` `` and a whitespace-only TEXT are never
    chips (R14).
  - There are no other escapes: `\n` copies two characters, and nothing is interpolated
    (A4).
- **`copyChipNormalizeSpan(raw, {table}) → content | null`.** `raw` is the whole span,
  delimiters included. The function:
  - strips equal-length backtick runs;
  - maps line endings to spaces;
  - strips one leading and one trailing space when both are present and the content is
    not all spaces;
  - in table context, unescapes `\|`.
- **`copyChipScanLine(line, {table}) → [{from, to, raw, label, text}]`.** It returns
  every chip span on one raw line, following CommonMark code-span rules:
  - any run length works (R3);
  - a backslash-escaped backtick does not open a span;
  - an unmatched run is literal;
  - every chip on the line is independent (R4);
  - spans inside `[[…]]` or a Markdown link label are skipped.

  Callers skip fenced lines.

- **`copyChipParseFenceInfo(info) → {label} | null`.** The language token must be
  exactly `bob-copy`. The label is the rest of the info string, trimmed, with
  surrounding `[ ]` stripped; blank means no label.
- **`copyChipFencePayload(bodyLines) → text`.** The body lines joined with LF:
  - a final empty body line keeps exactly one trailing LF;
  - the payload is never trimmed.

  The fence phase pins this against Obsidian's real code-block `source` and corrects it
  if needed.

- **`copyChipSerialize({label, text}, {table, indent}) → {ok, form, markdown} | {ok: false, reason}`.**
  - **Inline form**, when the text has no LF:
    - always write `copy: ` (with the space), or `copy[LABEL]: `;
    - the backtick run is one longer than the longest run inside the content;
    - pad both edges with one space when the content ends with a backtick;
    - escape `]` and `\` in the label;
    - escape `|` as `\|` in table context.
  - **Block form**, when the text has an LF:
    - use a backtick fence, or a tilde fence when the label contains a backtick;
    - the fence length is the larger of 3 and one more than the longest run of the fence
      character at the start of any body line;
    - `indent` prefixes every line;
    - write `bob-copy LABEL` as the info string;
    - a trailing LF becomes one extra empty body line.
  - **Refusal reasons:** "nothing to copy" for blank or whitespace-only text, and
    "labels are one line" for a label containing a newline.
  - CRLF in the text normalizes to LF.

**Tests:**

- **`scripts/fixtures/copy-chips-vectors.json`** is the one shared vector file (R1).
  Later phases' unit tests and e2e consume it too. It has these groups:
  - `chips`: line → expected chips. Include:
    - several chips on one line;
    - a double run with backticks inside;
    - a padded span;
    - a table row with `\|`;
    - Unicode text;
    - trailing spaces in the payload;
    - a labeled chip with `\]`.
  - `not_chips`:
    - `` `copy:` ``, `` `Copy: x` ``, and `` `copy x` ``;
    - a whitespace-only payload;
    - an escaped backtick;
    - a span inside a link label or wikilink;
    - `` `= dataview` ``.
  - `serialize`: input → exact Markdown.
  - `fence_info`.
- **`scripts/test-copy-chips-grammar.cjs`** runs the vectors.
- **`scripts/test-copy-chips-roundtrip.cjs`** is a seeded property test, run 5,000 times
  or more with no dependencies:
  - random text over an alphabet weighted toward `` ` ``, `\`, `[`, `]`, `|`, space,
    tab, and non-ASCII characters;
  - random labels, in and out of table context;
  - assert that `scanLine(serialize(x))` yields exactly one chip equal to `x`, and the
    same for fences through `copyChipFencePayload`;
  - print the seed on failure.

**Collision census.** Scan the vault read-only for inline spans that would become chips;
skip the gitignored `lit_review/` and `xlib/`. Report them in the phase notes. The
research found 3, all inside `.bob/ref-sources/`. If any user-facing note would change
meaning, record a `PROPOSED FOLLOW-UP:`.

## Phase: inline — Live Preview and Reading view chips

Shared fragments:

- **Feedback** (`030-feedback.js`):
  - `copyChipWrite(text)` awaits `navigator.clipboard.writeText` inside the gesture and
    resolves `{ok: true}` or `{ok: false, reason}` (R9);
  - the success and failure glyph states and their timers;
  - the shared live region, removed on unload.
- **Chip DOM** (`040-chip-dom.js`): `copyChipBuildInline(doc, chip, {interactive})`,
  used by Live Preview, Reading view, and later the author card's preview. It sets the
  classes `bob-copy-chip`, `is-labeled` or `is-literal`, and the transient `is-copied`
  or `is-failed`, plus the glyph slot `bob-copy-chip-glyph`.

**Live Preview** (`050-live-preview.js`): a `ViewPlugin` registered with
`registerEditorExtension`.

- **Which spans it decorates.**
  - Only when `editorLivePreviewField` is true; Source mode shows raw Markdown.
  - It walks `syntaxTree(state)` over `view.visibleRanges` only (R12). Span boundaries
    come from Obsidian's inline-code nodes, and the raw slice goes through
    `copyChipNormalizeSpan` and then `copyChipParseContent`.
  - It skips the contexts listed in design choice 4, and multi-line spans.
- **The decoration.** A `Decoration.replace` widget covers the whole span, backticks
  included, unless some selection range touches `[from, to]`. Then the source shows,
  which is the standard Live Preview reveal.
- **When it rebuilds.** Only from editor state: on document, viewport, selection, and
  syntax-tree changes. It does no DOM work outside its widgets and keeps no state across
  modes (R7).
- **Clicks.**
  - `ignoreEvent()` returns true, and `mousedown` calls `preventDefault`, so the caret
    never jumps into the source.
  - A copy fires only on a primary-button release within the chip after under 4 px of
    movement, so a drag never copies (R10).
  - The widget's `eq()` compares label, text, and position, so redraws keep feedback
    state stable.

**Reading view** (`060-reading.js`): `registerMarkdownPostProcessor`.

- It handles every `code` element not inside `pre` or `a`, and not inside an internal
  link.
- It parses `code.textContent` (already normalized by the renderer, R2) and replaces the
  element with a real `<button type="button">`. The button has
  `aria-label="Copy <face>"`, and Enter or Space activates it.
- This covers embeds, hover previews, Tasks and Dataview query results, and PDF export
  (R6), because they all run post-processors.
- Dataview `=` and `$=` spans are untouched (R11).

**Cursor-line command** (`070-commands.js`):

- Palette command "Copy chip on cursor line", id `copy-on-cursor-line`, an
  `editorCallback` with no default hotkey.
- It resolves through `chipAt` (the cursor rule) using the live buffer.
- It copies, then flashes:
  - a rendered widget pulses like a click;
  - if the cursor is inside the revealed source, the span's range flashes through a
    `StateEffect` mark decoration in `--bob-yank-flash-bg`.
- "No copy chip on this line" and the ambiguous case show a Notice.

**`api.copyChips` v1**, reached as
`app.plugins.plugins["bob-copy-chips"].api.copyChips`. It is frozen and never throws:

| Member                      | Returns                                                   |
| --------------------------- | --------------------------------------------------------- |
| `version`                   | `1`                                                       |
| `parse(content)`            | `{label, text}` or `null`                                 |
| `scanLine(line, opts)`      | the chips on one line                                     |
| `serialize(input, opts)`    | as in the grammar phase                                   |
| `chipAt({lines, line, ch})` | `{state: "none" \| "ambiguous" \| "ready", chip?, count}` |
| `copy(text)`                | `Promise<{ok, reason?}>`                                  |
| `copyChip(chip, {view?})`   | `Promise<{ok, reason?}>`                                  |
| `flash(chip, {view?})`      | `true` when it showed feedback                            |

- `chipAt` reads a zero-based line, skips fenced lines, and returns a chip shaped
  `{form, label, text, line, from, to}`.
- `copyChip` is `copy` plus the on-chip feedback.
- `flash` shows only the success feedback, with no clipboard write, for callers that
  already copied.

Document the namespace in `docs/develop/plugin-apis.md`.

**CSS.** Implement the [Look](#look) section.

**Tests:**

- **Unit tests.** Model and DOM tests run through the existing fake-node harness
  patterns in `scripts/`. They cover:
  - each vector line producing the expected widgets;
  - feedback timers;
  - the payload never reaching attributes;
  - the cursor rule in `chipAt`.
- **CPU budget.** `scripts/test-copy-chips-live-preview-performance.cjs` uses
  `scripts/cpu-work-perf-support.cjs`. Run it on a 2,000-line daily note holding 200
  chips, within the harness budget, and add it to `test:perf`.
- **Real-Obsidian e2e.** `scripts/obsidian-copy-chips-e2e.cjs` with its `-lib.cjs`, plus
  an offline `--self-test` wrapper in `test:unit`. It checks:
  - **Live Preview click:** a click copies, and the caret does not move.
  - **Drag:** a drag across a chip does not copy.
  - **Reveal:** moving into the span reveals the source.
  - **Reading view:** a button click copies, and so does keyboard Enter.
  - **Embeds:** an embedded note's chip copies.
  - **Several chips on one line:** each copies its own payload.
  - **Payload edge cases:**
    - a double-backtick payload with backticks inside;
    - a table-cell `\|`;
    - trailing spaces.
  - **Refusals:**
    - empty `copy:` is not a chip;
    - a `writeText` rejection stub gives the alert glyph and a Notice;
    - Dataview spans are untouched.
  - **Vim mode:** switching normal → insert → normal with chips on the line works.
  - **Cursor-line command:** it copies.
  - **Every vector line agrees in every surface:** the Live Preview payload equals the
    Reading view payload, and both equal the grammar's payload.
- **Docs.** Complete `docs/bob-copy-chips.md`:
  - syntax, the look and feedback, gestures, and the cursor rule;
  - the reliability contract;
  - non-goals: not a secret store (A8), no variables, no `obsidian://copy` handler.

## Phase: fence — the bob-copy card

- **Processor.** `registerMarkdownCodeBlockProcessor("bob-copy", …)`.
- **Payload.** It comes from the processor's `source` argument.
  - First pin Obsidian's exact `source` shape in the e2e, including the trailing newline
    and list-nested indentation, then make `copyChipFencePayload` agree.
  - An extra blank line before the closing fence keeps one trailing LF.
  - Never `trim()`.
- **Label.** It comes from `ctx.getSectionInfo(el)`'s opening line, through
  `copyChipParseFenceInfo`. When section info is null (callouts, some embeds), it falls
  back to "Copy · N lines", and the payload stays exact.
- **Card.**
  - **Header:** glyph, label, and a meta line in `--text-muted` with tabular figures: "N
    lines · M chars", plus "· ends with ⏎" when the payload ends with LF.
  - **Copy button:** fixed minimum width; it flips to "Copied ✓" for about 1.2 s, or to
    the failure state with a Notice.
  - **Body:** the text font with `white-space: pre-wrap`. It folds after 8 lines behind
    a "Show all N lines" toggle (session-only state).
  - **Clicks:** clicking the body selects text and never copies. Only the button copies.
- **Fences in `chipAt`.**
  - A cursor on the opening, body, or closing line of a `bob-copy` fence resolves that
    fence as a chip with `form: "block"`. Inline chips on the cursor line still win.
  - Fence detection must match the Copy card's list-nested backtick and tilde fence
    handling (`copyCardLineIsInFence` in bob-navigation-hotkeys).
  - `copy-on-cursor-line` flashes the card.
- **e2e cases.**
  - Live Preview and Reading view cards.
  - Payload exactness:
    - a body with an interior blank line copies byte-exact;
    - the trailing-newline rule;
    - a fence nested in a list item;
    - a fence inside a callout falls back to the generic label.
  - Fold and expand.
  - A body click does not copy.
  - The cursor-line command on a fence works.
- **Docs.** Add the fence section to `docs/bob-copy-chips.md`, with the guidance that
  `bob-copy` is for text that should read as a titled card (prompts, emails, addresses).
  Ordinary code listings keep the native fence copy button.

## Phase: copycard — the Copy card C row

Work in `plugins/bob-navigation-hotkeys`, in the Copy card fragments `101`–`103`, `641`,
and `717`.

- **Resolving the chip.** At open, alongside the task resolver and under the same guards
  (an editing mode and one selection), call
  `app.plugins.plugins["bob-copy-chips"]?.api?.copyChips?.chipAt`.
  - Feature-detect `version >= 1`.
  - When the plugin is absent or the call fails, there is no chip group.
- **The row.** A `chip` group after `TASK` with one row: id `chip`, key `c`.
  - **Ready:** the row label is "Chip · <face>". The value is the payload, previewed
    with `⏎` and middle truncation, like the template rows.
  - **Ambiguous:** a dimmed row, "N chips on this line · move onto one". Pressing `C`
    nudges it, like other dimmed rows.
  - **None:** no group.
- **Reserved key.** `c` joins the reserved keys even when the row is hidden. A
  `copy.templates` entry keyed `c` diagnoses "key c is Chip's" and stays copyable by
  `↑↓ ↵`. Bryan's config uses only `t`.
- **Copying.** Copy through the existing `copyCardRow` path, which keeps the card's
  Notice on success. Afterwards, call `copyChips.flash(chip, {view})` when the API
  exposes it, so the chip visibly answers.
- **Tests.**
  - Update the model, keys, and view tests (`scripts/test-navigation-copy-card-*.cjs`).
  - Extend `scripts/obsidian-copy-card-e2e.cjs`:
    - `Mod+Y C` copies the chip payload;
    - the ambiguous line refuses;
    - with bob-copy-chips disabled, the group is absent.
- **Docs.** Update `docs/copy-card.md`:
  - the Keys table;
  - the reserved list, `P A F N I C`;
  - one cross-link: "Authored payloads live on copy chips; live context lives on the
    Copy card."

## Phase: author — make-or-edit card and Alt+Click

**The command.** "Make or edit copy chip", id `make-or-edit`, an `editorCallback` with
the default hotkey `Mod+Shift+Y`. The research verified that no vault hotkey binds `Y`.
The card opens on the first context that applies:

| Context                              | Card opens with                                                        | Enter writes                                                                                                                                                                                                                      |
| ------------------------------------ | ---------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A non-empty one-line selection       | Text = selection; focus on Label                                       | An inline chip replacing the selection                                                                                                                                                                                            |
| A non-empty multi-line selection     | Text = selection; focus on Label                                       | A `bob-copy` fence replacing the selection, with line breaks added before or after when the selection starts or ends mid-line. The fence is indented to the list item's content column, reusing the line's own indent characters. |
| Cursor in a chip or `bob-copy` fence | Label and text prefilled; title "Edit copy chip"                       | The chip rewritten in place                                                                                                                                                                                                       |
| Cursor in a plain inline code span   | Text = span content; focus on Label                                    | That span converted to a chip                                                                                                                                                                                                     |
| Anything else                        | Clipboard text prefilled and preselected, when readable; focus on Text | Insert at the cursor                                                                                                                                                                                                              |

Several selections give a Notice: "One selection at a time".

**The card UI** (a Modal styled with theme variables in the Bob card language; see
`.bob-copy-card*` in bob-navigation-hotkeys `styles.css` for reference):

- **Fields.**
  - **Label:** one line, placeholder "optional — shown instead of the text".
  - **Text:** an auto-growing textarea in monospace, so trailing spaces and tabs are
    visible.
- **Preview row.** Built with the real chip and card renderers, non-interactive. Beside
  it, the form: "inline chip", or "block card · N lines".
- **Markdown row.** The exact Markdown the commit will write, monospace and scrollable,
  never elided.
- **Inline validation**, from `serialize`: "nothing to copy", "labels are one line", and
  a whitespace-only warning. Enter is disabled while the input is invalid.
- **Keys.**
  - Enter commits from either field.
  - Shift+Enter inserts a newline in Text, and the form switches to block live.
  - Tab and Shift+Tab move between fields.
  - A second `Mod+Shift+Y` commits.
  - Esc and `Ctrl+[` close; `q` types in the fields
    (`decisions:q-and-ctrl-bracket-mirror-esc`).
- **Footer hints:** "⏎ save · ⇧⏎ newline · Tab next field · Esc cancel".

**The write.**

- One editor transaction replaces exactly the target range: the selection, the chip
  span, the fence's lines including both fences, or the insertion point. This gives one
  undo step (R8).
- Table rows serialize with `{table: true}`.
- The stale guard (design choice 6) applies.
- The cursor lands just after the written chip.
- A Vim visual selection works, and the editor leaves visual mode after the commit.

**Alt+Click** on a Live Preview chip opens the card in edit mode for that chip. A plain
click still copies, and the tooltip gains "· Alt+Click to edit".

**Tests.**

- Unit tests for context resolution, indentation, mid-line splitting, and the stale
  guard.
- e2e cases:
  - wrap a one-line selection;
  - turn a multi-line selection under a list item into an indented fence;
  - edit a chip in place;
  - convert a plain span;
  - prefill from the clipboard;
  - text containing backticks gets a double run;
  - a table-cell chip;
  - Esc, `q`, and `Ctrl+[` behave as specified;
  - one `Mod+Z` restores the original;
  - Vim visual mode → card → commit;
  - Alt+Click opens the editor;
  - a second `Mod+Shift+Y` commits.

**Docs.** Add an Authoring section to `docs/bob-copy-chips.md` covering typing, the
card, conversion, and Alt+Click.

## Phase: adopt — vault, GTD prompts, memory

1. **Preflight.** Confirm all of these:
   - `bob plugins list` shows bob-copy-chips synced and enabled;
   - the Copy card build that carries the C row is deployed;
   - `~/.config/bob/config.yml` lists the plugin in `plugins.sync`.
2. **Vim.**
   - Check that CodeMirror Vim and `obsidian_vimrc.md` leave `gy` unbound.
   - Append this block after the zoom-image block, following the file's comment style.
     It must be `nnoremap`, per the file's header rule:

     ```vim
     " gy yanks the copy chip on the cursor line (one chip, or the one under the cursor) and flashes it like a yank.
     " Mod+Shift+Y makes or edits a copy chip in any mode.
     exmap bob_copy_chip obcommand bob-copy-chips:make-or-edit
     exmap bob_yank_chip obcommand bob-copy-chips:copy-on-cursor-line
     nnoremap gy :bob_yank_chip<CR>
     ```

3. **GTD prompts.**
   - Re-read `gtd_daily.md` fresh. In each **open** row (Blocked, Ready, and so on —
     never the Done & Canceled history), replace each double-quoted agent prompt, quotes
     included, with a labeled chip.
   - The payload must equal the old quoted text byte for byte.
   - Change nothing else in the row: no status, fields, dates, or other words
     (`decisions:task-status-is-derived`).
   - The current rows, with each chip's label and the prompt text it carries:
     - **Start last week's review draft** and **Weekly review:** label
       `last-week prompt`.
     - **Start last quarter's review draft** and **Quarterly review:** label
       `last-quarter prompt`.
     - **Start last year's review draft** and **Yearly review:** label
       `last-year prompt`.
     - **Summarize yesterday** (added by `bob-cli-nh` phase `adopt`), if present with a
       quoted prompt: label `summary prompt`.

     Each replacement keeps the words around it. The exact Markdown for the last-week
     rows and the Summarize yesterday row:

     ```markdown
     launch an agent with
     ``copy[last-week prompt]: Update the last-week period note (`bob periods show last-week` prints its path)``
     · done once the agent runs, … (still empty? launch
     ``copy[last-week prompt]: Update the last-week period note (`bob periods show last-week` prints its path)``
     now) launch an agent with
     ``copy[summary prompt]: Summarize yesterday's daily note (`bob periods brief yesterday` prints the instructions and evidence)``
     · done once the agent runs, …
     ```

   - Other rows that `bob-cli-nh` reworded may differ from this list. Convert whatever
     quoted agent prompts the open rows hold at the time.

4. **Coordinate with `bob-cli-nh`.**
   - Read its bead and notes (`sase bead read bob-cli-nh -r "<why>"`). A COPY CHIP note
     on it asks its adopt phase to write the Summarize yesterday prompt as a chip.
   - If that epic has not reached its GTD rows yet and the shipped syntax or label
     differs from that note, append a correcting note.
   - If its row already landed with a quoted prompt, step 3 converts it.
5. **Verify.**
   - Use a Node one-liner that loads `plugins/bob-copy-chips/main.js`'s `helpers`. Every
     chip in the open rows must parse, and its payload must equal the prompt it
     replaced.
   - Run `bob task reconcile`. It must leave the converted rows' text unchanged; it
     changes no task text.
   - Commit only `gtd_daily.md` and `obsidian_vimrc.md`.
6. **Memory**, as accepted.

   > [!decision] memory_decision_record Run `/sase_memory_write`, then add
   > `sase/memory/decisions/copy-chips-are-code-spans.md` in the decisions web's record
   > shape (Applies to / Claim / Why / Cost / Reopens when / Evidence).
   >
   > - **Claim:** authored click-to-copy text lives in a `copy:` inline code span or a
   >   `bob-copy` fence; payloads are literal, never interpolated, and copied byte-exact
   >   from source; chips never navigate, run code, or register a protocol handler.
   > - **Why:** the research's evidence (no escaping, parser inertness, mode-consistent
   >   rendering, the Jump to Link conflict, degradation).
   > - **Rejected:** `[label](copy:…)` links, button frameworks, and `obsidian://copy`.
   > - **Evidence:** the research ref and the shipped commits.
   >
   > Then run `sase memory init`.

   > [!decision] memory_glossary_copy_chip Run `/sase_memory_write`, then add the
   > `glossary:copy-chip` strand ("Copy Chip"; aliases: chip, copy card fence,
   > bob-copy). It defines the inline and block forms, the cursor rule, and the gestures
   > (click, `gy`, `Mod+Y C`, `Mod+Shift+Y`), and links to the plugin docs. Then run
   > `sase memory init`.

   For each memory decision left off, record a `PROPOSED FOLLOW-UP:` note on this phase
   bead instead, unless a human reviewer turned it off.

7. **Report.**
   - Ask Bryan to reload Obsidian on each host. Plugin folders are vault-ignored, so
     apollo and the Mac also need `bob plugins sync` (and `chezmoi apply` for the
     allowlist) before chips render there.
   - If Obsidian rewrote `community-plugins.json` before the reload, enabling the plugin
     in Settings → Community plugins fixes it.
8. **Proposed follow-ups.** Record each as a `PROPOSED FOLLOW-UP:` note on the phase
   bead:
   - the research's phase 3: a vault-wide "Copy a snippet…" picker, a masked `copy*`
     variant, rich-text card copy, and a per-note opt-out;
   - plain-text task surfaces show raw chip source: pickers, the Task Card title, and
     the Mac agenda. An `api.copyChips` face helper could feed them;
   - `.obsidian/plugins/bob-emoji/` is tracked in vault git and missing from the vault
     `.gitignore` (found while planning this epic).

## Non-goals

- Variables, templates, or JS in payloads. Live substitution stays with the Copy card's
  `copy.templates`.
- A secret store. Chips are plain text in git on three hosts.
- An `obsidian://copy` protocol handler, Publish interactivity, and a global typing
  trigger or autocomplete. The research rejected each.
- Any bob-cli, capture-grammar, or Mac Capture change.
