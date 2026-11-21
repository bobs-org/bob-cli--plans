---
tier: epic
title: Wikilink completion in Bob's prose prompts
goal: "Typing `[[` in any Bob-owned field that writes Markdown prose into the vault
  (intentions, Work/Rotten/Cancel/Schedule Log reasons, record text, notebook summaries,
  the Mac Pull-In reason) opens one consistent LINK band that inserts exactly the link
  it shows, never submits a half-typed link, never touches text after the caret, and
  never writes the vault. Identifier and filter fields stay literal.

  "
decisions:
  link_text:
    ask: Which link text should an accepted note row insert in Obsidian prompts?
    choices:
      path: Bob's vault-relative path, e.g. [[2026/2026-11-21]]; matches bob and the Mac
      shortest:
        Obsidian's shortest unique form, e.g. [[2026-11-21]]; the Mac keeps paths
    default: path
    why:
      Matches ~7.3k machine-written Bob links and stays unambiguous as the vault grows
    answer: path
  decision_record:
    ask: Record the `[[` scope rule and Enter rule as a new decisions strand?
    memory:
      - decisions
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: false
phases:
  - id: cli-link-fixes
    title: Fix bob's wikilink field range and ranking, publish shared vectors
    depends_on: []
    size: medium
    description:
      "cli-link-fixes: in bob-cli, end unclosed link fields at the caret, keep authored
      #subpath and |alias text when the target is replaced, break ranking ties by depth
      then inbound links, publish tests/fixtures/wikilink_complete/vectors.json, and
      write the new decisions strand."
  - id: link-engine
    title: Obsidian link engine (field model, ranking port, metadata index)
    depends_on:
      - cli-link-fixes
    size: medium
    description:
      "link-engine: in bob-navigation-hotkeys, add pure JS fragments for the field model
      and ranking that replay bob's vectors exactly, plus a lazy LinkIndex built on
      Obsidian's metadataCache. No UI."
  - id: link-band
    title: proseField controller, LINK band, and the Work summary pilot
    depends_on:
      - link-engine
    size: medium
    description:
      "link-band: build the two-mode proseField controller and the LINK band (rendering,
      CSS, keyboard contract, undo-safe insertion, a11y, unresolved-link hint) and
      api.proseField v1. Pilot it on block-id-prompt's Work summary prompt (S1) and
      verify undo, IME, and Esc layering in real Obsidian."
  - id: modal-prompts
    title: Standalone modal prompts
    depends_on:
      - link-band
    size: medium
    description:
      "modal-prompts: opt surfaces S2-S6, S12, and S13 into links with a correct
      sourcePath. Keep each host's guards, and close the Ctrl+[ gaps on these surfaces."
  - id: task-card-prompts
    title: Task Card stages and the scheduled inline reason
    depends_on:
      - link-band
    size: medium
    description:
      "task-card-prompts: opt the Task Card picker stages S7-S10 into links through the
      shared picker wiring. Add S11, the scheduled-stage reason tail, through minOffset
      so the date token never completes."
  - id: intention-card
    title: Intention Card fields
    depends_on:
      - link-band
    size: medium
    description:
      "intention-card: attach proseField to the Intention Card's intention input,
      summary input, and reflection textarea (S14). The band must survive repaints and [
      / ] travel, and it is the first rung of the card's back-out ladder."
  - id: record-card
    title: Record Card typed stages and link-stage convergence
    depends_on:
      - link-band
    size: medium
    description:
      "record-card: attach proseField to the Record Card's text, suggest, and list
      stages (S15). Converge the link stage onto the engine in field mode, retiring
      recordCardLinkCandidates and its bracket bug, and survive the per-keystroke
      repaint."
  - id: cli-links-only
    title: bob capture-complete --links-only
    depends_on:
      - cli-link-fixes
    size: small
    description:
      "cli-links-only: add -L/--links-only and -S/--source-note to bob capture-complete.
      They complete only wikilink fields in plain prose and never apply capture grammar.
      Includes help, docs-gen, and tests."
  - id: mac-pull-in
    title: Mac Pull-In reason link band
    depends_on:
      - cli-links-only
    size: medium
    description:
      "mac-pull-in: wire the Pull-In cancel reason field (S16) in bob-mac-capture to
      capture-complete --links-only. Render a LINK band in the recall band's slot with
      the same key contract, fail open on an older bob, and watch macOS CI to green."
  - id: prose-lint
    title: Prose-field role lint and end-to-end walkthrough
    depends_on:
      - modal-prompts
      - task-card-prompts
      - intention-card
      - record-card
    size: small
    description:
      "prose-lint: mark every text control in bob-plugins with a role (prose, literal,
      mixed, or none) and add a contract test that pins the S1-S15 allowlist and its
      wiring. Then run one isolated real-Obsidian walkthrough across representative
      surfaces."
proposed_by: bbugyi200.apollo.research.5o.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-21 10:32:14
status: wip
---

- **PROMPT:**
  [prompts/202611/prose_wikilink_completion.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/prose_wikilink_completion.md)

# Plan: Wikilink completion in Bob's prose prompts

## Summary

Bob already has `[[` completion in the Obsidian editor and in the Mac capture panel. It
does not work in the prompts launched from those surfaces, so Bryan types links like
`[[sase#^ref-sase-1n6-11-4-restart-readiness]]` by hand. A typo leaves a dangling link
without any warning. The write side already works: links in Work, Schedule, Rotten, and
Cancel Log entries become Task Mentions; the Intention Card renders links; and
`bob periods` repair retargets them. This plan adds the authoring side.

The design follows the consolidated research report
`research:202611/prose_prompt_wikilink_completion/prose_prompt_wikilink_completion.md`.
Bryan accepted all of its requirements. Its main points:

- One controller owns all prose fields.
- Candidates come from Obsidian's in-memory metadata, never from spawning a process per
  keystroke.
- The ranking is a JS port of bob's policy, pinned by shared fixture vectors.
- Enter and Tab accept a link and never submit the prompt.
- Completion only inserts text; it never writes the vault.
- Two verified bob defects are fixed first.

This plan makes four further calls:

- **Opt-in per surface.** The research proposed that existing recall surfaces get links
  with zero call-site changes. Instead, each surface opts in explicitly with its own
  `sourcePath`. That is one line per call site, every surface's link context becomes
  reviewable, and the role lint stops drift. The research itself calls `sourcePath`
  mandatory context.
- **A third defect.** `See [[sa▏#Overview|our plan]] ok` currently returns a replacement
  range of `{6,26}`. Accepting any row deletes `#Overview|our plan`. Phase
  `cli-link-fixes` fixes this alongside the two known defects.
- **No caret-anchored popover anywhere.** The band, in the recall band's slot, is the
  design on every surface.
- **`((` task links in prompts are a non-goal.** They need block-ID minting, which is a
  vault write.

