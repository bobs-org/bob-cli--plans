---
tier: epic
title: 'Go To Notes follow-ups: Recent, Archive scope, living-first [[, and <C-o>
  back'
goal: 'The `//` Go To Notes picker remembers what Bryan opened (a Recent group), reaches
  archived notes behind Command-X, lends its living-note tiers to `[[` link completion
  so machinery and zorg satellites stop crowding the 20 suggestion slots, and a line-less
  note open records a Vim jump so `<C-o>` returns to where Bryan was.

  '
decisions:
  link_tiers:
    ask: Drop archived and hidden notes from vault-wide `[[` suggestions instead of
      listing them after living notes?
    choices:
      demote: List them after every living match, so every note stays reachable from
        completion
      hide: Remove them from vault-wide suggestions; typing the path by hand still
        links them
    default: demote
    why: Living notes fill the 20 slots first, and linking a done/ note stays possible.
    answer: demote
phases:
- id: cli-archive-scope
  title: 'bob-cli: shared note tiers and `capture-complete -A` archive catalog'
  depends_on: []
  size: medium
  description: 'cli-archive-scope: give `note_catalog` one shared path-tier predicate
    (living, archive, hidden), add an archive catalog of top-level done/ notes, superseded
    library folders, and zorg daily satellites, and serve it behind a new `capture-complete
    -A/--archived-notes` flag with an `archived` marker; help, docs, and tests.

    '
- id: cli-link-tiers
  title: 'bob-cli: living notes first in vault-wide `[[` link completion'
  depends_on:
  - cli-archive-scope
  size: small
  description: 'cli-link-tiers: tag every `[[` note-index entry with the shared tier
    and order (or drop, per decision link_tiers) archive and hidden notes after living
    matches in `[[note`, `[[##`, `[[^^`, and record link-value suggestions, leaving
    target-scoped completion and link resolution unchanged.

    '
- id: mac-recent-notes
  title: 'Mac: Recent group in the empty `//` picker from an app-local open log'
  depends_on: []
  size: medium
  description: 'mac-recent-notes: record each successful `//` open in an app-local,
    schema-versioned open log, show up to eight recently opened notes as a Recent
    section after Today in the empty Notes picker, add a Settings action that clears
    the log, and land green on macOS CI.

    '
- id: mac-archive-scope
  title: 'Mac: Command-X toggles Go To · Archive in the `//` picker'
  depends_on:
  - cli-archive-scope
  - mac-recent-notes
  size: medium
  description: 'mac-archive-scope: claim the reserved Command-X in Notes to toggle
    a Go To · Archive scope fed by `capture-complete --archived-notes`, with stale-safe
    scope state, archive rows and groups, an old-bob hint, generated fixtures, docs,
    and green exact-SHA macOS CI.

    '
- id: plugin-open-notes-jump
  title: 'bob-plugins: line-less bob-open-notes opens record a `<C-o>` jump'
  depends_on: []
  size: small
  description: 'plugin-open-notes-jump: in bob-navigation-hotkeys, record a Vim jump
    from the prior caret to the landed note when `obsidian://bob-open-notes` opens
    a Markdown note without a line, with tests, docs, and a deployed sync.'
proposed_by: bbugyi200.athena.1cs
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-16 10:14:19
status: wip
bead_id: bob-cli-mb
---

- **PROMPT:** [prompts/202611/note_goto_followups.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/note_goto_followups.md)
- **BEAD:** [bob-cli-mb](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-mb/README.md)

# Go To Notes follow-ups: Recent, Archive scope, living-first `[[`, and `<C-o>` back

## Why

