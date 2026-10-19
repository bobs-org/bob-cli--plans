---
tier: tale
title: "Bob Refs: NEW section on top, with an ⌥N toggle"
goal:
  Bob Refs shows a precisely defined NEW section (renamed from Just added) at the top of
  the standing sections, the blue dot means exactly NEW, and ⌥N toggles a reference's
  NEW state.
size: medium
decisions:
  new_cap:
    ask: Should the NEW section list every NEW reference, dropping today's 5-row cap?
    choices:
      all: Dot and section always agree; on busy days Today can start below the fold
      cap5:
        Today stays higher, but NEW rows past the fifth keep their dot down in Ready or
        Next
    default: all
    why:
      One rule (dot = NEW = top section) is the clearer definition asked for; ⌥N keeps
      it short
    answer: all
  new_window:
    ask: How long should an unopened, unmarked reference stay NEW on its own?
    choices:
      three_days:
        3 local days, today's Just added window; about 5-11 rows on recent days
      seven_days: 7 local days; roughly twice as many rows, mostly agent reports
    default: three_days
    why: Keeps the uncapped section short; ⌥N can hold any older reference in NEW
    answer: three_days
  today_link:
    ask: Should a successful Link to Today (⌘N) in Bob Refs leave the reference NEW?
    choices:
      stays_new:
        It stays NEW, in the NEW section with its pink Today pill, until opened or ⌥N
      marks_seen:
        Linking also marks it seen, so it shows under Today after the next re-rank
    default: stays_new
    why:
      NEW keeps exactly three exits (open, ⌥N, leaving Ready/Next); the Today pill shows
      the link
    answer: marks_seen
  decision_record:
    ask: Should the plan add a decisions-web record for the Refs NEW state?
    memory:
      - decisions
    default: false
    answer: false
proposed_by: bbugyi200.athena.11v
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-19 11:13:49
status: wip
---

# Bob Refs: NEW on top, with ⌥N to toggle it

## Goal

In the Bob Mac Capture **Bob Refs** panel (the quick-open panel for reference PDFs):

1. Rename the browse section **Just added** to **NEW** and move it to the top of the
   standing sections, above Today.
2. Give NEW one precise definition, shared by the blue dot, the semibold title, and the
   section.
3. Add **⌥N** (Option-N) to toggle the selected reference's NEW state: **Mark as Seen**
   on a NEW row, **Mark as New** on any other Ready or Next row. Example: Bryan has
   acted on a reference without reading it. He will close its reading task in the
   morning GTD review, but until then he wants it gone from the top of the panel.

All code lives in the **`bob-mac-capture`** linked repo. Open it with `/sase_repo`
(`sase repo open bob-mac-capture -r "..."`), read its `AGENTS.md` first, and do all
reads and writes under the path it prints. Nothing changes in bob-cli, bob-plugins, or
the vault. The only possible bob-cli change is the memory note under the
`decision_record` decision.

## Current behavior (verified)

- `Sources/RefsCore/RefsRanking.swift` `browseSections` orders the sections as Just
  scanned → Today → Just added → Reading → Next → Ready → Recently opened → Library.
  Each item appears in the first section that admits it.
- Just added holds Ready/Next rows that are unopened (`RefsRanker.isUnopened`) and were
  added within `RefsRankingConstants.justAddedWindowDays = 3` local days. Rows are
  sorted by `compareAddedDesc` (added day only, then id) and capped at
  `justAddedCap = 5`. Over-cap rows fall through to their lane.
- The blue dot (`RefsRowView.gutter`, a 6 pt accent `Circle` in an 8 pt gutter) and the
  semibold title follow `RefsRowContent.isUnopened`, which is `RefsRanker.isUnopened`:
  Ready/Next, no open-log event, no Spotlight last-used date, no annotations. It has
  **no age limit**. About 200 legacy Ready rows can carry the dot, and dotted rows also
  show in Ready below the 5-row cap. This mismatch is why "NEW" feels undefined today.
- `RefsSectionKind.title` says "Never 'NEW': the review walk owns that chip". Bryan now
  wants the panel's section called NEW. The panel's NEW is a different concept from the
  review walk's NEW freshness tier, and the docs must say so (see the contract).
