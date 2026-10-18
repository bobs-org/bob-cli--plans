---
tier: epic
title: "Bob Refs ⌘0 Newest: browse references by capture date"
goal: "In the Bob Refs panel, ⌘0 toggles a Newest mode: every reference in a
  date-grouped timeline, newest capture first, still filterable by typing. bob also
  reports a capture timestamp that is complete, rename-safe, and accurate to the minute
  where the vault knows the time.

  "
decisions:
  newest_persistence:
    ask: Should Newest stay on after the panel hides, until you turn it off yourself?
    choices:
      reset:
        Every show opens on the Library sections, like scope and Tabs; press ⌘0 again
      remember:
        Newest stays on across shows and relaunches; Esc and Backspace no longer clear
        it
    default: reset
    why:
      Opening fresh keeps Today and Just scanned one glance away, like every other
      filter
    answer: reset
  memory_added_time_record:
    ask:
      Add a decisions record that ref added times come from created or git birth, never
      backfilled?
    memory:
      - decisions
    default: false
    answer: false
phases:
  - id: bob-added-at
    title: "bob: reliable added_at timestamps in ref rows"
    depends_on: []
    size: medium
    description: "bob-added-at: add the additive `added_at` row field, make `-g` date
      every modern row (v2 included) from its rename-following git birth commit, and
      document the `added` precedence.

      "
  - id: refs-core-newest
    title: "RefsCore: Newest ordering, date groups, captions, refs-rank"
    depends_on: []
    size: medium
    description: "refs-core-newest: decode `added_at`, keep git instants in the
      snapshot, and add the pure Newest listing (date groups, sort keys, captions,
      why-here, section jumps) plus `refs-rank --newest`.

      "
  - id: refs-panel-newest
    title: "Panel: ⌘0 toggle, Newest token, headers, footer, README"
    depends_on:
      - refs-core-newest
    size: medium
    description:
      "refs-panel-newest: route ⌘0, hold the mode in the panel model (exclusive with
      Tabs, Esc and Backspace ladders), draw the token, date headers, footer hints, and
      inspector time, wire the git-lane merge, and document the mode."
proposed_by: bbugyi200.athena.11j
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-18 15:40:18
status: wip
---

- **PROMPT:**
  [prompts/202610/refs_newest_mode.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/refs_newest_mode.md)

# Plan: Bob Refs ⌘0 Newest — browse references by capture date

## Why this shape

Two things have to be true for "sort by capture date" to be trustworthy:

1. **Every reference has a capture date, and it does not move.** Today `bob ref list`
   derives a day-only `added` date (`src/native/ref_library/mod.rs` `added_date`):
   frontmatter `captured`, then the date part of `created`, then the zorg `source_block`
   date. On the live vault (1,073 rows), that leaves 282 modern rows undated until `-g`
   backfills them from git. Even with `-g`, 3 v2 chat rows stay undated, because
   `fill_git_dates` skips v2 rows entirely. The git pass keys on the oldest commit that
   touched a `^ref` line _at the current path_, so a note renamed with `bob ref rename`
   can lose its history or take the rename date as its added date. In Newest, that would
   make an old note jump to the top of the timeline.
2. **Same-day captures order by time.** The panel captures 10–20 references on a busy
   day. Today, ties within a day fall back to title order. The vault already knows the
   time for most recent notes: frontmatter `created` is a full local datetime stamped
   once by the scan pipeline for every note it creates (since 2026-10-03; 187 notes),
   and the vault's git commits land roughly every 15 minutes.

So the epic is one bob-cli phase plus two Mac phases. The bob phase makes the timestamp
complete and precise. RefsCore adds the pure ordering and grouping. The panel phase adds
the key, the visuals, and the docs. The Mac phases follow the thin-client contract: bob
owns the date, and the app only orders, groups, and presents it.

"Capture date" means bob's `added` date: the day the reference entered the library
(web-capture day, else the day the scan wrote the note, else the zorg block day, else
git birth). The panel keeps its existing word, "added" ("added Oct 7", the Inspector's
Added fact), and the mode is called **Newest**.

## UX specification (shared by the Mac phases)

### Key and mode