Epic `bob-cli-ly` shipped the `//` Go To Notes picker on 2026-11-16. Its research report
(`research:202611/note_goto_double_slash/note_goto_double_slash.md`, "Follow-ups not in
v1") and plan (`plan:202611/note_goto_double_slash.md`, "Out of scope") list the
follow-ups. Bryan asked for the most useful ones, with no decision-record memory work.
This epic takes four of them.

| Follow-up                               | Value                                                                                                                                                                                                    | Evidence                                                                                                                               |
| --------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| **Recent group** in the empty `//` view | The notes Bryan just opened are his likeliest next targets. Today the empty view is a ~180-row map with no history                                                                                       | Research: "Recent group in the empty state (capture-local open history preferred)"                                                     |
| **Archive scope** behind ⌘X             | Every note becomes findable again (cdx's "deceptively incomplete search" concern). ⌘X was consumed in `//` to reserve it for this                                                                        | Research: "Attic scope: ⌘X reveals `done/`, satellites, `ref_history/`"                                                                |
| **`[[` adopts the tiers**               | On athena today, `[[2025111` fills 14 of its 20 slots with zorg satellites (`20251110_done.md`, `_habit`, `_poms`, …). `[[bob` puts `_meta/Bob Home.md` second. `[[cash` puts `done/cash_done.md` second | Live read-only probe, `bob 0.5.0`, 2026-11-16. Research: "Decide whether `[[` completion adopts the same hidden tier"                  |
| **`<C-o>` after a line-less open**      | Bryan navigates with the Vim jump list (walk landings record jumps). A `//` open is the one Bob landing that never records one                                                                           | bob-plugins `676-plugin-open-notes-uri.js`: the origin is captured, but on the no-line path it is dropped. Research: "Vim jump polish" |

**Deliberately not taken (still follow-ups):**

- **Option-↩ opens a ref row's PDF.** The Bob Refs panel already opens PDFs in
  Highlights, with frecency. `note_goto` ref rows carry no PDF path, and ⌥↩ already
  means "reveal in Finder" in Refs. Option-Return stays consumed in `//`.
- **`//note#heading`.** It needs grammar work and a with-line landing.
- **`///` full-text search.** No content index exists yet.
- **Active-leaf `bob-goto-note` lander and ⌘↩ "new tab".** Neither has evidence of need,
  and both need new plugin URI parameters.
- **The decision record.** Excluded by Bryan.

## Ground rules (every phase)

- **Repositories.**
  - `cli-*` phases work in this project's own bob-cli checkout.
  - `mac-*` phases work in the linked repo **bob-mac-capture**.
  - `plugin-open-notes-jump` works in the linked repo **bob-plugins**.
  - Open a linked repo with `sase repo open <name> -r "<reason>"`, read its `AGENTS.md`,
    and use only the path the command prints.
- **Thin client** (`decisions:mac-capture-is-a-thin-client`):
  - bob decides which notes exist, their tier, kind, title, keys, detail, group, and
    canonical order.
  - The app may filter or arrange what bob returned. The Recent group is that kind of
    arrangement: it re-shows bob's own rows in an order taken from the app's own open
    history.
  - The app never reads the vault or decides syntax.
- **Navigation never writes.** No phase adds a vault write on any `//`, `[[`, or open
  path. The Recent log lives in the app's Application Support directory, never in the
  vault, and is never logged with paths.
- **Additive JSON only.** `capture-complete` stays at `schema_version` 1 and grammar
  revision 28: a new flag and new fields are not grammar. The app decodes every new
  field with `decodeIfPresent`. Output for every context other than `note_goto`,
  wikilinks, and record link-value suggestions stays byte-identical.
- **Contract names are fixed:** `-A/--archived-notes`, `note_goto.archived`, kinds
  `archive` and `satellite`, group `done`, `Go To · Archive`, `NoteGotoOpenLogStore`,
  and `note-goto-open-log.json`. If a compile constraint forces a change, add an
  `INTERFACE CHANGE:` note on the phase bead and on the epic.
- **Fixtures are synthetic.** Never commit real vault titles or paths.
- **bob-cli gate.**
  - Run `just all`, plus `just docs-gen` and `just docs-check` when help or docs change.
  - Agents never edit the `Cargo.toml` version or `CHANGELOG.md`
    (`decisions:bob-cli-releases-ride-a-daily-train`).
  - Nothing here breaks a contract, so no subject carries `!`.
  - Read `sase memory read cli_rules.md` and `sase memory read docs.md` before changing
    help or docs. `docs/reference/capture-syntax.md` sits at 5,997 of 6,000 words and
    needs no change here; do not edit it.
- **bob-mac-capture gate.** Follow its `AGENTS.md` exactly:
  - Never land Swift you have not compiled.
  - Run `just portable-check`. It covers CaptureCore and RefsCore and runs on athena's
    Linux Swift.
  - Make a best-effort scratch compile on the Mac for `Sources/BobMacCapture*` and
    `Tests/BobMacCapture*`:
    - rsync a scratch copy without `.build`/`.git`;
    - run `Scripts/bundle.sh` and `just typecheck-tests`;
    - delete the copy;
    - never touch Bryan's checkout and never run `just install`.
  - Run `Scripts/macos-ci-check` under `/sase_monitor` (verify profile, a timeout of at
    least 95 minutes) before landing.
  - Format only changed lines (4-space indentation, 100 columns).
  - If the Mac is offline, say in the final report that the macOS targets are
    unverified.
- **Mac commit and CI loop.** This plan authorizes each `mac-*` phase to:
  1. run `git pull --rebase` on `master`;
  2. commit through `/sase_git_commit` from the bob-mac-capture checkout with a
     conventional subject;
  3. find the exact-SHA run with
     `gh run list -R bobs-org/bob-mac-capture --workflow CI --commit <sha> --json databaseId`;
  4. watch it through `/sase_monitor`
     (`gh run watch <id> -R bobs-org/bob-mac-capture --exit-status`);
  5. fix forward until it is green.

  A `mac-*` phase is done only when its exact-SHA run is green. Record the run URL and
  SHA on the bead.

- **Mac docs budget.** These pages mount into Bob's site under a 6,000-word reference
  cap:
  - `docs/task-pickers.md` is at 5,999 words (task `bob-cli-ma` tracks restoring its
    headroom): **do not add words there**;
  - `docs/keyboard.md` is at 5,840;
  - `docs/protocol.md` is at 5,916.

  New Notes copy goes in `docs/go-to-notes.md` (356 words). Edit rows in the other pages
  in place. Run `just docs-check` (it calls bob-cli's docs gate).

- **No memory edits** in any phase.

## Shared contract (decided; implement exactly)

### 1. Note tiers: one predicate (`note_catalog`)

`src/native/note_catalog/` gains one public crate predicate that every consumer shares:

```rust
pub(crate) enum NoteTier { Living, Archive, Hidden }
pub(crate) struct TierContext { /* library roots, fixed history dirs */ }
impl TierContext { pub(crate) fn for_vault(bob_dir: &Path) -> Self }
pub(crate) fn path_tier(bob_dir: &Path, relative: &Path, ctx: &TierContext) -> NoteTier
```

`relative` is a vault-relative `.md` or `.base` path. The first rule that matches wins:

| Tier      | Rule (reuse the existing predicates; never re-derive them)                                                            |
| --------- | --------------------------------------------------------------------------------------------------------------------- |
| `Hidden`  | Any path component (directory or file name) starting with `.`, or any **directory** component starting with `_` (H1). |
| `Hidden`  | The configured Highlights library roots `lib` and `xlib`, as `highlights_ref` resolves them (H3).                     |
| `Hidden`  | Any `X.md` with a sibling `X.pdf` (H3).                                                                               |
| `Archive` | The top-level `done/` tree, via `done_archive::is_archive_path` (H2).                                                 |
| `Archive` | The fixed history folders `old_lib/`, `ref_history/`, and `lit_review/` (formerly H3).                                |
| `Archive` | Zorg daily satellites `YYYY/YYYYMMDD_{habit,done,poms,events}.md` (H4).                                               |
| `Living`  | Everything else.                                                                                                      |

Requirements:

- The living walk in `scan` may keep its directory pruning for speed, but it must agree
  with `path_tier`. A test asserts that every row `scan` lists classifies `Living`, and
  that every regular, non-symlinked fixture `.md`/`.base` file `scan` skips classifies
  `Archive` or `Hidden`.
- The living catalog's rows, order, and fields stay exactly as today. The only change to
  living output is the new `archived: false` in §2.

### 2. `capture-complete -A/--archived-notes` (bob-cli)

- **Flag.** Add `-A, --archived-notes`, sorted alphabetically between `--all-tasks` and
  `--bob-dir`.
  - Short help: "List archived notes (top-level done/ archives, superseded library
    folders, and daily satellites) instead of living notes (note_goto context only)".
  - Add it to `capture_flag_guard.rs` `capture_complete_spec`: a long entry, plus `A` in
    `short_flags`.
  - Every other context ignores it. `-C` does not affect `note_goto`, and `-A` does not
    affect `task_goto`.
- **Catalog.** Add `note_catalog::scan_archive(bob_dir) -> NoteCatalog`.
  - It walks the vault, entering `done/` and the history folders, still pruning dot,
    `_`, and library directories, and never following symlinks.
  - It lists every `.md` file whose tier is `Archive`.
  - It reads only frontmatter (the same capped prefix reads), never writes, and turns
    unreadable files into bounded warnings.
  - `rank()` works on it unchanged.
- **Satellite rows** (kind `satellite`, `rank` 8):
  - `title` is `<ISO date> <suffix>`, for example `2026-11-14 habit`.
  - `match_keys` is `[title, stem]`. Weekday aliases are never keys.
  - `detail` is the full weekday name.
  - No `group`, `hue`, `status`, or `depth`.
- **Archive rows** (kind `archive`, `rank` 4, for every other archived note):
  - `title` and `match_keys` follow the living rule: a non-empty frontmatter `title:` or
    else the stem, then the stem when it differs from the title, then the aliases.
  - `detail` is the label of the frontmatter `parent:` wikilink when there is one (for
    example `[[abby]]` gives `abby`; take the target's last path segment, or the alias
    when the link has one). Otherwise it is the containing folder path, for example
    `ref_history/blogs`.
  - Rows under `done/` carry `group: "done"`. Other archive rows have no group.
  - No `hue` or `status`.
- **Canonical archive order.**
  1. `done` rows by case-folded title, then `note_path`.
  2. Remaining `archive` rows by case-folded title, then `note_path`.
  3. `satellite` rows by `note_path` descending (newest first).
- **Response.**
  - The `note_goto` object gains `archived`. It is always present: `true` behind `-A`,
    `false` otherwise.
  - `listed` counts the scope's unfiltered rows.
  - The picker descriptor, replacement range, text format (`kind  title  · note_path`),
    and empty shell output are unchanged.
  - T4 refusal and parsing are unchanged.
- **Budget.** Keep `capture-complete -A -c 2 -f json -- '//'` at or under 150 ms warm on
  athena's live vault. Record the time, row count, and byte size on the bead. The living
  `//` time must not regress by more than 10%.

### 3. `[[` link completion uses the tiers (bob-cli)

- **Tagging.** `capture_links::NoteIndex::read` tags each note entry with `path_tier`,
  using one `TierContext` per read. The index still contains every note, so insert
  targets, link resolution, target-scoped completion (`[[note#`, `[[note#^`, `[[#`,
  `[[#^`), warnings, and `((` are all unchanged.
- **Vault-wide lists.** The tiers apply to `[[note` (`note_candidates`), the vault-scope
  `[[##` and `[[^^` lists (`heading_candidates` and `block_candidates` with
  `LinkSearchScope::Vault`), and record link-value suggestions
  (`matching_note_inserts`).
- **Speed.** Detect PDF siblings without one extra `stat` per note where you can, for
  example from the directory listings the walk already reads. Record `[[bob` and
  `[[2025111` timings before and after on the bead. Neither may regress by more than
  10%.

> [!decision] link_tiers = demote Order each list as a stable partition: every `Living`
> match first in today's score order, then `Archive` matches, then `Hidden` matches,
> each in today's score order. Truncate to `RESULT_LIMIT` (20) only after partitioning.
> Help and docs say vault-wide suggestions "list living notes first; archived and hidden
> notes follow".

> [!decision] link_tiers = hide Drop `Archive` and `Hidden` entries from the four lists
> before ranking and truncating. Help and docs say vault-wide suggestions "omit archived
> and hidden notes; type the path to link one".

### 4. Recent group (bob-mac-capture, app-local)

**`NoteGotoOpenLogStore`** lives in CaptureCore and mirrors `RefsOpenLogStore` and
`FileCanceledDraftStashStore`:

- The file is
  `~/Library/Application Support/org.bobs.bob-mac-capture/note-goto-open-log.json` with
  `{"schema_version": 1, "opens": [{"vault": "<hex>", "note_path": "…", "at": "<ISO-8601>"}]}`.
- `vault` is the FNV-1a 64-bit hex hash of `note_goto.vault_root`, the same scheme as
  the inbox route key, so the log never stores the vault path.
- `load()` never creates the directory.
- `append` loads, appends, prunes, and saves atomically, so concurrent panels never lose
  each other's opens.
- Pruning drops events older than 365 days, then keeps the newest 500.
- The file is 0600 and the directory 0700.
- A corrupt file or a wrong schema is quarantined to `note-goto-open-log.corrupt.json`
  and the log starts empty.
- `reset()` deletes the file.
- Errors go to `onError` without paths.

**Recording.**

- Only the success branch of `acceptNoteGotoRow` (after the dispatch reports success)
  appends `{vault, note_path, now}`.
- A failed dispatch, a stale draft, or a double accept records nothing.
- `mac-archive-scope` adds: Archive-scope opens are never recorded.

**Presentation.** `NoteGotoPickerIndex` takes `recentNotePaths: [String]`, newest first
and distinct. The model keeps the log in memory (loaded lazily, updated on append) and
passes in the current vault's entries.

- In the grouped (empty-filter) view, a **Recent** section sits directly after Today
  (first when there is no Today section), before Inbox.
  - It holds up to **8** rows: recent paths that exist in bob's current rows and are not
    already in the Today section, in recency order.
  - Each row reuses that note's bob row presentation (glyph, hue, title, detail, status,
    path, detail strip).
  - Each row has a distinct row ID (`recent:` + `note_path`) and the same destination,
    so the note also stays in its bob group.
  - The section is omitted when it is empty.
- The hint row's "Type to search N more notes" count is unchanged.
- Filtered results are unchanged: no Recent rows, no recency boost.
- The warm-snapshot swap path renders Recent too.
- VoiceOver reads the section title "Recent".

**Settings.** Add a destructive **"Clear Go To Notes History…"** button with a
confirmation, in the capture settings (not the Refs section). It resets the store and
the in-memory copy. Accessibility hint: "Clears the Recent rows in Go To · Notes."

### 5. Archive scope (bob-mac-capture)

**Fetch.** `BobProcessClient.captureComplete` gains `includeArchivedNotes`, which
appends `--archived-notes`. If bob reports an unexpected argument, it throws
`BobClientError.unsupportedOption("--archived-notes")`, the same pattern as
`--closed-tasks`. Decode `noteGoto.archived` (`decodeIfPresent`, absent = `false`).

**Keys.**

- ⌘X in `.noteGoto` toggles Notes ⇄ Archive. Today it is `.consumeKey`.
- A key repeat is consumed.
- Every other Notes key keeps its v1 behavior.
- Option-Return and Shift-Return stay consumed.

**Scope state.** Notes needs stale-safe scope state with the semantics of
`CaptureTaskGotoScopeState`: a session ID, a generation bumped per request, desired and
applied scope, and a pending request. Generalize that type so both pickers share it; add
a sibling only if generalizing would churn the tested `/` behavior. Apply a response
only when all of these hold:

- context is `note_goto`;
- `noteGoto.archived` equals the requested scope;
- session and generation are current;
- the range matches.

**Toggle to Archive.**

1. Announce "Loading archived notes…" and show the quiet placeholder.
2. Fetch the full snapshot at `replacement.start` with `--archived-notes`, keeping the
   filter text.
3. Install the rows and announce "Showing archived notes, N notes".

**Toggle back.** Render the warm `NoteGotoSnapshotStore` rows immediately when they
exist, otherwise fetch without the flag. Announce "Showing notes, N notes".

**Warm store.** An archive response never touches `NoteGotoSnapshotStore`, which accepts
only `archived != true`. Archive rows may be kept in memory for the session only.

**Presentation.**

- Caption **Go To · Archive**; the scope token stays `//`.
- The Notes footer and its accessibility label add `⌘X Show Archive` / `⌘X Show Notes`.
- The grouped view maps group `done` to the section **Done Archives**.
- The hint row reads "Type to search N more archived notes".
- Kind glyphs (long-available SF Symbols): `archive` → `archivebox`, `satellite` →
  `calendar`, unknown → `doc.text`. VoiceOver kind labels: "Archived note" and "Daily
  log".
- Empty states: "No archived notes" (zero rows) and "No archived notes match"
  (filtered).
- Fix the adjacent `/` Completed copy as well: it now reads "No completed tasks" instead
  of "No open tasks in your capture notes".
- The Recent section never shows in Archive.

**Lifecycle.**

- `/`→`//` continuation always enters Notes.
- Backspace on an empty filter peels to `/` Tasks and resets the scope.
- Escape clears the filter, then leaves the `//` reopen chip, which keeps the scope. The
  chip reopen re-requests Archive, matching how `/` keeps Completed.
- Accept in Archive opens the archived note through the same line-less URL.

**Old bob.** On `unsupportedOption("--archived-notes")`, stay in Notes and show the
status "Update bob to show archived notes". Nothing is captured.

### 6. `<C-o>` after a line-less open (bob-plugins `bob-navigation-hotkeys`)

In `src/676-plugin-open-notes-uri.js`, the no-line branch of `focusBobOpenNotesPrimary`
activates the leaf and records nothing today. After activation, when the primary file is
Markdown and an origin was captured, record the transition with the existing helper:
`await this.finishVimLinkJump(jumpContext, file.path)`.

- Equal origin and destination record nothing (`vimJumpLocationsEqual`).
- No origin (for example, an empty active leaf) records nothing.
- PDFs, `.base`, and other non-Markdown files record nothing.
- The with-line path is unchanged and still records exactly one transition.
- Batch entries after the first are unchanged.

## Phase `cli-archive-scope`: shared tiers and the archive catalog

Implement contract §§1–2 in this repository.

- **Code.**
  - Add `NoteTier`, `TierContext`, and `path_tier` to `src/native/note_catalog/` (split
    out a `tiers.rs` if `mod.rs`, now 1,625 lines, grows).
  - Route `path_hidden` and the walk's pruning through the shared rules.
  - Add `scan_archive` and the satellite and archive row builders. Reuse `done_archive`,
    `highlights_ref` config, the daily-date parser, and the existing alias and `title:`
    helpers.
  - Add `NoteKindKey::Archive`/`Satellite` (snake_case `archive`/`satellite`) and
    `NoteGroup::Done`.
  - Thread the flag through `capture_complete/cli.rs` (the `-A` arg and its long-help
    sentence after the `//` sentence) and through `engine.rs` and
    `note_goto_candidates`.
  - Add `archived` to `NoteGotoContext`.
  - Add the example `bob capture-complete -A -c 2 -f json -- '//'` to the after-help.
- **Docs.** Update `docs/capture.md` "Going to a note with `//`":
  - the tier table gains the Archive tier;
  - add a short "Archive scope (`-A`)" paragraph;
  - add the row kinds `archive`/`satellite`, group `done`, and `note_goto.archived`.
- **Generated.** Run `just docs-gen` and update any help golden it changes.
- **Tests.**
  - Unit tests on a `TempDir` vault. Extend the existing tier fixture with
    `done/a_done.md` (`parent: "[[a]]"`), `ref_history/x/h.md`, `old_lib/o.md`, and the
    four satellite suffixes for two dates.
    - `path_tier` for every fixture path, including `_meta/`, `.bob/`, `lib/` plus a PDF
      sibling, and `projects/done/keep.md` (Living).
    - The living set is byte-identical to today's golden.
    - The exact archive set, with per-row fields and canonical order.
    - Satellites: `1114` returns them newest first, and `sat` never matches.
    - `a` matches the done twin through its title.
    - Hidden-tier files never appear in either scope.
    - The vault is byte-identical after both scans.
  - CLI tests in `tests/cli/capture/note_goto.rs`:
    - `-A` JSON for `//` and `//a` (`archived: true`, the §2 key sets);
    - living JSON now carries `archived: false`;
    - `-A` is ignored for `/`, `[[`, and `:`;
    - the human text format;
    - the flag guard accepts `-A`.
- **Gate and probe.** Run `just all` and `just docs-check`. Then run
  `cargo build --release` and probe the live vault read-only with the built binary
  (never install it) for `-A -c 2 -f json -- '//'` and plain `//`. Record time, rows,
  bytes, and the per-tier counts on the bead.
- **Commit:** `feat(capture): list archived notes for // behind capture-complete -A`.

## Phase `cli-link-tiers`: living notes first in `[[`

Implement contract §3 in this repository, on top of `path_tier`.

- **Code.**
  - Add a `tier` field to `capture_links::NoteEntry`, set in `NoteIndex::read`.
  - Apply the partition (or the filter) in `note_candidates`, in the `Vault` branches of
    `heading_candidates`/`block_candidates`, and in `matching_note_inserts`.
  - Keep the 20-row `RESULT_LIMIT` after ordering.
- **Help and docs.**
  - In the `capture-complete` long help's wikilink paragraph, add the sentence from the
    chosen §3 branch.
  - In `docs/capture.md`, add the same rule to its wikilink completion section with a
    pointer to the `//` tier table.
  - Run `just docs-gen`.
- **Tests** (in `capture_links.rs` tests, on a `TempDir` vault):
  - A done twin, a `_meta/` page, a `lib/` sidecar beside a PDF, and satellites rank
    after (or are absent from, per the branch) every living match for `[[`, `[[##`, and
    `[[^^`.
  - A date query returns living dailies before satellites.
  - When more than 20 notes match, the 20 visible rows are all living.
  - An explicit `[[done/a_done#` still completes that note's headings.
  - Link resolution of `[[done/a_done]]` is unchanged.
  - The empty `[[` query orders living notes first.
  - Record link-value suggestions follow the same rule.
  - Update existing goldens only where the new order is intended.
- **Gate and probe.** Run `just all` and `just docs-check`. Then run a release build and
  a read-only live probe of `[[cash`, `[[bob`, and `[[2025111`. Record the before and
  after rows and timings on the bead.
- **Commit:** `feat(capture): suggest living notes first in [[ link completion`. Under
  `link_tiers = hide`, the subject reads
  `omit archived and hidden notes from [[ suggestions`.

## Phase `mac-recent-notes`: Recent group from an app-local open log

Implement contract §4 in bob-mac-capture. No bob change is needed.

- **CaptureCore.**
  - Add `NoteGotoOpenLogStore` and its event and file types, modeled on
    `RefsCore/RefsStores.swift`. Keep a pure FNV-1a helper so the vault hash is
    testable.
  - Add the `recentNotePaths` input to `NoteGotoPickerIndex.groupedPresentation()`, with
    `recent:`-prefixed row IDs that resolve to the same `CaptureNoteGotoDestination`.
- **Model and controller.**
  - Inject the store into `CapturePanelModel` (closure or protocol, like
    `taskGotoOpener`, so tests use a temp file).
  - Append only in the success branch of `acceptNoteGotoRow`.
  - Feed recent paths into every Notes index build: the real `//` response, the warm
    snapshot swap (`presentWarmNoteGotoSnapshot`), and the chip reopen.
  - Make sure acceptance of a `recent:` row dispatches the same URL.
- **Settings.** Add the §4 button and confirmation, wired through `AppDelegate`.
- **Tests.**
  - `CaptureCoreTests`:
    - the store's append, prune, quarantine, schema rejection, permissions, no directory
      creation on `load()`, and `reset()`;
    - presentation: Recent after Today, first without Today, cap 8, newest first,
      missing paths skipped, Today duplicates skipped, distinct IDs, omitted when empty,
      hint count unchanged, filtered view unchanged.
  - Fake-bob panel tests:
    - a successful accept appends one event;
    - a failed dispatch, a stale draft, and a double accept do not;
    - Recent shows on the next empty `//`, including the warm swap;
    - accepting a Recent row opens the same `bob-open-notes` URL;
    - clearing the history empties Recent.
- **Docs.**
  - `docs/go-to-notes.md`: a short Recent paragraph saying it is app-local, never in the
    vault, and cleared in Settings.
  - `docs/settings.md`: the new button.
  - Do not edit `task-pickers.md`.
- Follow the commit and CI loop. **Commit:**
  `feat(capture): show recently opened notes in the // empty state`.

## Phase `mac-archive-scope`: Command-X toggles Go To · Archive

Implement contract §5 in bob-mac-capture. Build the bob used for fixtures from bob-cli
at or after `cli-archive-scope`.

- **Fixtures.**
  - Extend `Scripts/generate-note-goto-fixtures.sh`'s synthetic vault with a `done/`
    note (with `parent:`), a `ref_history/` note, and satellites.
  - Generate `note-goto-archive.json` (`-A -c 2 -f json -- '//'`) and a filtered archive
    fixture, and regenerate the living fixtures (they gain `archived: false`).
  - Check in one hand-written old-bob "unexpected argument '--archived-notes'" error
    fixture.
  - If the generated keys differ from §2, record an `INTERFACE CHANGE:` note.
- **Code.**
  - `BobProcessClient`: the flag and the unsupported-option error.
  - Decode `noteGoto.archived`.
  - Router: in `pickerCommand` `case KeyCode.x`, `.noteGoto` now returns a new toggle
    command. Map it in `CapturePanelController`.
  - Scope state (generalized per §5).
  - Model: toggle, fetch, settle, and failure. Gate `NoteGotoSnapshotStore` and Recent
    recording on the scope. Carry the scope in the chip, and reset it on peel and on
    continuation.
  - Presentation: caption, footer, `done` group title, archive hint copy, glyphs,
    VoiceOver labels, empty states, and the `/` Completed empty-state copy fix.
- **Tests.**
  - Update `CaptureNoteGotoPanelTests.testRouterReservesNotesKeys…`: ⌘X now toggles; its
    repeat is consumed.
  - Scope-state unit tests.
  - Presentation tests: Done Archives section, hint copy, glyph table, empty states,
    Completed copy.
  - Fake-bob panel tests:
    - toggle there and back, with exact `--archived-notes` argv;
    - a stale race: toggle twice fast and let the late living response arrive; it must
      not install over Archive, and the reverse;
    - the warm store is untouched by an archive response;
    - the chip reopen keeps Archive;
    - the peel resets the scope;
    - accepting in Archive opens the `done/` path and records no Recent event;
    - the old-bob hint;
    - no `capture` call ever.
  - Existing `CaptureTaskGotoPanelTests` and `CaptureTaskGotoScopeStateTests` stay
    green.
- **Docs.**
  - `docs/go-to-notes.md`: an Archive paragraph. Replace "Command-X … consumed" with the
    toggle; Shift-Return and Option-Return stay consumed.
  - `docs/keyboard.md`: edit the Go To Notes Command-X row in place.
  - `docs/protocol.md`: add `--archived-notes` and `note_goto.archived` to the
    `note_goto` capability row and the compatibility table, within its 84-word headroom.
- Follow the commit and CI loop. **Commit:**
  `feat(capture): toggle a // archive scope with Command-X`.

## Phase `plugin-open-notes-jump`: `<C-o>` after line-less opens

Implement contract §6 in bob-plugins, plugin `bob-navigation-hotkeys`.

- **Code.**
  - Edit only `plugins/bob-navigation-hotkeys/src/676-plugin-open-notes-uri.js`, plus
    tests and docs.
  - Run `npm run build`; never hand-edit `main.js`. The fragment stays at or under 1,000
    lines.
  - Follow the repo's manifest and version conventions for a behavior change (copy what
    recent `feat(navigation)` commits did).
