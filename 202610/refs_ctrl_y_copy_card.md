---
tier: epic
status: done
title: Bob Refs ⌃Y Copy card
goal: 'In Bob Mac Capture''s Refs panel, ⌃Y turns the inspector column into a Copy
  card where one keypress copies a precise handle for the selected reference (marker
  ID, title, wiki link, sources, paths, the PDF file, the reading task, research ref,
  or one or all of your notes). The copy is checked, Refs hides so the next ⌘V lands
  where you came from, and a glass HUD confirms what was copied. A highlight picker
  and ⌥ alternates follow.

  '
decisions:
  return_to_capture:
    ask: When Refs was opened over a Capture draft, should a copy reopen Capture so
      ⌘V pastes into the draft?
    default: true
    why: Opening Refs from Capture is a lookup for the draft, so the paste belongs
      there
    answer: true
phases:
- id: bob-marker-id
  title: bob-cli serializes the frontmatter id as marker_id
  depends_on: []
  size: small
  description: 'bob-marker-id: make `bob ref list/show/find` JSON rows carry the frontmatter
    `id` as an additive `marker_id` field (null when absent), with tests and docs/ref.md
    updated.'
- id: refs-copy-core
  title: RefsCore decoding, copy catalog, formats, key map, and card state machine
  depends_on: []
  size: medium
  description: 'refs-copy-core: in bob-mac-capture''s portable RefsCore, decode marker_id,
    research_ref, annotation block_id/link, and own_notes; build the pure fixed-letter
    copy catalog, payload formatters, card layout capacity, key map, and the RefsCopyCard
    state machine, all Linux-tested.'
- id: refs-copy-components
  title: App building blocks for the Copy card and HUD (not yet wired)
  depends_on:
  - refs-copy-core
  size: medium
  description: 'refs-copy-components: in the app target, add the checked pasteboard
    write, the single-flight exact-path ref-show cache, the RefsCopyCardView, and
    the glass copy HUD with design fixtures, without wiring ⌃Y yet.'
- id: refs-copy-integration
  title: Wire ⌃Y, the card lifecycle, hide-on-copy, ⌘K, footer, and README
  depends_on:
  - bob-marker-id
  - refs-copy-components
  size: medium
  description: 'refs-copy-integration: route ⌃Y and every card key, own the card lifecycle
    in RefsPanelModel, copy then hide Refs (or return to Capture) and show the HUD,
    fold ⌘K''s copy items into Copy…, update the footer, the Esc ladder, and the README,
    and add tests.'
- id: refs-copy-highlight-picker
  title: H highlight picker layer
  depends_on:
  - refs-copy-integration
  size: medium
  description: 'refs-copy-highlight-picker: drop `-c` from the inspector''s ref-show
    fetch and add the H layer that reaches every highlight and standalone note (digits
    for the visible window, arrows for the rest, A for all highlights).'
- id: refs-copy-alternates
  title: ⌥ alternates and the Obsidian link row
  depends_on:
  - refs-copy-highlight-picker
  size: medium
  description: 'refs-copy-alternates: holding ⌥ swaps previews in place for alternate
    payloads (titled wiki link, ~ path, absolute note path, quote only, narration
    file), and add the O Obsidian URI row.'
proposed_by: bbugyi200.apollo.research.19.linker.w0
decided_by: auto
create_time: 2026-10-14 09:32:29
status: wip
bead_id: bob-cli-7k
---

- **PROMPT:** [prompts/202610/refs_ctrl_y_copy_card.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/refs_ctrl_y_copy_card.md)
- **BEAD:** [bob-cli-7k](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-7k/README.md)

# Plan: Bob Refs ⌃Y Copy card

## Context

The design follows the consolidated research report
`research:202610/refs_panel_ctrl_y_copy_card/refs_panel_ctrl_y_copy_card.md`. Read it
with
`sase artifact read research:202610/refs_panel_ctrl_y_copy_card/refs_panel_ctrl_y_copy_card.md "<why>"`
when you need its measurements or rationale. Bryan accepted every requirement it
recommends (A1–A11). The open questions are settled as follows:

- **Default row:** the Marker ID, or the Wiki link when there is no ID.
- **After a copy:** Refs hides. The plan decision `return_to_capture` only chooses
  whether Capture comes back afterwards.
- **Notes:** a note payload keeps its `— Page · [[block link]]` provenance line.
- **Surface:** an in-panel card, not an `NSMenu`.

This plan is the single source of truth for the contract below. Every phase implements
its slice of it exactly as written, and changes behaviour only when a phase section says
so.

Baseline: bob-mac-capture `44051a0`, bob-cli `ae9e6ef`.

Repositories:

- `bob-marker-id` changes **bob-cli**, the primary repo.
- Every other phase changes the linked repo **bob-mac-capture**. Open it with your
  `/sase_repo` skill (`sase repo open bob-mac-capture -r "<why>"`), use only the path it
  prints, and read that repo's `AGENTS.md` before editing.
- `decisions:mac-capture-is-a-thin-client` governs the app side. Every payload is either
  a bob-authored string copied verbatim or a deterministic presentation format of bob
  fields. The app never reads frontmatter, never parses Markdown or the PDF for
  payloads, and never derives the marker ID from the PDF stem.