## Scope

### The rule

`[[` completes in every Bob-owned text field whose value Bob writes into the vault as
Markdown prose. Every other field stays literal. This includes picker filters, block
IDs, rename fields, bank and album fields, Command Mode argv, copy-chip payloads (where
bash `[[ -f x ]]` is literal), and email To, Subject, and compose fields.

### Prose surfaces (the allowlist)

Repo prefixes: **nav** = bob-plugins `plugins/bob-navigation-hotkeys/src`, **bip** =
`plugins/block-id-prompt/src`, **ldg** = `plugins/bob-ledger-tools/src`, **mail** =
`plugins/bob-mail/src`, **mac** = bob-mac-capture.

| ID  | Surface                                                                    | Where                                                                      | Recall today              | `sourcePath`                                                         | Phase             |
| --- | -------------------------------------------------------------------------- | -------------------------------------------------------------------------- | ------------------------- | -------------------------------------------------------------------- | ----------------- |
| S1  | Unlink / task-link-open Work summary                                       | bip `090-prompt-modals.js` `WorkSummaryPromptModal`                        | work, ownKeys             | link-task: `source.sourcePath`; task-link-open: `source.target.path` | link-band         |
| S2  | Move-to-Next summary                                                       | nav `196-task-link-lane-modal.js`                                          | work, ownKeys             | the tasks' note if they share one, else `null`                       | modal-prompts     |
| S3  | Lane-release summary                                                       | nav `190-summary-modals.js` `LaneReleaseSummaryModal`                      | work, ownKeys             | released task's note                                                 | modal-prompts     |
| S4  | Batch keep reason                                                          | nav `192-freshness-refresh-summary-modal.js`                               | rotten, ownKeys, no scope | single shared entry note, else `null`                                | modal-prompts     |
| S5  | Rotten keep / decay Stay reason                                            | nav `195-rotten-keep.js` (hosted by 192 and `060-decay-card-modal.js`)     | rotten, ownKeys           | `keep.entry` / decay entry note                                      | modal-prompts     |
| S6  | Successor inline cancel reason                                             | nav `206d-successor-picker-verdict-render.js`                              | cancel, ownKeys, dense    | `row.notePath`                                                       | modal-prompts     |
| S7  | Task Card cancel-reason stage                                              | nav `380-picker-cancel-and-lane.js`                                        | cancel, picker            | `picker.filePath` (or a single link-session path)                    | task-card-prompts |
| S8  | Task Card lane-release-reason stage                                        | nav `380-picker-cancel-and-lane.js`                                        | work, picker              | same as S7                                                           | task-card-prompts |
| S9  | Task Card schedule-work-log stage                                          | nav `370-picker-schedule-work-log.js`                                      | work, picker              | same as S7                                                           | task-card-prompts |
| S10 | Task Card schedule-review reason + summary                                 | nav `360-picker-schedule-review.js` + `185` `setupScheduleReviewLogRecall` | schedule/work, picker     | same as S7                                                           | task-card-prompts |
| S11 | Task Card scheduled inline reason (`3 waiting on [[api`)                   | nav `360`/`410` typed row, `280` `matchTypedSchedulePrefix`                | none                      | same as S7                                                           | task-card-prompts |
| S12 | Notebook create Summary                                                    | nav `685-plugin-notebook-create.js`                                        | none                      | `null`                                                               | modal-prompts     |
| S13 | Task from email                                                            | mail `040-task-modal.js`                                                   | none                      | `null` (bob capture routes it)                                       | modal-prompts     |
| S14 | Intention Card intention, summary, reflection                              | ldg `297c*`                                                                | none                      | `() => modal.subjectPath`                                            | intention-card    |
| S15 | Record Card text / suggest / list stages, plus the link stage (field mode) | ldg `242o`–`242s`                                                          | none                      | `modal.recordPath`                                                   | record-card       |
| S16 | Mac Pull-In cancel reason                                                  | mac `CapturePullInChoiceView.swift` `CapturePullInReasonField`             | cancel band               | the task's note, when the row carries it                             | mac-pull-in       |

**Never completes:**

- Every picker filter, including the shared `FilteredPickerModal` filter when it is not
  in S7–S11.
- Block-ID and rename fields, and notebook ID and Title.
- Theme and relation-label managers.
- Bank, album schema, and emoji slot fields.
- The record choice filter, and the record date, number, and URL stages.
- Gmail search, compose To and Subject.
- Command Mode sheet and args, copy and run chip payloads, and every settings field.
- The **Mac event-note editor**: `escape_note_inline` rewrites every `[` to `\[` in gcal
  private notes, so a completed link would be corrupted.

## Design

### Architecture

```text
 host prompt ──attach──▶ proseField (bob-navigation-hotkeys)            one keyboard owner
                           ├─ recall mode  — USED BEFORE (existing 183–185, unchanged)
                           ├─ link mode    — LINK band (new)
                           │    ├─ field model   pure, UTF-16, mirrors bob after the fix
                           │    ├─ ranker        pure, replays bob's vectors byte-for-byte
                           │    └─ LinkIndex     lazy, app.metadataCache + vault events
                           └─ hint line   — unresolved-link notice (informational)
 Mac Pull-In reason ──▶ bob capture-complete --links-only (bob decides; Mac renders)
```

**Why the controller lives in nav.** The recall band, which is the main keyboard
competitor, is already there, so one owner arbitrates both modes and no cross-plugin
yielding protocol is needed. Ledger-tools and mail cards feature-detect `api.proseField`
the same way block-id-prompt feature-detects `logRecall`. When the API is missing they
fall back to plain typing. The layering stays as it is today: UI kit in nav, data in
ledger-tools. The band's CSS lives in nav's `styles.css`, which is loaded whenever the
API exists.

**Why the ranking is ported, not spawned.** `bob capture-complete` costs 0.4–1.2 s per
call because `NoteIndex::read` walks the whole vault. The thin-client decision
(`decisions:mac-capture-is-a-thin-client`) binds the Mac app, not Obsidian plugins. A
port pinned by byte-identical vectors keeps exactly one policy: if either side changes
it, the other side's CI fails. This is the same arrangement as `log_recall`.

### Field model

The JS model must match the Rust model after `cli-link-fixes`, vector for vector.

- **Opener.** The field opens at the last unescaped `[[` or `![[` before a collapsed
  caret on the same line, provided:
  - it is not closed before the caret;
  - it is not inside an inline or fenced code span;
  - it is not escaped (`\[[`).

  A nested `[[` restarts scanning at the inner opener.

- **Stages.**
  - target: until `#` or `|`;
  - heading: after `#`;
  - block: after `#^`;
  - current-note heading or block: `[[#…`, `[[#^…`;
  - vault-wide: `[[##…`, `[[^^…`.

  With the caret in the `|alias` tail, nothing completes; the controller shows the alias
  hint instead.