- ⌥N is unbound in the panel. It reaches the search field, where on a US layout it is a
  dead key that starts composing `˜`. Keys go through `RefsPanelController`'s local
  monitor → `RefsKeyRouter.command(keyCode:modifiers:context:)` →
  `RefsPanelModel.perform`. When `perform` returns false, the key passes through to the
  field.
- Open history lives Mac-locally in `refs-open-log.json` (`RefsOpenLogStore` in
  `Sources/RefsCore/RefsStores.swift`: atomic writes, 0600 file in a 0700 dir,
  quarantine on corruption, 365-day / 2000-event pruning, `rename(from:to:)` after
  `bob ref rename`). `RefsLibrary.recordOpen` reloads the log and publishes
  `signals.opens` synchronously.
- The listing is frozen: a library publish refreshes row content in place and never
  reorders (`RefsListing.refreshingContent`). Re-sectioning happens only on a fresh
  listing: panel presentation (`prepareForPresentation` → `rerankSelectingFirst`, which
  selects the first row), query/scope/order edits, and ⌘R.

## The NEW contract

This section is authoritative. Code, tests, captions, and the README all follow it.

**Definition.** A reference row is NEW when both hold:

1. Its reading state is **Ready or Next**, blocked included. Reading, Read, Dropped, and
   Unknown rows are never NEW, whatever their marks. So when the morning review closes
   or drops the reading task, the row leaves NEW by itself.
2. One of these holds:
   - **Marked new**: its ⌥N mark is `new`, and the row has not been opened since the
     mark. An open-log event or a Spotlight last-used date later than the mark clears
     NEW. Annotations and age do not.
   - **Arrived**: it has no ⌥N mark, its `added` day is within `newWindowDays` local
     days of today (inclusive edge, panel calendar — the same arithmetic as today's Just
     added), and `RefsRanker.isUnopened` holds (never opened, never annotated).

   A `seen` mark makes the row not NEW.

**Ways out:** open the PDF (anything that logs an open or updates Spotlight's last-used
date), ⌥N, the reading task leaving Ready/Next, or, for unmarked rows only, aging past
the window. **Ways in:** arrival, or ⌥N on a Ready/Next row.

**Never.** NEW is presentation state on this Mac only. It never runs `bob`, never writes
the vault, and never changes reading state, task lanes, Today, or review freshness. It
is **not** the review walk's NEW tier, which is task freshness in Obsidian. README text
and the `RefsSectionKind` doc comment must say this in one sentence.

**Presentation invariant.** A row shows the blue dot and a semibold title exactly when
it is NEW, in every mode (browse, search, Newest, Tabs). In browse mode, every NEW row
is listed in the NEW section, except rows already shown in Just scanned.

> [!decision] new_cap = cap5 Keep a `newCap = 5` constant and take `prefix(newCap)`. The
> invariant then reads "the NEW section lists the newest five NEW rows; any others keep
> their dot in their lane". Keep the existing cap test, renamed. README and the doc
> comment state the cap.

**Section order (browse, ranked):** Just scanned → **NEW** → Today → Reading → Next →
Ready → Recently opened → Library. Just scanned stays first. It is a transient answer to
⌘S (15 minutes), its rows are selected after a scan, and putting NEW above it would
swallow most of a scan's rows. A Today-linked NEW row sits in NEW with its pink Today
pill (see `today_link`).

**NEW order:** by **new-since**, newest first. A marked row's new-since is the mark
instant. An arrived row's new-since is `addedAt` when known, or else the start of its
`added` day in the panel calendar. This puts timed rows ahead of untimed rows on the
same day, as Newest does. Ties break on id ascending. A reference marked new today lands
at the top of NEW.

**Toggle (⌥N) semantics: the smallest mark wins.** ⌥N flips the selected row's NEW state
and stores the smallest mark that produces the flipped state. The flipped state is
compared with what the arrival rule alone would give (the unmarked result). If they
match, the row's mark is **removed**. Otherwise a `new` or `seen` mark is stored with
the current instant. So ⌥N ⌥N always restores the starting state, and an arrived row
keeps aging out on schedule after an accidental double press.

## Interaction design

- **⌥N** (`flags.subtracting(.capsLock) == .option`, key code 45) maps to a new
  `RefsCommand.toggleNew`. It passes through only while the search field holds marked
  (IME-composing) text, like ⌘Y and ⌘N. ⌥⇧N, ⌃N (move next), and ⌘N (Link to Today) keep
  their current behavior. While a card (Copy, Rename, Link to Today) is up, its own key
  handling owns the keyboard, so ⌥N does nothing there.