## The design contract

### What the user sees

1. A reference is selected in Refs. **⌃Y** turns the inspector column into the **Copy
   card**. The selected list row stays visible beside it, so the target is never in
   doubt. Below the 760 pt inspector threshold, the card covers the list column instead.
2. Every visible row has a fixed keycap, a label, and a literal one-line preview of the
   exact text that will be copied.
3. **One more keypress copies.** The pasteboard write is checked. Refs hides
   immediately, so key focus returns to the app you came from, and a small glass **HUD**
   confirms the copy (`✓ Copied Marker ID` over the value). With `return_to_capture`,
   Capture comes back instead when Refs was opened over a Capture draft.
4. **esc** turns the card back into the inspector. The query, scope, and selection are
   left untouched.

### The catalog

The catalog is closed and fixed. Letters never move: a row a reference lacks is hidden,
and the remaining rows keep their letters. Rows render in this order, with a hairline
between groups. Only the notes group has a header (`YOUR NOTES`).

| Key     | Label             | Exact payload                                                                     | Source                                | Shown when                       | Phase      |
| ------- | ----------------- | --------------------------------------------------------------------------------- | ------------------------------------- | -------------------------------- | ---------- |
| `I`     | Marker ID         | `marker_id` verbatim, plus a `shared ×N` tag when N ≥ 2                           | bob `marker_id`                       | present                          | core       |
| `T`     | Title             | the raw bob `title`                                                               | `RefItem.rawTitle`                    | always                           | core       |
| `W`     | Wiki link         | bob `link` verbatim (`[[ref/chat/x]]`)                                            | `RefItem.link`                        | always                           | core       |
| `U`     | Source URL        | `urls[0]` verbatim                                                                | `RefItem.urls`                        | a URL exists                     | core       |
| `L`     | Markdown link     | `[<escaped title>](<encoded urls[0]>)`                                            | title + URL                           | a URL exists                     | core       |
| `X`     | arXiv             | `arXiv:<id>`                                                                      | `identity.arxiv`                      | present                          | core       |
| `D`     | DOI               | bare DOI                                                                          | `identity.doi`                        | present                          | core       |
| `N`     | Note path         | vault-relative note path (`ref/chat/x.md`)                                        | `RefItem.id`                          | always                           | core       |
| `O`     | Obsidian link     | `obsidian://open?…` from `ObsidianOpenURL`                                        | absolute note path                    | URL builds                       | alternates |
| `P`     | PDF path          | absolute PDF path, plus a `missing` tag when the file is absent                   | `RefsLibrary.pdfURL(for:)`            | the URL resolves                 | core       |
| `F`     | PDF file          | the file itself, written as a file URL                                            | same URL                              | the file exists                  | core       |
| `R`     | Reading task      | bob `task.link` verbatim; the label reads `Reading task · archived` when archived | `RefItem.task`                        | task with a link                 | core       |
| `S`     | Research ref      | bob `research_ref` verbatim                                                       | `RefItem.researchRef`                 | present                          | core       |
| `1`–`9` | `p. N  ↳ comment` | note block (see the payload formats below)                                        | the inspector's `commentedHighlights` | loaded, and the row fits         | core       |
| `A`     | All your notes    | the commented highlights and standalone notes as blocks, then `own_notes`         | the same show row                     | any of these exists              | core       |
| `H`     | Pick a highlight… | opens the picker layer                                                            | the unfiltered show row               | any highlight or standalone note | picker     |

**Default row.** `↵` and a second `⌃Y` copy the selected row. The selection opens on
`I`, or on `W` when `I` is hidden. Nothing remembers the last-used row.

**Shared marker IDs.** `RefItem.markerIDShareCount` counts how many notes in the
snapshot carry the same `marker_id`, including this one. Superseded notes are already
excluded from `ref list`. Count over every snapshot **record**, including notes without
a PDF, not just the PDF-backed items. When the count is 2 or more, `I` shows an orange
`shared ×N` capsule with the help text "N notes share this ID — `bob ref show` needs the
note path for this one (press N)".

**Rows that load.** `1`–`9` and `A` come from the inspector's `bob ref show` row.
Opening the card uses the cached row when one exists. Otherwise it fetches at once, with
no 120 ms settle, and the notes section fills in at the bottom without moving the rows
above it. While it loads:

- The notes header shows a small spinner, but only when `annotationCount > 0` or
  `commentCount > 0`. Otherwise nothing renders until data arrives, so 88% of cards
  never flicker.
- If the load fails, the header reads a dim "Notes unavailable".

**Rows that fit.** Notes never scroll; the card's height decides how many show. With
`cap` note slots, `visible = min(9, total ≤ cap ? total : cap − 1)`. When
`total > visible`, the last line reads `+N more — A copies all`. The picker phase
extends it to `… · H picks one`.

### Payload formats

All formats live in RefsCore (`RefsCopyFormat`) and are pure.