- **Replacement range.**
  - _Closed link_: replace only the stage's component (target, heading text, or block
    ID). Authored `#subpath` and `|alias` text is kept, the existing `]]` is reused
    exactly once, and the caret lands after `]]`.
  - _Unclosed link_: replace from the component start to the caret, never past it.
    Append `]]` unless the character after the caret is `#` or `|`, since an authored
    continuation must stay intact. The caret lands after the inserted text.
- **Alias rows.** An alias-match row inserts `target|Alias`. When the link already has
  an authored `|label`, it inserts the target alone: the author's display text wins.
- **Offsets.** JS uses UTF-16 throughout and never splits a surrogate pair. Bob's byte
  spans are converted only on the Mac path.
- **Opening rules.** These belong to the controller, not to the vectors.
  - The band opens only on a user `input` event that leaves a collapsed caret in a
    field.
  - Never on paste or drop (`insertFromPaste`, `insertFromDrop`), during IME composition
    (re-check on `compositionend`), on a programmatic recall fill, or with a
    non-collapsed selection.
  - Caret-only moves never open the band. A caret that leaves the link closes it.
- **`minOffset`.** Hosts may set it so that openers before an offset never open. S11
  uses it to restrict links to the reason tail after the date token.
- **`mode: "field"`.** The whole value is one link target, for the Record Card link
  stage. If the value already starts with `[[`, the inline model applies. Otherwise the
  entire value is the query, and accepting a row replaces the whole value with
  `[[insert]]`.

### Candidates and ranking

- **Index.**
  - Built lazily on the first opening, from `vault.getMarkdownFiles()` and
    `metadataCache.getFileCache()`:
    - path and stem;
    - aliases via `parseFrontMatterAliases`;
    - headings with their levels;
    - block IDs from `BlockCache.id`;
    - tier;
    - depth (number of `/` in the path).
  - Event listeners (`metadataCache` `changed`/`resolved`, vault
    `create`/`rename`/`delete`) are registered only after the first build and are
    cleaned up through `plugin.registerEvent`.
  - Inbound counts are the number of distinct other notes that link to the note, taken
    from `metadataCache.resolvedLinks`. They are computed lazily and cached until the
    next `resolved` event.
  - Unsaved buffers are visible through the metadata cache, so a note Bryan just created
    can be linked immediately.
- **Tier.** This is a port of `src/native/note_catalog/tiers.rs`:
  - hidden: dot or underscore components, the default library roots `lib/` and `xlib/`,
    and a `.md` next to a same-stem `.pdf`;
  - archive: top-level `done/`, `old_lib/`, `ref_history/`, `lit_review/`, and dated
    satellites such as `2026/20261121_habit.md`;
  - living: everything else.
- **Order.** Sort by tier (living, archive, hidden), then match tier (exact, prefix,
  word-boundary, acronym, substring), then component (alias, stem, path; for prefix
  matches alias and stem rank equally), then path depth (shallower first), then inbound
  count (more first), then `compare_normalized(path)`.
  - Matching uses ASCII-only case folding, Rust's `words()` split, and byte-order string
    comparison, exactly as in Rust.
  - Headings and blocks use the same note-level tie-break, then the line ordinal.
  - Block queries match the block ID only.
- **Cap.** Rank everything, keep the top 50, and show a window that follows the
  selection.
- **Empty `[[` (nearby list).** This list is a prompt-only convenience and is not in the
  vectors. In order:
  1. the source note's parent, through the existing parent-note model in `208`/`682` (if
     no resolver exists, omit these rows and never invent a parent rule);
  2. notes the source links to, in `resolvedLinks[sourcePath]` order;
  3. `workspace.getLastOpenFiles()`.

  Each row is deduplicated, never the source note itself, never archive or hidden, and
  capped at 8. It never adds new usage tracking.

- **Block previews.** Previews are display-only. They are hydrated lazily from
  `vault.cachedRead` for visible rows only, cached by `(path, mtime)`, and trimmed like
  Rust's `block_preview` (text before `^id`, 80 chars). A request token discards stale
  results.

### Link text

> [!decision] link_text = path An accepted note row inserts bob's `insert_target`: the
> vault-relative path without `.md`, as in `[[sase]]` and `[[2026/2026-11-21]]`. This
> matches bob, the Mac panel, and the vectors' `insert` strings exactly.

> [!decision] link_text = shortest An accepted note row inserts
> `app.metadataCache.fileToLinktext(file, sourcePath ?? "", true)`. Headings and blocks
> append `#Heading` / `#^id` to that form. The JS vector replay checks ranges and order
> against bob, and checks `insert` only after mapping through a `path → shortest` table
> in the test. The Mac (S16) keeps bob's path form.

### Keyboard contract

There is one arbiter. Routing order: **IME → link band → recall → host** (stage, picker,
or modal). With `ownKeys: true` the controller listens on the input in the capture
phase. When given a modal `scope`, it also registers a catch-all that returns "handled"
only when it consumed the key, so Esc with the band open never reaches the modal's Esc.
Fix the existing `captureKeydown`, which returns `undefined` today, so the scope handler
reports the real value. Picker hosts (`ownKeys: false`) already offer every key to
`handle.handleKeydown` before their own Enter, Tab, and arrow logic.

