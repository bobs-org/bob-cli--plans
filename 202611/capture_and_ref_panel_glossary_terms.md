---
tier: tale
title: Add Capture Panel and Ref Panel glossary terms
goal:
  The glossary web defines Bob Mac Capture's capture window (Capture Panel) and its Bob
  Refs window (Ref Panel) with lean read closures, and the published glossary page
  matches.
size: small
decisions:
  capture_panel_term:
    ask:
      Add the Capture Panel glossary term as the new strand glossary/capture-panel.md?
    memory:
      - glossary:capture-panel
    default: true
    requested:
      Can you help me add excellent new "capture panel" and "ref panel" terms to the
      glossary that specify the bob-mac-capture app's capture panel and its ref panel,
      respectively?
    answer: true
  ref_panel_term:
    ask:
      Add the Ref Panel glossary term (aliases Bob Refs, refs panel) as the new strand
      glossary/ref-panel.md?
    memory:
      - glossary:ref-panel
    default: true
    requested:
      Can you help me add excellent new "capture panel" and "ref panel" terms to the
      glossary that specify the bob-mac-capture app's capture panel and its ref panel,
      respectively?
    answer: true
  capture_strand_link:
    ask:
      Also link the Capture strand's "Bob Mac Capture's panel" phrase to the new Capture
      Panel term?
    memory:
      - glossary:capture
    default: false
    answer: true
proposed_by: bbugyi200.apollo.et
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-17 18:10:45
status: wip
---

# Add Capture Panel And Ref Panel Glossary Terms

## Goal

Add two glossary terms to the `glossary` memory web: **Capture Panel** for the capture
window in Bob Mac Capture (the `bob-mac-capture` linked repo) and **Ref Panel** for its
reference-library window, which the app calls **Bob Refs**. Each definition says what
the panel is, how it opens, what it shows, what it is allowed to write (always through
`bob`), how the two panels relate, and what it is not. The published glossary page is
regenerated to match.

## Context (verified while planning)

- Strands live at `sase/memory/glossary/<slug>.md`. Frontmatter holds `keyword:` and an
  optional quoted `aliases:` list. The body is one dense paragraph of about 250–350
  words that ends with "Not …" contrasts and a "See `docs/...`" pointer.
  `sase/memory/glossary/capture.md` and `sase/memory/glossary/command-mode.md` show the
  house style.
- `sase memory init` rebuilds the roster block in `sase/memory/glossary.md`, plus
  `AGENTS.md`, `CLAUDE.md`, and `sase/memory/README.md`. Never hand-edit those.
- `docs/reference/glossary.md` is generated from the web by
  `uv run --project tools/docs bob-docs gen` (memory pages are part of `just docs-gen`).
  `just docs-check` fails when that page drifts, and also when it names a bead ID
  (`bob-mac-capture-<id>`). The bare repo name `bob-mac-capture` is fine. The tree had
  no drift at plan time (`bob-docs gen --check` passed).
- Mac Capture docs are mounted into `docs/reference/mac-capture/` but that directory is
  gitignored. Cite them the way `capture-alias.md` does, as Bob Mac Capture's
  `docs/<page>.md`.
- **Mention closure (`closure: mentions`).** `sase memory read glossary:<x>` inlines the
  full definition of every term whose keyword or alias appears in plain text in `<x>`,
  and does so recursively. Matching is case-insensitive, takes the longest phrase, and
  accepts plurals. An authored `[[slug]]` link appears only as a "Linked References"
  row; it is not inlined. Tested on a scratch copy of `sase/memory`:
  - A bare word still matches inside a link that has no hyphen, so `[[pomodoro]]` or
    `[[glossary:pomodoro]]` inlines Pomodoro. Hyphenated slugs such as
    `[[reference-note]]` do not match, and code spans never do.
  - The word "Capture" inside "Bob Mac Capture" inlines Capture. Almost every large term
    reaches the same cluster of about 4,300 words (Capture → Pomodoro → Task Type → Task
    Link → …). Other traps found: "keeps" matches Keep Streak, "type" and "theme" match
    Task Type, and plain "reference note" matches Reference Note (about 5,000 words with
    its closure).
  - The phrase "capture panel" resolves to Capture Panel, not Capture. The plural "Refs
    panel" does **not** match the keyword "Ref Panel", so it needs an alias.
