---
tier: epic
title: 'Bob Refs: a quick-open panel in Bob Mac Capture that opens reference PDFs
  in Highlights'
goal: 'One keystroke (⌃⇧⌘R anywhere, or the Open… key while Highlights is frontmost)
  shows a prewarmed glass panel listing every PDF-backed reference note from `bob
  ref list`, with kind and reading state on every row, the working set first when
  the query is empty, tiered relevance when it is not, rows that never move under
  the cursor, and a kind-adaptive inspector. Return opens the original PDF in Highlights
  and changes nothing in the vault.

  '
decisions:
  highlights_open_key:
    ask: Which key should open Bob Refs while Highlights is the frontmost app?
    choices:
      cmd_o: Take over ⌘O, Highlights' stock Open… key; File › Open… stays in its
        menu
      ctrl_o: Take over ⌃O, exactly as written in the request; ⌘O keeps Highlights'
        dialog
      global_only: No takeover; only the global ⌃⇧⌘R opens Bob Refs
    default: cmd_o
    why: Highlights' Open… is ⌘O and the dotfiles remap neither key; Settings can
      change it anytime
    answer: cmd_o
  refs_decision_memory:
    ask: Add a decision record extending the thin-client rule to Bob Refs (opening
      never mutates the vault)?
    memory:
    - decisions
    default: false
    answer: false
phases:
- id: cli-blocked
  title: bob-cli exposes Blocked on ref rows
  depends_on: []
  size: small
  description: 'cli-blocked: add an always-present `blocked` boolean to `bob ref list/show/find`
    rows (true when the note''s single ^ref tracker is Blocked [?]), document it in
    docs/ref.md as an overlay on the reading lane, and test it.'
- id: mac-groundwork
  title: Hotkey registry and CI render artifacts in Bob Mac Capture
  depends_on: []
  size: small
  description: 'mac-groundwork: replace the single-key HotKeyManager with a HotKeyRegistry
    that routes by EventHotKeyID, keep Capture''s hotkey working through it, add a
    shared PNG render helper for design tests, and make CI render and upload design
    fixtures as an artifact.'
- id: refs-core-model
  title: RefsCore target — decoding, item model, fetcher, and stores
  depends_on:
  - cli-blocked
  size: medium
  description: 'refs-core-model: add the Foundation-only RefsCore target with lossy
    decoding of `bob ref list` and `bob plan` JSON, the RefItem/RefKind/RefState/RefScope
    model, the BobRefsFetcher over BobProcessClient, the snapshot cache and open log
    stores, fake-bob ref/plan branches, synthetic fixtures, and tests.'
- id: refs-core-ranking
  title: RefsCore ranking — browse sections, search tiers, stability, explanations
  depends_on:
  - refs-core-model
  size: medium
  description: 'refs-core-ranking: implement browse sections, tiered search scoring
    with named constants, captions, the why-here explanation, content-only refresh
    of a frozen listing, golden tests over a synthetic library, and the refs-rank
    tuning CLI.'
- id: refs-panel-model
  title: Refs library service and panel model
  depends_on:
  - refs-core-ranking
  size: medium
  description: 'refs-panel-model: build the RefsLibrary refresh service (cache, watcher,
    git-date pass, Today, Spotlight sweep, missing PDFs, open log) and the RefsPanelModel
    (query, scope, frozen listing, selection, commands, open dispatch, banners) with
    fake-bob tests.'
- id: refs-panel-ui
  title: Refs panel window, list, basic inspector, and keyboard
  depends_on:
  - refs-panel-model
  - mac-groundwork
  size: medium
  description: 'refs-panel-ui: build the borderless non-activating glass panel, search
    bar, two-line rows, section headers, basic inspector, footer, empty and error
    states, key router, motion and accessibility, with geometry, router, and rendered
    design tests checked from the CI artifact.'
- id: refs-entry-points
  title: Hotkeys, Highlights takeover, menu, settings, and coexistence with Capture
  depends_on:
  - refs-panel-ui
  - mac-groundwork
  size: medium
  description: 'refs-entry-points: wire the library and panel into AppDelegate, register
    the global ⌃⇧⌘R hotkey and the Highlights-frontmost takeover, add the References
    status-menu item and Settings section, keep only one Bob panel visible, refresh
    Today after captures, and test it all.'
- id: refs-inspector
  title: Kind-adaptive inspector and actions menu
  depends_on:
  - refs-panel-ui
  size: medium
  description: 'refs-inspector: upgrade the inspector with PDFKit thumbnails or outlines
    by kind, Bottom-line and abstract excerpts, reading-time estimates, commented
    highlights from `bob ref show -c`, open history, and a ⌘K actions menu, all lazy,
    cancellable, and cached.'
- id: refs-closeout
  title: README coherence, optional memory record, final CI, and Bryan's checklist
  depends_on:
  - cli-blocked
  - refs-entry-points
  - refs-inspector
  size: small
  description: 'refs-closeout: make the README''s Bob Refs section coherent, apply
    or record the memory decision, confirm final CI and render fixtures, record proposed
    follow-ups, and leave Bryan a Mac verification checklist.'
proposed_by: bbugyi200.apollo.5z
decided_by: auto
create_time: 2026-10-08 19:32:39
status: done
bead_id: bob-cli-5s
---

- **PROMPT:** [prompts/202610/bob_refs_panel.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/bob_refs_panel.md)
- **BEAD:** [bob-cli-5s](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-5s/README.md)

# Bob Refs: a quick-open panel that opens reference PDFs in Highlights

## Why

Finding a reference PDF through Highlights' Open… dialog is slow. The dialog shows about
310 snake_case filenames in `lib/chat/`, with no title, kind, reading state, or date,
and about ten new agent reports arrive every day. Highlights has no library view, no
quick-open, and no scripting. Bob already knows every fact the dialog lacks.