- **⌘0** (ANSI 0, key code 29) toggles Newest. It is free in the panel: ⌘1–⌘5 are the
  scopes and nothing else uses 0. The local key monitor already delivers ⌘-digits to the
  non-activating panel. Caps Lock never blocks it. While IME-composing, it passes
  through. Shift-⌘0 and the keypad 0 are not bound.
- **Membership never changes, only order.** ⌘0 shows exactly the rows the current scope
  and query show without it, re-ordered and grouped by date. Scope (⌘1–⌘5) composes with
  Newest.
- **Tabs and Newest are exclusive views.** The Tabs view keeps its purpose-built order
  (current tab, then tab recency), and unmatched or colliding tab rows have no date. ⌘0
  in Tabs leaves Tabs and enters Newest; ⌘T in Newest leaves Newest and enters Tabs. The
  query and scope survive either switch, as they do for ⌘T today.
- Toggling re-ranks and selects the first row (the newest capture), like ⌘1–⌘5. It is
  unanimated, like every other re-rank. VoiceOver announces "Newest first" or "Library
  sections".

### Timeline (empty query)

All scoped references, sorted by:

1. `added` day, newest first;
2. then `added_at` instant, newest first, with timed rows before untimed rows on the
   same day;
3. then id (note path) ascending.

Rows without `added` come last, in id order. Consecutive rows form date groups, using
the panel calendar (`signals.now`, `signals.calendar`), so groups never shift a day in
US evenings:

| Group               | Rule                                  | Pinned header   | Trailing detail |
| ------------------- | ------------------------------------- | --------------- | --------------- |
| Today               | day ≥ today (future dates clamp here) | `TODAY`         | `Oct 18`        |
| Yesterday           | 1 day ago                             | `YESTERDAY`     | `Oct 17`        |
| Weekday             | 2–6 days ago                          | `THURSDAY`      | `Oct 16`        |
| Month               | older, same year as today             | `OCTOBER`       | —               |
| Month, earlier year | older, earlier year                   | `DECEMBER 2025` | —               |
| Undated             | no `added`                            | `UNDATED`       | —               |

The weekday window matches the existing caption rule in `RefsCaption.absoluteDay`
(weekday for 2–6 days ago), so headers and captions speak one language. The weekday
groups and the month group can share a month: on Oct 18, the `OCTOBER` group holds Oct
1–11. Headers keep today's style: 26 pt, pinned, uppercase title with the count after
it. The trailing detail is drawn the way the Just scanned header draws its relative time
(`caption2`, monospaced digits, tertiary). There is no Just scanned or Today-ledger
section in Newest: a scan's new notes carry `created` = scan time, so they land at the
top of TODAY on their own. The pink Today pill, the unopened dot, and the trailing marks
render on rows exactly as they do now.

Sketch (880 pt, list column):

```
🔍 Search references              [🗓 Newest ✕]        14 open · 1073
TODAY 3                                                       Oct 18
 •▣ Attention Is All You Need         Paper · Vaswani · added 9:41 AM · 15 pp
  ▣ Agent memory survey               Article · added 8:02 AM
YESTERDAY 18                                                  Oct 17
  …
OCTOBER 61
  ▣ Context engineering notes         Chat · added Oct 7
  …
DECEMBER 2025 22
UNDATED 3
```

### Typing filters, Newest keeps order

With a query, membership is exactly `RefsRanker`'s tiered search (T0–T3, every token
must match). Rows then sort and group exactly as in the timeline, under the same date
headers. Title, stem, and secondary highlights still render. The count reads "`k of M`".
The first row is the newest match.

### Row captions in Newest

The caption keeps its current shape: kind, author, date phrase, pages, then the matched
part when searching. Only the date phrase changes, to name the date that explains the
row's position:

- Day groups (Today, Yesterday, weekdays): `added 9:41 AM`, the short time of `added_at`
  in the panel calendar's time zone and locale. Untimed rows show no date phrase,
  because the header already names the day.
- Month groups: `added Oct 7` (`RefsCaption.absoluteMonthDay`).
- Undated: no date phrase.
- Git-sourced dates keep the existing `≈` mark: `added ≈ 3:12 PM`, `added ≈ Sep 27`.