- **Note block** (one commented highlight). Sections are joined by one blank line, with
  no trailing newline:

  ```markdown
  > the dolla dolla bills didn't hurt

  Comment: Replace with cash and cool 😎 .

  — Page 11 · [[ref/chat/sase_launch_post_introduction#^h-bf02e5a34bcb]]
  ```

  - Every quote line gets `> `. An empty quote line becomes `>`.
  - The comment's first line gets `Comment: `. Later comment lines stay verbatim and are
    never re-wrapped.
  - The provenance line is `— <displayPageLabel> · <link>`. Omit whichever part is
    missing, and drop the line when both are missing. Never invent a page.

- **Standalone note** (`kind == "note"`): `Note: <first line>`, then the remaining
  lines, then the provenance line. There is no quote.
- **All your notes** (`A`):
  - The commented highlights and standalone notes, as blocks in stored order, joined by
    a blank line.
  - If `own_notes` is not blank, then `### Notes`, a blank line, and the trimmed
    `own_notes`.
  - Distinct comments are never deduplicated.
- **Markdown link** (`L`):
  - In the title, escape `\` → `\\`, `[` → `\[`, and `]` → `\]`.
  - In the URL, percent-encode only `(` → `%28`, `)` → `%29`, and space → `%20`.
- **Previews.**
  - For every value row (`I T W U L X D N P R S`, and later `O`), the preview string
    **equals** the payload text. The view truncates it only with an ellipsis: `.middle`
    for IDs, paths, and links, `.tail` for titles.
  - `F` previews the file name.
  - A note row's detail is a short page label: `Page 28` → `p. 28`, and any other label
    as is. Its preview is `↳ <first comment line>`, and its `.help` shows the full
    payload.
  - `A` previews `N notes · Markdown`, with ` + your notes` appended when `own_notes`
    exists.
- Plain-text payloads everywhere except `F`, which writes a file URL.

### Keys while the card is open

The card captures **every** keyDown, so nothing leaks into the search field.

- **Normalizing the event.** Before matching, drop `capsLock`, `numericPad`, `function`,
  and `help` from the flags; arrow keys carry `numericPad` and `function`. Match letters
  and digits on `charactersIgnoringModifiers`, lowercased, when the remaining flags are
  empty or `shift`. That way mnemonics follow the printed layout, and Shift or Caps Lock
  changes nothing.

| Key                              | Effect                                                                                                                                   |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Letter or digit of a visible row | Copy that row                                                                                                                            |
| Letter of a hidden row           | The message line explains why the row is missing (strings below). Nothing is copied                                                      |
| Digit or `A` while notes load    | Queue it, with the message "Waiting for notes… esc cancels". The copy happens when data lands. After 5 s, show an error and copy nothing |
| `↵`, keypad Enter, `⌃Y`          | Copy the selected row                                                                                                                    |
| `↓` `⌃N` `⌃J` / `↑` `⌃P` `⌃K`    | Move the selection over visible rows, wrapping                                                                                           |
| `esc`, `⌃[`                      | Close the card and return to the inspector                                                                                               |
| `⌘K`                             | Close the card, then open Actions                                                                                                        |
| Any other printable key `c`      | Transient message "Nothing on ‘c’" for 1.5 s. No beep, no shake                                                                          |
| Anything else                    | Consumed                                                                                                                                 |

Mouse:

- Hovering a row highlights it, and a click copies it. Use `onTapGesture` with an
  accessibility button trait and action, like the inspector's disclosure row, so the
  search field keeps focus.
- Clicking a list row (`.select` or `.activate`) closes the card first.

When a hidden row's key is pressed, the message line shows one of these exact strings:

- `I`, field absent: "No marker ID on this note — N copies its path"
- `I`, older bob (no `marker_id` key at all): "Marker IDs need a newer bob — N copies
  the note path"
- `U` / `L`: "No source URL for this reference"
- `X`: "No arXiv ID for this reference"
- `D`: "No DOI for this reference"
- `P`: "No PDF path for this reference"
- `F`: "The PDF is missing — P copies its path"
- `R`: "No reading task for this reference"
- `S`: "No research ref for this reference"
- Note digits:
  - `k` with no such note: "No note k"
  - `k` past the visible rows: "Note k doesn't fit — A copies all"
- Notes failed to load: "Notes unavailable — nothing copied"
- `A` with nothing to copy: "No notes on this reference"
- A queued copy timed out: "Notes didn't load — nothing copied"
- A pasteboard write failed: "Couldn't copy — ↵ tries again"

The last three are errors and render red. When no message is set, the line reads
`↵ copies <selected label> · ↑↓ choose · esc back`.

### After a copy

1. Build the payload. If it is empty or unavailable, stop with a message and leave the
   clipboard untouched.
2. Write it **once** and check the result.
3. **On success:**
   - Close the card.
   - Hide Refs. With `return_to_capture`, Capture comes back when appropriate.
   - Show the HUD centred on Refs' former frame.
   - Post a VoiceOver announcement, e.g. "Copied Marker ID".
4. **On failure:** keep the card open, select the failed row, and show "Couldn't copy —
   ↵ tries again". `↵` retries the same frozen payload.

Confirmation titles are `Copied <label>`, for example "Copied Marker ID", "Copied PDF
File", "Copied Note 3", and "Copied 5 Notes". The HUD's value line is the row preview.

### Reliability contract

- **Frozen target.** Opening the card freezes the target: the `RefItem` snapshot, its
  note path, the PDF URL and existence at open time, and a monotonically increasing
  token. Library refreshes never change what a letter copies mid-gesture.
- **Selection changes close the card.** If `selectedID` changes to anything other than
  the frozen id (a click, a scan re-rank, or a fresh listing), the card closes.
- **Exact path match only.** A show result counts only when its `path` equals the frozen
  path. Remove the loader's `response.refs.first` fallback, so the inspector shows
  "Notes unavailable" rather than another reference's notes.
- **One pending copy at most.** `esc`, hiding Refs, `prepareForPresentation`, or a newer
  card revokes it, using the token. A late subprocess result may finish, but it can
  never reach the clipboard.
- **Older bob.** A bob without `marker_id` decodes as `.unsupported`. `I` hides, and
  pressing it says "needs a newer bob". The rest of the library decodes unchanged.
- **⌃Y never leaks.** In Refs, `⌃Y` is always consumed: it opens the card, or does
  nothing when no available row is selected. It passes through only while an IME is
  composing. Otherwise Cocoa's `yank:` reaches the search field.

### Visual spec

The card is the inspector turned over. It reuses `RefsKindTile`, the inspector's type,
the list's selection capsule, the Capture keycap style, and the panel's glass. It never
adds a second glass surface and never resizes the panel.

| Element       | Spec                                                                                                                                                                                                                                                                |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Header        | `RefsKindTile` at 28 pt; "COPY" in `.caption.weight(.semibold)` `.secondary`; the title in `.headline`, one line, tail-truncated                                                                                                                                    |
| Row           | 24 pt tall: an 18×18 keycap, a 92 pt `.callout` label, then a preview filling the rest                                                                                                                                                                              |
| Keycap        | Radius 4.5; fill `.primary.opacity(0.06)`; stroke `.primary.opacity(0.14)` at 0.5 pt (0.30 / 1 pt under Increase Contrast); SF Mono 11 semibold, uppercase glyph. The selected row uses an accent fill with a white glyph                                           |
| Selected row  | The list's selection capsule: radius 8, accent at 0.16 (0.30 under Increase Contrast). Hover uses `.primary.opacity(0.06)`                                                                                                                                          |
| Preview       | SF Mono 11.5, `.secondary`                                                                                                                                                                                                                                          |
| Tags          | Capsule, `.caption2`: `shared ×N` orange, `missing` orange, `archived` secondary                                                                                                                                                                                    |
| Notes header  | "YOUR NOTES", count in `.caption2.monospacedDigit()` `.tertiary`, then `· quote + comment + link` in `.tertiary`                                                                                                                                                    |
| Message line  | The card's last line, `.caption` `.secondary` (red for errors). It doubles as the hint line                                                                                                                                                                         |
| Panel footer  | While the card is open: "Copy — press a key". When idle, add `⌃Y Copy` after `⌥↵ Reveal`, with `ViewThatFits` fallbacks that drop `⌘1–5 Scope`, then `⌘S Scan`                                                                                                      |
| Motion        | Cross-fade plus a 6 pt rise, 0.14 s easeOut. Under Reduce Motion, cross-fade only. Selection moves instantly                                                                                                                                                        |
| Accessibility | The card is one container, labelled "Copy menu for <title>, N options". Each row reads "<label>, <preview>, key <glyph>" and carries the button trait and an action. Hidden rows are absent from the tree. On open, announce "Copy menu. Press a letter or number." |

**HUD.** `RefsCopyHUDController` owns a borderless `.nonactivatingPanel`:

- Window behaviour: `canBecomeKey` and `canBecomeMain` are false, `ignoresMouseEvents`
  is true, the level is `.floating`, and it uses the Refs collection behaviour. Show it
  with `orderFrontRegardless()`.
- Content: a glass capsule (`NSGlassEffectView`) hosting a SwiftUI view. On the left, a
  green `checkmark.circle.fill`. Beside it, the title at 13 pt semibold over the value
  in SF Mono 11.5 `.secondary`, middle-truncated at a maximum width of 380 pt. Under
  Reduce Transparency, use an opaque `windowBackgroundColor` capsule.
- Timing: fade in over 0.1 s, hold 1.0 s, fade out over 0.3 s. A new show restarts the
  cycle. Showing or re-presenting Refs removes the HUD at once.

### Module map

- **RefsCore (portable, Linux-tested):**
  - decoding additions;
  - `RefsClipboardPayload` (`.text(String)` / `.fileURL(URL)`);
  - `RefsCopyKey`, `RefsCopyRow`, `RefsCopyCatalog`, `RefsCopyFormat`;
  - `RefsCopyCardLayout` (capacity math; the app passes in its metrics);
  - `RefsCopyKeyModifiers` (an `OptionSet` the app adapts `NSEvent` flags into) and
    `RefsCopyKeyMap`;
  - the `RefsCopyCard` state machine.
- **App target:**
  - checked pasteboard writes;
  - the loader's single-flight show cache;
  - `RefsCopyCardView` and `RefsCopyHUDController` with its view;
  - router, model, controller, `⌘K`, footer, and coordinator wiring.

## Verification rules (every bob-mac-capture phase)

Follow bob-mac-capture's `AGENTS.md` exactly. Never land Swift you have not compiled.

- **RefsCore.** Run `just portable-check` (swift-format lint, then build and test the
  portable targets) on a tailnet host with Swift. The agent hosts have none; athena
  does. Read `sase memory read tailnet.md -r "<why>"` for host access.
- **App and app tests** (`Sources/BobMacCapture*`, `Tests/BobMacCapture*`). Type-check a
  scratch copy on the Mac: rsync it without `.build`/`.git`, run `Scripts/bundle.sh` and
  then `just typecheck-tests`, and delete the copy afterwards. Never use Bryan's
  checkout or `just install`. CI runs the tests and uploads the render fixtures. If the
  Mac is offline, say in the final report that the macOS targets are unverified.
- Keep swift-format lint clean; a lint parse error skips every later CI step.

## Phase: bob-marker-id

Expose bob's stable marker ID to JSON consumers. This happens in bob-cli, so the app
never guesses it: the stem is wrong or missing for 83 of 406 PDF-backed rows.

- `src/native/ref_library/row.rs`:
  - On `RefRow.id`, replace `#[serde(skip_serializing)]` with
    `#[serde(rename = "marker_id")]`. It stays an `Option<String>`, so absent serializes
    as `null`, like the other optional fields.
  - Keep the Rust field name `id`, so query resolution (`MatchKind::Id`, `find.rs`) is
    untouched.
  - Update the field's doc comment: it is serialized as `marker_id` and resolution reads
    it.
- This adds a field under `REF_SCHEMA_VERSION` 1. `ref list`, `ref find` (match `ref`
  objects), and `ref show` (the flattened row) all inherit it. Human and Markdown output
  do not change.
- `docs/ref.md`, in the JSON envelope paragraph's list of `ref` object fields: add
  `marker_id`, "the frontmatter `id` (the stable marker ID `bob ref create --include-id`
  writes), `null` when absent". Add `marker_id` to the sentence that lists the fields
  additive under `schema_version` 1.