| Key                             | LINK band open                                                                                                                                                                     | Band closed (unchanged)                 |
| ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------- |
| ↓ / Ctrl+N, ↑ / Ctrl+P          | Move the selection by row key. It clamps with the 2px edge bump and never wraps.                                                                                                   | Recall, then the host                   |
| Enter (also Ctrl/Cmd/Alt+Enter) | Insert the selected row and keep the prompt open. With zero rows: close the band and insert nothing. In the alias stage: finish the link (append `]]` if missing, caret after it). | Host submit                             |
| Shift+Enter                     | Textarea: a newline (the host's behaviour; the newline ends the link, so the band closes). Input: same as Enter.                                                                   | Host                                    |
| Tab                             | Insert the selected row. With zero rows: pass to the host.                                                                                                                         | Host traversal                          |
| Shift+Tab                       | Pass to the host. The band closes on blur.                                                                                                                                         | Host                                    |
| `#` / `^`                       | On the note stage, with a non-empty query and a selected row: replace the query with `target#` or `target#^` and stay open in the heading or block stage. Otherwise it types.      | Types                                   |
| `\|`                            | On the note stage, with a selected row: insert `target\|` and enter the alias stage. Otherwise it types.                                                                           | Types                                   |
| Esc / Ctrl+[                    | Close the band only: one layer per press. Remember `(opener offset, query)`.                                                                                                       | Recall back-out, then the host's ladder |
| `q`                             | Types (per `decisions:q-and-ctrl-bracket-mirror-esc`)                                                                                                                              | Types                                   |
| ← → Home End                    | Move the caret. The band closes if the caret leaves the link.                                                                                                                      | Host                                    |
| Backspace through `[[`          | The band closes.                                                                                                                                                                   | Host                                    |
| Click a row                     | `mousedown` calls `preventDefault()` so focus never moves (S6 commits on blur). `click` inserts.                                                                                   | —                                       |
| Hover                           | Hover tint only; never changes the selection.                                                                                                                                      | —                                       |

- **Held Enter.** An Enter that accepted a row arms a swallow. It eats repeated Enter
  keydowns until Enter's `keyup`, so a held key cannot accept and then save. Host guards
  (the Alt+F repeat swallow, the successor picker's Enter guard, the decay card's
  chord-release swallow, the Record Card repeat swallow) stay authoritative above the
  band. The band never claims Alt+F.
- **Dismissal memory.** After Esc, the band stays closed while the link's opener and
  query are unchanged. Host repaints, recall back-out, caret moves, and focus changes do
  not reopen it. Any edit that changes that link's text reopens it, which matches IDE
  suggest conventions.
- **Recall suppression.** While the band is open, the USED BEFORE band is hidden and its
  keys belong to the band. When the band closes, recall re-renders against the current
  draft.

### Visual design

The LINK band renders in the recall band's slot. Where a host has no recall, it renders
in a host-provided slot directly under the field. It grows downward only, so the field
and everything above it never move. Its height is fixed for the whole open session:

- one header line, five 24px rows (roomy) or three 22px rows (dense, S6), and one detail
  strip;
- empty slots keep the height, exactly like recall's pinned slots;
- stage changes and re-filtering swap content without animation;
- opening fades in over 120 ms (opacity only under `prefers-reduced-motion`).

```text
 Move to Next — what did you work on?
 ┌──────────────────────────────────────────────────────────────────────┐
 │ Outlined the post in [[sase_bl▏                                      │
 └──────────────────────────────────────────────────────────────────────┘
  🔗 LINK · notes                                                1 of 3
 ▎ ▤  sase_blog_zettel
   ▤  sase_blog                                          ref/
   ▤  sase_blog_zettel_waves                             done/  Archived
  ·
  ·
  ↵ [[sase_blog_zettel]]                     ⇥ insert  # heading  ^ block  | alias
```

```text
  🔗 LINK · sase › headings                                      2 of 9      ← after typing # on a row
   #  Overview                                                        H2
 ▎ #    Restart readiness                                             H3
  ↵ [[sase#Restart readiness]]                                  ⇥ insert

  🔗 LINK · sase › blocks                                        1 of 4
 ▎ ¶  Restart readiness checklist for 1n6…              ^ref-sase-1n6-11-4
  ↵ [[sase#^ref-sase-1n6-11-4]]                                 ⇥ insert

  🔗 LINK · nearby                                                          ← empty [[
 ▎ ▤  projects/sase                                                 parent
   ▤  sase_blog_zettel                                              linked
   ▤  2026/2026-11-21                                               recent

  🔗 LINK · notes
          No note matches “sase_bx” — keep typing, or esc
  ↵ closes this list · nothing is inserted                       esc close

  🔗 LINK · alias
          Type the label — ↵ finishes the link
  ↵ [[sase_blog_zettel|my zettel]]
```

**Row grammar.** This mirrors the Mac `CompletionRowContent`, so both surfaces read
alike.

- **Icon.** Lucide via `setIcon`, muted:
  - `file-text` for a note;
  - `calendar-days` for daily and period notes (`YYYY/YYYY-MM-DD`, root `YYYY`,
    `YYYYMM`, `YYYY-Www`, `YYYY-Qn`);
  - `book-open` for `ref/`;
  - `hash` for a heading, indented 8px per level past 1, capped at 3 levels;
  - `pilcrow` for a block.
- **Primary text.** Stem, alias, heading, or block preview. Every matched span is
  emphasised (accent color, weight 600) with safe text nodes, never `innerHTML`.
- **Folder.** The muted parent folder, middle-truncated so the filename always survives.
  Root notes show nothing.
- **Capsule.** A right-aligned quiet capsule appears only where it changes the meaning:
  - `Alias`, where the secondary text reads `→ stem`;
  - `H2`;
  - a mono `^id`;
  - `Archived` or `Hidden`, which also dims the row to 60% opacity;
  - nearby reasons: `parent`, `linked`, `recent`.

**Header and detail strip.**

- The header is `🔗 LINK · <stage crumb>`, with a right-aligned `n of N`. It uses the
  same type ramp as `USED BEFORE`.
- The detail strip shows `↵` plus the **exact resulting link**, from `[[` to `]]` and
  including any preserved `#subpath|alias`, in `--font-monospace`.
- Right-aligned keycap hints reuse the recall keycap style:
  - note stage: `⇥ insert  # heading  ^ block  | alias`;
  - heading and block stages: `⇥ insert`;
  - dense: `⇥ insert` only.

**Selection and theming.**

- Selection uses the recall treatment: a 2px `--interactive-accent` left bar plus a
  `color-mix` accent fill at 10%.
- Hover is a 5% tint.
- Only theme variables are used (`--interactive-accent`, `--background-modifier-border`,
  `--radius-m`, `--text-faint`). Raise contrast under `prefers-contrast: more`. The band
  must be legible in light, dark, and high-contrast themes.

**Quiet states.**

- Same-note `[[#` with `sourcePath: null`: "Same-note links need one home note here —
  type a note name first".
- A heading stage whose target does not resolve: "No note named “foo”".
- If the index cannot load or the matcher throws, show no band and no notice. Never
  replace a host's write error or retained retry.

**Unresolved-link hint.** This implements requirement A4.

- When the band is closed and the draft holds a complete `[[…]]` whose target resolves
  to nothing, one faint single line appears under the band slot, for example:
  - `⚠ [[sase_blgo]] — no such note`;
  - `— no heading “Foo” in sase`;
  - `— no block ^x in sase`.
- A trailing `+1 more` appears when several links are unresolved.
- Resolution uses `metadataCache.getFirstLinkpathDest(linkpath, sourcePath ?? "")` plus
  the resolved file's headings and blocks.
- It is debounced 250 ms after typing settles.
- It is informational only: Bob never refuses Bryan's text, and Enter still submits.

**Classes.** Use `bob-link-band-*` and `bob-link-hint`, defined in nav `styles.css` next
to the recall band block. Reuse `--bob-log-recall-row-height` and the keycap classes.

### Accessibility

- Use the editable-combobox pattern on inputs:
  - the input gets `aria-autocomplete="list"`, `aria-expanded`, `aria-controls`, and
    `aria-activedescendant`;
  - the list gets `role="listbox"`, and each row gets `role="option"` and
    `aria-selected`.
- Focus always stays in the field.
- A polite live region announces "Link 2 of 8: sase_blog_zettel".
- The reflection `<textarea>` keeps textbox semantics. It carries `aria-controls` and
  `aria-activedescendant`, without a role stamp.
- Clear every ARIA attribute on close and on `detach`.

### Reliability rules (release criteria for every phase)

- **R1.** Never destroys text after the caret. Authored `#subpath` and `|alias` survive
  retargeting.
- **R2.** Never submits while the band is open. A held Enter cannot accept and then
  save.
- **R3.** One native undo restores the typed query.
  - Insert by selecting the replacement range and calling
    `document.execCommand("insertText", false, text)`.
  - Fall back to `setRangeText` plus an `input` event, marked `insertReplacementText`.
  - Never use value assignment for link insertion.
- **R4.** Survives:
  - host repaints (`rebind`), stage exits, and modal close;
  - plugin unload, where `detach` is idempotent and the API is replaced on unload like
    `logRecall`;
  - pop-out windows: use `inputEl.ownerDocument` and `ownerDocument.defaultView` for
    events, `execCommand`, and live regions.
- **R5.** Fails open:
  - no nav plugin, an old nav, or a missing `proseField` means the prompt works
    unchanged;
  - a thrown matcher or index means no band and the text stays intact;
  - every API call is wrapped in `try`/`catch`.
- **R6.** Stays quiet in identifier fields, code spans, `\[[`, pasted text, IME
  composition, and non-collapsed selections.
- **R7.** Never writes the vault: no note creation, no block-ID minting, no navigation.
  Candidate lookup is read-only.
- **R8.** Privacy: paths, aliases, headings, and previews are never logged, persisted,
  or sent anywhere.
- **R9.** Performance: on a synthetic 10k-note index, the keystroke-to-rows p95 is ≤ 8
  ms in node, and the first build is ≤ 40 ms. Enforce both in `test:perf`.
- **R10.** No drift:
  - the vectors are byte-identical in both repos;
  - every prose surface is explicitly wired with a `sourcePath`;
  - the role lint (`prose-lint`) fails on an undeclared text control.

### API

`bob-navigation-hotkeys.api.proseField`, version 1. The root API stays at version 3;
feature-detect the namespace.

```js
const nav = app.plugins.plugins["bob-navigation-hotkeys"]?.api;
const prose =
  nav && nav.proseField && Number(nav.proseField.version) >= 1 ? nav.proseField : null;
const handle = prose
  ? prose.attach({
      inputEl, // HTMLInputElement | HTMLTextAreaElement (required)
      bandHost, // element directly under the field (required)
      sourcePath, // string | null | () => string | null  — resolves [[# / [[#^ and the nearby list
      recall, // null | "work" | "schedule" | "rotten" | "cancel" (USED BEFORE kind)
      links, // default true
      mode, // "inline" (default) | "field"
      minOffset, // () => number; openers before it never open
      density, // "roomy" | "dense"
      ownKeys,
      scope, // as logRecall: true = controller listens itself (capture phase + modal scope)
      pinnedSlots,
      arrowKeys,
      maxRows, // recall pass-through
    })
  : null;
// handle: handleKeydown(event) → boolean, backOut() → boolean, isLinkOpen(), isRecalling(),
//         rebind({ inputEl?, bandHost? }), setSourcePath(p), setKind(k), setPinnedSlots(n), detach()
```

- **Composition.** `attachProseField` creates two child hosts inside `bandHost`, one for
  recall and one for the link band. It attaches the existing `attachLogRecall` to the
  recall child with `ownKeys: false`, and owns all key routing itself.
- **`logRecall.attach(opts)`.** This stays byte-for-byte unchanged unless
  `opts.links === true`. In that case it returns
  `attachProseField({ ...opts, recall: opts.kind })`. These are additive options;
  `logRecall` stays at version 1. A host passing `links` while running against an older
  nav simply gets recall only.
- **Explicit consumers.** These are hosts whose own listeners can run before the
  input's, such as the Record Card's document capture listener and card scope ladders.
  They must call `handle.backOut()` as the first rung of their Esc/`q`/Ctrl+[ ladder.
  They must also offer Enter, Tab, arrows, and Ctrl+N/P to `handle.handleKeydown` before
  their own handling.

## Phases

General rules for every bob-plugins phase:

- Open bob-plugins with `/sase_repo` and read its `AGENTS.md`.
- Edit only `src/` fragments and add new ones to `src/fragments.json`. Keep every
  fragment at or under 1000 lines (nav's `480`, `350`, `470`, `570`, and `700` are
  already at the cap, so put new code in new fragments).
- Add new test files to the `test:unit` list (or `test:perf`).
- Run `npm run build`, `npm test`, and `npm run validate`, plus `npm run docs:check`
  when docs change.
- Bump the changed plugins' manifest versions (minor) and run `npm run docs:gen`.
- Finish with `bob plugins sync`, following `docs/develop/deployment.md` if sync
  refuses.
- If phases running in parallel touch the same plugin, rebase and rebuild `main.js`;
  never hand-merge generated output.

### Phase cli-link-fixes — bob-cli

1. **Unclosed range.** In `src/native/capture_links.rs`:
   - `note_field` and `subpath_field` end an unclosed link's replacement at the cursor,
     never at the line end;
   - synthesize `]]` unless the next character after the cursor is `#` or `|`.

   Do not change `scan_links`, `wikilink_spans`, or `is_protected_markup_range`:
   capture's protected-markup semantics and span highlighting stay exactly as they are.

2. **Authored subpath and alias survive.**
   - In the target stage of a link with a heading, block, or alias component, replace
     only the target component. `cursor_after` still lands after an existing `]]`.
   - Alias-match candidates drop their `|Alias` suffix when the link already has an
     authored alias.
   - Regression: `See [[sa▏#Overview|our plan]] ok` must keep `#Overview|our plan`.
3. **Tie-break.**
   - `note_candidates` and the vault-scope `heading_candidates` / `block_candidates`
     order by tier, then match rank, then path depth (ascending), then inbound count
     (descending), then `compare_normalized`. Heading and block candidates then order by
     ordinal.
   - Compute inbound counts in `NoteIndex::read`, which already reads every note:
     - scan each note's wikilinks with the existing scanner;
     - resolve each one by exact path or `insert_target`, else by a unique stem
       (case-insensitive);
     - count distinct source notes, excluding self.
   - Add a test-only constructor that builds a `NoteIndex` from entries, so the vectors
     need no filesystem for ranking.
4. **Vectors.** Add `tests/fixtures/wikilink_complete/vectors.json`, modelled on
   `tests/fixtures/log_recall/vectors.json`. It contains `version: 1`, `source`, and
   three sections, all encoding-agnostic (strings, never offsets):
   - `fields`: `before` and `after` caret strings, plus
     `expect: null | {context, scope, target?, query, replaced, accept?: {insert, result}}`,
     where `result` marks the caret with `▏`.
   - `tiers`: path plus optional `pdf_siblings`, mapped to `living | archive | hidden`,
     using the default `lib`/`xlib` roots.
   - `rank`: one note table (path, aliases, headings with levels, block IDs, inbound)
     and cases giving `before`/`after` and an optional `current_note`, mapped to the
     ordered top-N `insert` strings.

   Cover at least these cases:
   - `Discuss [[pla▏ tomorrow` (trailing prose survives);
   - `[[pla▏]]` (the closer is reused once);
   - `[[Old▏#Overview|our plan]]`;
   - an alias row inside an authored `|label`;
   - `` `[[x▏` `` and `\[[x▏`, both inert;
   - `![[` embeds;
   - `[[#He▏`, `[[#^bl▏`, `[[##He▏`, `[[^^bl▏`;
   - `[[note#He▏ading]]` and the unclosed `[[note#He▏ more`;
   - a nested `[[a [[b▏`;
   - two links on one line;
   - emoji and RTL neighbours;
   - caret in the alias (`null`);
   - root-vs-nested ties (the `[[sase` regression: a root `sase_*` note ranks ahead of
     `chat/` and `plans/`);
   - inbound-count ties;
   - archive and hidden ordering;
   - empty-query order.

   A Rust test asserts the file. Copy it into bob-plugins in `link-engine`.

5. **Docs.** Update `docs/capture.md`'s wikilink completion contract: the field end
   rule, authored subpath and alias preservation, and the ranking tie-breaks. The JSON
   shape is unchanged, so the commit is `fix(capture): …` with no `!`. Never edit
   `Cargo.toml`'s version or `CHANGELOG.md`.
6. **Decision record.**

   > [!decision] decision_record Use `/sase_memory_write` to add the strand
   > `sase/memory/decisions/prose-fields-complete-wikilinks.md` with
   > `origin: requested`, `metadata.status: accepted`, and `decided: 2026-11-21`, then
   > run `sase memory init`. Content:
   >
   > - **Applies to:** bob-plugins prose prompts and cards, Bob Mac Capture prose
   >   fields, and bob-cli `capture-complete`.
   > - **Claim:**
   >   - `[[` completes in every Bob-owned field that writes Markdown prose into the
   >     vault, through nav's one `proseField` controller. Identifier and filter fields
   >     never complete.
   >   - While the band is open, Enter and Tab finish a link and never submit. With zero
   >     rows, Enter closes the band and inserts nothing.
   >   - Completion only inserts text: it never creates notes, mints block IDs, or
   >     writes the vault.
   >   - Obsidian ranks locally from `metadataCache`, with policy pinned to
   >     `capture_links.rs` by shared vectors.
   >   - The Mac asks bob through `--links-only`.
   >   - The inserted text form follows `link_text`.
   >   - The Mac event-note editor is excluded while gcal escapes `[`.
   > - **Why:** cite the research ref and this plan. Rejected alternatives:
   >   - per-prompt implementations, which drift (for example
   >     `recordCardLinkCandidates`);
   >   - spawning bob per keystroke in Obsidian (0.4–1.2 s);
   >   - a caret-anchored popover;
   >   - Tab-to-leave;
   >   - note creation from prompts.
   > - **Cost:** policy changes land in Rust and are replayed in JS; each new prose
   >   prompt must opt in and declare its role.
   > - **Reopens when:** a prose field needs literal `[[` text, or bob ships an
   >   in-process index Obsidian can call.
   >
   > Link `[[decisions:mac-capture-is-a-thin-client]]` and
   > `[[decisions:q-and-ctrl-bracket-mirror-esc]]`.

   If `decision_record` is off, skip this step and follow `/sase_memory_write`'s
   declined-change rule.

Acceptance:

- `just all` is green.
- `bob capture-complete -c 9 -f json -- 'Read [[da because of the outage'` returns a
  replacement `{7,9}`.
- The `[[sa▏#Overview|our plan]]` probe keeps the subpath.
- The vectors test passes.

### Phase link-engine — nav

All work is in new fragments, for example `186-wikilink-field-model.js`,
`186a-wikilink-rank.js`, `186b-wikilink-index.js`, and an exports fragment after `724`.
The functions must be pure where stated and never throw.

- **Field model.**
  - `wikilinkFieldAt(text, caret, { minOffset, mode })` returns
    `{ context, scope, target, query, replaceStart, replaceEnd, linkStart, closed, appendClose, hasAuthoredAlias }`
    or `null`.
  - `wikilinkAliasAt(text, caret)` covers the alias hint state.
  - `applyWikilinkInsert(text, field, insert)` returns
    `{ start, end, replacement, caret }`.
  - `completeWikilinksIn(text)` lists the complete links for the unresolved hint.
- **Ranker.**
  - `wikilinkTier(path, ctx)` and the match scoring follow the order above.
  - `rankWikilinkRows(entries, field, opts)` returns `{ rows, total }`. Each row carries
    `key`, `kind`, `path`, `stem`, `folder`, `tier`, `alias?`, `heading?`, `level?`,
    `blockId?`, `matchKind`, `matchRanges`, `insert`, and `reason?`.
  - The `link_text` decision is applied at `insert`.
- **LinkIndex.**
  - `createWikilinkIndex(plugin)` returns a lazy singleton per plugin instance, with
    `entries()`, `inbound(path)`, `nearby(sourcePath)`,
    `resolve(linkpath, subpath, sourcePath)`, `blockPreview(path, id)`, and `dispose()`.
  - It is wired into the plugin lifecycle without building at load.
- **Tests.**
  - Copy bob-cli's vectors byte-identically to
    `scripts/fixtures/wikilink-complete-vectors.json` with the same `source`.
  - Add `scripts/test-navigation-wikilink-model.cjs` to replay every vector. Also cover
    JS-only `minOffset`, field mode, the opening rules as pure predicates, surrogate
    pairs, and the nearby list against a fake `metadataCache`.
  - Add `scripts/test-navigation-wikilink-index-performance.cjs` (R9) to `test:perf`.

### Phase link-band — nav plus a block-id-prompt pilot

- **Controller and band.** Put these in new fragments, for example
  `187-prose-field-controller.js` and `187a-prose-field-band.js`.
  - Implement everything in Keyboard contract, Visual design, Accessibility, and R1–R9.
  - Add `api.proseField` v1 in a new fragment called from `createDependencyNavApi`,
    which is in `480` (that file is at the cap, so the namespace factory goes in the new
    fragment).
  - Make `logRecall.attach` delegate when `links: true`.
  - Replace the API on unload like `logRecall`.
- **Recall coexistence.** Recall moves into its child host. Verify that every existing
  recall-only surface is visually and behaviourally unchanged when `links` is not
  passed. The existing log-recall tests stay green.
- **Pilot S1.** In bip `090-prompt-modals.js`, pass `links: true` and the right
  `sourcePath` for both the link-task and task-link-open paths.
- **Real-Obsidian spike.** Use the isolated e2e harness pattern in bob-plugins
  `scripts/obsidian-*-e2e.cjs`. Before marking the phase done, prove in S1 and in a
  throwaway `<textarea>` host:
  - one Ctrl+Z restores the typed query after acceptance;
  - IME composition never opens or accepts;
  - Esc with the band open closes only the band, not the modal;
  - Ctrl+[ does the same;
  - a held Enter never accepts and then submits.

  Record the results in the phase bead notes. If `execCommand` undo fails, ship the
  fallback and note the undo gap in the docs.

- **Recall-history check.** This is the 10-minute check from the research. Confirm that
  `[[`-bearing phrases entering the recall catalog through `noteUsed` behave sensibly
  after renames. If not, file a task bead with `/sase_new_task`; do not redesign recall
  here.
- **Docs.**
  - `docs/develop/plugin-apis.md`: the `proseField` v1 row and prose, plus `logRecall`'s
    additive options.
  - `docs/bob-navigation-hotkeys.md`: a new "Wikilink completion in prompts" section
    with the key table, states, and the scope rule.
  - `docs/block-id-prompt.md`: one line for S1.
- **Tests.** Use the harnesses (`navigation-hotkeys-harness.cjs`, `modal-harness.cjs`)
  to cover:
  - band state transitions (open, filter, drill `#`/`^`, alias `|`, accept, zero-row
    Enter, Esc dismissal memory, reopen on edit);
  - key routing with and without recall;
  - the held-Enter swallow;
  - paste and IME suppression;
  - `rebind` preserving state;
  - `detach` idempotence;
  - ARIA attributes;
  - the unresolved hint debounce.

  Add an HTML preview script, like `scripts/preview-*.cjs`, that renders the band's
  states (notes, headings, blocks, nearby, no match, alias, dense, archived row,
  unresolved hint) in light and dark themes for visual review.

### Phase modal-prompts — nav plus mail

- **S2–S6.** Pass `links: true` and the `sourcePath` from the allowlist table. Preserve
  each host's guards:
  - S4 and S5: the Alt+F and Ctrl+Alt+F swallow;
  - S5 in the decay card: chord-release key swallowing and the repeated-Enter ignore;
  - S6: the 350 ms Enter guard and blur commit. S6 rebuilds its input on every
    `renderResults`, so keep the band state with `rebind` where the element is replaced,
    or reattach with the draft preserved.
- **S12 and S13.** Attach `proseField` explicitly (`recall: null`, `sourcePath: null`)
  with a band host under the field.
  - S13 has no keydown handler today. Enter continues to do nothing there while the band
    is closed; do not add submit-on-Enter in this phase.
- **Close keys.** Close the Ctrl+[ gaps on S4, S5 (including the decay card Stay
  reason), S6, S12, and S13: Ctrl+[ must mirror Esc's current layer after the band
  layer, per `decisions:q-and-ctrl-bracket-mirror-esc`.
- **Docs.** Update the user docs for the affected surfaces, including
  `docs/notebook-gestures.md` and `docs/bob-mail.md`.

### Phase task-card-prompts — nav

- **S7–S10.** `attachPickerLogRecall` in `185-log-recall-picker.js` passes `links: true`
  and `sourcePath: () => picker.filePath ?? <single link-session path> ?? null`.
  - Picker back-out (`pickerLogRecallBackOut`) and `offerPickerLogRecallKey` already
    route Esc, Ctrl+[, and all keys first. Verify Enter, Tab, and arrows reach the band
    before `openItemAtIndex`, S10's Tab field cycling, and S9's Alt+P (which keeps
    priority).
  - S10 rebuilds its inputs on `renderAll` and switches handles on focus. Keep band
    state across those switches.
- **S11.** When the scheduled value stage is active, attach `proseField` to the picker's
  main filter input with `recall: null` and
  `minOffset: () => end of the typed date token plus whitespace`, computed from
  `matchTypedSchedulePrefix`. With no valid date prefix, return `Infinity`.
  - Detach when the stage changes, through the same wrapper pattern as
    `installPickerLogRecallCleanup`.
  - The value-stage keydown offers keys to the handle before date-row navigation and
    before Enter, Shift+Enter, Ctrl+Enter, and Ctrl+R.
  - With the band closed, today's typed-row behaviour is unchanged.
- **Docs.** Update `docs/task-card.md`'s scheduled stage and reason sections.

### Phase intention-card — ledger-tools

- **S14.** Feature-detect nav's `proseField`. On stage open, attach it to the intention
  `<input>`, the summary `<input>`, or the reflection `<textarea>`, with a band host
  inside the stage under the field and `sourcePath: () => modal.subjectPath` (which
  follows `[`/`]` travel). Detach on stage close and on modal close.
- **Repaints.** `intentionCardEditorPaintAll` rebuilds the stage on paint, so call
  `handle.rebind({ inputEl, bandHost })` after every paint. The band re-renders
  identically, with no fade replay and with the caret, selection key, and dismissal
  memory preserved.
- **Ladder and keys.** `handle.backOut()` is the first rung of the card's back-out
  ladder (`bobCardWireCloseKeys`, `onEscapeKey`, `intentionOnKeyDown`).
  `intentionOnKeyDown` offers keys to the handle first. In the reflection field,
  Shift+Enter is a newline, Enter saves only while the band is closed, and
  Ctrl/Cmd+Enter (save and close) is consumed by the band while it is open.
- **Unchanged behaviour.** The pending-write swallow and the multi-line paste collapse
  in intention inputs are unchanged. `bob periods update` writes are unchanged, and YAML
  quoting of `[[` is already handled by bob's `quote_yaml`.
- **Docs.** Update `docs/intention-card.md`.

### Phase record-card — ledger-tools

- **S15 typed stages.** Attach `proseField` to the text `<textarea>`, the suggest
  stage's free-text input, and list-item inputs, with `sourcePath` set to
  `modal.recordPath`. Date, number, and URL stages stay literal.
- **Per-keystroke repaint.** `recordTypedInput` calls `paintRecordCard` on every input,
  which creates a new input element. Either stop recreating the input for typed stages,
  or call `rebind` after each paint.
  - Requirement: no flicker, no lost IME state, and the caret, band state, and undo
    history all survive.
  - Prefer not recreating the input. Native undo cannot survive element replacement.
- **Ladder and keys.** The document capture-phase Esc/`q`/Ctrl+[ listener calls
  `handle.backOut()` first. The typed-stage key handler offers keys to the handle
  before:
  - the repeat-swallowed Enter save;
  - Tab (`recordTab`);
  - list Enter (add item);
  - suggest-stage arrow cycling.
- **Link stage convergence.** The link stage becomes a field-mode consumer
  (`mode: "field"`) of the same engine. Arrows move band rows, and Enter or Tab accept
  into the draft. A second Enter saves once the band is closed.
  - Retire `recordCardLinkCandidates` and the link branch of `recordMoveSuggest`. This
    removes the brackets-in-filter bug.
  - Keep `recordCardParseLinkDraft`'s accepted forms, and extend them to
    `[[path|Alias]]` if they do not already accept it.
  - Without nav, the stage is plain typing.
- **Docs.** Update `docs/record-card.md`. Its claim that suggestions resolve relative to
  the record becomes true through `sourcePath`.

### Phase cli-links-only — bob-cli

Add two options to `bob capture-complete`, sorted alphabetically, each with a short
alias:

- **`-L, --links-only`.** Treat TEXT as plain prose.
  - Only wikilink fields complete: note, heading, block, `[[#`, `[[##`, `[[^^`. Every
    other caret position returns an empty success.
  - It never parses capture grammar: no routes, markers, records, Work Log phrases, or
    `((`.
  - The Work Log block-link suppression does not apply.
- **`-S, --source-note <PATH>`.** The vault-relative note that same-note `[[#…` and
  `[[#^…` resolve against. It requires `--links-only`. Without it, same-note fields
  return an empty success.

Requirements:

- The JSON shape is unchanged (`schema_version` 1, same `grammar`), so the commit is
  `feat(capture): …`.
- Write excellent `--help` text, per `cli_rules`.
- Run `just docs-gen`, and update the `docs/capture.md` `capture-complete` section.
- Add tests: prose drafts with `@`, `#`, `+`, `x`, and `=` stay literal; a same-note
  field without `-S` is empty.

### Phase mac-pull-in — bob-mac-capture

Read its `AGENTS.md`: never land uncompiled Swift; run `Scripts/macos-ci-check`; watch
the exact-SHA macOS run to green with `/sase_monitor`.

- **Data.**
  - When the Pull-In reason draft contains `[[`, every text or caret change calls
    `BobProcessClient` with `capture-complete --links-only`, plus
    `--source-note <task note>` when the row carries a note path, `--cursor`, and
    `--format json -- <reason>`. This is a cheap pre-filter only; bob decides whether
    the caret is in a field.
  - Debounce 60–80 ms. A generation token discards stale responses, and the previous
    rows stay visible while a request is in flight, with no spinner flash.
  - On `unsupportedOption("--links-only")`, disable the feature for the session. Any
    other error means no band.
- **Presentation.**
  - The LINK band replaces `CapturePullInRecallBand` in its slot with the same anatomy:
    a hairline, an SF Symbol `link` header reading `LINK`, trailing keycaps, 22pt rows
    rendered with `CompletionRowContent`, and `.caption.monospacedDigit()` metadata.
  - A detail line shows `↵` plus the exact resulting link, built from bob's replacement.
  - Selection is by row key, and hover never selects.
- **Keys.** Through `CaptureKeyCommandRouter`, with the band open:
  - ↑/↓ and Ctrl+N/P move the selection;
  - Return and Tab insert bob's replacement at bob's byte range, converted with the
    existing byte↔`String.Index` helpers, and place the caret at `cursor_after`;
  - zero rows: Return closes the band and never confirms the Pull-In;
  - Esc and Ctrl+[ close the band only;
  - insertion goes through the field editor so `NSUndoManager` records one step.
- **Exclusion.** The event-note editor stays excluded.
- **Docs.** Update the Mac wikilink-completion doc and the Pull-In doc, then run
  `just docs-check`.

### Phase prose-lint — bob-plugins

- **Markers.** Every text-control creation site in `plugins/*/src/*.js` (and in
  hand-edited `main.js` for plugins without `src/`) carries a
  `// bob-text: <role> <id-or-why>` comment within the 3 lines above it.
  - Sites to scan: `createEl("input"`, `createEl("textarea"`, `.addText(`,
    `.addTextArea(`, `.addSearch(`, `new TextComponent(`, `new TextAreaComponent(`,
    `createElement("input"|"textarea")`, and `contenteditable`.
  - Roles:
    - `prose S<n>`;
    - `literal <why>`;
    - `mixed <why>`, for a shared input whose prose stages are wired elsewhere, such as
      the `FilteredPickerModal` filter used by S7–S11;
    - `none <why>`, for checkbox, radio, range, date, color, and file inputs.
- **Contract test.** `scripts/fixtures/prose-fields.json` lists S1–S15, each with its
  plugin, creation file, and wiring file. The new test checks that:
  - every `prose` marker's ID is listed;
  - every listed ID has a marker;
  - each wiring file mentions `links: true` or `proseField`;
  - no creation site lacks a marker.
- **Docs.** Add a short "Prose fields" rule to `docs/develop/architecture.md` (or
  `plugin-apis.md`): new prose prompts attach `proseField` with a `sourcePath` and
  declare `bob-text: prose`.
- **Walkthrough.** Run one isolated real-Obsidian e2e walkthrough over S1, S10, S11, S14
  (reflection), and S15 (text and link stages). In each, type `[[`, filter, drill `#`,
  accept, undo once, check Enter safety and Esc layering, and confirm the written note
  text. Attach screenshots as artifacts with `/sase_artifact`.

## Non-goals

- **`((` task-link completion in prompts.** Accepting an ID-less task mints a block ID,
  which is a vault write from inside a modal. Revisit after living with `[[`.
- **Side effects from completion.** No note creation, block-ID minting, or navigation.
- **Caret-anchored popovers.** The band is the design everywhere.
- **The Mac event-note editor.** It stays out until gcal's private-note escaping
  contract changes under its own decision.
- **Capture grammar changes.** No changes to `capture-parse` spans or to the
  protected-markup scanning of unclosed links.
- **Ranking exceptions.** Letting an exact archive title beat living matches is
  deferred.

## Risks

- **`execCommand` undo.** It is deprecated and Electron behaviour could regress. The
  spike proves it, the fallback keeps the text correct, and the docs record any undo
  gap.
- **Obsidian Scope ordering.** Esc could close the modal before the band sees it.
  Mitigations: the input capture listener, the corrected scope catch-all return value,
  and an e2e assertion.
- **Element-replacing hosts** (Record Card, Intention Card, S6, S10). Mitigations:
  `rebind` with state keyed by text offsets rather than elements, and the Record Card
  preferring a stable input.
- **Rust and JS drift.** Mitigations: byte-identical vectors, both test suites, and a
  `source` field asserted on both sides.
- **Release train.** The Mac needs a released bob with `--links-only`, so it fails open
  until the daily train ships it.