### Inspector

- Why-here in the Newest timeline: `Added Oct 16 at 9:41 AM`, `Added Oct 16` (untimed),
  `Added ≈ Sep 27 at 3:12 PM · from vault history` (git), or
  `Undated · bob has no added date for this note`.
- Newest search keeps the existing search why-here (the match reason).
- The Added fact shows the time whenever `added_at` exists, in every mode:
  `Oct 17 at 9:41 AM (created)` and `≈ Sep 27 at 3:12 PM`. The existing source suffix
  rules are unchanged.

### Search bar token

A Newest token sits in the Tabs token's slot, before the scope token. It is an accent
capsule with SF Symbol `calendar.day.timeline.left` (fall back to `calendar` if the
symbol does not resolve), the text "Newest", and an `xmark`. Activating it turns Newest
off. Its accessibility label is "Sorted newest first. Activate to show library
sections."

### Ladders and footer

> [!decision] newest_persistence = reset

- **Esc ladder:** cards → banner → query → Tabs **or Newest** → scope → close.
- **Backspace on an empty query:** removes the Tabs or Newest token first, then the
  scope token.
- `prepareForPresentation` turns Newest off on every show, like scope and Tabs.

> [!decision] newest_persistence = remember

- Newest is a remembered preference rather than a removable filter. Esc goes cards →
  banner → query → Tabs → scope → close, and Backspace removes Tabs then scope; neither
  ever clears Newest. Only ⌘0 and the token's ✕ turn it off.
- `prepareForPresentation(source: .library)` restores the remembered value; a
  `.highlightsTabs` show opens Tabs and leaves the preference untouched.
- The preference persists in `UserDefaults` (key `refsNewestOrder`) through an
  injectable store, so tests use a throwaway suite.

**Footer hints** (`RefsFooter.hintVariants`, widest first, using `ViewThatFits`):

- Library: core + `⌘S Scan  ⌘0 Newest  ⌘1–5 Scope  ⌘T Tabs  esc Close`, then core +
  `⌘S Scan  ⌘0 Newest  esc Close`, then core + `esc Close`.
- Newest: the way back is in every variant, as Tabs always shows "⌘T Library". Core +
  `⌘S Scan  ⌘0 Library  ⌘1–5 Scope  ⌘T Tabs  esc Close`, then core +
  `⌘0 Library  esc Close`, then core + `⌘0 Library`.
- The Tabs ladder is unchanged.

**Section jumps:** Option-↑/↓ move between date groups in Newest whether or not a query
is typed. In Library search they still pass through, as today.

## Phase bob-added-at — reliable `added_at` timestamps (bob-cli repo)

Work in this repository (bob-cli). Keep the change additive under `schema_version` 1: no
human or Markdown output changes, and no CLI option changes.

**Row field.** In `src/native/ref_library/row.rs`, add `added_at: Option<String>` right
after `added_source`. It serializes as `null` when absent, like its neighbors. Update
every `RefRow` literal (for example `rename/plan.rs`, `show.rs`, `output.rs` tests).

**Index-time rule** (next to `added_date` in `src/native/ref_library/mod.rs`):
`added_at` is the most precise instant known on the row's `added` day.

- Set it when frontmatter `created` parses as a full datetime with an offset. Accept the
  scan's `%Y-%m-%dT%H:%M:%S%z` form (`2026-10-17T09:12:44-0400`) and RFC 3339. The date
  part, in its own offset, must also equal the row's `added`.
- Emit RFC 3339 with seconds and a numeric offset: `2026-10-17T09:12:44-04:00`.
- Outcome by source: `created` rows always get it when the value parses. `captured` rows
  get it when `created` falls on the same day. `zorg_block` rows and unparseable or
  date-only values get `null`.

**`-g` git birth pass** (`src/native/ref_library/list.rs` `fill_git_dates`):

- Every **modern** row (v1 _or_ v2) whose `added` is still null gets three values from
  the note's birth commit: `added` (the date part of the instant as written, matching
  today's `%cs`), `added_source: "git"`, and `added_at` (that commit's `%cI`).