The consolidated research report
`research:202610/highlights_quick_open_refs_panel/highlights_quick_open_refs_panel.md`
(2026-10-08, five researchers plus a lead) recommends a second hotkey panel inside Bob
Mac Capture. The panel is a thin client of `bob ref list` and opens the original PDF in
Highlights. This epic builds that panel. It follows the research except where
[Departures from the research](#departures-from-the-research) says otherwise.

Live numbers measured on 2026-10-08 while writing this plan:

- **References:** 342 PDF-backed notes. By kind: chat 314, docs 11, blogs 9, papers 8.
- **States:** read 295, abandoned 19, ready 13, next 8, wip 7. One row is Blocked `[?]`.
- **Dates:** only 60 rows carry `added` without `-g`.
- **Titles:** 74 titles contain backtick code spans.
- **Cost:** `bob ref list -R all -A -f json` takes 0.12 s and returns 743 KB. `-g` adds
  about 1.3 s on the Mac. `bob plan -f json` takes 0.74 s.

## The experience (north star)

1. **Open from Highlights.** Bryan is reading in Highlights and presses ⌘O.
   - Instead of the Open… dialog, the Bob Refs panel appears near the top of the screen:
     a Liquid Glass slab whose large search field is already focused.
   - It appears in well under 50 ms because it is prewarmed and paints from memory.
   - Highlights stays the active app.
2. **The empty-query list shows his working set in intent order.**
   - The sections are **Today**, **Just added**, **Reading**, **Next**, **Ready**,
     **Recently opened**, and then the whole **Library**, newest activity first.
   - Today holds the reports linked under today's BLOG Pomodoro. Each wears a pink
     `BLOG` pill.
   - Just added holds this morning's unopened reports, shown semibold with a blue dot.
   - Every row says what the reference is (a tinted kind tile: Chat, Paper, Article,
     Doc) and where it stands (the vault's own status glyph).
   - The right half of the panel describes the selected reference.
3. **Typing collapses the sections into one ranked list.**
   - He types `omni`. Matched letters light up in the accent color.
   - The report he is reading today floats above an older finished one with the same
     word in its title.
   - A weak fuzzy match never beats a real title-word match because it happens to be
     recent.
4. **Return opens the PDF.** The panel vanishes and the PDF opens, or comes forward, in
   Highlights. Nothing in the vault changes.
5. **Other keys:**
   - ⌘↵ opens the reference note in Obsidian.
   - ⌥↵ reveals the PDF in Finder.
   - ⌘2 narrows the list to Chats.
   - Esc clears the query, then the scope, then closes the panel.
6. **From anywhere else on the Mac,** ⌃⇧⌘R opens the same panel.

## Departures from the research

| Research said                                     | This plan does                                                                       | Why                                                                                                                                                                                                                            |
| ------------------------------------------------- | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| A `task_status` field such as `blocked` or `next` | A boolean `blocked` on every row                                                     | Blocked is a derived overlay on the reading lane, not a status (`decisions:task-lanes-are-sticky`; `plan:202610/blocked_ref_task_status.md` rejected an authored blocked status). A boolean says exactly that and nothing more |
| A user-resizable width that is remembered         | A fixed 880 × 560 pt panel, clamped to the screen; it is list-only below 760 pt wide | Spotlight-style panels don't resize. One less persisted state to break. 880 pt fits about 50-character titles                                                                                                                  |
| Article tint pink                                 | Article tint brown                                                                   | Pink is the Pomodoro and Today hue in capture (the NOW pill), and the Today pill uses it here                                                                                                                                  |
| Click accepts, as in capture's pickers            | Click selects, double-click opens                                                    | This list exists to browse and inspect. Capture's pickers insert                                                                                                                                                               |
| ⌘Y Quick Look                                     | Dropped                                                                              | `QLPreviewPanel` needs responder-chain control that fights a non-activating panel. The inspector is the preview, and Highlights is one keystroke away                                                                          |
| Live `NSMetadataQuery`                            | A one-shot Spotlight metadata sweep after each snapshot refresh                      | It only needs last-used date and page count. A sweep is simpler and cannot leak a live query                                                                                                                                   |
| Acronym bonus inside the word tier                | No acronym rule                                                                      | FuzzyMatcher's boundary bonuses already rank initials well inside the fuzzy tier                                                                                                                                               |
| ⌘K "Capture follow-up…", "Play narration"         | Omitted, or "Open Narration" (default app)                                           | Cross-panel draft seeding is follow-up work                                                                                                                                                                                    |

## Ground rules (every phase)

- **Repositories.**
  - App code lives in the linked repo **bob-mac-capture**. Open it with
    `sase repo open bob-mac-capture -r "<reason>"`.
  - If that fails because the linked repo's primary checkout is missing on this host,
    use `sase repo open gh:bobs-org/bob-mac-capture -r "<reason>"` instead.
  - Either way, work only in the path the command prints, and read that repo's
    `AGENTS.md` if it names one.
  - The `cli-blocked` phase and the closeout memory step live in **bob-cli** (this
    project).
- **Thin client (`decisions:mac-capture-is-a-thin-client`).**
  - `bob` owns every reference fact: which references exist, kind, status, Blocked,
    dates, annotations, and Today.
  - The app filters, ranks, sections, and presents what `bob` returns. It also reads PDF
    intrinsics (pages, outline, thumbnail, text excerpts) with PDFKit; those are file
    facts, not vault semantics.
  - The app never parses frontmatter, note bodies, or `reading_state_source`, and never
    writes the vault.
- **Opening never mutates.**
  - Opening a reference never changes reading state, task lanes, review state, or any
    vault file (`decisions:task-lanes-are-sticky`; an open is a retrieval, not a status
    gesture).
  - The open history lives only in the app's Application Support directory.
- **JSON contract.**
  - The `schema_version` values the app expects are: `ref list` and `ref show` 1,
    `plan` 2. Reject any other version; treat it like any other refresh failure.
  - Decode every optional field with `decodeIfPresent`, so an older `bob` still works.
    For example, a `bob` without `blocked` decodes every row as not blocked.
  - Ignore unknown fields.
  - Read stdout fully. `bob` panics on a broken pipe.
- **No Swift toolchain on agent hosts.**
  - GitHub Actions (`.github/workflows/ci.yml`, macOS 26) is the only compiler for the
    AppKit targets, so write conservatively:
    - Swift 5 language mode;
    - `@MainActor` on every AppKit-touching type;
    - `@available(macOS 26.0, *)` on views, as capture does;
    - long-stable APIs only;
    - swift-format default style (4-space indent, lines ≤ 100 columns);
    - no long `+` expression chains (Linux Swift 6.0.3 type-checks them slowly).
  - If a Swift toolchain exists on the host (athena has had Linux Swift 6.0.3), run
    `swift test` on a scratch copy of the tree. Linux builds only `CaptureCore`,
    `RefsCore`, and `refs-rank`.
  - Best effort: if `ssh mac` answers, copy the tree to a temporary directory there, run
    `./Scripts/xcode-swift.sh build`, then delete the copy. The Mac has only the Command
    Line Tools, so it has no XCTest.
- **Commit and CI loop for bob-mac-capture phases.** This plan explicitly authorizes
  every bob-mac-capture phase to do the following:
  1. Commit to `master` with `/sase_git_commit` from the bob-mac-capture checkout, using
     conventional headers (`feat(refs): …`, `refactor(hotkeys): …`).
  2. Find the CI run with
     `gh run list -R bobs-org/bob-mac-capture --workflow CI --commit <sha> --json databaseId`.
  3. Watch it through `/sase_monitor`
     (`gh run watch <id> -R bobs-org/bob-mac-capture --exit-status`).
  4. Fix forward until the whole job is green. On failure, read
     `gh run view <id> --log-failed` and grep for ` error:`.

  A phase is not done until its CI run is green. Its bead note records the run URL and
  SHA.

- **Look at the pixels.**
  - After `mac-groundwork`, CI uploads every design-test PNG as the artifact
    `render-fixtures`.
  - Every phase that changes Refs visuals downloads it
    (`gh run download <id> -R bobs-org/bob-mac-capture -n render-fixtures -D <tmpdir>`),
    opens its Refs PNGs with the Read tool, and checks them against this spec before
    declaring done.
  - Fix misalignment, clipping, contrast, or truncation it sees.
- **Shared contracts.**
  - Later phases depend on the type and method names in this spec. Keep them.
  - If a compile constraint forces a rename, note it on the phase bead as
    `INTERFACE CHANGE:` so the dependent phases see it.
- **Documentation.** Each bob-mac-capture phase documents its slice in one README
  section, `## Bob Refs`, which the first Refs phase creates. Document the user-facing
  contract, not implementation trivia.
- **Privacy.**
  - Signposts carry names only, never titles or paths.
  - The open log stores note paths and timestamps only: mode 0600, in a 0700 directory.
  - The snapshot cache follows the same file-mode pattern as
    `FileCanceledDraftStashStore`.
- **Tests match their neighbors.** Each test file keeps its own `waitUntil`, fixture
  loading via `#filePath`, and `fakeBobPath()` patterns, the way its neighbors do.
  Fixtures are compact JSON with a trailing newline.
- **Never commit real vault titles.** Every fixture uses synthetic titles and paths.
  Keep the live key set and key order (copy them from one live
  `bob ref list -R all -A -f json` row).

## Design specification (decided; implement exactly)

### 1. Data and refresh

| Source            | Command                                       | Lane             | When                                                                                                                                                                                                 |
| ----------------- | --------------------------------------------- | ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Snapshot          | `bob ref list -R all -A -f json`              | `refs-list`      | Launch (after the cache loads); a change under `<vault>/ref` or `<vault>/lib` (FSEvents, 0.5 s trailing debounce); panel open when the snapshot is older than 60 s; ⌘R; Recheck Bob; wake from sleep |
| Git added dates   | `bob ref list -R all -A -g -f json`           | `refs-git`       | After a snapshot refresh, only if some row has no `added` and its path is not yet in `gitAddedDates`                                                                                                 |
| Today             | `bob plan -f json`                            | `refs-plan`      | Every panel open (in the background); every successful Capture submit; every 10 minutes                                                                                                              |
| Notes (inspector) | `bob ref show <note path> -f json -c`         | `refs-show`      | The selection has settled for 120 ms                                                                                                                                                                 |
| Spotlight sweep   | `MDItemCreateWithURL` / `MDItemCopyAttribute` | background queue | After each snapshot refresh: `kMDItemLastUsedDate` and `kMDItemNumberOfPages` per PDF                                                                                                                |
| Missing PDFs      | `FileManager.fileExists`                      | background queue | After each snapshot refresh                                                                                                                                                                          |

**Snapshot and refresh rules:**

- Keep only rows with a non-empty `source_pdf`.
- Reject a `source_pdf` that is absolute or has a `..` component; treat it as missing.
- Swap a refreshed snapshot in atomically, only after a complete decode.
- A failed refresh keeps the last good snapshot and records `.failed(message, at)`. It
  never empties the list.
- Persist the snapshot, including `gitAddedDates`, so a cold launch paints immediately.
- Only one refresh per lane is in flight. A trigger that arrives during a refresh
  schedules exactly one follow-up.

**Git dates:**

- Merge a git-backfilled `added` value (`added_source == "git"`) only into rows whose
  `added` is null.
- Never use a git-sourced `finished` date (218 rows share one bulk date). Use `finished`
  only when `finished_source != "git"`.

**Today:**

- Today is the set of `today_tasks[]` whose `path` matches a row's `path`.
- Order is first occurrence in the array.
- The Pomodoro name is `entry_name`.

**The vault root** is resolved exactly as `AppDelegate` resolves it today:
`settings.bobDirectory`, or `~/bob` when that is empty. Extract one shared helper.

### 2. Item model and taxonomy (`RefsCore`)

`RefItem` is built from one decoded row plus the snapshot's git dates. Its `id` is the
note `path` (`ref/chat/x.md`), never the PDF path, because PDF stems repeat
(`harness_engineering.pdf` exists in both `blogs/` and `papers/`).

| `ref_type`    | `RefKind`     | Label                           | SF Symbol                      | Tint      | Scope       |
| ------------- | ------------- | ------------------------------- | ------------------------------ | --------- | ----------- |
| `chat`        | `.chat`       | Chat (byline "Agent report")    | `bubble.left.and.bubble.right` | purple    | ⌘2 Chats    |
| `papers`      | `.paper`      | Paper                           | `graduationcap`                | teal      | ⌘3 Papers   |
| `blogs`       | `.article`    | Article                         | `newspaper`                    | brown     | ⌘4 Articles |
| `docs`        | `.doc`        | Doc                             | `book.closed`                  | indigo    | ⌘5 Docs     |
| `books`       | `.book`       | Book                            | `books.vertical`               | secondary | All only    |
| `slides`      | `.slides`     | Slides                          | `rectangle.on.rectangle`       | secondary | All only    |
| other or null | `.other(raw)` | title-cased raw, or "Reference" | `doc`                          | secondary | All only    |

**`RefState`** is derived from bob's `status`:

| `status`    | `RefState` |
| ----------- | ---------- |
| `wip`       | `.reading` |
| `next`      | `.next`    |
| `ready`     | `.ready`   |
| `read`      | `.read`    |
| `abandoned` | `.dropped` |

Any other or missing `status` falls back to `reading_state`:

| `reading_state` | `RefState` |
| --------------- | ---------- |
| `started`       | `.reading` |
| `queued`        | `.ready`   |
| `finished`      | `.read`    |
| `dropped`       | `.dropped` |
| anything else   | `.unknown` |

**Blocked.** `isBlocked` comes only from the new `blocked` field. It overrides the
displayed glyph and label but not the lane: a blocked `next` row is still in the Next
lane.

| State             | Palette case (`CaptureEditorPalette.taskStatus`) | Glyph                    | Color     | Label   |
| ----------------- | ------------------------------------------------ | ------------------------ | --------- | ------- |
| reading           | `.inProgress`                                    | `circle.lefthalf.filled` | orange    | Reading |
| next              | `.next`                                          | `circle.inset.filled`    | blue      | Next    |
| ready             | `.todo`                                          | `circle`                 | secondary | Ready   |
| blocked (overlay) | `.blocked`                                       | `pause.circle`           | secondary | Blocked |
| read              | `.done`                                          | `checkmark.circle`       | green     | Read    |
| dropped           | `.canceled`                                      | `xmark.circle`           | secondary | Dropped |
| unknown           | `.other`                                         | `circle.dashed`          | secondary | Unknown |

Never use the word "NEW". The review walk owns that chip, so the section is "Just
added".

Other derived fields:

- **`title`** is `TaskDisplayText(parsing: rawTitle)`. Backticks become code segments,
  and all matching runs on its `text`.
- **`stem`** is the PDF filename without `.pdf`.
- **`titleDisambiguator`** is the stem when another item has the same folded display
  title.
- **`parentLabel`** is `parent` with its wikilink brackets stripped and a trailing
  `_ref` removed.
- **`added`** and **`finished`** are `RefDay` values (a calendar-day ordinal parsed from
  `YYYY-MM-DD`, comparable, timezone-free). `addedIsApproximate` is true when the date
  came from git.
- **Unopened** is not stored on the item. `RefsRanker.isUnopened(_:signals:)` returns
  true when the state is ready or next (blocked included), no open-log event exists,
  Spotlight has no last-used date, and `annotation_count == 0`.

The complete `RefItem` field list, which later phases rely on:

```swift
public struct RefItem: Identifiable, Equatable, Sendable {
    public let id: String                 // note path
    public let link: String               // bob's `link`, e.g. "[[ref/chat/x]]"
    public let rawTitle: String
    public let title: TaskDisplayText
    public let stem: String
    public let titleDisambiguator: String?
    public let kind: RefKind
    public let state: RefState
    public let isBlocked: Bool
    public let pdfPath: String            // vault-relative `source_pdf`
    public let parentLabel: String?
    public let author: String?
    public let published: String?
    public let added: RefDay?
    public let addedSource: String?       // created | captured | zorg_block | git
    public let addedIsApproximate: Bool
    public let finished: RefDay?          // nil when finished_source == "git"
    public let annotationCount: Int
    public let commentCount: Int
    public let snapshotSyncedAt: String?
    public let audioPath: String?
    public let urls: [String]
    public let arxivID: String?
    public let doi: String?
    public let isAgentReport: Bool        // origin == "agent-report"
}
```

### 3. Browse order (empty or whitespace-only query)

Each item appears once, in the first section whose rule admits it. Empty sections are
omitted, and a header never stands alone. The scope filter applies first. "Last opened"
means `max(picker open log, Spotlight last-used)`.

| #   | Section         | Admits                                                      | Order within                                                              | Cap  |
| --- | --------------- | ----------------------------------------------------------- | ------------------------------------------------------------------------- | ---- |
| 1   | Today           | id in Today                                                 | Today order                                                               | none |
| 2   | Just added      | ready or next, `unopened`, `added ≥ today − 3 days`         | added desc                                                                | 5    |
| 3   | Reading         | state reading                                               | last opened desc (nil last), added desc                                   | none |
| 4   | Next            | state next; blocked rows last                               | last opened desc, added desc                                              | none |
| 5   | Ready           | state ready; blocked rows last                              | added desc                                                                | none |
| 6   | Recently opened | state read, dropped, or unknown, last opened within 14 days | last opened desc                                                          | 5    |
| 7   | Library         | everything else                                             | last activity desc: `max(added, last-opened day, finished)` with nil last | none |

- The final tie-break everywhere is `id` ascending, so identical inputs always produce
  identical order.
- An item over a cap falls through to the next section that admits it.
- Named constants:
  - `justAddedWindowDays = 3`
  - `justAddedCap = 5`
  - `recentlyOpenedWindowDays = 14`
  - `recentlyOpenedCap = 5`

### 4. Search ranking (typed query)

**Normalization:**

- Split the query on whitespace.
- Within each token, split on `_ - / . :`.
- Strip leading and trailing non-alphanumerics from each piece (so `` `%auto` `` becomes
  `auto`), then drop empty pieces.
- If no pieces remain, use browse order.
- Matching folds case, diacritics, and width (FuzzyMatcher already does).
- **Words** are maximal letter-or-digit runs of the folded display title, or of the
  stem.

**Every token must match somewhere (AND).** A token can match in one of these ways:

| Kind of match | Rule                                                                                                                                                                                 |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Word          | The token is a prefix of a title or stem word                                                                                                                                        |
| Fuzzy         | Tokens of ≥ 3 characters: a FuzzyMatcher subsequence in the title or stem, with quality `q ≥ fuzzyFloor` (0.55). Tokens of 2 characters: a contiguous substring of the title or stem |
| Secondary     | Word or fuzzy against `author`, `parentLabel`, kind words, or cached outline headings (from the inspector phase)                                                                     |

The kind words are:

- chat: "chat", "agent", "report"
- paper: "paper"
- article: "article", "blog"
- doc: "doc", "docs"

A token of one character only matches as a word prefix.

**Tiers.** A row's tier is its worst token's best tier. Rows never cross tiers.

| Tier         | Rule                                                                                                                                                                                                                                                   |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| T0 Exact     | The normalized whole query equals the normalized title or stem; or the trimmed raw query equals an arXiv id (case-insensitive, `arxiv:` prefix and `vN` suffix ignored), a DOI (case-insensitive), or a URL (scheme, `www.`, and trailing `/` ignored) |
| T1 Word      | Every token is a word match in title or stem                                                                                                                                                                                                           |
| T2 Fuzzy     | Every token matches the title or stem, at least one only fuzzily                                                                                                                                                                                       |
| T3 Secondary | At least one token matches only a secondary field                                                                                                                                                                                                      |

**Within-tier score.** These are named constants, pinned by golden tests:

```text
score = m + P + W + L + F + R
  q(token) = min(1, best FuzzyMatcher score of token in the field it matched
                  ÷ FuzzyMatcher score of token matched against FuzzyField(token))
  m  mean q over tokens (T0: 1)
  P  prefixBonus     0.15  if the first token's match starts at offset 0 of title or stem
  W  wholeWordBonus  0.10  if every token equals a whole word
  L  lane prior: Today 0.20 (wins over the lane), reading 0.15, next 0.12, ready 0.08,
     blocked uses its lane's prior − 0.02, read 0, unknown 0, dropped −0.10
  F  frecencyWeight 0.25 · log1p(f) / log1p(fMax), f = Σ opens 0.5^(ageDays / 14);
     0 when fMax is 0
  R  recencyWeight 0.10 · 0.5^(daysSinceAdded / 30); 0 without an added date
ties: shorter display title, newer added, then id
```

**Golden expectations.** Build analogous synthetic rows for each case, with the
structure described.

1. **Today beats a title prefix.** Query `omnigent`, three T1 rows. The Today + Reading
   row whose title ends in the word ranks first. A finished row whose title starts with
   it ranks second.
2. **`blog post` keeps both words.**
   - Two Today + Reading rows titled "… Blog Post …" lead.
   - Then a finished "blog00 launch post review".
   - Then a dropped "… blog post" row.
   - An Article row whose title lacks both words ranks below all of them, in T3, through
     its kind word. Bare kind words never hard-filter.
3. **`harness` separates kinds and sinks the dropped row.**
   - Two `harness_engineering` stems (Paper/Ready and Article/Read) both match, and
     their kind tiles tell them apart.
   - A dropped row titled "Harness for …" ranks below a Ready non-prefix match.
4. **`omnimeta` is T2 fuzzy only.** It matches "Introducing Omnigent: A Meta-Harness …".
   A scattered subsequence row with `q` below the floor is absent.
5. **`liu` reaches T3 through the author** "Yangze Liu", below every T1 or T2 row.
6. **Identity is exact.** A pasted arXiv id selects its row first (T0), and so does a
   DOI.
7. **Whitespace falls back.** Whitespace-only input yields browse order.

If the specified constants fail an expectation, adjust them minimally. State the change
and the reason in the README's ranking paragraph. The tier invariant is not negotiable.

Each ranked row carries highlight ranges:

- **Title ranges** come from the winning title alignments, coalesced into ranges.
- **Stem ranges**, when a token matched only the stem. The caption then shows the stem.
- **Secondary ranges**, when the row is T3. The caption then shows the matched author,
  area, or heading.

### 5. Stability rules (tested)

1. A query or scope edit selects the first row of the new ranking. Opening the panel
   selects the first row of the first section. After that, the selection is an **id**,
   and it stays put while the user navigates.
2. While the panel is open, data that arrives late changes row **content** in place
   (`RefsListing.refreshingContent`). Examples: a refreshed snapshot, git dates, Today,
   Spotlight, missing-PDF checks, and inspector hydration. Order changes only on the
   next query or scope edit, ⌘R, or the next open.
3. The panel's height is fixed when it opens. Filtering never resizes it.
4. Identical inputs produce identical order, down to the id tie-break.
5. A frozen id that is gone from the new snapshot is shown as unavailable: dimmed, with
   the caption "No longer in your library". Return then shows that message instead of
   opening anything. Never open whatever row slid into its index.

### 6. Window and motion

- **Panel class.** `RefsPanel: NSPanel`, created with
  `styleMask: [.borderless, .nonactivatingPanel]`, `backing: .buffered`, `defer: false`.
  - Override `canBecomeKey` to return `true` and `canBecomeMain` to return `false`.
  - `isFloatingPanel = true`, `level = .floating`.
  - `collectionBehavior = [.canJoinAllSpaces, .fullScreenAuxiliary, .transient, .ignoresCycle]`.
  - `isOpaque = false`, `backgroundColor = .clear`, `hasShadow = true`.
  - `hidesOnDeactivate = false`, `isReleasedWhenClosed = false`,
    `isMovableByWindowBackground = false`.
  - Never call `NSApp.activate`. Highlights, or whatever app was frontmost, stays
    active.
- **Shell.** The content view is an `NSGlassEffectView` (`cornerRadius = 20`) whose
  `contentView` is the `NSHostingView` for `RefsPanelView`. Set the hosting view's
  `sizingOptions = []`.
  - If the SDK's `NSGlassEffectView` API differs, use SwiftUI
    `.glassEffect(.regular, in: .rect(cornerRadius: 20))` on the root view instead.
  - Under Reduce Transparency, the root view paints an opaque
    `Color(nsColor: .windowBackgroundColor)` rounded rect beneath its content.
  - Call `invalidateShadow()` after the first layout.
- **Size.** Width is `min(880, visibleFrame.width − 80)`. Height is
  `min(560, visibleFrame.height − 80)`. Both are fixed while the panel is visible.
  - The inspector shows only when width ≥ 760.
  - The list column takes `round(width × 0.52)` and the inspector the rest.
- **Position.** Use the screen containing `NSEvent.mouseLocation`, falling back to
  `NSScreen.main`.
  - Center horizontally in `visibleFrame`.
  - Place the top edge at `visibleFrame.maxY − round(visibleFrame.height × 0.16)`, then
    clamp to `visibleFrame`.
  - Recompute on every show.
- **Prewarm.** Build the panel and hosting view at launch, the way capture's `prewarm()`
  does.
- **Show.**
  1. Call `model.prepareForPresentation()` (blank query, All scope, fresh listing, first
     row selected).
  2. Set the frame, set `alphaValue = 0`, and call `makeKeyAndOrderFront(nil)`.
  3. Fade to 1 over 0.12 s with `NSAnimationContext`. At the same time, the SwiftUI
     content scales from 0.98 to 1.0 with `.easeOut(duration: 0.12)`.
  4. Focus the search field.
  - Under Reduce Motion, use the fade only.
- **Hide.** Call `orderOut(nil)` immediately, with no fade-out. Key status returns to
  the frontmost app on its own.
- **Click outside.** When the panel resigns key (a click in another window or app, or
  Settings opening), it hides, like Spotlight. The hide is skipped while the ⌘K actions
  menu is tracking.
- **Signposts.** `refs-hotkey-received`, `refs-panel-show`, `refs-open-dispatch`, and a
  `refs-refresh` interval.

### 7. Layout

```text
╭────────────────────────────────────────────────────────────────────────────────╮  glass, r=20
│  ⌕  Search references                          [▣ Chats ✕]   27 open · 342   ◌  │  search bar 52 pt
│ ╭───────────────────────────────────────┬────────────────────────────────────╮ │  content well
│ │ TODAY  3                              │  ▣  Chat   ◐ Reading   ⏱ BLOG       │  r=12, 8 pt inset
│ │ ● [▣] Team Fit Review, After Omni… ⏱BLOG ◐ │  Team Fit Review, After Omnigent │  .regularMaterial
│ │       Chat · opened 2h ago · 11 pp         │  Agent report · sase             │
│ │   [▣] The First Blog Post: A Re…  ⏱BLOG ◐ │                                   │
│ │       Chat · added Oct 7                   │  Added      Oct 7 (created)      │
│ │ JUST ADDED  2                         │  Pages      11                     │
│ │ ● [🎓] What Does a Harness Buy?…        ○ │  Opened     2 hours ago · 4 times│
│ │       Paper · Liu · added today · 22 pp    │  Notes      3 highlights         │
│ │ READING  4                            │                                    │
│ │   …                                   │  In Today · BLOG Pomodoro          │
│ │                                       │  lib/chat/team_fit_review.pdf      │
│ ╰───────────────────────────────────────┴────────────────────────────────────╯ │
│  ↵ Open in Highlights  ⌘↵ Note  ⌥↵ Reveal  ⌘1–5 Scope  esc Close   Updated 2m ago │  footer 30 pt
╰────────────────────────────────────────────────────────────────────────────────╯
```

**Shell.** The outer padding is 8 pt. The content well's radius is 12, which is
concentric with the 20 pt glass (20 − 8). The well is `.regularMaterial`, matching
capture's picker cards, and has a 0.5 pt `primary.opacity(0.08)` stroke.

**Search bar** (52 pt tall, drawn directly on the glass):

- A `magnifyingglass` glyph at 17 pt, secondary.
- The search field: 20 pt system font, placeholder "Search references".
  - Reuse `CapturePickerFilterNSTextField`, adding an accessibility-identifier
    parameter. Use `org.bobs.bob-mac-capture.refs-filter`.
  - Give it capture's focus-repair behavior.
- When the scope is not All, a scope token follows the field. It is a capsule containing
  the kind glyph, the label, and an `xmark` glyph.
  - Its fill is the kind tint at 0.16 opacity, and its text is the tint color.
  - Clicking the `xmark` removes it, and so does ⌫ on an empty query.
- At the trailing edge:
  - A count in `.callout.monospacedDigit()`, secondary.
    - In browse mode it reads "`N open · M`", where open means reading, next, and ready
      (blocked included).
    - In search mode it reads "`k of M`".
  - A small `ProgressView` while a snapshot refresh runs.

**Columns.** The list and the inspector are separated by a 0.5 pt
`primary.opacity(0.10)` hairline (0.25 under Increase Contrast).

**List:**

- A `ScrollView` containing a `LazyVStack(pinnedViews: [.sectionHeaders])`, with 6 pt
  vertical padding.
- The selection stays visible through `ScrollViewReader.scrollTo(id, anchor: nil)`, with
  no animation.

**Section header** (26 pt tall, pinned, `.regularMaterial` background):

- The title is `.caption.weight(.semibold)`, uppercase, with 0.5 tracking, in secondary.
- The count follows in `.caption2.monospacedDigit()`, tertiary.
- It carries the `.isHeader` accessibility trait.

**Footer** (30 pt tall, on the glass):

- **Left:** keycap hints styled like `CapturePickerKeyHints`. When the selected row's
  PDF is missing, the first hint reads "↵ Open note".
- **Right:** the library status, in `.caption`, secondary:
  - "Updated 2m ago" (relative, refreshing each minute while the panel is visible);
  - "Updating…";
  - "Update failed · ⌘R to retry", in orange, when the last refresh failed.

### 8. Rows (44 pt, two lines)

Each row is an `HStack(spacing: 10)` with 8 pt leading and 12 pt trailing padding:

1. **Unopened gutter (8 pt).** A 6 pt `accentColor` circle when the item is `unopened`.
2. **Kind tile (28 × 28).**
   - `RoundedRectangle(cornerRadius: 7)` filled with the tint at 0.16 opacity, with a
     0.5 pt stroke of the tint at 0.28 (0.5 under Increase Contrast).
   - The kind symbol at 13 pt semibold, in the tint color.
3. **Text column**, a `VStack(alignment: .leading, spacing: 2)`.
   - **Line 1:** the title as rich text, using `CapturePickerRichText.displayText` with
     the title segments and match ranges.
     - 13.5 pt, semibold when `unopened`, otherwise regular.
     - One line, tail truncation.
     - When there is a `titleDisambiguator`, ` · <stem>` follows in
       `.caption.monospaced()`, tertiary.
   - **Line 2:** the caption in 11.5 pt, secondary, `monospacedDigit()`. The parts are
     joined with `·`:
     - the kind label;
     - the author, for paper, article, and doc;
     - a date phrase;
     - `N pp`, when the page count is known;
     - in search mode, the matched stem or secondary field with its highlight ranges.
4. **Spacer** (minimum 8 pt).
5. **Trailing marks, in order:**
   - A **Today pill** for Today items: a pink `Capsule` containing a white `timer` glyph
     and the Pomodoro name in `.caption2.weight(.bold)`. The name truncates at 12
     characters, and an empty name reads "TODAY". It matches capture's NOW pill.
   - **Annotations:** `highlighter` plus the count, in `.caption` secondary
     monospacedDigit, when `annotation_count > 0`.
   - **Narration:** `waveform`, in `.caption` secondary, when `audio` is set.
   - **State glyph:** 15 pt, in a fixed 18 pt column so the glyphs align down the list.
     A missing PDF replaces it with `exclamationmark.triangle.fill` in orange, and the
     caption gains "PDF missing".

**Date phrase.** It names the date that explains the row's position:

| Where the row sits                                 | Phrase                                            |
| -------------------------------------------------- | ------------------------------------------------- |
| Just added, Ready, Library with only an added date | "added Oct 7"                                     |
| Reading, Next, Recently opened                     | "opened 2h ago", falling back to the added phrase |
| Read rows with a non-git finish date               | "read Oct 3"                                      |
| Search mode                                        | The last-activity phrase                          |

Dates are formatted as "today", "yesterday", the weekday for the last 6 days, "Oct 7"
within the year, and "Oct 7, 2025" otherwise. A git-approximated date reads "added ≈ Sep
12".

**Selection and hover:**

- The selection is a `RoundedRectangle(cornerRadius: 8)` inset 6 pt horizontally, filled
  `accentColor.opacity(0.16)` (0.30 under Increase Contrast).
- Hover is `primary.opacity(0.06)` (0.12 under Increase Contrast).
- Single click selects. Double-click opens in Highlights.
- An unavailable row is drawn at 0.45 opacity.

**No row separators and no thumbnails in the list.** Whitespace and the selection carry
the structure.

### 9. Typography and color summary

- **List:** titles 13.5 pt, captions 11.5 pt.
- **Inspector:** title `.title3.weight(.semibold)`, body `.callout`, labels `.caption`.
- **Digits:** every count and date uses `.monospacedDigit()`.
- **Hue meanings:**
  - The kind tints from §2 are purple, teal, brown, and indigo.
  - The status hues are orange, blue, and green. They never encode kind.
  - Pink means Today or Pomodoro only.
  - The accent color means selection, matches, and the unopened dot.
- **Dark mode** relies on system colors only. Never hard-code RGB.

### 10. Inspector

The inspector fills instantly from the list item and is upgraded lazily. It sits in a
`ScrollView` with 18 pt padding and a `VStack(alignment: .leading, spacing: 14)`.

| Block      | `refs-panel-ui` (basic)                                                                                                                                                                                                                                                                       | `refs-inspector` (full)                                                                                                                                                                                        |
| ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Hero       | A 44 pt kind tile (radius 11, 20 pt glyph). Beside it: a chips row (kind chip, state chip with glyph and label, Today pill), the title (`.title3.weight(.semibold)`, rich, up to 4 lines), and the byline                                                                                     | Papers, articles, and docs replace the tile with a 112 × 145 pt page-1 thumbnail (radius 6, 0.5 pt hairline, soft shadow). Chats keep the tile                                                                 |
| Byline     | Author · published · source host (from `urls[0]`) · arXiv or DOI. For chats: "Agent report · `parentLabel`"                                                                                                                                                                                   | Chats add "Consolidated report" for a stem ending in `__final`, or "Researcher draft (`x`)" for `__x`                                                                                                          |
| Facts      | A `Grid` of a right-aligned `.caption` secondary label (84 pt) and a `.callout` value. Rows: Added (with source, and ≈ when from git), Finished (non-git only), Pages, Opened ("2 hours ago · 4 times" / "Never opened"), Notes ("3 highlights · 1 comment" plus the snapshot age), Narration | Adds reading time to Pages ("22 · ≈ 70 min · 3 Pomodoros"), and Tasks ("2 open") from `ref show`                                                                                                               |
| Summary    | —                                                                                                                                                                                                                                                                                             | Chats: the first paragraph or up to 3 bullets under "Bottom line" (or TL;DR, Summary, Executive summary, Key findings). Papers: the Abstract excerpt. Label it `SUMMARY` or `ABSTRACT`                         |
| Contents   | —                                                                                                                                                                                                                                                                                             | Top-level outline headings (up to 6), as a compact list with `chevron.right` bullets. It is the main visual for chats                                                                                          |
| Your notes | —                                                                                                                                                                                                                                                                                             | Up to 3 commented highlights. Each shows its page label (`.caption`), the quote (italic, secondary, 3 lines, with a 2 pt yellow leading bar), and the comment (`.callout`). Then "+N more · ⌘↵ opens the note" |
| Footer     | The why-here line (`.caption`, tertiary), then the vault PDF path (`.caption.monospaced()`, middle truncation, selectable)                                                                                                                                                                    | Same                                                                                                                                                                                                           |

**Honesty rules:**

- Absence stays absence. Omit a fact row rather than printing "—".
- A publication date is not a file date.
- A highlight count is not progress.
- Git-sourced dates carry "≈".
- Reading time is labeled as an estimate.

**Missing PDF.** The inspector shows an orange callout: "The PDF this note points to is
missing: `<path>`. ↵ opens the note instead."

### 11. States and messages

| Situation                                             | Behaviour                                                                                                                                                                            |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| First launch, no cache, refresh in flight             | 8 skeleton rows (`.redacted(reason: .placeholder)`). The field stays interactive, and the first real snapshot replaces the skeleton in place                                         |
| No cache and the refresh failed (or `bob` unresolved) | A centered callout card with an `exclamationmark.triangle` glyph. Title "Bob couldn't load your references", the bounded error text, and two buttons: Retry (⌘R) and Copy Diagnostic |
| Refresh failed with a cache                           | The last good list stays. The footer reads "Update failed · ⌘R to retry"                                                                                                             |
| No matches                                            | A centered 28 pt `magnifyingglass` glyph (tertiary), then "No references match “xyz”". When scoped: "No Papers match “xyz” · ⌘1 searches All". Return does nothing                   |
| PDF missing                                           | See §8 and §10. ↵ opens the note                                                                                                                                                     |
| Selected row unavailable                              | See §5.5                                                                                                                                                                             |
| Highlights not found                                  | A banner above the list: "Highlights isn't installed where Bob Refs can find it." with buttons Open in Default App and Choose Highlights… (opens Settings). The panel stays open     |
| Highlights reported an error                          | The panel is shown again with the query and selection intact. Banner: "Highlights couldn't open “Title”." plus the error, with buttons Try Again and Open in Default App             |

**Banners** use orange `exclamationmark.triangle.fill`, a rounded radius-10 card with an
orange 0.12 fill, and `.callout` text. They are dismissed by the next query edit, Esc,
or a successful open.

### 12. Keyboard

The local key monitor is gated on `panel.isKeyWindow`, as in capture.
`RefsKeyRouter.command(for:context:)` is a pure function. Printable keys fall through to
the field.

| Key                                         | Command                                                                    |
| ------------------------------------------- | -------------------------------------------------------------------------- |
| ↵, Tab, double-click                        | Open in Highlights. For a missing PDF, open the note                       |
| ⌘↵                                          | Open the note in Obsidian (`ObsidianOpenURL`)                              |
| ⌥↵                                          | Reveal the PDF in Finder                                                   |
| ↑ ↓, ⌃P ⌃N, ⌃K ⌃J                           | Move (wraps; `CapturePickerNavigation`)                                    |
| PgUp PgDn                                   | Page by `visibleRowBudget − 1` (clamped)                                   |
| ⌘↑ ⌘↓, Home End                             | First, last                                                                |
| ⌥↑ ⌥↓                                       | First row of the previous or next section (browse only; no-op in search)   |
| ⌘1 … ⌘5                                     | Scope All, Chats, Papers, Articles, Docs                                   |
| ⌘R                                          | Refresh library, Today, and git dates now; re-rank                         |
| ⌘K                                          | Actions menu (`refs-inspector`)                                            |
| ⌫ on an empty query                         | Remove the scope token                                                     |
| Esc, ⌃[                                     | Dismiss the banner, else clear the query, else reset the scope, else close |
| ⇧Tab                                        | Consumed (no-op)                                                           |
| Global hotkey or takeover key while visible | Close                                                                      |

Return while the field editor `hasMarkedText()` (IME composition) passes through to the
text system.

### 13. Opening

1. **Highlights URL.** Resolve it, in order:
   - the Settings override path, if it exists;
   - `NSWorkspace.shared.urlForApplication(withBundleIdentifier: "net.highlightsapp.universal")`;
   - otherwise nil, which shows the "Highlights not found" banner.
2. **On ↵:** capture the selected **id** and its URL (vault root + `source_pdf`).
   - A missing file opens the note instead.
   - An unavailable id shows its message.
3. **Hide the panel at once.** Then call
   `NSWorkspace.shared.open([url], withApplicationAt: highlights, configuration: cfg)`,
   with `cfg.activates = true`.
4. **Completion error:** show the panel again with the banner, as in §11.
5. **Success:** append `{path: id, at: now}` to the open log.
   - Only Highlights and default-app opens count.
   - Note opens, reveals, selections, and previews are never recorded as opens.
6. **Constraints:**
   - Never build a shell command.
   - Never open a copy of the PDF: Highlights' annotations must land on the library
     original that bob's scan reads.
7. **Other targets:**
   - Note: `ObsidianOpenURL.url(forAbsolutePath:)` on vault root + `path`, through
     `NSWorkspace.open`.
   - Reveal: `NSWorkspace.shared.activateFileViewerSelecting([url])`.
   - Default app: `NSWorkspace.shared.open(url)`.

### 14. Entry points and coexistence with Capture

- **Hotkey registry** (`mac-groundwork`). One Carbon event handler reads the
  `EventHotKeyID` from each event and routes it to an action:
  - `HotKeyAction.capture` (id 1)
  - `.refs` (id 2)
  - `.refsHighlightsOpen` (id 3)

  Unknown ids return `eventNotHandledErr`.

- **Global Refs hotkey.** ⌃⇧⌘R (`kVK_ANSI_R`) is on by default. Settings has a toggle. A
  registration failure is reported in Diagnostics as "Refs hotkey conflict: …".
- **Highlights takeover.** `HighlightsTakeover` observes
  `NSWorkspace.shared.notificationCenter`: `didActivateApplicationNotification`,
  `didDeactivateApplicationNotification`, and `didTerminateApplicationNotification`.
  - It registers `.refsHighlightsOpen` only while the frontmost app's bundle id matches
    Highlights (or the override app's bundle id) and the setting is not off.
  - At launch it checks `frontmostApplication`.
  - Other apps never lose ⌘O or ⌃O.
  - The panel is non-activating, so Highlights stays frontmost while the panel is
    visible, and the same key closes the panel.
  - The setting's default is decision `highlights_open_key`.
- **Status menu.** Insert "Bob Refs…" directly after "Capture". It shows ⌃⇧⌘R as its key
  equivalent, for display only, and its action toggles the panel. Update
  `testStatusMenuOffersRestartBeforeQuit` and any glyph tests that count menu items.
- **Settings › References**, a new `Form` section after Hotkey:
  - Toggle: "Open Bob Refs with Control-Shift-Command-R".
  - Picker: "In Highlights, open Bob Refs with: Off / Command-O / Control-O".
  - Highlights app:
    - It shows the resolved app's name, version, and path, or "Not found".
    - "Choose…" opens an `NSOpenPanel` limited to `.app`.
    - "Use Default" clears the override.
  - "Reset Open History…", behind a confirmation dialog.
  - UserDefaults keys: `refsHotkeyEnabled` (default true), `refsHighlightsOpenKey`
    (`cmdO` | `ctrlO` | `off`), and `refsHighlightsAppPath` (default empty).
  - Changes re-register immediately through `@Published` sinks. Capture's hotkey remains
    re-registered only at launch and on Recheck Bob, as today.
- **One Bob panel at a time.** `BobPanelCoordinator` owns the rule.
  - Showing Refs while Capture is visible first calls Capture's
    `model.closeRetainingDraft()`, so the draft is preserved.
  - Showing Capture while Refs is visible hides Refs.
  - Refs work runs on its own `refs-*` lanes, so it can never cancel a capture submit.
- **Capture success refreshes Today.** Subscribe to `model.$successAnnouncementTick` to
  refresh Today, and mark the snapshot stale.
- **Recheck Bob** also re-points the Refs fetcher and restarts the Refs watcher.

The takeover default follows decision `highlights_open_key`. Whatever the default, the
takeover code ships and the Settings picker can change the key at any time.

> [!decision] highlights_open_key = cmd_o `refsHighlightsOpenKey` defaults to `cmdO`: ⌘O
> opens Bob Refs while Highlights is frontmost, and Highlights' File › Open… menu item
> still opens its own dialog.

> [!decision] highlights_open_key = ctrl_o `refsHighlightsOpenKey` defaults to `ctrlO`:
> ⌃O opens Bob Refs while Highlights is frontmost, and ⌘O keeps opening Highlights'
> dialog.

> [!decision] highlights_open_key = global_only `refsHighlightsOpenKey` defaults to
> `off`: only ⌃⇧⌘R opens Bob Refs until Bryan picks a takeover key in Settings.

### 15. Accessibility

- **Rows** combine into one element: "Title, Paper, Ready, added today, 22 pages, Today
  in BLOG Pomodoro, never opened". The selected row carries the `.isSelected` trait.
- **Result counts** are announced through `AccessibilityNotification.Announcement` 600
  ms after typing stops, never on every keystroke.
- **Images.** The thumbnail is `accessibilityHidden`. Kind tiles carry the kind label.
- **Display settings:**
  - Increase Contrast raises the selection fill, the hairline, and the tile strokes.
  - Reduce Transparency paints an opaque base.
  - Reduce Motion uses the fade only.

## Phase `cli-blocked`: bob-cli exposes Blocked on ref rows

**Repo:** bob-cli.

**Why.** The panel must show Blocked `[?]` references without parsing
`reading_state_source`. Today a blocked row reads `status: "next"`, and the `[?]`
appears only inside `reading_state_source: "ref_task:[?]"`.

**Change:**

1. **`StatusOutcome`** (`src/native/ref_library/status.rs`) gains `pub blocked: bool`.
   `decide_status` sets it from its tracker hits: true exactly when there is one `^ref`
   tracker and its mark is `?`. This holds whether or not that tracker is usable, so a
   `[?]` with no frontmatter status is still blocked. Several trackers mean false.
2. **`RefRow`** (`src/native/ref_library/row.rs`) gains `pub blocked: bool` directly
   after `reading_state_source`. It is always serialized. `build_row` (`mod.rs`) copies
   it from the outcome.
3. **Test-only row constructors** in `output.rs`, `show.rs`, and `coverage.rs` set
   `blocked: false`. Because `ShowRow` flattens `RefRow` and `find` embeds it,
   `ref show` and `ref find` gain the field automatically.
4. **`docs/ref.md`:**
   - In the reading-state section, add a paragraph saying that `blocked` is true when
     the note's single `^ref` tracker is Blocked `[?]`.
   - Explain that it is a derived overlay: `status` and `reading_state` still describe
     the reading lane (`decisions:task-lanes-are-sticky`).
   - Say that clients such as Bob Mac Capture's Refs panel read Blocked from this field
     and never parse `reading_state_source`.
   - Add `blocked` to the row field list in the JSON envelope section, and note that it
     is additive under `schema_version` 1.
5. **No other changes.** Human and markdown output, CLI flags, completion, and help are
   unchanged.

**Tests:**

- `src/native/ref_library/tests.rs`: the existing `ref/papers/blocked_mark.md` fixture
  row has `blocked == true`, and an ordinary tracker row has `false`.
- If no fixture covers several trackers where one is `[?]`, add one and update the
  hard-coded fixture counts it shifts.
- `tests/cli/ref_library/list.rs`: assert `document["refs"][i]["blocked"]` is true for
  the blocked fixture and false for another row.
- `tests/cli/ref_library/show.rs`: assert the flattened field is present.

**Done when:**

- `just check` passes. If an unrelated pre-existing failure appears, note it and leave
  it alone.
- A live `bob ref list -R all -A -f json` shows exactly one row with `"blocked":true`
  (or however many `[?]` trackers the vault holds at the time).

## Phase `mac-groundwork`: hotkey registry and CI render artifacts

**Repo:** bob-mac-capture.

1. **`HotKeyRegistry`** replaces `HotKeyManager` in `HotKeyManager.swift`; rename the
   file to `HotKeyRegistry.swift`.

   ```swift
   enum HotKeyAction: UInt32, CaseIterable { case capture = 1, refs = 2, refsHighlightsOpen = 3 }
   final class HotKeyRegistry {
       init(registrar: HotKeyRegistering = CarbonHotKeyRegistrar(),
            onPressed: @escaping (HotKeyAction) -> Void)
       func register(_ action: HotKeyAction, configuration: HotKeyConfiguration) throws
       func unregister(_ action: HotKeyAction)
       func isRegistered(_ action: HotKeyAction) -> Bool
       func invalidate()
       func handle(hotKeyID: EventHotKeyID) -> OSStatus   // test seam; the C callback calls it
   }
   ```

   - **Registration:** every key uses signature `'BOBC'` (`0x424F4243`) with
     `id = action.rawValue`. Registering an action replaces only that action's key. A
     failed registration throws `HotKeyRegistrationError` and leaves every other
     registration intact.
   - **The handler** reads the id with
     `GetEventParameter(event, EventParamName(kEventParamDirectObject), EventParamType(typeEventHotKeyID), …)`
     and calls `handle(hotKeyID:)`. `handle` dispatches the matching action and returns
     `noErr`. It returns `eventNotHandledErr` for a foreign signature or an unknown id.
   - **Presets:** keep `HotKeyConfiguration.development` and `.production`, and add:
     - `.refs` (⌃⇧⌘R, `kVK_ANSI_R`, "Control-Shift-Command-R");
     - `.highlightsCommandO` (⌘O, "Command-O");
     - `.highlightsControlO` (⌃O, "Control-O").
   - **AppDelegate:** creates one registry and registers `.capture` exactly as before.
     Its callback switches on the action. `.refs` and `.refsHighlightsOpen` are no-ops
     until `refs-entry-points` wires them.

2. **Tests** (`HotKeyRegistryTests`, using the existing `FakeHotKeyRegistrar` pattern):
   - distinct ids per action;
   - routing through `handle`;
   - an unknown id and a foreign signature return `eventNotHandledErr`;
   - re-registering replaces the earlier key;
   - a failure keeps the other registrations;
   - `invalidate` unregisters everything.

   Keep `testHotKeyRegistrationConflictIsReported` passing, adapted to the registry.

3. **Render helper.** Add `Tests/BobMacCaptureTests/RenderFixtureWriter.swift`:
   - `static func write<V: View>(_ view: V, name: String, width: CGFloat, appearance: NSAppearance.Name) throws`;
   - it renders with `ImageRenderer` at scale 2 onto an opaque `windowBackgroundColor`
     base;
   - it writes `<name>-<light|dark>.png` to `BOB_MAC_CAPTURE_RENDER_DIR`;
   - it throws `XCTSkip` when that variable is unset.

   New design tests use it. Existing design tests may keep their private copies.

4. **CI.**
   - The Test step sets `BOB_MAC_CAPTURE_RENDER_DIR: ${{ runner.temp }}/render-fixtures`
     and runs `mkdir -p` on that directory before `./Scripts/xcode-swift.sh test`.
   - A new step, `Upload render fixtures`, runs with `if: always()` and
     `actions/upload-artifact@v4`, `name: render-fixtures`, `if-no-files-found: ignore`,
     and `retention-days: 14`.
   - Confirm on the first green run that the artifact exists and contains capture's
     existing design PNGs.
5. **README:** update the hotkey paragraph to describe the registry. The `## Bob Refs`
   section is created later.

**Done when:** CI is green, the `render-fixtures` artifact downloads, and pressing the
production hotkey behaves as before. The registry tests prove that last point.

## Phase `refs-core-model`: RefsCore decoding, model, fetcher, and stores

**Repo:** bob-mac-capture.

1. **`Package.swift`.**
   - Outside the `#if os(macOS)` block, add:
     - `.target(name: "RefsCore", dependencies: ["CaptureCore"], swiftSettings: [.swiftLanguageMode(.v5)])`;
     - `.testTarget(name: "RefsCoreTests", dependencies: ["RefsCore", "CaptureCore"], …)`.
   - Add `"RefsCore"` to the `BobMacCapture` and `BobMacCaptureTests` dependencies.
   - RefsCore imports Foundation and CaptureCore only.
2. **`BobProcessClient`.** Promote the private
   `decode<T: Decodable & SchemaVersioned>(…)` helper to `public`, with an added
   `timeout:` parameter defaulting to `defaultTimeout`, so RefsCore reuses its error
   mapping (`processFailed`, `emptyStdout`, `malformedJSON`, `schemaMismatch`). Do not
   duplicate it.
3. **Decoding** (`Sources/RefsCore/RefsDecoding.swift`):

   ```swift
   public struct RefsListResponse: Decodable, SchemaVersioned, Equatable, Sendable
       // schemaVersion (expected 1), generatedAt: String?, refs: [RefRecord], skippedRowCount: Int
   public struct RefRecord: Codable, Equatable, Sendable
       // path, link, title (required); refType, origin, status, readingState: String?;
       // blocked: Bool (?? false); parent, author, published, added, addedSource,
       // finished, finishedSource, sourcePDF, audio, snapshotSyncedAt: String?;
       // urls: [String] (?? []); arxiv, doi: String? (from identity);
       // annotationCount, commentCount: Int (?? 0)
   public struct RefsPlanResponse: Decodable, SchemaVersioned, Equatable, Sendable
       // schemaVersion (expected 2), todayTasks: [RefsPlanTodayTask] (?? [])
   public struct RefsPlanTodayTask: Codable, Equatable, Sendable  // path, entryName, statusSymbol?
   ```

   - **Lossy rows.** `refs` decodes element by element through a wrapper. A row missing
     `path` or `title` is skipped and counted in `skippedRowCount`; it never fails the
     whole response.
   - **Encoding.** `RefRecord` encodes with the same snake_case keys, so the snapshot
     cache round-trips.

4. **Model** (`RefsModel.swift`):
   - `RefDay`;
   - `RefKind`, with `label`, `symbolName`, and `scope`. The tint lives in the app,
     keyed by kind;
   - `RefScope: Int, CaseIterable` (`all = 1`, `chats`, `papers`, `articles`, `docs`),
     with `label` and `contains(_:)`;
   - `RefState`;
   - `RefItem`, with exactly the fields described in §2;
   - `RefsCatalog.items(from: RefsSnapshot) -> [RefItem]`, sorted by id.

   The snapshot itself:

   ```swift
   public struct RefsSnapshot: Codable, Equatable, Sendable {
       public static let currentSchemaVersion = 1
       public var schemaVersion: Int; public var fetchedAt: Date
       public var records: [RefRecord]; public var gitAddedDates: [String: String]
       public var pathsNeedingGitDates: [String] { get }   // added == nil && not in gitAddedDates
   }
   ```

5. **Today** (`RefsToday.swift`): `RefsToday { entries: [String: RefsTodayEntry] }`.
   Each entry is `order: Int` plus `pomodoroName: String`, built with
   `init(plan: RefsPlanResponse)`. The first occurrence wins. It is `Codable`, so the
   last value is cached in memory and on disk next to the snapshot.
6. **Fetcher** (`RefsFetching.swift`):

   ```swift
   public protocol RefsFetching: Sendable {
       func list(gitDates: Bool) async throws -> RefsListResponse
       func plan() async throws -> RefsPlanResponse
   }
   public final class BobRefsFetcher: RefsFetching { public init(client: BobProcessClient) }
   ```

   Arguments are `["ref", "list", "-R", "all", "-A", "-f", "json"]` plus `-g` for the
   git pass, and `["plan", "-f", "json"]`, on the lanes from §1.

7. **Stores** (`RefsStores.swift`), following `FileCanceledDraftStashStore`. That means:
   - an injected file URL, with the default
     `~/Library/Application Support/org.bobs.bob-mac-capture/`;
   - load never creates the directory;
   - a corrupt file or a schema mismatch renames it to `*.corrupt.json` and reports
     through `onError`;
   - saves are atomic, with a 0600 file in a 0700 directory;
   - `.secondsSince1970` dates and `.sortedKeys`.

   The stores themselves:
   - **`RefsSnapshotStore`** (`refs-snapshot.json`): holds the snapshot and the last
     `RefsToday`.
   - **`RefsOpenLogStore`** (`refs-open-log.json`,
     `{schema_version: 1, opens: [{path, at}]}`):
     - `append(_ event:)` loads, appends, prunes, and saves. Pruning drops events older
       than 365 days, then keeps the newest 2000.
     - `load()` and `reset()`.
   - **`RefsOpenStats`** is built with `init(events:now:)`. It provides
     `lastOpened(_:)`, `count(_:)`, `frecency(_:)` (`Σ 0.5^(ageDays/14)`), and
     `maxFrecency`.

8. **fake-bob.** Add a `ref)` branch:
   - `ref list` prints `${FAKE_BOB_REFS_LIST_FIXTURE:-refs-list.json}`, or
     `refs-list-git.json` when the arguments contain `-g`.
   - `ref show` prints `refs-show.json`, unless `FAKE_BOB_REFS_SHOW_FIXTURE` names
     another fixture.

   Add a `plan)` branch that prints `${FAKE_BOB_PLAN_FIXTURE:-refs-plan.json}`. Keep the
   global knobs (`FAKE_BOB_STDOUT`, `_EXIT`, `_DELAY_SECONDS`, and the rest) working
   ahead of the dispatch. `bash -n` must pass.

9. **Fixtures** (synthetic; see Ground rules):
   - **`refs-list-golden.json`:** the full live envelope, with about 40 rows covering
     every case in §3, §4, and §11. That includes:
     - Today items;
     - 7 just-added candidates (to exercise the cap);
     - an added-today item that has annotations;
     - Reading, Next, and Ready rows, with blocked rows in the Next and Ready lanes;
     - 6 recently opened candidates;
     - library rows with git-only dates and with git `finished` dates;
     - the duplicate `harness_engineering` stems;
     - two rows with the same title;
     - a missing-PDF path;
     - every kind, plus an unknown `ref_type`;
     - backtick titles;
     - arXiv and DOI identities;
     - the golden-expectation rows of §4;
     - a legacy row without `source_pdf`;
     - a row with no `title`, which is skipped.
   - `refs-list.json`: small, 6 rows.
   - `refs-list-git.json`: the same 6 rows with git `added` dates.
   - `refs-plan.json`: schema 2, Today tasks for two of the rows plus a non-ref task.
   - `refs-plan-schema3.json`: for the rejection test.
   - `refs-open-log-golden.json`: open events for the recently-opened and frecency
     cases.
10. **Tests** (`RefsCoreTests`):
    - decoding: the golden envelope, lossy skips, a missing `blocked`, a schema
      mismatch, and a round-trip through the cache encoding;
    - kind, state, and scope mapping tables;
    - blocked overlay semantics;
    - `RefDay` parsing and comparison;
    - disambiguators;
    - `parentLabel`;
    - the git-date merge, including that git `finished` is ignored;
    - `RefsToday` ordering and dedupe;
    - store load, save, corruption, and file modes, plus open-log pruning;
    - frecency math;
    - a `BobRefsFetcher` test against fake-bob that records argv and lanes.
11. **README:** create `## Bob Refs`, with an overview and a "Data" paragraph covering
    the commands, schema versions, and what is cached where.

## Phase `refs-core-ranking`: browse, search, stability, explanations

**Repo:** bob-mac-capture.

1. **API** (`Sources/RefsCore/RefsRanking.swift` and its siblings):

   ```swift
   public struct RefsSignals: Sendable {
       public var today: RefsToday; public var opens: RefsOpenStats
       public var externalLastUsed: [String: Date]   // keyed by item id
       public var pageCounts: [String: Int]; public var missingPDFs: Set<String>
       public var outlineHeadings: [String: [String]]
       public var now: Date; public var calendar: Calendar
   }
   public enum RefsSectionKind: CaseIterable, Sendable { case today, justAdded, reading, next, ready, recentlyOpened, library }
   public enum RefsMatchTier: Int, Comparable, Sendable { case exact, word, fuzzy, secondary }
   public struct RefsMatch: Equatable, Sendable {
       public let tier: RefsMatchTier; public let score: Double; public let breakdown: RefsScoreBreakdown
       public let titleRanges: [Range<Int>]; public let stemRanges: [Range<Int>]
       public let secondary: RefsSecondaryMatch?   // field (author|area|kind|heading), text, ranges
   }
   public struct RefsSection: Equatable, Sendable { public let kind: RefsSectionKind?; public let ids: [String] }
   public struct RefsListing: Equatable, Sendable {
       public enum Mode: Equatable, Sendable { case browse, search(String) }
       public let mode: Mode; public let scope: RefScope; public let sections: [RefsSection]
       public let orderedIDs: [String]; public let matches: [String: RefsMatch]
       public let totalCount: Int; public let openCount: Int
       public func refreshingContent(availableIDs: Set<String>) -> (listing: RefsListing, unavailable: Set<String>)
   }
   public enum RefsRanker {
       public static func listing(_ items: [RefItem], query: String, scope: RefScope, signals: RefsSignals) -> RefsListing
   }
   public enum RefsRankingConstants { /* every constant named in §3 and §4 */ }
   public enum RefsCaption {
       public static func caption(for item: RefItem, in section: RefsSectionKind?, signals: RefsSignals) -> String
       public static func datePhrase(…) -> String
   }
   public enum RefsExplanation {
       public static func whyHere(_ item: RefItem, listing: RefsListing, signals: RefsSignals) -> String
   }
   public enum RefsSelectionPolicy {   // wraps CapturePickerNavigation over listing.orderedIDs
       public static func initial(in listing: RefsListing) -> String?
       public static func move(_ step: RefsMove, from id: String?, in listing: RefsListing, pageSize: Int) -> String?
   }
   ```

2. **Section metadata.** `RefsSectionKind` provides `title` ("Today", "Just added",
   "Reading", "Next", "Ready", "Recently opened", "Library").
3. **Why-here strings.** These exact forms are used, and tests pin them:
   - "In Today · BLOG Pomodoro"
   - "Just added Oct 8 · never opened"
   - "Reading · opened 2 hours ago"
   - "In your Next lane · blocked"
   - "Ready · added Oct 3"
   - "Opened 3 days ago"
   - "Library · last activity Sep 2"
   - Search mode: "Title word match · Reading", "Fuzzy title match", "Matched author",
     "Matched area", "Matched kind", "Matched heading", or "Exact arXiv id".
4. **Highlight ranges.** Copy capture's `ActiveTaskMatchHighlights.coalesced` into
   RefsCore as `RefsHighlights.coalesced`, because the original is internal.
5. **Performance.**
   - Precompute `FuzzyField`s and word lists once per item set, cached by snapshot.
   - Ranking 342 items must be far under 16 ms.
   - Add a sanity test: 10,000 synthetic items rank in under 1 s on CI, as a guard
     against accidental O(n²).
6. **`refs-rank` CLI.**
   - Add an `.executableTarget(name: "refs-rank", dependencies: ["RefsCore"])` outside
     the macOS block. It is never bundled; check that `Scripts/bundle.sh` copies only
     `BobMacCapture`.
   - Usage:
     `swift run refs-rank [--plan plan.json] [--opens open-log.json] [--scope chats] [--now 2026-10-08T09:00:00] [query…] < ref-list.json`.
   - It prints the sections, or the ranked rows, as aligned text:

     ```text
     rank  tier  score (m P W L F R)  state  kind  title
     ```

     followed by the why-here line.

   - It is for tuning against live `bob ref list` output on athena or the Mac.

7. **Golden tests.**
   - Browse: every section's membership and order, the caps and fall-through, the
     blocked rows last, omitted empty sections, and that the scope applies.
   - Search: every expectation in §4; the short-query policy (1, 2, and 3 or more
     characters); AND semantics; that tiers never cross; deterministic ties; and that
     whitespace-only input means browse.
   - Captions and date phrases, including "≈".
   - Why-here strings.
   - `refreshingContent` keeps the order and reports unavailable ids.
   - Selection: wrap, clamp, section jumps, and that the initial selection is the first
     row of the first section.
8. **README:** add "Sorting" (the browse table and the tier table) and "Tuning" (the
   `refs-rank` usage).

## Phase `refs-panel-model`: library service and panel model

**Repo:** bob-mac-capture. Code goes in `Sources/BobMacCapture/Refs/`.

1. **`VaultTargetWatcher`.** Generalize it to
   `init(paths: [String], latency:onChange:onFailure:)` and keep `init(path:…)` as a
   convenience. Capture's behavior is unchanged.
2. **`RefsLibrary`** (`@MainActor final class: ObservableObject`):

   ```swift
   @Published private(set) var items: [RefItem]
   @Published private(set) var signals: RefsSignals
   @Published private(set) var refreshState: RefsRefreshState   // .idle | .refreshing | .failed(message: String, at: Date)
   @Published private(set) var lastSuccessAt: Date?
   @Published private(set) var hasSnapshot: Bool
   init(fetcher: RefsFetching?, snapshotStore: RefsSnapshotStore, openLogStore: RefsOpenLogStore,
        vaultRoot: @escaping () -> URL, fileExists: @escaping (URL) -> Bool,
        spotlight: RefsSpotlightProviding, now: @escaping () -> Date)
   func start()                         // load cache + open log, then refresh
   func setFetcher(_ fetcher: RefsFetching?)
   func refresh(reason: RefsRefreshReason)     // coalesced per §1
   func refreshIfStale(maxAge: TimeInterval = 60)
   func refreshToday()
   func recordOpen(id: String)
   func resetOpenHistory()
   func pdfURL(for item: RefItem) -> URL?      // nil when unsafe
   func noteURL(for item: RefItem) -> URL
   ```

   - It owns the refs watcher on `<vault>/ref` and `<vault>/lib` (latency 0.5), the
     10-minute Today timer, and an `NSWorkspace.didWakeNotification` observer that
     refreshes the snapshot and Today after sleep.
   - The Spotlight sweep sits behind `protocol RefsSpotlightProviding`. The concrete
     `SpotlightRefsSignals` uses `MDItemCreateWithURL` / `MDItemCopyAttribute` on a
     utility queue. Tests inject a fake.
   - Missing-PDF checks and the sweep finish off the main actor, then publish together.

3. **`RefsPanelModel`** (`@MainActor final class: ObservableObject`):

   ```swift
   @Published var query: String
   @Published private(set) var scope: RefScope
   @Published private(set) var listing: RefsListing
   @Published private(set) var selectedID: String?
   @Published private(set) var unavailableIDs: Set<String>
   @Published private(set) var banner: RefsBanner?          // kind, message, actions
   @Published var visibleRowBudget: Int                    // from list geometry
   var panelDismisser: () -> Void
   var panelPresenter: () -> Void
   var settingsPresenter: () -> Void
   init(library: RefsLibrary, opener: RefsOpening, highlights: RefsHighlightsLocating)
   func prepareForPresentation()
   func queryDidChange()
   @discardableResult func perform(_ command: RefsCommand) -> Bool
   func rowContent(for id: String) -> RefsRowContent?      // item + match + caption + flags
   var selectedItem: RefItem? { get }
   ```

   - `RefsCommand` cases: `open(RefsOpenTarget)` (`.highlights`, `.note`, `.reveal`,
     `.defaultApp`); `move(RefsMove)`; `setScope(RefScope)`; `escape`;
     `deleteBackwardOnEmpty`; `refresh`; `select(id:)`; `activate(id:)` (double-click);
     `showActions`, which is a no-op until `refs-inspector`.
   - **Library publications while presented** go through `refreshingContent` (§5). They
     never re-rank. Re-ranking happens only on `queryDidChange`, `setScope`, `refresh`,
     and `prepareForPresentation`.
   - **Protocols:**
     - `RefsOpening`:
       - `openInHighlights(_ pdf: URL, app: URL, completion: @escaping (Error?) -> Void)`;
       - `openNote(_ url: URL)`;
       - `reveal(_ url: URL)`;
       - `openWithDefaultApp(_ url: URL, completion: …)`.

       The concrete `WorkspaceRefsOpener` uses `NSWorkspace` exactly as in §13.

     - `RefsHighlightsLocating`: `highlightsAppURL() -> URL?`. The concrete
       `HighlightsLocator` takes an injected override-path closure, wired to Settings in
       `refs-entry-points`.

   - The open flow, the Esc chain, the banners, and the unavailable handling follow §11
     to §13 exactly.

4. **Preview hook.**
   `installForPreviews(items:signals:query:scope:selectedID:banner:refreshState:)` lets
   design tests skip processes, like capture's `installPickerForPreviews`.
5. **Tests** (`RefsLibraryTests`, `RefsPanelModelTests`; fake-bob plus a fake opener,
   locator, and Spotlight provider):
   - **Refresh:**
     - cache-first paint;
     - a refresh replaces the snapshot;
     - a failure keeps the last good snapshot and sets `.failed`;
     - a schema mismatch is treated the same way;
     - lossy rows survive;
     - the git pass runs only when it is needed, and merges;
     - coalescing (two triggers during a delayed refresh run exactly one follow-up);
     - Today loads, and a schema 3 plan keeps the last Today.
   - **Presented updates:** while presented, a refresh updates content without
     reordering; a vanished selected id becomes unavailable; Return on it does not open.
   - **Opening:**
     - Open hides first, then dispatches, then records on success.
     - An error re-presents the panel with the banner and keeps the query.
     - A missing PDF opens the note and records nothing.
     - Highlights not found shows the banner, and the panel stays.
     - `.note` and `.reveal` never record.
   - **Navigation and editing:**
     - the Esc chain order;
     - ⌫ on an empty query removes the scope;
     - a scope or query edit selects the first row;
     - navigation wraps and clamps;
     - section jumps work in browse mode and are a no-op in search.
   - **Open history:** `resetOpenHistory` clears both the log and the stats.

## Phase `refs-panel-ui`: window, list, basic inspector, keyboard

**Repo:** bob-mac-capture. Code goes in `Sources/BobMacCapture/Refs/`.

1. **`RefsPanelController`** (`@MainActor final class: NSObject`):
   - It provides `prewarm()`, `show()`, `hide()`, `toggle()`, and `isVisible`, plus the
     local key monitor and focus repair. It implements §6 exactly.
   - `makePanel()` is `static` so tests can assert the configuration.
2. **Views** (one file each, all `@available(macOS 26.0, *)`):
   - `RefsPanelView` (the shell, the well, the column split, the motion state);
   - `RefsSearchBar`;
   - `RefsListView`, `RefsSectionHeader`, `RefsRowView`;
   - `RefsKindTile`, `RefsStateGlyph`, `RefsTodayPill`;
   - `RefsInspectorView` (the basic column of §10);
   - `RefsFooter`, `RefsBannerView`, `RefsEmptyStateView`, `RefsSkeletonList`;
   - `RefsVisualTokens` (kind tints and every size constant from §6 to §9, as named
     statics).

   Reuse `CapturePickerRichText.displayText` for titles and
   `CaptureEditorPalette.taskStatus` for the state glyphs.

3. **`RefsKeyRouter`** implements the §12 table. Its context is: banner visible, query
   empty, scope is All, marked text present, and list mode.
4. **Tests:**
   - **`RefsPanelGeometryTests`:**
     - the style mask contains `.borderless` and `.nonactivatingPanel`;
     - `canBecomeKey` is true and `canBecomeMain` is false;
     - the level is floating;
     - the collection behavior;
     - the size clamp on a 1280 × 800 and a 700 × 500 visible frame;
     - the inspector threshold;
     - the top-edge position.
   - **`RefsKeyRouterTests`:** every row of §12, plus IME passthrough.
   - **`RefsPanelDesignTests`** (through `RenderFixtureWriter`, light and dark):
     - `refs-browse-880` (all sections; a Today pill; unopened dots; a blocked row; a
       missing-PDF row);
     - `refs-search-880` (query `omni`, with highlights and a stem-match caption);
     - `refs-scoped-empty-880`;
     - `refs-missing-pdf-selected-880`;
     - `refs-refresh-failed-880` (a footer state);
     - `refs-load-failed-880` (the callout card);
     - `refs-skeleton-880`;
     - `refs-banner-880`;
     - `refs-browse-700` (list only).

     `ImageRenderer` cannot draw `NSViewRepresentable`, so give `RefsSearchBar` a
     preview mode that draws the query as `Text`.

5. **Look at the artifact** (Ground rules). In particular check:
   - title and caption baselines;
   - that the state glyph column aligns;
   - that pills don't crowd the title;
   - code spans inside titles;
   - contrast of secondary text on the material in dark mode;
   - that pinned headers occlude rows cleanly.

   Iterate until the renders match this spec.

6. **README:** add "The panel" (layout, keyboard table, states).

**Temporary entry.** The panel is not reachable from the UI until `refs-entry-points`
wires it in. Do not add a debug entry point.

## Phase `refs-entry-points`: hotkeys, takeover, menu, settings, coexistence

**Repo:** bob-mac-capture.

1. **AppDelegate launch.**
   - Construct the stores, `BobRefsFetcher` (sharing the existing `BobProcessClient`),
     `RefsLibrary`, `RefsPanelModel`, and `RefsPanelController` in
     `applicationDidFinishLaunching`, after capture's panel.
   - Call `prewarm()` and `library.start()`.
   - Launch must never block, and must stay healthy when `bob` is absent: CI's launch
     smoke test has no `bob`.
   - `applicationWillTerminate` invalidates the watcher and the timer.
2. **Hotkeys.**
   - Register `.refs` with `HotKeyConfiguration.refs` when `refsHotkeyEnabled` is set.
   - Route `.refs` and `.refsHighlightsOpen` to `BobPanelCoordinator.toggleRefs()`.
   - Record the status and conflicts in Diagnostics, as capture does.
3. **`HighlightsTakeover`** (§14): an injected notification center, registry, and
   frontmost-app provider, so it is testable.
4. **`BobPanelCoordinator`** (§14). Capture's existing show path goes through it too:
   the status menu, the hotkey, and the notification `showCapture` route.
5. **The status-menu item and the Settings section** (§14), plus the `AppSettings` keys
   with `@Published` sinks that re-register the hotkeys live.
6. **Capture-success refresh.** Subscribe to `successAnnouncementTick`, as in §14.
   **Recheck Bob** re-points the fetcher and restarts the refs watcher.
7. **Tests:**
   - the coordinator: showing Refs hides Capture through `closeRetainingDraft()`, so a
     draft survives; showing Capture hides Refs; toggling closes;
   - the takeover: it registers on Highlights activation, unregisters on any other
     activation, termination, or setting change, honors the setting value (including
     off), matches the override app's bundle id, and checks the frontmost app at launch;
   - Settings defaults and persistence, with a `UserDefaults` suite;
   - the menu order and actions;
   - live re-registration when the toggle or the picker changes;
   - a Refs registration conflict is reported.
8. **README:** add "Opening Bob Refs" (the hotkeys, the takeover and why it is scoped to
   Highlights, the Settings section) and the Diagnostics lines.

## Phase `refs-inspector`: kind-adaptive inspector and actions

**Repo:** bob-mac-capture.

1. **RefsCore, testable on Linux:**
   - **`RefsShowResponse`** (schema 1). Each row has `annotations` (`pageLabel`, `kind`,
     `quote`, `comment`), `tasks` (`checked`, `mark`, `text`), and `annotationsStatus`.
     All of these are lossy and use `decodeIfPresent`.
   - `RefsFetching` gains `show(path:)`, with arguments
     `["ref", "show", path, "-f", "json", "-c"]` on lane `refs-show`.
   - **`RefsSummaryExtractor`:**
     - `bottomLine(fromLeadText:) -> RefsSummary?` returns either one paragraph of at
       most 360 characters, cut at a word boundary with "…", or up to 3 bullets of at
       most 140 characters each.
     - The heading must match
       `^\s*(\d+(\.\d+)*\s*)?(bottom line|tl;?dr|summary|executive summary|key findings)\s*:?\s*$`
       (case-insensitive).
     - The excerpt is the text after the heading, up to the next blank line or the next
       heading-like line.
     - `abstract(fromLeadText:)` returns at most 420 characters, starting after a line
       equal to "Abstract" or a line beginning "Abstract—" or "Abstract:".
   - **`RefsReadingTime.estimate(words:) -> RefsReadingTime`:**
     - 230 words per minute, minimum 1 minute;
     - at 20 minutes or more, round to the nearest 5;
     - Pomodoros = `ceil(minutes / 25)`;
     - it formats as "≈ 17 min · 1 Pomodoro".
   - **`RefsOutline.clean(_:)`:**
     - trim each title;
     - drop empty titles and "Contents" / "Table of Contents";
     - dedupe;
     - keep at most 6.
   - **Tests** cover each, with text fixtures modeled on pandoc report page text.
2. **App:**
   - **`RefsPDFIntrinsicsLoader`** (an actor using PDFKit):
     - It produces the page count, the outline (top level; descend once when the root
       has a single child with children), a page-1 thumbnail at 2× for 112 × 145 pt
       using the `.cropBox`, lead text from up to 3 pages (capped at 12k characters),
       and a word estimate (`words(first ≤ 5 pages) / n × pageCount`).
     - Results are cached by `(path, fileSize, mtime)`:
       - in memory, as an LRU of 64;
       - on disk in `~/Library/Caches/org.bobs.bob-mac-capture/refs/`
         (`intrinsics.json`, capped at 1000 entries, plus a `thumbs/<sha256>.png` per
         thumbnail).
     - A locked document reports "Preview unavailable: encrypted".
     - A load over 3 s reports "Preview unavailable" and is cancelled.
   - **`RefsInspectorLoader`:**
     - It waits for the selection to settle for 120 ms, then cancels the previous load.
     - It loads intrinsics and `ref show` concurrently.
     - It keeps an LRU of 32 `ref show` results keyed by `(id, snapshot.fetchedAt)`.
     - It publishes content updates only, so the inspector fills in without layout
       jumps.
     - Page counts and outline headings feed back into `RefsSignals`, as a content
       update, so rows gain `N pp` and T3 can match headings.
   - **`RefsInspectorView`:** upgrade to the full column of §10. Reserve fixed space for
     the thumbnail so nothing jumps.
   - **`RefsActionsMenu` (⌘K).**
     - It is an `NSMenu` popped up below the selected row; when the row frame is
       unknown, at the list's center.
     - Items:
       - Open in Highlights (↵), Open Note (⌘↵), Reveal in Finder (⌥↵);
       - a separator;
       - Copy Wiki Link (bob's `link` field, verbatim);
       - Copy PDF Path (absolute);
       - Open Source URL, shown only when `urls` is non-empty;
       - Open Narration, shown only when `audio` is set; it opens with the default app;
       - a separator;
       - Open in Default App;
       - Refresh Library (⌘R).
     - A copy shows a 1.5 s footer toast ("Copied wiki link").
     - Add "⌘K Actions" to the footer hints.
   - **fake-bob:** add a `ref show` fixture with commented highlights, tasks, and a
     missing-annotations variant.
3. **Tests:**
   - the loader cancels on a selection change and caches;
   - the show-failure path keeps the basic inspector and adds a dim "Notes unavailable";
   - the actions menu items vary with `urls` and `audio`;
   - copy writes the expected pasteboard strings (use an injected pasteboard);
   - design renders:
     - `refs-inspector-chat-880` (outline plus Bottom line);
     - `refs-inspector-paper-880` (a thumbnail stand-in image plus the abstract and
       notes);
     - `refs-inspector-encrypted-880`;
     - each in light and dark.
4. **Look at the artifact.** Check:
   - the thumbnail's shadow and hairline;
   - the outline's rhythm;
   - quote-bar alignment;
   - that nothing jumps between the basic and full states (render both).
5. **README:** add "Inspector" (what each kind shows, the honesty rules, the caches).

## Phase `refs-closeout`: coherence, memory, final verification

1. **README.** Read the whole `## Bob Refs` section end to end. Fix contradictions,
   duplicated paragraphs, and stale names. Add "Privacy" (the Privacy ground rule) and
   "Troubleshooting":
   - the hotkey conflict;
   - the takeover not firing (check the Settings picker and that Highlights is
     frontmost);
   - "Highlights not found";
   - "Update failed";
   - where the caches live.
2. **bob-cli.** Confirm that `docs/ref.md` names Bob Mac Capture's Refs panel as a
   consumer of the `ref list` envelope.
3. **Memory decision `refs_decision_memory`:**

   > [!decision] refs_decision_memory Write the `refs-panel-is-a-thin-client` decision
   > strand described below.

   > [!decision] refs_decision_memory = no Write no memory. Handle the off case as
   > described below.
   - **If accepted:** use `/sase_memory_write` to add a `decisions` strand,
     `refs-panel-is-a-thin-client`, with the claim "Bob Refs Ranks bob's Reference Index
     And Never Mutates It". It follows the decisions-web template: Applies to, Claim,
     Why (with rejected alternatives), Cost, Reopens when, and Evidence. Evidence is the
     research report, this plan, and the phase commits. Link
     `[[mac-capture-is-a-thin-client]]` and `[[task-lanes-are-sticky]]`. Then run
     `sase memory init`.
   - **If a human turned it off:** do nothing.
   - **If `%auto` left it off:** record a `PROPOSED FOLLOW-UP:` note on this phase's
     bead.

4. **Final checks:**
   - CI is green on the final `master`.
   - Download `render-fixtures` and review every `refs-*` PNG one last time.
   - `just check` passes in bob-cli.
   - Linux `swift test` passes, if a toolchain exists.
5. **Proposed follow-ups.** Record them as `PROPOSED FOLLOW-UP:` notes on the phase
   bead. Do not create beads.
   - Spotlight full-text "Found in text" section.
   - Learned query→pick latching.
   - Open at the furthest highlight page (`highlights://` absolute-path deep links,
     after a Mac spike).
   - "Open in Highlights" on capture's "Already in your library" card, and a "Capture
     follow-up…" action.
   - A `bob ref list` PDF-only filter, if payload size ever matters.
   - Unindexed `lib/` PDFs, if orphans appear.
   - Quick Look.
6. **Leave Bryan the checklist below** in the final report. State plainly what was and
   was not verified.

## Manual verification for Bryan (after `just install` on the Mac, and a bob-cli install there)

**Highlights.** These spike items could not be verified from agent hosts:

1. ↵ opens the PDF in Highlights when Highlights is not running, and when it is.
2. Opening an already-open PDF focuses its existing window, not a duplicate.
3. With Highlights frontmost, the configured key (⌘O by default) opens Bob Refs instead
   of Highlights' Open… dialog. Other apps' ⌘O and ⌃O still work. File › Open… in
   Highlights still works.
4. ⌃⇧⌘R works from any app, and nothing else on the Mac claims it.
5. After a few opens, "Recently opened" and "opened … ago" reflect them.

**The panel:**

6. Hotkey to a painted panel feels instant. The `refs-panel-show` signpost is available
   in Instruments if needed.
7. Rows never jump while you are moving through the list during a refresh. To test,
   touch a ref note while the panel is open.
8. While a Capture draft is open, ⌃⇧⌘R swaps to Refs. Reopening Capture shows the draft
   intact.
9. Light, dark, Increase Contrast, Reduce Transparency, and Reduce Motion all look
   right.

**Timing.** Time three tasks against Highlights' Open… dialog:

10. Reopen yesterday's reference.
11. Find one by a partly remembered title or author.
12. Retrieve an old finished one.

## Out of scope

- Anything that writes the vault or changes reading state, lanes, or review state.
- A separate `bob-mac-refs` app, or extracting a shared kit.
- A new `bob ref catalog` command or a `--pdf` filter. There are 0 orphan PDFs today.
- An `xlib/` intake section. On the Mac it is gitignored and normally empty.
- Phase-4 research items, listed under the closeout's proposed follow-ups.