- **Naming.** In the app's own menus and docs the two panels are "Capture" and "Bob
  Refs" (`BobPanelCoordinator`: only one Bob panel is ever visible, and Ctrl+Tab
  switches between them). bob-cli docs call the second one the "Refs panel"
  (`docs/ref.md`, `docs/vault-git-sync.md`). "Capture" cannot be an alias because it is
  already the Capture keyword, so the Capture Panel strand states the naming difference
  itself.

## Design choices

- **Capture Panel** has no aliases. "Mac Capture panel" already matches the keyword.
  Inlining Capture is intended, because a capture panel submit _is_ one Capture. A read
  of `capture-panel` is about 4,650 words, in line with neighbouring terms.
- **Ref Panel** gets the aliases `Bob Refs` (the product name used in the app's menus,
  docs, and decision records) and `refs panel` (the bob-cli docs name). It must stay
  **lean**: no plain-text mention of any other glossary term. That means it names the
  app as `` `bob-mac-capture` `` in a code span, links [[reference-note]],
  [[reference-task]], [[task-link]], and [[capture-panel]], and never uses the bare
  words capture, Pomodoro, keeps, type, or theme. A read is about 540 words.
- The `refs panel` alias makes Annotation Section's existing phrase "the Mac Refs panel
  group the same way" a mention. Its read grows from 258 to about 780 words and now
  explains which panel that is. No edit to `annotation-section.md` is needed.
- No new terms beyond these two, and no decision records. Decision records are immutable
  and none is wrong.

## Steps

1. Use `/sase_memory_write` first. Accepted `capture_panel_term`, `ref_panel_term`, and
   (if accepted) `capture_strand_link` decisions authorize exactly the edits below.
   Touch no other memory note.
2. Open the linked repo with `/sase_repo` (`sase repo open bob-mac-capture`). Check
   every factual claim in the two drafts below against its current
   `docs/capture-panel.md`, `docs/refs.md`, `docs/refs-inspector.md`,
   `docs/keyboard.md`, `docs/event-notes.md`, and `docs/protocol.md`: hotkeys, footer
   actions, Escape, Ctrl+Tab, stash behaviour, sections, card names, and `bob` commands.
   Fix any claim that has drifted, and keep the lean rules above while doing it.