- **Birth** means the oldest commit that added the file, following renames. Use one
  oldest-first `git log --reverse --format=%x1e%cI --name-status -M --diff-filter=AR`
  over the ref dir, with `-z` or `-c core.quotePath=false` so non-ASCII and spaced paths
  match row paths.
- An `A` record sets a path's birth only if the path has none yet. An `R` record gives
  the new path the old path's birth, or the rename commit's time if the old path has
  none.
- Strip the vault prefix with the same logic `load_ref_history` uses, so a vault nested
  in a larger repo still matches.
- Legacy rows are untouched. The `finished` backfill keeps its current `^ref` pass and
  stays v1-only, so the v2 `finished` guarantee in
  `tests/cli/ref_library/tasks.rs::git_dates_never_backfills_v2_rows` still holds.
  Rename that test so it names what it pins (finished).
- Run each pass only when some row needs it. If either pass fails, set
  `coverage.git_dates: "unavailable"` and print one stderr warning. This is never fatal.
- Cost: the birth pass measured about 0.5 s on the live vault, comparable to the
  existing `^ref` pass.

**Forward guarantee.** Every new ref note already gets `created` from the sync executors
(`metadata.created = Some(new_note_created_timestamp())` in `highlights_ref/sync.rs`).
Confirm that each creation route (v1 and v2 executors) reaches it. Add or extend a test
that pins `created` on a freshly scanned note, so `added_at` can never silently regress
for new captures.

**Docs** (`docs/ref.md`):

- Document the `added` precedence (captured → created date → zorg block → git birth with
  `-g`) and the `added_source` values (`captured`, `created`, `zorg_block`, `git`).
- Document `added_at` and its same-day rule.
- Describe the rename-following birth pass.
- Replace "`-g/--git-dates` never backfills v2 rows" with the accurate statement: `-g`
  dates v2 rows but never backfills a v2 row's `finished`.
- Add `added_at` to the JSON row field list.

**Tests:**

- Unit: `added_at` from both `created` forms; a different-day `created` gives null; a
  `captured`-plus-`created` same day gives the `created` instant; zorg and date-only
  values give null.
- CLI (`tests/cli/ref_library/list.rs`), using temp git repos:
  - `-g` dates a v2 row;
  - a note renamed in a later commit keeps its original birth `added`/`added_at`;
  - a non-ASCII or spaced note path is dated;
  - `added_at` is present (null) without `-g`;
  - legacy rows are untouched;
  - git-unavailable behavior is unchanged.

**Verify:**

- `just check`.
- Against the live vault, read-only:
  `cargo run --release -- ref list -R all -A -f json -g`. Report the count of null
  `added`, which should be 0, down from 3. Report the count of non-null `added_at` and
  the `added_source` distribution.

> [!decision] memory_added_time_record

Use `/sase_memory_write`, then add a `decisions` strand titled "Reference Added Times
Are Derived, Never Backfilled". It records that bob derives `added`/`added_at` at read
time from `captured`/`created`/zorg/git birth, and never writes inferred dates into note
frontmatter. Its rejected alternative is a one-time `created` backfill of ~282 notes.
Cite this plan and the phase commit as evidence. Then run `sase memory init`.

## Phase refs-core-newest — pure Newest ordering (bob-mac-capture repo, RefsCore)

Open the `bob-mac-capture` linked repo with `/sase_repo` and read its `AGENTS.md`. This
phase changes only `Sources/RefsCore`, `Sources/refs-rank`, `Tests/RefsCoreTests`, and
the README's Data, Sorting, and Tuning subsections. All of these are verifiable with
`just portable-check` on a Linux host with Swift. It needs nothing from bob-added-at:
build fixtures from the contract above.

**Data:**

- `RefRecord.addedAt` decodes `added_at` with `decodeIfPresent`; an older bob decodes it
  as nil.
- `RefItem.addedAt: Date?` is parsed once at catalog build (`RefItem.make`) with
  `ISO8601DateFormatter` `.withInternetDateTime`. A malformed value becomes nil and
  never fails decoding.
- Git rows take their instant from a new `RefsSnapshot.gitAddedAt: [String: String]`,
  persisted as `git_added_at` with decode-if-present defaulting to empty.