- Tests, next to the existing ref-library JSON tests:
  - A note with frontmatter `id: foo` serializes `"marker_id":"foo"`.
  - A note without one serializes `"marker_id":null`.
  - `ref show -f json` carries it.
  - Update any exact-JSON goldens this changes.
- `just check` passes.

## Phase: refs-copy-core

Everything here lives in `Sources/RefsCore` with tests in `Tests/RefsCoreTests`. No
AppKit.

### Decoding (additive; every new key uses `decodeIfPresent`)

- **`RefRecord`:**
  - `markerID: String?` and `reportsMarkerID: Bool`. The flag is true when the row JSON
    contains the `marker_id` key, even as `null`.
  - `researchRef: String?` (`research_ref`).
  - Encoding must round-trip through the snapshot cache: when `reportsMarkerID` is true,
    write the value, or `encodeNil` when the value is nil. Then a cached snapshot still
    tells "absent" from "older bob". Old caches decode as `.unsupported` until the next
    refresh; there is no snapshot schema bump.
- **`RefItem`:**
  - `markerID: RefMarkerID` (`.present(String)` / `.absent` / `.unsupported`);
  - `markerIDShareCount: Int`;
  - `researchRef: String?`.
  - `RefsCatalog.items(from:)` computes the share counts over every snapshot record, the
    way it already computes `titleDisambiguator`. Carry every new field through its
    re-init.