3. > [!decision] capture_panel_term Create `sase/memory/glossary/capture-panel.md` with
   > this content:

   ```markdown
   ---
   keyword: Capture Panel
   ---

   Bob Mac Capture's capture window: the pre-warmed, non-activating `NSPanel` that the
   global hotkey (Control-Shift-Command-I by default, plus `Command-\`) or the menu-bar
   item's Capture row opens over any app. It stacks a one-line editor that grows with
   the draft, an auxiliary region, and footer actions that always stay visible (Capture,
   Preview, Discard, Stash, and the `⌃↩ Open` chip). The auxiliary region shows today's
   Pomodoro agenda while the draft is empty
   ([[decisions:idle-capture-shows-ledger-agenda]]), then completion and picker cards,
   bob's live preview (`bob capture --dry-run`), and errors as you write. Every show
   puts the input line at one fixed eye line; height follows content, and width stays
   resizable. It is a thin client ([[decisions:mac-capture-is-a-thin-client]]): spans,
   completions, rewrites, previews, and picker rows all come from `bob capture` and
   `bob capture-*` JSON, and submitting sends the whole draft, one item or a
   [[bulk-capture]], as exactly one [[capture]]. The panel hides only after bob reports
   success; a failure leaves the full draft open beside an actionable error. Escape,
   `Ctrl+[`, and Ctrl+Tab hide it with the draft and caret retained; Discard clears the
   draft; Control-C, Quit, and Restart stash a nonempty draft that Control-S restores.
   It also works without capturing: Go To (`/` tasks, `//` notes) and event note mode
   (`Command-E`) open inside it over the retained draft. Only one of it and the
   [[ref-panel]] is visible at a time, and Ctrl+Tab switches between them. The app's
   menus and docs call it just "Capture"; in this glossary a capture is one
   `bob capture` run, so say "capture panel" for the window. Not the Obsidian
   [[command-mode]] sheet, the app's menu-bar item or Settings window, or the
   Hammerspoon [[mac-menu-bar-pomodoro-indicator]]. See Bob Mac Capture's
   `docs/capture-panel.md`.
   ```

4. > [!decision] ref_panel_term Create `sase/memory/glossary/ref-panel.md` with this
   > content:

   ```markdown
   ---
   keyword: Ref Panel
   aliases:
     - "Bob Refs"
     - "refs panel"
   ---

   The reference-library window of the `bob-mac-capture` menu-bar app, named **Bob
   Refs** in its menus and docs: a near-full-screen, non-activating glass panel that
   quick-opens any PDF-backed [[reference-note]] while Highlights stays frontmost,
   hiding again when it loses key focus. Control-Shift-Command-O or `Command-]` toggles
   it from any app; while Highlights is frontmost, `Command-O` opens it in place of
   Highlights' Open dialog, and `Command-T` and `Command-Y` open it on Highlights' open
   tabs or on the focused PDF's Copy card. A search field filters two-line rows (kind
   tile, title, reading state, a NEW dot, and a Now pill while the row's
   [[reference-task]] has a [[task-link]] under one of today's open ledger entries)
   grouped under pinned headers such as Just scanned, NEW, NOW, Reading, Next, and
   Ready; `Command-0` regroups them by date added and `Command-T` narrows them to
   Highlights' open tabs. An inspector column beside the list fills from `bob ref show`.
   Return opens the PDF in Highlights (the note when the PDF is missing) and changes
   nothing in the vault; other keys open the note or the reading task in Obsidian,
   reveal the PDF in Finder, or raise in-panel cards: Copy (IDs, links, code blocks,
   highlights; [[decisions:code-copy-reads-render-time-catalog]]), Rename
   (`bob ref rename`; [[decisions:reference-renames-keep-identity]]), and Link to Today
   (`bob ref today`). `Command-S` runs `bob ref scan -a -w` and reports what it filed
   ([[decisions:open-pdf-writes-are-coordinated]]). Escape or `Ctrl+[` backs out of a
   card or banner, then hides the panel with its view remembered for the next show. Like
   the [[capture-panel]], it is a thin client
   ([[decisions:mac-capture-is-a-thin-client]]): rows come from `bob ref list` and
   `bob plan`, and it never parses notes or writes the vault itself. Its NEW section,
   Seen marks, and open history are app-local state, never the `]s` walk's NEW tier.
   Only one of the two panels is visible at a time; Ctrl+Tab switches between them, and
   each comes back as it was left. Not Highlights, not the reference's Obsidian note,
   and not `bob ref` CLI output. See `docs/refs.md` and `docs/refs-inspector.md` in the
   `bob-mac-capture` repo.
   ```

5. > [!decision] capture_strand_link = true In `sase/memory/glossary/capture.md`,
   > replace "or Bob Mac Capture's panel, which submits" with "or the [[capture-panel]],
   > which submits", keeping the line wrapping tidy. Change nothing else in that strand.
   > Skip this step when the decision is false.
6. Run `sase memory init`. Confirm the roster in `sase/memory/glossary.md` now lists
   `Capture Panel` and `Ref Panel (Bob Refs, refs panel)` in alphabetical order.
7. Run `uv run --project tools/docs bob-docs gen` (or `just docs-gen`). Among docs
   files, only `docs/reference/glossary.md` should change, gaining
   `### Capture Panel {#capture-panel}` and `### Ref Panel {#ref-panel}` with working
   anchor links. If other generated files change, find out why before committing them;
   the tree had no drift at plan time.

## Verification

- `sase memory show glossary:ref-panel | grep -E '^#+ '` prints only `# Ref Panel`,
  `## Linked References`, and numbered `### N.` rows. Any other heading is a stray
  plain-text mention; reword it into a hyphenated `[[link]]` or a code span.
- `sase memory show glossary:capture-panel | grep -E '^#+ '` inlines Capture (expected)
  and never `Ref Panel`.
- `sase memory show glossary:annotation-section | grep -E '^#+ '` now inlines only
  `## Ref Panel` beyond itself, at about 780 words in total.
- `sase memory show glossary:capture | wc -w` stays about 4,280 words. Capture Panel
  must never be inlined there; with `capture_strand_link`, it appears only as a Linked
  References row.
- `sase memory web show glossary capture-panel -b` and
  `sase memory web show glossary ref-panel -b` show the keywords and aliases.
- `just docs-check` passes.
- `git status` shows only the new strand files, `capture.md` if relinked,
  `sase/memory/glossary.md`, `sase/memory/README.md`, `AGENTS.md`, `CLAUDE.md`, and
  `docs/reference/glossary.md`.

## Out of scope

- Any change in the `bob-mac-capture` repo or its docs.
- Edits to decision records, `annotation-section.md`, or the other strands that name Mac
  Capture cards (`inbox-file.md`, `project-note-capture.md`, `task-successor.md`,
  `capture-alias.md`). The roster already makes the new terms discoverable.