- Carry `addedAt` through `RefsCatalog.items`' disambiguator rebuild.
- `pathsNeedingGitDates` also lists dateless rows whose git date has no instant yet. An
  old cache then refills once.
- Add a pure `RefsSnapshot.mergingGitDates(from records: [RefRecord], needing:)` that
  merges git-sourced `added` and `added_at` for still-dateless paths. The app's `-g`
  lane switches to it in the panel phase.

**Ordering and groups** (`RefsRanking.swift`):

- Add `public enum RefsListOrder { case ranked, newest }`.
- `RefsRanker.listing(_:query:scope:order:signals:)` gains
  `order: RefsListOrder = .ranked`, so every existing call site keeps today's behavior.
- `RefsListing` gains `order` (default `.ranked`); `refreshingContent` carries it.
- For `.newest`, browse membership is all scoped items, and search membership is
  `searchRows`' ids, with `matches` kept. Both then sort and group by the timeline rules
  above.
- Add `RefsSectionKind.added(RefsAddedGroup)`. Drop the unused `CaseIterable`; nothing
  enumerates the kinds.
- `RefsAddedGroup` carries a stable id (`day-2026-10-16`, `month-2026-09`, `undated`),
  `title`, optional `detail`, and an `accessibilityLabel`:
  - "Added today, October 18"
  - "Added Thursday, October 16"
  - "Added in October"
  - "Added in December 2025"
  - "Undated"
- These strings are computed from `signals.now` and `signals.calendar` at listing time.
  `RefsSectionKind.title` returns the group title.

**Captions and why-here** (`RefsCaption.swift`):

- Implement the Newest date phrases and the why-here forms above.
- Format time with `Date.FormatStyle` short time, using the panel calendar's time zone
  and locale. Tests pin `en_US_POSIX` and `America/New_York`.
- Gate the matched stem or secondary caption part on `match != nil` instead of
  `section == nil`, so Newest search keeps it. This is equivalent for every existing
  listing.

**Selection** (`RefsSelection.swift`): `moveSection` also works when
`listing.order == .newest`, in browse or search.

**`refs-rank --newest`:** print the Newest listing for live tuning, in browse or search.
Show one header line per group (title, count, detail), then one row per line: `added_at`
(or the bare day), state, kind, title. Document the flag in the README's Tuning
subsection.

**README:**

- Data: `added_at`, and the `git_added_at` cache key.
- Sorting: add a Newest paragraph with the timeline order, the group table, and "⌘0
  changes order, never membership".
- Tuning: `--newest`.

**Tests** (`Tests/RefsCoreTests`; fixed now, `America/New_York`, `en_US_POSIX`):

- Group boundaries: today, a future date, yesterday, 2 and 6 days ago, 7 days ago into
  the month group, the month boundary inside the weekday window, and the year suffix for
  earlier years.
- Sort keys: timed before untimed on the same day, instant desc, id tie-break, undated
  last.
- Scope filtering.
- Newest search membership equals ranked search membership for several queries,
  including T2 and T3.
- Highlights are retained.
- Captions and why-here strings, including `≈`.
- `added_at` decoding: present, absent, and malformed.
- Snapshot round-trip with and without `git_added_at`.
- `mergingGitDates`.
- `moveSection` in Newest search.
- Existing golden ranking tests stay green, unchanged.

**Verify:**

- `just portable-check`.
- When bob-added-at has landed, also pipe `bob ref list -R all -A -f json -g` into
  `swift run refs-rank --newest`. Report the group headers and confirm UNDATED is empty.

## Phase refs-panel-newest — the ⌘0 experience (bob-mac-capture repo, app target)

Implement the UX specification in `Sources/BobMacCapture/Refs`.

**Router** (`RefsKeyRouter.swift`):

- Add `KeyCode.zero = 29`. ⌘0 (Caps Lock ignored) returns a new `.toggleNewest` command;
  with marked text it returns nil.
- `RefsKeyContext` gains `newestOn`. Backspace on an empty query fires when Tabs,
  Newest, or a non-All scope is on (only Tabs and scope under `remember`).
- Option-↑/↓ section moves are allowed in Newest even while searching. Replace
  `listModeIsBrowse` with a `sectionJumps` flag that `RefsPanel` computes as browse mode
  or Newest.