- `perform(.toggleNew)` **always returns true**, even with no row selected, so ⌥N never
  leaks the `˜` dead key into the search field. Accepted cost: ⌥N no longer composes
  `˜`/`ñ` in the Refs search field.
- **On an eligible row** (selected, available, has an item, Ready/Next):
  1. Write the mark through `RefsLibrary`. The write is synchronous on the main actor,
     like `recordOpen`, so a fast second press sees the new state.
  2. Leave the row **in place**: same section, same index. The selection does not move,
     so pressing ⌥N again undoes. This is the Mail "unread" pattern: a message you just
     read stays in the Unread view until you leave it.
  3. Post a VoiceOver announcement: "Marked seen" or "Marked new".
  4. The row re-sections on the next fresh listing (panel reopen, ⌘R, query, scope, or
     order change).
- **On anything else** (Reading/Read/Dropped/Unknown rows, unavailable rows, Tabs
  fallback rows, no selection): consume the key, call `NSSound.beep()` (the macOS "can't
  do that" cue, which honors the user's alert and flash-screen settings), and announce
  "Only Ready and Next references can be New". Write nothing. Inject the beep as a
  closure on the model (default `NSSound.beep`) so tests can count it.
- **⌘K menu:** for an eligible row, add **Mark as Seen** (when NEW) or **Mark as New**
  (when not), with key equivalent `n` + `.option`. Place it right after **Link to
  Today…**, so the ⌘N/⌥N pair sits together. Omit it for ineligible rows, the same way
  the menu omits other actions that don't apply.
- The footer hints stay unchanged, so the footer's `ViewThatFits` variants and their
  tests don't shift. Discoverability comes from the NEW header hint and the ⌘K menu.

## Visual design

- **Row dot.** Keep the 6 pt accent dot in the 8 pt gutter. It now means NEW. Animate it
  on change:
  - Appearing: scale from 0.3 to 1 with opacity,
    `spring(response: 0.3, dampingFraction: 0.6)`, a small pop.
  - Disappearing: scale to 0.3 with opacity, `easeOut(duration: 0.16)`.
  - Under Reduce Motion: opacity only (0.12 s).

  The title weight (semibold ↔ regular) changes under the same animation. Rename
  `RefsRowContent.isUnopened` → `isNew` and compute it with `RefsRanker.isNew`.

- **NEW header.** `RefsSectionHeader` for `.new` leads with the same 6 pt accent dot
  before the uppercase title "NEW", so the section names the dot. Then the count as
  usual. While the selection is a row of the NEW section, add a quiet trailing hint in
  the existing trailing-detail style (`.caption2`, `.tertiary`, monospaced digits):
  - "⌥N Mark Seen" when that row is NEW.
  - "⌥N Mark New" when it was just marked seen in place, which doubles as the undo
    affordance.

  No hint otherwise. Expose the hint from the model (for example
  `RefsPanelModel.newSectionHint: String?`) so tests pin it without building views. The
  count stays the frozen `section.ids.count`, like every other header. The accessibility
  label stays "New, N references". The hint is visual only, because the ⌘K item and the
  announcements cover VoiceOver.

- **Inspector chip.** When the selected row is NEW, add a **New** chip after the state
  chip and before the Today pill. Use an accent-tinted capsule in the kind chip's style
  (`.caption.weight(.semibold)`, 8×4 padding, accent at 0.16 background) whose label is
  a small `circle.fill` glyph plus "New". Fade it in and out with the row dot's timing
  (opacity only under Reduce Motion).
- **Captions.** The row date phrase for `.new` stays the added phrase, as Just added
  had. The inspector why-here line for rows in the NEW section:
  - Arrived: `New · added Oct 18 · never opened`. With no added date:
    `New · never opened`.
  - Marked: `New · marked Oct 19`. This is the mark's local month-day; add a small
    `Date` month-day helper beside `RefsCaption.absoluteMonthDay` if none exists.
  - Marked seen in place, so no longer NEW:
    `Seen · leaves New next time you open Bob Refs`.
- **VoiceOver row label.** Replace the trailing "never opened" with "new" whenever the
  row is NEW (for example "Title, Paper, Ready, added today, 22 pages, new").
- Update the `RefsVisualTokens` comment "the accent means selection, matches, and the
  unopened dot" to say "the New dot".

## Implementation

### RefsCore (portable; built and tested on Linux)

1. **`RefsRanking.swift`**
   - Constants: rename `justAddedWindowDays` → `newWindowDays`, set to `3`
     (`new_window = three_days`) or `7` (`seven_days`). Delete `justAddedCap`
     (`new_cap = all`), or rename it to `newCap` (`cap5`). Update the doc comments.
   - `RefsSignals`: add `public var newMarks: RefsNewMarks` with an empty default in
     `init`, so existing call sites compile unchanged.
   - `RefsSectionKind`: rename `.justAdded` → `.new`, with title `"New"` (headers
     uppercase it). Replace the "Never 'NEW'" comment with the contract's "Never"
     sentence.
   - Add `RefsNewness` (`.arrived`, `.marked(Date)`) and these `RefsRanker` helpers:
     - `newness(_:signals:) -> RefsNewness?` implements the contract.
     - `isNew(_:signals:) -> Bool`.
     - `arrivedNew(_:signals:) -> Bool`: the unmarked arrival rule, ignoring marks. The
       toggle uses it.
     - `newSince(_:_:signals:) -> Date`.

     Keep `isUnopened` as the arrival rule's "never opened or annotated" check.

   - `browseSections`: move the NEW block directly after Just scanned and before Today.
     Candidates are unplaced rows with `newness != nil`, sorted by new-since desc then
     id, with no cap (or `prefix(newCap)`). Update the "a scanned row never also shows
     in…" comment.

2. **New `RefsNew.swift`** holds the model and the pure toggle logic:
   - `RefsNewMark { state: .new | .seen; at: Date }` (Codable state raw values `"new"`
     and `"seen"`).
   - `RefsNewMarkEntry { path, state, at }` for the file. It has a lossy decode: an
     unknown state drops that entry, not the file.
   - `RefsNewMarks` (Equatable, Sendable) maps note path → `RefsNewMark`. Build it from
     entries; on a duplicate path, the latest `at` wins.
   - `RefsNewToggle.result(for item: RefItem, signals: RefsSignals, now: Date) -> RefsNewToggleResult`
     returns either `.ineligible` or `.apply(mark: RefsNewMark?, becomesNew: Bool)`. A
     nil `mark` means remove. This implements the smallest-mark rule.
3. **`RefsStores.swift`**: add `RefsNewMarksFile` and `RefsNewMarkStore`, mirroring
   `RefsOpenLogFile`/`RefsOpenLogStore` line for line:
   - File `refs-new-marks.json` in the same Application Support directory, holding
     `{"schema_version": 1, "marks": [{"path", "state", "at"}]}` with the shared
     `RefsStoreCodecs`.
   - Quarantine file `refs-new-marks.corrupt.json`, plus the matching
     unreadable/save-failure messages.
   - Load never creates the directory.
   - `apply(path:mark:)` loads, replaces or removes that path's entry, prunes, and saves
     atomically, so concurrent writers can't lose each other's marks.
   - `rename(from:to:)` keeps each entry's state and instant.
   - Pruning drops marks older than 365 days, then keeps the newest 2000. Reuse the
     open-log constants' values; define separate constants.
4. **`RefsCaption.swift`**: `datePhrase` maps `.new` to the added phrase. `whyHere`
   `.new` follows the three caption forms above. A row in the NEW section that is no
   longer NEW gets the `Seen · …` line.
5. **`Sources/refs-rank/main.swift`**: add `--new-marks refs-new-marks.json`, which
   decodes `RefsNewMarksFile` into `signals.newMarks`, plus a usage-comment line. Browse
   output prints the `New` section where Just added used to print.

### BobMacCapture (macOS 26 targets)

6. **`RefsLibrary.swift`**
   - Inject a `RefsNewMarkStore` beside `openLogStore`, through every initializer and
     every preview or test factory.
   - Load it in `start()` into `signals.newMarks`.
   - Add `setNewMark(id:mark:)`: call `store.apply`, reload, set `signals.newMarks`, and
     bump `signals.now` synchronously, the way `recordOpen`/`publishOpenSignals` do.
   - In the successful `rename` branch, call `newMarkStore.rename(from:to:)` and reload,
     next to the open-log rename.
7. **`AppDelegate.swift`**: build the store from `RefsNewMarkStore.defaultFileURL()`,
   with the same `onError` logging the open log uses.
8. **`RefsKeyRouter.swift`**: in the `KeyCode.n` branch, add
   `if flags.subtracting(.capsLock) == .option { return context.markedTextPresent ? nil : .toggleNew }`.
   Update the doc comment.
9. **`RefsPanelModel.swift`**
   - Add `RefsCommand.toggleNew` and `perform` handling per the interaction design.
     Return true always.
   - Add an injectable `beep` closure.
   - Rename `RefsRowContent.isUnopened` → `isNew`, and use it everywhere.
   - Add `newSectionHint`.
   - Pass the selected row's NEW eligibility and state into `RefsActionsMenu.sections`,
     and handle the new action in `performAction` by running the same toggle path.
   - If `today_link = marks_seen`: see the callout below.
10. **`RefsActionsMenu.swift`**: add `.markSeen` / `.markNew` (titles "Mark as Seen" and
    "Mark as New", key `n`, mask `.option`). Add a `sections` parameter carrying the
    row's NEW state (none / NEW / not NEW). Insert the item after `.linkToday`.
11. **`RefsRowView.swift`**: the dot animation and title weight described above, plus
    the VoiceOver "new".
12. **`RefsListView.swift`**: the NEW header dot and trailing hint. Thread the model's
    hint into the `.new` header in both header call sites.
13. **`RefsInspectorView.swift`**: the New chip.
14. **`RefsVisualTokens.swift`**: the comment fix. Add named tokens for the dot size and
    animation timings if that reads cleaner than literals.

> [!decision] today_link = marks_seen In `finishTodayLink`, after a successful
> (non-guarded) link, run the toggle logic for that item only if it is currently NEW,
> recording the smallest mark that makes it not NEW. Add "or link it to Today from Bob
> Refs" to the contract's ways out, and add a model test for it. Links made elsewhere
> (Obsidian, Capture) do not clear NEW.

### Docs (bob-mac-capture `README.md`, Bob Refs section)

- **Sorting**: the new order, the NEW definition in brief, and new-since ordering
  instead of "Just added (…capped at 5)". Day counts now mention NEW.
- **Data**: the new `refs-new-marks.json` file, next to the open log.
- **The panel**: "the New dot", the NEW header (dot and ⌥N hint), the inspector New
  chip, and the VoiceOver example ending in "new".
- **Key table**: an Option-N row: "Mark the selected Ready or Next reference Seen, or
  New again; the row stays put until the next re-rank; other rows beep".
- **⌘K list**: Mark as Seen / Mark as New (⌥N).
- **Privacy**: the new file name.
- **Tuning**: `--new-marks`.
- **Keyboard** (the "Bob Refs shortcuts" table): the Option-N row.
- Add one short **"NEW"** paragraph or subsection with the contract's definition, the
  ways out, the "Never" sentence, and the smallest-mark rule.

## Tests

Update every existing reference to `.justAdded`, `justAddedWindowDays`, `justAddedCap`,
"Just added", `isUnopened` on row content, and "never opened" in labels. Known sites:

- `Tests/RefsCoreTests/RefsRankingTests.swift` (around lines 153, 212, 629, 902–938,
  1206).
- `Tests/BobMacCaptureTests/RefsPanelDesignTests.swift:134`.
- `Tests/BobMacCaptureTests/RefsPanelModelTests.swift:1692`.

Then add:

- **RefsCoreTests (ranking/newness)**:
  - Section order: Just scanned → New → Today. A Today-linked unopened row lands in New.
    A Just scanned row is not repeated in New.
  - With `all`, more than 5 NEW rows are all listed. With `cap5`, the renamed cap test
    applies.
  - New-since order: a mark instant beats an older arrival, timed beats untimed on the
    same day, id tie-break.
  - A `seen` mark hides an arrived row.
  - A `new` mark shows an old, annotated, opened-before row.
  - An open-log event or Spotlight last-used after a `new` mark clears it.
  - Read, Reading, and Dropped rows with a `new` mark are not NEW.
  - The window edge (keep the midnight-UTC local-day test, renamed).
  - The three why-here forms and the `Seen · …` form.
- **RefsCoreTests (toggle)**: smallest-mark table. Arrived → `seen` → removed. Old →
  `new` → removed. A pre-existing `seen` mark on an arrived row → removed. Ineligible
  states → `.ineligible`. ⌥N twice restores the original `isNew`.
- **RefsStoresTests**:
  - Missing file → empty, with no directory created.
  - Apply/replace/remove round trip.
  - Duplicate path → latest wins.
  - Pruning by age and count.
  - Rename keeps state and instant.
  - Corrupt file or wrong schema → quarantined, with the message.
  - Unknown state entry dropped.
  - Permissions 0600/0700.
- **RefsKeyRouterTests**: ⌥N → `.toggleNew`. ⌥N with Caps Lock → `.toggleNew`. ⌥N with
  marked text → nil. ⌥⇧N → nil. ⌃N and ⌘N unchanged.
- **RefsPanelModelTests**:
  - ⌥N on a NEW row flips `rowContent.isNew`, keeps `listing.orderedIDs` and
    `selectedID`, persists the mark, and the second press removes it.
  - The header hint switches "⌥N Mark Seen" ↔ "⌥N Mark New".
  - An ineligible row calls the injected beep once, returns true, and writes nothing.
  - With no selection, ⌥N returns true.
  - After `prepareForPresentation`, a seen row leaves New.
  - ⌘K sections show Mark as Seen / Mark as New / nothing per row state.
- **RefsLibraryTests**: `setNewMark` persists and publishes synchronously. A successful
  rename carries the mark to the new path.
- **RefsPanelDesignTests**: refresh fixtures to include a NEW section (header dot and
  hint), a row marked seen in place, and the inspector New chip. Keep the
  `ImageRenderer` fixture conventions already used there.

## Verification

Never land Swift you have not compiled (`bob-mac-capture/AGENTS.md`):

1. On athena (Linux, Swift present): `just portable-check` in the bob-mac-capture
   checkout. It lints every Swift file and builds and tests CaptureCore, RefsCore, and
   refs-rank. It must pass.
2. Smoke test the ranker against live data on athena:
   `bob ref list -R all -A -f json | swift run refs-rank`. Browse output must show `New`
   before `Today`, holding only Ready/Next rows from the window, uncapped (or capped per
   the decision). Then hand-write a one-entry `refs-new-marks.json` with a `seen` mark
   on one listed path, pass it with `--new-marks`, and confirm that row leaves New.
3. BobMacCapture targets (best effort, per that repo's `AGENTS.md`): rsync a scratch
   copy to the Mac without `.build`/`.git`, run `Scripts/bundle.sh` and
   `just typecheck-tests`, then delete the copy. Never touch Bryan's checkout or run
   `just install`. Use `sase memory read tailnet.md` for access. If the Mac is offline,
   say in the final report that the macOS targets are unverified.

## Memory

> [!decision] decision_record Add the bob-cli strand
> `sase/memory/decisions/refs-new-is-a-local-seen-state.md`: keyword "Refs NEW Is A
> Local Seen State", `origin: requested` only if Bryan asked, otherwise follow the
> `/sase_memory_write` routing. It contains:
>
> - **Applies to**: Bob Mac Capture's Bob Refs panel.
> - **Claim**: the contract's definition, the ⌥N smallest-mark toggle, the Mac-local
>   `refs-new-marks.json`, never vault/bob/reading state/review freshness, and distinct
>   from the review walk's NEW tier.
> - **Why, with rejected alternatives**:
>   - A vault field written through a new `bob` command: git churn per toggle, and it
>     would store review-like state in fields, which
>     `decisions:ready-is-freshness-gated` rules out.
>   - Moving the row immediately on ⌥N: breaks the frozen listing and the undo.
>   - Auto-advancing the selection: breaks toggle undo.
>   - Keeping "dot = never opened" with no age limit: about 200 legacy dots.
> - **Cost**: per-Mac only, and ⌥N no longer composes `˜`.
> - **Reopens when**: a second machine needs the same NEW state.
> - **Evidence**: this plan and the README section.
>
> Link `[[decisions:mac-capture-is-a-thin-client]]` and run `sase memory init`. Use
> `/sase_memory_write` before writing.

## Non-goals

- No bulk "Mark All Seen". The request is a per-reference toggle. File a task bead if it
  is wanted later.
- No change to Just scanned, Today, Reading, Next, Ready, Recently opened, Library, the
  footer hints, Newest, Tabs, search ranking, or any `bob` JSON contract.
- No change to the Settings "Reset Open History…" action. NEW marks are separate,
  explicit decisions.