- **`RefsShowAnnotation`:** `blockID: String?` (`block_id`) and `link: String?`.
- **`RefsShowRow`:**
  - `ownNotes: String` (`own_notes`, default `""`);
  - computed `standaloneNotes` (`kind == "note"` with a non-blank comment);
  - computed `yourNotes` (commented highlights plus standalone notes, in stored order).
  - `commentedHighlights` is unchanged.

### Catalog, formats, and layout

- **`RefsCopyKey`:** one case per catalog row, `note(Int)` for 1–9, plus `allNotes`. It
  has a `glyph` and parses from a typed character. Leave room for the later
  `obsidianLink` and `highlightPicker` cases.
- **`RefsCopyRow`:** key, group (`identity`, `sources`, `files`, `notes`), label,
  payload, preview, truncation style, optional tag (`shared(Int)` / `missing` /
  `archived`), optional detail (the note's page), optional help (the full note payload),
  and the confirmation title.
- **`RefsCopyCatalog`:**
  - Inputs:
    - the frozen item;
    - the absolute PDF path and whether the file exists (both from the app);
    - the notes state (`.loading` / `.loaded(RefsShowRow)` / `.unavailable`);
    - the note capacity.
  - Outputs: the visible rows in catalog order, the default key, and the hidden-row
    reason for any key, using the exact strings from the design contract above.
- **`RefsCopyFormat`:** implements the payload formats above.
- **`RefsCopyCardLayout`:** the capacity math. It takes the card height, the count of
  non-note rows, the group count, and a metrics struct (header, row, separator,
  notes-header, message-line, and padding heights), and returns `cap`. It also
  implements the `visible` rule from the catalog section.

### Key map and state machine

- **`RefsCopyKeyMap.command(characters:keyCode:modifiers:)`.** Returns one of:
  - `.key(RefsCopyKey)`;
  - `.copySelected`;
  - `.move(±1)`;
  - `.close`;
  - `.showActions`;
  - `.unknown(Character)`;
  - `.consume`.

  It implements the key table above with the flag normalization. Key codes: Return 36,
  keypad Enter 76, Esc 53, `[` 33, Y 16, N 45, P 35, J 38, K 40, ↓ 125, ↑ 126.

- **`RefsCopyCard`** (value type).
  - State:
    - token, frozen item id, and inputs;
    - rows;
    - `selectedKey` (it opens on the default key);
    - `pending: RefsCopyKey?`;
    - `message` (text, error flag, and whether it is transient).
  - Transitions. Each returns an effect: `.none`, `.write(RefsCopyRow)`, `.close`, or
    `.closeThenActions`.
    - `handle(_ command:)`
    - `notesLoaded(path:row:)`: ignored unless the path matches. Resolves a pending key
      into a write or a reason.
    - `notesFailed(path:)`
    - `pendingTimedOut()`
    - `writeFailed(key:)`: selects that row and sets the error.
    - `setNoteCapacity(_:)`: recomputes the rows and keeps the selection when it is
      still visible, or else returns to the default.
  - Move wraps over the visible rows.
  - After `.close`, any later `notesLoaded` returns `.none`.

### Tests (Linux)

- **Decoding:**
  - `marker_id` present, `null`, and missing map to the three states, and each survives
    a snapshot round trip.
  - `research_ref`.
  - Share counts include notes without a PDF.
  - Annotation `block_id`/`link` and `own_notes`.
  - Older payloads still decode.
- **Catalog:**
  - A chat fixture yields `I T W N P F R`, in order.
  - A paper fixture with a URL, an arXiv ID, and notes yields the full card.
  - Letters are fixed when rows hide.
  - A marker ID that differs from the stem copies the marker ID.
  - `shared ×N` appears for shared IDs.
  - A missing PDF hides `F` and tags `P`.
  - An archived task changes the `R` label.
  - With an older bob, `I` hides with the "newer bob" reason.
  - The default is `I`, else `W`.
  - Every value row's preview equals its payload.
- **Formats:**
  - A multi-line comment is kept intact.
  - Every quote line is prefixed.
  - A missing page, a missing link, and both missing.
  - Standalone notes.
  - `A` with `own_notes`.
  - Markdown-link escaping: brackets and a backslash in the title; parentheses and a
    space in the URL.
- **Layout:** capacity and the visible/overflow edge cases.
- **Key map:**
  - Letters with Shift or Caps Lock.
  - Arrows carrying `function`/`numericPad`.
  - `⌃N`/`⌃P`/`⌃J`/`⌃K`; Esc and `⌃[`; `↵`, keypad Enter, and `⌃Y`; `⌘K`.
  - `⌘V` is consumed.
  - An unknown printable key.
- **State machine:**
  - A visible key writes.
  - A hidden key gives its reason and no write.
  - A queued digit followed by a matching load writes.
  - A load for a mismatched path is ignored.
  - A failed load gives an error and no write.
  - A timeout gives an error.
  - Close revokes a pending copy.
  - Move wraps.
  - `writeFailed` selects the row and sets the error.
  - Capacity changes keep the selection.

## Phase: refs-copy-components

This phase adds app-target building blocks only. Nothing the user sees changes: ⌃Y stays
unmapped until the next phase.

- **Pasteboard** (`RefsActionsMenu.swift`):
  - `RefsPasteboardWriting` becomes
    `func write(_ payload: RefsClipboardPayload) -> Bool`, with a protocol-extension
    `copy(_ string:)` that keeps the diagnostic call sites working.
  - `SystemRefsPasteboard` clears the pasteboard once, then writes text with `setString`
    or a file URL with `writeObjects([url as NSURL])`. It also sets
    `org.nspasteboard.source` to the bundle identifier, so clipboard managers attribute
    the entry. It returns the result of the primary write.
  - Update the test `FakePasteboard` to record payloads and to fail on demand.
  - Existing copy actions keep their behaviour.
- **Show cache** (`RefsInspectorLoader.swift`):
  - Add `showRow(for item: RefItem) async -> RefsShowRow?`. It is single-flight per
    `(id, snapshotFetchedAt)` key: concurrent callers share one unstructured fetch task,
    and canceling one awaiting caller never cancels the fetch. This matters because the
    `refs-show` lane cancels the previous call in the lane, so a duplicate fetch would
    kill the inspector's.
  - It keeps the existing LRU of 32, never caches a failure, and accepts only an exact
    `path` match (remove the `?? response.refs.first` fallback).
  - Add a synchronous `cachedShowRow(for:)` accessor.
  - The inspector's own `load` goes through `showRow(for:)`.
- **`RefsCopyCardView`** (new, `Sources/BobMacCapture/Refs/`):
  - Renders a `RefsCopyCard` exactly per the visual spec.
  - Reports its measured height through a callback, as the list reports
    `visibleRowBudget`.
  - Exposes row tap and hover callbacks.
  - Has a `previewMode` static layout for fixtures, like `RefsInspectorView`.
  - Add the card metrics to `RefsVisualTokens` and feed them to `RefsCopyCardLayout`.
- **`RefsCopyHUDController` plus `RefsCopyHUDView`** (new): the HUD exactly per the
  visual spec, with `show(title:value:centeredOn:)` and `dismiss()`. Window placement
  math is a static function so geometry tests can assert it.
- **Tests:**
  - Loader: an exact-path mismatch yields nil and "Notes unavailable"; two concurrent
    `showRow` calls produce one fetch; the cached accessor works.
  - Pasteboard fake behaviour.
  - HUD frame math.
  - `RefsCopyCardDesignTests` renders through `RenderFixtureWriter`, light and dark, at
    inspector-column width:
    - the chat card;
    - an annotated paper with two notes;
    - a shared ID plus a missing PDF;
    - notes overflow;
    - notes loading with a queued digit;
    - an error message.
  - Plus a HUD view fixture.

## Phase: refs-copy-integration

This phase wires the feature end to end.

- **Router.**
  - `RefsKeyRouter`: keyCode 16 with flags exactly `.control` maps to `.showCopyCard`,
    or nil while marked text is present.
  - Add `RefsCommand.showCopyCard`. `perform(.showCopyCard)` always returns true, so the
    key is consumed even with no selection.
- **`RefsPanelModel`.**
  - `@Published copyCard: RefsCopyCard?`.
  - `openCopyCard()`:
    - Needs a selected, available item. Unavailable rows do nothing.
    - Freezes the target: the item, `library.pdfURL(for:)`, and a `FileManager`
      existence check.
    - Seeds the notes from `cachedShowRow`, or else starts `showRow` at once.
    - Announces the open.
  - `handleCopyCardKey(characters:keyCode:modifiers:)` and `copyCardRowTapped(_:)` feed
    the state machine.
  - Effects:
    - `.write` → `pasteboard.write`, then on success the copy-completion path, or on
      failure `writeFailed`.
    - `.close` → nil.
    - `.closeThenActions` → close, then `actionsPresenter()`.
  - Timers: clear transient messages after 1.5 s; pending timeout after 5 s.
  - `copyCardNoteCapacity` is set from the view.
  - Lifecycle:
    - The Esc ladder becomes card → banner → query → scope → close.
    - `prepareForPresentation` and `panelDidHide` clear the card and revoke any pending
      copy.
    - A `selectedID` change away from the frozen id closes the card.
    - `.select` and `.activate` close the card first.
- **Controller** (`RefsPanel.swift`).
  - While `model.copyCard != nil`, `processKeyEvent` routes every keyDown to the card,
    with `charactersIgnoringModifiers` and flags adapted into `RefsCopyKeyModifiers`,
    and always consumes it.
  - On copy success:
    1. Capture `panel.frame`.
    2. Dismiss Refs through `copyDismisser`, which defaults to `hide()`.
    3. Show the HUD centred on that frame.
    4. Post "Copied …".
  - `show()`/`represent()` dismiss the HUD at once.
- **⌘K** (`RefsActionsMenu.swift`, `performAction`).
  - Replace `.copyWikiLink` and `.copyPDFPath` with one `.copy` item titled "Copy…",
    with key equivalent `y` and mask `.control`. It opens the card.
  - Remove `copyWikiLink()`, `copyPDFPath()`, the toast, and the footer toast branch;
    they are now dead.
- **Footer.**
  - Idle hints gain `⌃Y Copy` after `⌥↵ Reveal`, with the `ViewThatFits` fallbacks.
  - While the card is open, the left side reads "Copy — press a key".
  - Keep `RefsFooter.hintsText` testable.
- **Panel view.**
  - When `copyCard` is set, the inspector column renders `RefsCopyCardView` in place of
    `RefsInspectorView`, using the specified transition.
  - Below the inspector threshold, the card replaces the list column.

> [!decision] return_to_capture `BobPanelCoordinator` records whether Capture was
> visible when `showRefs()` opened Refs. `AppDelegate` sets the controller's
> `copyDismisser` to `coordinator.dismissRefsAfterCopy()`. That method hides Refs and,
> when the flag is set, clears it and calls `presentCapture()`. The retained draft then
> comes back, and ⌘V pastes into it. The HUD still shows.

> [!decision] return_to_capture = no `copyDismisser` stays `hide()`, and Capture never
> reopens after a copy.

- **README** (bob-mac-capture):
  - Add a `Ctrl-Y` row to the Bob Refs key table.
  - Change the Esc row to "Copy card, then banner, then query, then scope, then close".
  - Add a "### Copy card" subsection: the catalog, keys, hide-on-copy and HUD, shared
    IDs, and the older-bob behaviour.
  - Rewrite the ⌘K paragraph to describe "Copy… (⌃Y)" and drop the toast sentence.
  - Update the router's doc comment where it names the keyboard table.
- **Tests:**
  - Router: `⌃Y`; `⌃Y` while marked text is present; plain `y` passes through.
  - Model, using `FakePasteboard`:
    - no selection → consumed, and no card;
    - frozen payloads across a library publish;
    - a selection change closes the card;
    - the Esc ladder;
    - resets on presentation and on hide;
    - success writes once and fires the completion with the confirmation;
    - failure keeps the card with an error and no completion;
    - a queued digit is resolved by the loader;
    - `⌘K` from the card;
    - the actions sections now contain `.copy`, and `.copy` opens the card.
  - Controller: with the card open, `w` is consumed and copies.
  - Coordinator: `dismissRefsAfterCopy` with and without the Capture flag.
  - Footer hint variants.
  - Full-panel design fixtures: the wide card, and the narrow card covering the list.
  - Update the existing tests that asserted the removed copy items and the toast.

## Phase: refs-copy-highlight-picker

`H` reaches every highlight, including uncommented ones, without breaking the
single-keypress promise for the common case.

- **Fetch.**
  - `BobRefsFetcher.show` drops `-c`: the unfiltered show JSON is at most 13.6 KB, with
    a median fetch of 0.2 s.
  - The inspector keeps its `commentedHighlights` projection, so its notes are
    unchanged.
  - Update the README's inspector paragraph accordingly.
- **RefsCore.**
  - `RefsHighlightPicker` state:
    - Entries are every `highlight` with a non-empty quote or comment, plus every
      standalone `note`, in stored order. Images are excluded.
    - Selection and a window of up to 9 rows that scrolls minimally to keep the
      selection visible. The window size comes from the picker's measured height.
  - Payloads:
    - a commented highlight → the note block;
    - an uncommented highlight → the quote block plus provenance;
    - a note → the standalone format;
    - "all highlights" → every entry's block, joined by blank lines.
- **Catalog.**
  - Add `H` "Pick a highlight…", previewing `N highlights`. It shows once loaded and
    when entries exist, and queues like `A` while notes are loading.
  - Its hidden reason is "No highlights on this reference".
  - The overflow line and the past-the-rows digit message gain "H picks one".
- **Picker layer.**
  - It replaces the card's rows in the same column. The header reads `HIGHLIGHTS <n>`
    over the title.
  - Rows are a fixed 36 pt and two lines:
    - line 1: the digit keycap for the window position, a short page label, and the
      quote, italic, `.secondary`, tail-truncated;
    - line 2: `↳ comment` when there is one, or else the quote continues.
  - The message line reads `↵ copies · 1–9 pick · A copies all · esc back`.
- **Keys.**
  - Digits copy that window row.
  - Moves follow the card's keys.
  - `↵` and `⌃Y` copy the selected row.
  - `A` copies all highlights ("Copied 29 Highlights").
  - `esc` and `⌃[` go back to the card, with its state intact.
  - `⌘K` closes everything, then opens Actions.
  - Any other printable key gives "Nothing on ‘c’".
  - A single copy confirms as "Copied Highlight · p. 11".
- **Tests:**
  - RefsCore: entries, windowing, payloads, and keys.
  - Model: H opens and closes, digit copy, and A copy.
  - Design fixtures: the picker with 29 entries, mid-scroll.
  - README key and catalog updates.

## Phase: refs-copy-alternates

This phase uses the native menu-alternate idiom: holding ⌥ swaps previews in place and
never shifts the layout.

- **Controller.**
  - Also monitor `.flagsChanged`, but only while Refs is key and the card or picker is
    open.
  - Set `model.copyAlternatesActive` from the option key.
  - At card open, seed it from `NSEvent.modifierFlags`.
- **RefsCore.**
  - `RefsCopyRow` gains an optional `alternate` (label, payload, preview, confirmation).
  - The key map accepts `option` (alone or with Shift) for letters and digits as
    `.key(_, alternate: true)`. A row without an alternate copies its primary, as native
    menus do.
- **Alternates:**
  - `W` → "Wiki link · titled": `[[<link target>|<title>]]`. Strip `[`, `]`, and `|`
    from the alias text, and fall back to the plain link when nothing remains.
  - `P` → "PDF path · ~": `~`-abbreviated.
  - `N` → "Note path · absolute".
  - `1`–`9` and picker rows → "Quote only": the highlight's quote verbatim.
  - `F` → "Narration file": the audio file URL, when `audio` resolves and exists.
- **Obsidian row.** Add `O` "Obsidian link" in the files group, right after `N`. Its
  payload is `ObsidianOpenURL.url(forAbsolutePath:)` for the note, as an
  `absoluteString`, and it shows when the URL builds.
- **Tests:** RefsCore alternates, key map, and the `O` row; model flag handling; design
  fixtures of the card with ⌥ held; README updates.

## Bryan's Mac checks

These are manual, after `refs-copy-integration` lands and is installed. They cover what
CI and agents cannot verify:

1. `⌃Y` reaches the Refs panel. No Karabiner, Hammerspoon, or system binding eats it.
2. After a copy, `⌘V` pastes immediately in Highlights, Obsidian, and Terminal. With
   `return_to_capture`, it pastes into the restored Capture draft.
3. `F` attaches the PDF in Finder, Slack, Mail, Messages, Claude, and ChatGPT.
4. The HUD never takes key and shows on the active Space over full-screen Highlights.

## Out of scope

- **Stamping marker IDs on more notes.** 77 PDF-backed notes have none, because
  `bob ref create` adds an ID only with `--include-id`. This is a separate bob-cli
  question; file a task bead if Bryan wants it.
- An HTML flavour for `L`, dragging rows out, a "smart ⌘C", citation/BibTeX formats,
  summary excerpts, and app-side capture tokens. All were rejected or deferred by the
  research.