**Model** (`RefsPanelModel.swift`):

- Add `@Published private(set) var listOrder: RefsListOrder = .ranked`, plus commands
  `.toggleNewest` and `.setNewest(Bool)`.
- Entering Newest from Tabs runs the same teardown as `setTabsFilter(false)` without its
  re-rank, then re-ranks once. `setTabsFilter(true)` sets `.ranked`.
- Pass `order: listOrder` to every `RefsRanker.listing` call site:
  `rerankSelectingFirst`, `freshListingKeepingSelection`, `freshListing(selecting:)`,
  `buildFreshListingForScan`, and the preview installer, which gains a `listOrder`
  parameter for fixtures.
- Implement the Esc and Backspace ladders and the `prepareForPresentation` behavior for
  the chosen `newest_persistence` branch.
- A scan that completes in the Newest timeline selects the first created row, as the
  existing logic does.
- Post the toggle announcement.

**Views:**

- `RefsSearchBar`: the Newest token.
- `RefsListView`/`RefsSectionHeader`: added-group title, count, trailing `detail`, and
  the group `accessibilityLabel` with ", N references".
- `RefsFooter`: the hint ladders.
- `RefsInspectorView`: the Added fact with time.

**Git lane** (`RefsLibrary.runGitDatePass`): replace the inline merge loop with
`RefsSnapshot.mergingGitDates` so git instants persist.

**README** (Bob Refs → The panel, keyboard table):

- Add
  `Command-0 | Toggle Newest: every reference by date added, newest first, grouped by day then month; typing still filters`.
- Update the Esc row (`…banner, query, Tabs or Newest, scope, close`), the Backspace
  row, and the Option-Up/Down row ("browse and Newest").
- Describe the token and headers in "The panel" prose.

**Tests:**

- `RefsKeyRouterTests`: ⌘0, Caps Lock, marked text, Shift-⌘0 unbound, Backspace with
  Newest, Option-↑ in Newest search.
- `RefsPanelModelTests`:
  - toggling selects the first row, and membership is unchanged with and without a
    query;
  - exclusivity in both directions keeps query and scope;
  - the Esc and Backspace ladders and presentation follow the chosen branch;
  - ⌘R keeps the selection in Newest;
  - a scan selects the created row in Newest.
- `RefsFooterTests`: both ladders exactly.
- `RefsPanelDesignTests`: add render states for the Newest timeline (TODAY, YESTERDAY, a
  weekday, OCTOBER, a prior-year month, UNDATED), Newest search with a scope token and
  highlights, and the narrow 700 pt list. Assert that the token symbol resolves.

**Verify:**

- `just portable-check`.
- Per the repo `AGENTS.md`, compile a scratch copy on the Mac (best effort): rsync
  without `.build`/`.git`, run `Scripts/bundle.sh` and `just typecheck-tests`, then
  delete the copy. Never use Bryan's checkout or `just install`. If the Mac is offline,
  say in the final report that the macOS targets are unverified.
- CI runs the XCTest suites.

## Rejected alternatives

- **Backfilling `created` into ~282 older notes.** Rejected because it writes inferred
  values into a set-once-at-creation field, churns the vault, and hides provenance.
  Read-time derivation keeps `added_source` honest, and git is already the vault's
  durable record.
- **Sorting by `published`.** That is publication date, not capture.
- **Using ref-jobs `created_at`.** It covers only `bob capture` URL jobs, while
  `created` (scan time) and git commits are within the ~15-minute cadence for every
  route.
- **Composing Newest with Tabs.** Tabs has its own tab order, and synthetic tab rows
  have no date.
- **Relevance order or best-match selection inside Newest search.** Either would break
  "⌘0 changes order, never membership" and make the first row unpredictable.
- **Finder-style "Previous 7 Days / Previous 30 Days" buckets.** Named weekdays, then
  calendar months, match the panel's existing caption rules and make Option-↑/↓ jump by
  meaningful units.
- **A ⌘K menu entry.** The ⌘K menu holds per-row actions; Tabs (⌘T) has no entry either.
  Discoverability comes from the footer hint and the token.