- **Tests.** In `scripts/test-navigation-open-notes-uri.cjs` (its harness already stubs
  `captureGoToTaskJumpContext`):
  - a line-less Markdown open records exactly one transition from the captured origin to
    the landed caret;
  - an already-active note at the same caret records nothing;
  - no origin records nothing;
  - `.base` and PDF opens record nothing;
  - the with-line open still records exactly one;
  - a reused background tab records its preserved caret as the destination.
- **Docs.** In `docs/open-notes-uri.md` (mounted into Bob's site), add one sentence
  saying a line-less open of a Markdown note records a Vim jump, so `<C-o>` returns to
  the previous caret. Run `npm run docs:gen` if the inventory changes, then
  `npm run docs:check`.
- **Gate.**
  - Run `npm test` (it includes `build:check`) and the focused suite
    `node --test scripts/test-navigation-open-notes-uri.cjs scripts/test-navigation-goto-task-uri.cjs scripts/test-navigation-jump-history.cjs`.
  - Then deploy: `bob plugins sync --repo "$PWD" --no-pull -p bob-navigation-hotkeys`.
- **Coordination.** Epic `bob-cli-m0` (Ctrl+W window keys) is in progress in the same
  plugin.
  - `git pull --rebase` before building.
  - If sync refuses to overwrite another checkout's deploy, follow `AGENTS.md` and
    `docs/develop/deployment.md` exactly. Never pass `-F` yourself.
- **Commit** through `/sase_git_commit` from the bob-plugins checkout:
  `feat(navigation): record a jump on line-less bob-open-notes opens`.

## Rollout and compatibility

| Mismatch         | Result                                                                                                                                                       |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Old bob, new app | ⌘X in Notes shows "Update bob to show archived notes" and stays in Notes. Recent works, since it needs no bob change. Living rows decode without `archived`. |
| New bob, old app | The old app never passes `-A`, so ⌘X stays consumed. It ignores `archived`. `[[` ordering improves for every client.                                         |
| Old plugin       | `//` opens work as before, just without the jump.                                                                                                            |

- bob-cli ships on the daily train. Neither bob-cli commit is a contract break.
- `cli-archive-scope`, `mac-recent-notes`, and `plugin-open-notes-jump` can start in
  parallel.
- `cli-link-tiers` waits for the shared predicate.
- `mac-archive-scope` waits for the flag and for `mac-recent-notes`, because both edit
  `CapturePanelModel.swift` and `NoteGotoPickerPresentation.swift`.

## Out of scope

- Option-↩ on ref rows opening the PDF in Highlights.
- `//note#heading` landings.
- `///` full-text search.
- A cursor-restoring `bob-goto-note` lander, and ⌘↩ "open in new tab".
- A recency boost in filtered `//` results.
- Syncing Recent across Macs.
- A `bob doctor` tier-drift report.
- Any decision-record or other memory edit.
