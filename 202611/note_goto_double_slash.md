---
tier: epic
title: Double-slash Go To Notes in Bob Mac Capture
goal: 'Typing a second real `/` after the `/` Go To trigger (or pasting a whole-draft
  `//query`) opens a Go To · Notes picker over the vault''s living notes; picking a row
  opens that note in Obsidian without writing anything, while `\\` still types one
  literal backslash.

  '
decisions:
  meta_dir:
    ask:
      Hide the legacy `_meta/` pages from `//` along with every other `_`-prefixed
      folder?
    default: true
    why:
      Measured contents are zorg migration machinery; restoring them later is one tier
      line.
    answer: true
  bases:
    ask: List the vault's Obsidian Bases (`.base` files) as View rows in `//` v1?
    default: false
    why:
      bob-open-notes is unverified on non-Markdown paths; a fast-follow can add them
      safely.
    answer: true
  memory_note_goto_record:
    ask: Write a decisions strand recording the `//` Go To Notes design after it ships?
    memory:
      - decisions
    default: false
    answer: false
phases:
  - id: cli-note-catalog
    title: bob-cli note catalog module (tiers, kinds, rows, map, ranking)
    depends_on: []
    size: medium
    description: "cli-note-catalog: build the read-only `note_catalog` module that walks
      the vault, applies the hidden tiers, classifies kinds, and produces ranked rows
      plus the empty-state map, with golden tests on a synthetic tier vault.

      "
  - id: cli-note-goto
    title: bob-cli `//` grammar claim, note_goto completion, Tasks continuation, docs
    depends_on:
      - cli-note-catalog
    size: medium
    description: "cli-note-goto: claim whole-draft `//query` as `note_goto` across
      parse, refusal, and completion; serve catalog rows; advertise `/` as the bare Go
      To continuation key; bump the grammar; update help, docs, and CLI tests.

      "
  - id: mac-note-core
    title: CaptureCore Notes source, index, row presentation, open URL
    depends_on: []
    size: medium
    description: "mac-note-core: decode the note_goto contract, add the `.noteGoto`
      source, a note index with grouped map and filtered ranking, note row anatomy,
      captions, and the line-less bob-open-notes URL, all unit-tested in
      CaptureCoreTests.

      "
  - id: mac-note-panel
    title: Panel wiring, keys, accept and open, truthful fixtures, Mac docs
    depends_on:
      - cli-note-goto
      - mac-note-core
    size: medium
    description: "mac-note-panel: wire `note_goto` into the panel model and key router
      (continuation, peel, consume rules), open notes read-only, add generated fixtures
      and panel tests, document the picker, and land green on macOS CI.

      "
  - id: mac-note-polish
    title: No-blink scope swap, warm notes snapshot, accessibility, closeout
    depends_on:
      - mac-note-panel
    size: medium
    description:
      "mac-note-polish: keep the card mounted across `/` and `//`, serve rows from a
      warm in-memory snapshot, announce scope changes, run the acceptance closeout, and
      write the decision strand only when its memory decision says yes."
proposed_by: bbugyi200.athena.1cr
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-15 16:46:56
status: wip
---

- **PROMPT:**
  [prompts/202611/note_goto_double_slash.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/note_goto_double_slash.md)

# Double-slash Go To Notes in Bob Mac Capture

## Why

Bob Mac Capture already answers "go to that task" from any app: `/` (or `\` on a blank
draft) opens the Go To picker over every open task. "Go to that **note**" is the missing
sibling. Today Bryan must switch to Obsidian and press ⌘O, whose Quick Switcher lists
machinery, archives, and sidecars alongside real notes.

This epic implements the consolidated research report
`research:202611/note_goto_double_slash/note_goto_double_slash.md` (2026-11-15). Bryan
accepted every requirement adjustment it recommends (R1–R9). Its open questions are
settled here:

- **`_meta/`** — hidden with every `_`-prefixed folder (decision `meta_dir`).
- **`.base` views** — not in v1 (decision `bases`).
- **Landing** — `obsidian://bob-open-notes` with no line: an already-open note is reused
  in any window, otherwise the note opens in an empty active leaf or a new far-right
  tab. No bob-plugins change.
- **⌘↩** — the same as Return, matching `/`.

The design rests on facts verified against the code and the live vault on 2026-11-15:

- `/` Go To is a whole-draft bob token (`task_goto_query_token`,
  `src/native/capture_language/tokens.rs`), and `//` is currently prose **by test**
  (`capture_language/tests/goto.rs`). `capture-rewrite` already leaves a `/` right after
  another `/` literal, so `//` is never turned into `/#`.
- After `/` opens, focus moves into the picker filter field. The Mac router already
  turns a bob-advertised `action_continuation_keys` key typed into an empty filter into
  `continuePickerOperator`, which types the key into the draft and re-analyzes (used by
  `+`, `!`, and `- ^`). That is exactly the hand-off a second `/` needs.
- The vault holds 6,869 Markdown files outside dot-directories (7,268 with them). About
  3,700 are notes Bryan would jump to. The rest are machinery (`_generated` 1,131,
  `.bob` 397, `_meta` 13), archives (`done/` 54), Highlights sidecars (`lib/` 192, 191
  of them beside a same-stem PDF), and zorg-era daily satellites (1,836:
  `YYYYMMDD_habit` 826, `_done` 818, `_poms` 188, `_events` 4 — no other suffix exists).
- A frontmatter-prefix walk (8 KiB cap) of the living tier takes about 85 ms in Python
  on athena with a warm page cache, so v1 needs **no persistent cache**.
- `obsidian://bob-open-notes` already accepts `count=1&path1=<abs>` with no `line1`,
  never moves the caret, and never writes (`docs/reference/plugins/open-notes-uri.md`).
  `ObsidianOpenURL.openNotesURL(for:)` in CaptureCore already builds it.

Following `decisions:mac-capture-is-a-thin-client`, **bob-cli owns the token, catalog,
tiers, kinds, titles, match keys, and ranking data; the app owns keys, pixels, and URI
dispatch.** `decisions:bob-cli-releases-ride-a-daily-train` applies: previously
capturable `//…` drafts now refuse, so the grammar commit is marked `!`.

## The experience (north star)

1. Bryan summons the panel and types `/`. The Go To card opens, captioned **Go To ·
   Tasks**. Its footer gains a quiet **`/ Notes`** chip.
2. He types a second `/`. The card stays put: the scope capsule morphs from `/` to `//`,
   the caption becomes **Go To · Notes**, and task rows cross-fade into a table of
   contents:
   - **Today** — today's daily note, then this week's note;
   - **Inbox** — `mac_inbox` first;
   - **Areas** — tree order, sub-areas indented;
   - **Projects** — open projects, with status capsules;
   - **Notebooks**, then **Albums**;
   - a closing hint row: _Type to search 3,612 more notes_ (bob's real count).

   The status line reads "Pick a note to open — press Tab to browse".

3. He types `cash`. The list narrows across title, file name, and aliases first, then
   paths. `cash.md` (an area, tinted with its area hue) ranks above the 151 references
   that merely mention cash. A row matched through an alias wears a quiet `aka …` badge.
4. Each row says what it is:
   - a kind glyph (tray, stack, hammer, book, calendar, …) tinted with bob's area hue;
   - the title, with matched characters emphasized;
   - a one-line detail (breadcrumb, notebook summary, weekday, or period span);
   - the vault path, middle-truncated so the file name survives.

   The detail strip reads "↩ Opens cash.md in Obsidian — nothing is written".

5. He presses Return (or Tab, ⌘↩, ⌃↩, or clicks the row). Obsidian comes forward on that
   note — reusing its tab if it is already open, with caret, scroll, and Reading/Source
   mode untouched — and the panel hides with the draft discarded. The vault is
   byte-identical.
6. `//1114` lists every November 14 daily note, newest first. Letters such as `sat`
   never flood with Saturdays: daily notes match only their date, never their weekday
   aliases.
7. Backspace on an empty `//` filter peels one slash back to Go To · Tasks; Backspace on
   an empty `/` filter returns to the idle agenda. Escape clears the filter, then
   collapses to the reopen chip.
8. `\` still works exactly as today. On a blank draft it becomes `/` and arms a one-shot
   literal, and a second `\` types one literal `\`. Only a real `/` enters Notes, so `\`
   then `/` opens Notes, while `\\` never does.

## Departures from and refinements of the research

| Topic                                 | Research                                                  | This plan                                                                                | Why                                                                                                                                                                                       |
| ------------------------------------- | --------------------------------------------------------- | ---------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Persistent catalog cache              | `(mtime, size)`-fingerprinted cache under bob's cache dir | **None in v1**; one frontmatter-prefix walk per call, measured and recorded              | A measured ~85 ms Python walk means Rust is well under budget. The Mac's warm in-memory snapshot hides the rest. Reopen if the measured warm `capture-complete` exceeds 250 ms on the Mac |
| Row payload                           | `target` per row, or `vault_root` + relative join         | Top-level `note_goto.vault_root`; rows carry only `note_path`                            | Saves ~150 KB over ~3,700 rows. Joining a root and a path is transport, not grammar                                                                                                       |
| Match fields                          | `title`, `aliases`, path, `date` fields                   | Bob-authored **`match_keys`** (primary) plus `note_path` (secondary)                     | Bob alone decides what is searchable, including the date-keyed daily rule. The app applies one generic rule and never special-cases kinds                                                 |
| Today group                           | Today's daily note                                        | Today's daily note **then this week's note**                                             | The week note is the next most-opened period note. Both rows are omitted when the file is missing, and nothing is ever created                                                            |
| Grouped map                           | Notebooks & Albums as one group                           | Separate **Notebooks** and **Albums** groups                                             | Separate headers read more clearly and keep each group's order honest                                                                                                                     |
| Backspace peel from a seeded `//cash` | Peel one slash                                            | Remove everything after the first `/`, so the result is `/` (Tasks with an empty filter) | The filter was already emptied, so re-seeding Tasks with `cash` would surprise                                                                                                            |
| Mac phases                            | Two (core, panel)                                         | Three: core, panel, **polish** (no-blink swap and warm snapshot)                         | This keeps each worker's slice of the 21.7k-line `CapturePanelModel.swift` bounded, and lets "beautiful" land as its own reviewed step                                                    |
| `mac-note-core` order                 | After bob-cli                                             | **In parallel** with the bob-cli phases                                                  | The contract below is fixed and the core uses synthetic fixtures. `mac-note-panel` regenerates them from real bob output                                                                  |

## Ground rules (every phase)

- **Repositories.** The `cli-*` phases work in this project's own bob-cli workspace. The
  `mac-*` phases work in the linked repo **bob-mac-capture**: open it with
  `sase repo open bob-mac-capture -r "<reason>"`, read its `AGENTS.md`, and work only in
  the path the command prints. bob-plugins is not touched.
- **Thin client.** bob decides what a draft means, which notes exist, what each row is
  called, what is searchable, and the tie-break order. The app reacts to `capture-parse`
  `needs` and the `capture-complete` `context`/`picker`. It never parses `//` itself and
  opens Notes **only when bob reports `note_goto`**.
- **Navigation never writes.** No phase adds a vault write on the `//` path. That rules
  out:
  - `capture` and `capture-task-id` calls;
  - creating today's note or a missing note;
  - freshness stamps, ledger edits, and walk moves.

  Tests in both repos assert it.

- **The contract below is fixed.** Keep these names exactly: `note_goto`, `NoteGoto`,
  `note_goto_query_token`, `match_keys`, `note_path`, `vault_root`, `.noteGoto`,
  `Go To · Notes`, `Go To · Tasks`, the T4 string, and the group and kind keys. If a
  compile constraint forces a change, add an `INTERFACE CHANGE:` note on the phase bead
  and the epic.
- **Additive JSON only.** `capture-parse` and `capture-complete` stay at
  `schema_version` 1. Output for `:`, `+`, `!`, `&`, `[[`, and `((` stays
  byte-identical. The only `task_goto` change is the additive `picker` descriptor
  (§4.4). The app decodes every new field with `decodeIfPresent`.
- **Fixtures are synthetic.** Never commit real vault titles or paths.
- **Tests match their neighbours.**
  - bob-cli: in-module `TempDir` tests, `capture_language/tests/`, and the
    `tests/cli/capture/` harness.
  - bob-mac-capture: `CaptureCoreTests`, the fake-bob panel tests (`#filePath` fixtures,
    `waitUntil`, closure injection for the opener and panel dismisser), and
    `Scripts/generate-*-fixtures.sh` for truthful fixtures.
- **Swift toolchain.** Follow bob-mac-capture's `AGENTS.md` exactly:
  - Never land Swift you have not compiled.
  - `just portable-check` on a host without Xcode; Linux Swift on athena builds
    `CaptureCore`.
  - Best-effort scratch compile on the mac (`bundle.sh`, `just typecheck-tests`) for
    `Sources/BobMacCapture*`.
  - `Scripts/macos-ci-check` before landing.
  - Format only changed lines (4-space indentation, 100 columns).
  - Swift 5 mode, `@MainActor` on AppKit code, long-stable APIs only, no long `+`/`??`
    chains.
  - If the mac is offline, say in the final report that the macOS targets are
    unverified.
- **Commit and CI loop for bob-mac-capture phases.** This plan explicitly authorizes
  each `mac-*` phase to:
  1. `git pull --rebase` on `master`;
  2. commit with `/sase_git_commit` from the bob-mac-capture checkout, using
     conventional headers (`feat(capture): …`, `test(capture): …`, `docs: …`);
  3. find the run with
     `gh run list -R bobs-org/bob-mac-capture --workflow CI --commit <sha> --json databaseId`;
  4. watch it through `/sase_monitor`
     (`gh run watch <id> -R bobs-org/bob-mac-capture --exit-status`);
  5. fix forward until the whole job is green, reading `gh run view <id> --log-failed`.

  A `mac-*` phase is not done until its exact-SHA run is green. Record the run URL and
  SHA on the bead.

- **bob-cli gate.** Run `just all` (fmt, lint, test) and `just docs-check` when docs
  change. Agents never edit the `Cargo.toml` version or `CHANGELOG.md`.

## Shared contract (decided; implement exactly)

### 1. The `//query` claim (bob-cli grammar)

`note_goto_query_token(item) -> Option<Token>` sits beside `task_goto_query_token` in
`capture_language/tokens.rs`. A draft claims it when all of these hold:

- the item has exactly one line;
- that line tokenizes to exactly one whitespace-delimited token;
- the token starts with `//`, and its third byte, if any, is not `/`;
- the token equals `item.source.trim()`.

The returned token covers the whole query, sigil included. The query is everything after
`//` and may contain further slashes. Execution, the editor parse, and completion all
call this one predicate. `task_goto_query_token` is untouched; it already rejects any
token with a second `/`.

| Draft                                | Reading                                          |
| ------------------------------------ | ------------------------------------------------ |
| `//`                                 | Notes, query `""`                                |
| `//cash`                             | Notes, query `cash`                              |
| `//prj/sase`                         | Notes, query `prj/sase`                          |
| `//2026-11-14`                       | Notes, query `2026-11-14`                        |
| `  //cash\n`                         | Notes, query `cash`                              |
| `/`, `/fix`                          | Tasks (unchanged)                                |
| `///`, `///x`                        | Prose (reserved for a future full-text mode)     |
| `/Users/x`, `/r/rust`, `/a/b`        | Prose (unchanged: the second `/` is not leading) |
| `// cash`, `//a b`, `Buy //x`        | Prose: several tokens                            |
| `//x` with a child line              | Prose: several lines                             |
| `//x`, a blank line, then `Buy milk` | Prose: two items, both captured as usual         |
| `@@work` on one line, then `//x`     | Prose: `//x` routes to `work` as usual           |

### 2. `bob capture-parse`

A claimed draft reports:

- mode `incomplete`;
- `needs: ["note_goto"]`;
- `body: ""`;
- one `interactive_placeholder` span over the whole token;
- no diagnostics.

A `@@` declaration never applies, exactly like `task_goto`. Add `Need::NoteGoto`
(`"note_goto"`) directly after `task_goto` in the needs order. Grammar revision 27 → 28;
its doc comment reads "Revision 28 claims whole-draft `//query` as `note_goto`
(previously prose)".

### 3. `bob capture` refusal (T4)

`bob capture` and `--dry-run` refuse a claimed draft before any other item parser,
beside the T3 check in `item.rs`. Forced flags (`-r`, `-s`) change nothing, the whole
batch rolls back, and no vault file changes. The error keeps the usual
`capture item 1 starting on line L: …` prefix and reads:

`` `//cash` opens the Go To Notes picker; pick a note to open it in Obsidian — it never captures ``

The quoted token is the exact typed token, so a bare `//` reads `` `//` opens … ``. T3
(the `/` message) is unchanged.

### 4. `bob capture-complete`: context `note_goto`

#### 4.1 Field and response

- **Query and replacement.** The query is the token text between `//` and the cursor. It
  is empty at the token start, at offset 1, and at offset 2. The replacement covers the
  whole token, so refetching at `replacement.start` returns the unfiltered snapshot.
- **Response.** `context: "note_goto"` plus the candidates (§4.2) and:
  - `picker`:
    `{kind: "note_goto", scope: "vault", scope_token: "//", marker_range: <token>, trigger_removal_range: {start: token.start + 1, end: token.end}}`,
    with no continuation keys. Deleting the removal range always leaves exactly `/`.
  - `note_goto`:
    `{vault_root: "<absolute vault path>", listed: <row count of the unfiltered catalog>}`.
  - No `query` field.
- **Flags.** `-C/--closed-tasks`, `-a`, `-n`, and `-F` do not affect this context.
- **Warnings.** Unreadable or non-UTF-8 files go into the usual `warnings` array
  (bounded, deduplicated) and are skipped. They are never fatal.

#### 4.2 Row (`NoteGotoCandidate`)

| Key           | Presence                 | Meaning                                                                                                                   |
| ------------- | ------------------------ | ------------------------------------------------------------------------------------------------------------------------- |
| `replacement` | always `""`              | Decoder compatibility; never inserted                                                                                     |
| `note_path`   | always                   | Exact vault-relative path with extension; the app's row ID                                                                |
| `kind`        | always                   | One of `inbox`, `area`, `project`, `notebook`, `album`, `record`, `ref`, `period`, `daily`, `bank`, `mail`, `note` (§5.2) |
| `title`       | always                   | Display title (§5.3)                                                                                                      |
| `match_keys`  | always, non-empty        | Primary searchable strings in priority order (§5.3); `match_keys[0] == title`                                             |
| `detail`      | optional                 | One-line secondary text (§5.3)                                                                                            |
| `status`      | optional                 | Capsule text for projects and refs (§5.3)                                                                                 |
| `hue`         | optional                 | Bob's hue name, the same `Hue` serialization the agenda uses (for example `"blue"`)                                       |
| `rank`        | always                   | Kind tier, 0–8; lower sorts first at equal match score (§5.2)                                                             |
| `group`       | optional                 | Map group key: `today`, `inbox`, `areas`, `projects`, `notebooks`, or `albums` (§5.4)                                     |
| `depth`       | optional, omitted when 0 | Map indentation, at most 2 (area tree)                                                                                    |

Example row:
`{"replacement":"","note_path":"cash.md","kind":"area","title":"cash","match_keys":["cash","money"],"detail":"life","hue":"green","rank":1,"group":"areas"}`.

#### 4.3 Order and rendering

- **Empty query.** Rows come in canonical order (§5.5).
- **Non-empty query.**
  - Score `match_keys` with the existing `capture_link_tasks::match_score`, which
    applies AND across whitespace terms.
  - Rows with a primary match come first, by score descending.
  - Rows whose only match is `note_path` (without `.md`) come next, by score descending.
  - Ties keep canonical position. There is never a row cap.
- **Text format.** One line per row: `kind  title  · note_path`.
- **Shell format.** Yields no rows, because nothing is insertable.

#### 4.4 The `task_goto` continuation descriptor

Every `task_goto` response gains an additive `picker` descriptor:
`{kind: "task_goto", scope: "vault", scope_token: "/", marker_range: <token>, trigger_removal_range: <token>}`.
When the token is exactly `/`, it also carries `action_continuation_keys: ["/"]`. bob
thereby stays the authority that a second `/` escalates. Old apps route `task_goto` by
`context` and ignore the descriptor (verified: `CapturePickerDescriptor.kind` decodes as
a free `String`, and continuation keys are read only for parent-task, complete, and move
sources). Update the `/` section of `docs/capture.md`, which today says `task_goto`
carries "no `picker` descriptor".

### 5. The note catalog (`src/native/note_catalog/`)

**The rule:** `//` lists the documents Bryan writes or reads as notes. It hides what bob
or Obsidian owns as machinery, files that duplicate a listed note, and attachments that
have their own opener. It never hides a real note for being old, empty, or unnamed:
there are no `Untitled`/`tmp`/size heuristics. The tiers apply to `//` only; the shared
`[[` walker and `ALWAYS_EXCLUDED_NOTE_DIRECTORY_NAMES` stay unchanged.

#### 5.1 Hidden tiers (pruned during the walk, never listed)

| Tier            | Rule                                                                                                                                                                                                                                                                                                | Live-vault ≈ |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -----------: |
| H0 Non-Markdown | Only regular `*.md` files are candidates; symlinked directories are never followed (match the existing note walkers)                                                                                                                                                                                |            — |
| H1 Machinery    | Any dot-prefixed directory **or file name** at any depth (`.bob`, `.git`, `.obsidian`, `.sase`, `.trash`, a file literally named `.md`). Any `_`-prefixed directory at any depth (`_conflicts`, `_generated`, `_templates`, `_zorg_templates`, and `_meta` per decision `meta_dir`)                 |        1,550 |
| H2 Archive      | The top-level `done/` tree only, via `done_archive::is_archive_root`. A nested `projects/done/` and a root `done.md` stay live                                                                                                                                                                      |           54 |
| H3 Library      | The configured Highlights library roots (lib and xlib as `highlights_ref` resolves them, honouring `BOB_HIGHLIGHTS_LIB_DIR`/`BOB_HIGHLIGHTS_XLIB_DIR`; defaults `lib`, `xlib`); the fixed top-level `old_lib/`, `ref_history/`, `lit_review/`; and any `X.md` whose sibling `X.pdf` exists anywhere |          193 |
| H4 Satellite    | `YYYY/YYYYMMDD_{habit,done,poms,events}.md`, where `YYYY` is a top-level four-digit directory and `YYYYMMDD` is eight digits                                                                                                                                                                        |        1,836 |

> [!decision] meta_dir = no Keep `_meta/` live: the H1 `_` rule exempts exactly the
> top-level `_meta` directory, and its notes classify as plain `note` rows. Every other
> `_` directory stays hidden.

> [!decision] bases When `bases` is yes, `.base` files outside the hidden tiers are
> listed as kind `view` (rank 4, title = stem, `match_keys` = [stem]). `mac-note-core`
> adds a `tablecells` glyph. `mac-note-panel` must confirm on the Mac that
> bob-open-notes opens a `.base` path. If it does not, the panel phase filters `view`
> rows out and records a `PROPOSED FOLLOW-UP:` instead of shipping a broken row.

#### 5.2 Kinds and rank tiers (first match wins)

| Kind       | Rule (reuse existing predicates; never re-derive them)                                                                                       | `rank` |
| ---------- | -------------------------------------------------------------------------------------------------------------------------------------------- | -----: |
| `daily`    | `YYYY/YYYYMMDD.md` under a top-level four-digit directory, with a valid calendar date                                                        |      8 |
| `inbox`    | An inbox file as `capture_targets` classifies it (`CaptureTargetKind::Inbox`: `mac_inbox`, `inbox`, `gkeep_inbox`, `gmail_inbox`, …)         |      0 |
| `area`     | `frontmatter_is_area`                                                                                                                        |      1 |
| `project`  | `frontmatter_is_project`; open (non-terminal) projects                                                                                       |      2 |
| `project`  | The same, closed (done or canceled per the projects catalog): demoted, never hidden                                                          |      5 |
| `notebook` | The notebooks module's notebook predicate                                                                                                    |      3 |
| `album`    | The album registry                                                                                                                           |      3 |
| `record`   | A record note per the records catalog                                                                                                        |      5 |
| `ref`      | Frontmatter `type: "[[ref]]"` or under the configured ref directory                                                                          |      7 |
| `period`   | A bob period note (week, quarter, year, `all_time`), or any other note whose frontmatter `type` is `[[day]]` (legacy monthly and zorg notes) |      6 |
| `bank`     | The bank hub or a bank month note, per the bank module                                                                                       |      4 |
| `mail`     | Under the configured Gmail folder (`gmail.folder`, default `mail/`)                                                                          |      4 |
| `note`     | Everything else, including root type notes, `AGENTS.md`, `lit/`, `prompts/`, empty files, and `Untitled*.md`                                 |      4 |

#### 5.3 Title, keys, detail, status, hue

- **`title`.**
  - `daily`: the ISO date `YYYY-MM-DD`.
  - Otherwise: a non-empty string frontmatter `title:`, or else the file stem.
- **`match_keys`.**
  - `daily`: `[ISO date, stem]`. Weekday aliases (`2026-11-14 Sat`) are **never** keys.
  - Otherwise: `[title, stem when ≠ title, each frontmatter alias]`. Aliases come from a
    list or a single string, are trimmed and non-empty, and are deduplicated
    case-insensitively against earlier keys.
- **`detail`** (one line, omitted when empty):
  - `daily`: the full weekday name (`Saturday`).
  - `period`: the period's date span, from the existing `periods` label helpers when bob
    recognizes the note (for example `Nov 9 – 15, 2026`).
  - `notebook`: `summary:` when present, else the breadcrumb.
  - Other kinds: the breadcrumb.

  The **breadcrumb** is the ancestor chain from the same `pomodoro_homes` home walk that
  yields `hue` (its `residence_chain`, excluding the note itself), root first, labels
  joined by `›`. It keeps at most the nearest three ancestors and never includes type
  notes such as `area`.

- **`status`.**
  - Projects: the label `bob projects` prints for that project's `^prj` line (for
    example `WIP`, `Blocked`, `Done`, `Canceled`).
  - Refs: the frontmatter `status:` value, capitalized (`Read`, `Next`, …).
  - Omitted otherwise.
- **`hue`.** `Home.hue` from `pomodoro_homes::HomeResolver` (nearest area or project
  `color:`, else the top-level area stem hash). It is omitted when the note has no area
  home. Resolve hue and breadcrumb from one indexed source built over the frontmatter
  prefixes the walk already read, so no note is read twice.

#### 5.4 The empty-state map (`group`, `depth`)

A row appears once. Map rows carry `group`, and every other listed row has none. Group
order and membership:

1. `today`: today's daily note (`pomodoro::day_file_for`, honouring `BOB_DAY_FILE` and
   `BOB_NOW`), then the current ISO week note. Each is included only when its file
   exists.
2. `inbox`: inbox targets in `capture-targets` order (`mac_inbox` first).
3. `areas`: area notes in the area tree's order (`src/native/areas/graph.rs`), with
   `depth` = tree depth capped at 2.
4. `projects`: open projects in `capture-targets` order.
5. `notebooks`: notebook targets in `capture-targets` order.
6. `albums`: album notes in album-registry order.

#### 5.5 Canonical order

Map rows come first, in §5.4 order. All remaining listed rows follow, sorted by
`(rank, within-rank key)`:

- `daily` and `period` rows sort by `note_path` **descending** (newest first);
- every other kind sorts by case-folded `title`, then `note_path`.

#### 5.6 Budget and safety

- One walk.
- Frontmatter-only reads: stop at the closing `---`, capped at 8 KiB. Never parse
  headings, blocks, or tasks.
- No persistent cache.
- Never writes, never creates a file, never touches `.obsidian`.
- Target ≤ 150 ms warm for `capture-complete -c 2 -f json -- '//'` on athena's live
  vault, with JSON under 1 MB. Record the measured time, row count, and byte size on the
  bead.

### 6. Opening a note (app)

Accepting a row builds
`ObsidianOpenURL.openNotesURL(for: CaptureOpenPlan(... one note ...))` for the absolute
path `vault_root + "/" + note_path`, with **no line, ref, or block**. That yields
`obsidian://bob-open-notes?count=1&path1=<percent-encoded absolute path>`.

- There is never a fallback to `obsidian://open?path=`, which has no leaf reuse and
  would clobber the active tab.
- The flow is: stale-draft guard, then a dispatch lock that blocks double accepts, then
  dispatch through the injected opener. On success the panel hides and the draft is
  discarded, never stashed. On failure the picker stays open with "Couldn't open
  Obsidian".

### 7. App behaviour (bob-mac-capture)

| You type (blank draft)                                | Draft               | Result                                                             |
| ----------------------------------------------------- | ------------------- | ------------------------------------------------------------------ |
| `/`                                                   | `/`                 | Go To · Tasks; footer adds `/ Notes` when bob advertised the key   |
| `\`                                                   | `/` (literal armed) | Go To · Tasks (unchanged)                                          |
| `\` `\`                                               | `\`                 | One literal backslash (unchanged)                                  |
| `/` `/`                                               | `//`                | **Go To · Notes** (via bob's continuation key; the arm is cleared) |
| `\` `/`                                               | `//`                | **Go To · Notes**                                                  |
| `/` `\`                                               | `/`                 | Consumed (unchanged)                                               |
| `/` `/` `/`                                           | `//`                | Third `/` consumed on an empty Notes filter (reserved)             |
| `/fix` then `/`                                       | `/fix`              | Filter text; mode switches only on an empty filter                 |
| fast `//cash` or pasted `//cash`                      | `//cash`            | bob claims it; Notes opens with filter `cash`                      |
| `//` ⌫ on an empty filter                             | `/`                 | Back to Tasks (bob's `trigger_removal_range`)                      |
| `/` ⌫ on an empty filter                              | blank               | Idle agenda (unchanged)                                            |
| `\` on an empty Notes filter                          | —                   | Consumed                                                           |
| `/` held down (key repeat), or during IME marked text | —                   | Never escalates                                                    |
| ⌘X in Notes                                           | —                   | Consumed (reserved for a future archive scope)                     |
| Shift-↩, Option-↩ in Notes                            | —                   | Consumed                                                           |
| ⌃C in Notes                                           | —                   | Discard without stashing (same as `/`)                             |
| Return / Tab / click / ⌘↩ / ⌃↩                        | —                   | Open the note (§6)                                                 |
| Escape                                                | —                   | Clear a non-empty filter, then cancel to the `//` reopen chip      |

Compatibility:

| Mismatch              | Result                                                                                                                                       |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Old bob, new app      | No continuation key: a `/` on an empty Tasks filter is consumed and the status line shows "Update bob to search notes". No junk capture ever |
| New bob, old app      | `//` parses as `note_goto`; the old app opens no picker, the preview shows T4, and submit refuses                                            |
| Missing or old plugin | Dispatch fails; the picker stays open with "Couldn't open Obsidian"                                                                          |

## Phase `cli-note-catalog`: bob-cli note catalog module

Implement contract §5 as a read-only library module in this repository. No grammar, no
CLI surface.

- **Module.** Add `src/native/note_catalog/` (for example `mod.rs`, `tiers.rs`,
  `kinds.rs`, `rows.rs`, `rank.rs`, `tests.rs`) and declare it from `src/native.rs`.
  - Public crate API, used verbatim by `cli-note-goto`:
    - `scan(bob_dir: &Path) -> NoteCatalog`, where
      `NoteCatalog { vault_root: PathBuf, rows: Vec<NoteRow>, warnings: Vec<String> }`
      holds rows in canonical order (§5.5);
    - `rank<'a>(catalog: &'a NoteCatalog, query: &str) -> Vec<&'a NoteRow>` (§4.3).

    `NoteRow` carries exactly the §4.2 fields except `replacement`, with
    `kind: NoteKindKey` (snake_case serialization).

  - Until `cli-note-goto` wires it in, put `#[cfg_attr(not(test), allow(dead_code))]` on
    the `mod note_catalog;` line, with a comment naming that phase. `cli-note-goto`
    removes it.

- **Reuse, never re-derive.** Use these existing sources:
  - `done_archive::is_archive_root`;
  - `frontmatter_is_area`/`frontmatter_is_project`;
  - the notebooks, albums, records, bank, and periods predicates;
  - `capture_targets` (inbox kinds and map order);
  - `areas::graph` (tree order and depth);
  - the projects catalog (open/closed state and status label);
  - `highlights_ref` library-root resolution;
  - the Gmail folder config;
  - `pomodoro::day_file_for`;
  - `pomodoro_homes::HomeResolver` (hue and breadcrumb), fed by an indexed source over
    the prefixes already read.

  Where a predicate is private, widen it to `pub(crate)` rather than copying it.

- **Tests** (`TempDir` synthetic vault with one example of every tier and kind):
  - Hidden files:
    - `.bob/x.md`, `.sase/x.md`, a root file named `.md`;
    - `_generated/x.md`, `_meta/x.md` (per `meta_dir`), `_zorg_templates/x.md`;
    - `done/old.md`;
    - `lib/papers/p.md` + `p.pdf`, `xlib/q.md`, `old_lib/r.md`, and a stray
      `notes/s.md` + `notes/s.pdf`;
    - `2026/20261114_done.md`, `_habit`, `_poms`, `_events`;
    - a symlinked directory;
    - `x.base` and `img/a.png`.
  - Live files:
    - `projects/done/keep.md` and `done.md`;
    - `2026/20261114.md` and `2025/20251114.md`;
    - `2026/2026-W46.md`, `2026.md`, `all_time.md`, `202511.md` (`type: [[day]]`);
    - an area with `color:`, a nested sub-area;
    - an open project and a closed project;
    - a notebook with `summary:`, an album with one record;
    - `ref/papers/a.md` with `title:` and `status: read`;
    - `mail/2026/m.md`, `bank.md`;
    - `mac_inbox.md`, `inbox.md`;
    - a plain note with `aliases:`, `Untitled.md`, an empty file.

  Golden assertions:
  - the exact listed `note_path` set;
  - per-row `kind`/`rank`/`title`/`match_keys`/`detail`/`status`/`hue`/`group`/`depth`;
  - the canonical order, including newest-first dailies;
  - `today` present only when the day file exists (with `BOB_NOW`), and no file is
    created;
  - the daily keys exclude weekday aliases;
  - ranking cases:
    - `cash` area above a ref mentioning cash;
    - `1114` returns both dailies, newest first;
    - `sat` does not match a daily;
    - an alias-only match;
    - a path-only match ranked after primary matches;
    - AND terms;
  - an unreadable file becomes one warning;
  - the vault is byte-identical after `scan` (hash every file before and after).

- **Measure.** Build release and time `scan` against the live vault read-only through a
  `#[ignore]` test or a scratch binary (never install). Record the time, the row count,
  and the per-tier hidden counts on the bead.
- **Commit** subject: `feat(capture): add read-only note catalog for // Go To Notes`.

## Phase `cli-note-goto`: grammar claim, `note_goto` completion, Tasks continuation, docs

Implement contract §§1–4 in this repository on top of `note_catalog`.

- **Grammar.**
  - Add `note_goto_query_token` (§1), `parse_editor_note_goto_item` plus
    `Need::NoteGoto` (§2), and the T4 refusal with its error helper beside
    `task_goto_picker_error` (§3).
  - Add `note_goto_completion_field_at` plus `CompletionContext::NoteGoto` (§4.1).
  - Grep every `task_goto_query_token`, `TaskGoto`, and `task_goto` site in
    `capture_language/`, `capture_complete/`, and `capture_parse.rs`, and mirror each
    one for notes. That includes the `notebook.rs` guard and the `editor_parse.rs`
    `needs == [Need::TaskGoto]` check.
  - Bump `GRAMMAR_REVISION` to 28 (§2) and remove the `dead_code` allowance on
    `note_catalog`.
- **Candidates.**
  - Add `NoteGotoCandidate`, `Candidates::NoteGoto`, and a `note_goto` top-level result
    object (`vault_root`, `listed`).
  - Add the §4.1 `picker` descriptor (new `PickerKind::NoteGoto`) and the §4.4
    `task_goto` descriptor (new `PickerKind::TaskGoto`).
  - Add the text renderer line and the empty shell arm.
- **Rewrite golden.** `rewrite_draft("//", Some(2))` and
  `rewrite_draft("//cash", Some(6))` are no-ops (`/` must never become `#` here).
  `rewrite_draft("/", Some(1))` is unchanged.
- **Help.** Read `sase memory read cli_rules.md` first. There are no new options or
  subcommands.
  - `capture-parse`: add a sentence after the `/` sentence saying a whole draft that is
    one `//` token reports `incomplete` needing `note_goto` and `bob capture` refuses
    it. Add the example `bob capture-parse -f json -- '//cash'` and add `note_goto` to
    **Needs**.
  - `capture-complete`: add a sentence after the `/` sentence. It says a whole-draft
    `//` token completes `note_goto` over the living notes, with `note_path` rows and a
    top-level `vault_root`, and that the replacement is never inserted. Add the example
    `bob capture-complete -c 2 -f json -- '//'` and add `note_goto` to **Contexts**.
  - Update the `-C` help to say `task_goto` context only (unchanged meaning).
  - Run `just docs-gen` and update any help golden this changes.
- **Docs.** Read `sase memory read docs.md` first.
  - `docs/capture.md`:
    - add a `//<query>` row to "Grammar at a glance" and a Contents entry;
    - add `### Going to a note with '//'` directly after "Going to a task with '/'". It
      covers the §1 table, T4, the catalog rule and the §5.1 tier table, kinds and
      ranks, date-keyed dailies, the map, the row fields, "Go To Notes never writes",
      and a pointer to the [Open notes URI](reference/plugins/open-notes-uri.md);
    - in the `/` section, replace the `//` prose row with a pointer to the new section
      (keep `/Users/x` and `/r/rust` as prose), and document the §4.4 descriptor;
    - insert `note_goto` into the needs order and the contexts list.
  - `docs/reference/capture-syntax.md`: add the `//` token row.
- **Tests.**
  - **Unit** (`capture_language/tests/note_goto.rs`, registered in `tests/mod.rs`):
    every §1 row through `execute`, `editor`, and `field`; cursors at offsets 0, 1, 2,
    and mid-token; `task_goto_query_token` still `None` for every Notes row and
    `note_goto_query_token` `None` for every Tasks row; the `@@` and two-item cases.
    Update `goto.rs`'s `//` prose assertion to expect `note_goto`.
  - **CLI** (`tests/cli/capture/note_goto.rs`, registered in `mod.rs`):
    - `capture-parse` JSON for `//` and `//cash`;
    - every prose row keeps its parse;
    - `capture` and `--dry-run` refuse `//`, `//cash`, and `//cash` with `-r inbox`
      using T4, with vault bytes identical;
    - `capture-complete -f json` for `//` and `//ca` returns the exact §4.1 shape and
      §4.2 key sets on a synthetic vault (reuse the catalog fixture builder);
    - the bare `/` response carries `action_continuation_keys: ["/"]` and `/fix` does
      not;
    - the text and shell formats.
  - Existing `:`, `/`, `+`, `!`, `&`, `[[`, and `((` tests stay green unchanged, except
    the asserted `/` descriptor addition.
- **Gate and probe.** Run `just all` and `just docs-check`. Then run
  `cargo build --release` and probe the live vault read-only with `target/release/bob`.
  Do not install it.
  - Check the `capture-complete -c 2 -f json -- '//'` row count, warm time, and byte
    size.
  - Check that `capture --dry-run -- '//x'` exits with T4.

  Record both results on the bead.

- **Commit** subject (breaking: `//…` drafts no longer capture):
  `feat(capture)!: claim whole-draft //query as Go To Notes`.

## Phase `mac-note-core`: CaptureCore Notes source, index, presentation, URL

Implement the CaptureCore half of §§4, 6, and 7 in bob-mac-capture. This phase uses
hand-written synthetic fixtures that follow the contract exactly. `mac-note-panel`
regenerates them from real bob output.

- **Decode.** Add `decodeIfPresent` fields on `CaptureCompletionCandidate`: `matchKeys`,
  `rank`, `status`, `hue`, plus `kind`, `title`, `detail`, `group`, `depth`, and
  `notePath` where they do not already exist. Reuse existing names and types where a
  field already exists (`kind`, `group`, `depth`, `detail`, and `notePath` already do
  for other contexts). Add a top-level `noteGoto` (`vaultRoot`, `listed`) on the
  completion response.
- **Source.** Add `CapturePickerSource.noteGoto`:
  - scope token `//`;
  - captions **Go To · Notes** and, renamed, **Go To · Tasks** for `.taskGoto`
    (`Go To · Completed` stays);
  - accessibility names "Go To Notes picker" and "Go To Tasks picker".

  Update every exhaustive switch.

- **Index.** Add `NoteGotoPickerIndex` and `CapturePickerIndex.noteGoto`. Row ID =
  `note_path`, never `replacement`.
  - **Grouped (empty filter):** sections in first-appearance order of `group`, titled
    Today, Inbox, Areas, Projects, Notebooks, Albums. Area rows are indented by `depth`
    (existing 14 pt step, max 2). The list closes with one non-selectable hint row,
    "Type to search N more notes", where N is the count of rows without `group`,
    formatted with a grouping separator. The hint row is hidden when N is 0.
  - **Filtered:**
    - `FuzzyQuery(filter, leadingSigil: "//")`, AND across tokens.
    - Each token's best `FuzzyMatcher` score over `match_keys` (primary) and over
      `note_path` minus `.md` (secondary, scored below any primary match).
    - Rows with any primary match sort before path-only rows. Within each class, sort by
      summed score descending, then `rank` ascending, then bob index.
    - Never cap rows.
- **Row anatomy.**
  - **Kind glyph** (long-available SF Symbols): `inbox` `tray`, `area`
    `square.stack.3d.up`, `project` `hammer`, `notebook` `book.closed`, `album`
    `rectangle.stack`, `record` `doc.richtext`, `ref` `book`, `daily` and `period`
    `calendar`, `bank` `banknote`, `mail` `envelope`, `note` `doc.text`, and any unknown
    kind `doc.text`. Tint it with bob's `hue` through the existing `PomodoroHomeColor`
    mapping, else secondary.
  - **Title** with matched-character emphasis when the best primary key is
    `match_keys[0]`.
  - A quiet **`aka <key>`** badge when the best primary key is at index ≥ 1 and is not
    the file stem. The stem is already visible in the path line.
  - **Status capsule** from `status`. Row fills stay native so color never fights
    selection.
  - **Secondary line:** `detail` · path. The path is middle-truncated so the basename
    survives, with emphasis when the path matched.
  - **Detail strip:** "↩ Opens <basename> in Obsidian — nothing is written".
  - **VoiceOver:** "<kind>, <title>, <detail>, <path>", with the hint "Opens in
    Obsidian".
- **URL.** Add `NoteGotoDestination` (`vaultRoot`, `notePath`, `absolutePath`) and a
  helper that returns `ObsidianOpenURL.openNotesURL(for:)` for one line-less note (§6).
- **Tests** (`CaptureCoreTests`, runnable on Linux Swift on athena):
  - decoding a `note-goto.json` fixture and an old-bob response without the new fields;
  - grouped presentation (section order, indentation, hint count);
  - filtered ranking (title over path, rank tie-break, bob-index tie-break, AND terms,
    `1114` daily ordering, the alias badge rule, no cap);
  - a 4,000-row synthetic filter run that stays under a generous time bound;
  - the glyph table;
  - captions;
  - URL encoding for spaces, Unicode, `%`, `#`, `&`, and `+`, with no `line1`;
  - `TaskGoto` presentation tests updated for the caption rename.
- Follow the commit and CI loop. Subject:
  `feat(capture): add Go To Notes source, index, and row presentation`.

## Phase `mac-note-panel`: panel wiring, keys, open, fixtures, docs

Wire Notes into `CapturePanelModel`, `CaptureKeyCommandRouter`, and
`CapturePanelController` per §§6–7.

- **Truthful fixtures.** Add `Scripts/generate-note-goto-fixtures.sh`, following
  `generate-record-fixtures.sh` conventions:
  - an isolated synthetic vault and an explicit `BOB_BIN` built from bob-cli at or after
    `cli-note-goto`;
  - documented inputs;
  - generated `capture-parse` fixtures for `/`, `//`, and `//cash`, and
    `capture-complete` fixtures for `/` (with continuation keys), `//`, and `//cash`.

  Replace `mac-note-core`'s hand-written fixture with the generated one. Check in one
  hand-written old-bob `/` fixture without the descriptor. If the generated keys differ
  from §4, record an `INTERFACE CHANGE:` note.

- **Analysis.**
  - Track `lastAnalysisNeedsNoteGoto` beside the task flag. Skip the doomed live dry run
    and show "Pick a note to open — press Tab to browse".
  - Route `context == "note_goto"` to a handler that refetches the full snapshot at
    `replacement.start`.
  - Seed the filter from the text after the two-byte sigil.
  - Install `.noteGoto` only when the response context is `note_goto` for the current
    draft and generation. A late `task_goto` response must never install over Notes, or
    the reverse.
- **Router and continuation.**
  - Include `.taskGoto` in `pickerOperatorContinuationKeys` (bob's keys, empty filter
    only) and accept `.taskGoto` in `continuePickerOperator`. Clear
    `backslashLiteralArmed` when it fires.
  - Key repeat and IME marked text consume instead of escalating.
  - With no advertised key (old bob), a `/` on an empty Tasks filter is consumed and the
    status line shows "Update bob to search notes".
  - In Notes:
    - Backspace on an empty filter applies bob's `trigger_removal_range`;
    - `/` and `\` on an empty filter are consumed;
    - ⌘X, Shift-↩, and Option-↩ are consumed;
    - ⌃C discards without stashing (extend the `draftIsTaskGoto` stash guard);
    - Escape clears, then shows a `//` reopen chip.
- **Accept** (§6): Return, Tab, click, ⌘↩, and ⌃↩ all call `acceptNoteGotoRow(id:)`. It
  reuses the dispatch-lock and opener pattern of `acceptTaskGotoRow`, through the
  injected opener closure, so tests can assert the exact URL.
- **Footers.** Go To · Tasks shows a `/ Notes` key chip while its filter is empty and
  bob advertised the key. Go To · Notes shows
  `↑↓ Move · ↩ Open · ⌫ Tasks · esc Clear / Cancel`.
- **Tests** (`CaptureNoteGotoPanelTests`, fake-bob):
  - every §7 key row, including:
    - `\` `/` → Notes;
    - `\\` → literal `\` (regression);
    - `/` `\` consumed;
    - a third `/` consumed;
    - the peel `//` → `/` → blank;
  - a fast `//cash` draft;
  - the Escape chip and reopen;
  - accept dispatches exactly `obsidian://bob-open-notes?count=1&path1=…` with no
    `line1`, then hides and discards with no `capture` invocation recorded by fake-bob;
  - failure keeps the picker with "Couldn't open Obsidian";
  - a double accept dispatches once;
  - ⌃C does not stash;
  - the old-bob hint;
  - key repeat;
  - a stale-response race between `/` and `//`.

  Existing `CaptureTaskGotoPanelTests` and `CaptureBackslashGotoTests` stay green, apart
  from the caption rename.

- **Docs** (bob-mac-capture `docs/`, mounted into Bob's site; run `just docs-check`):
  - `task-pickers.md`: add a `## Go To Notes Picker` section after Go To Picker, and
    rename that picker's caption to Go To · Tasks with its new `/ Notes` hint;
  - `keyboard.md`: the §7 keys;
  - `protocol.md`: a capability row for `note_goto` (need, context, `note_goto` object,
    descriptors) and the compatibility table;
  - `pickers.md`: list Go To Notes (`//`).
- **Bases.** When decision `bases` is yes, confirm on the Mac that bob-open-notes opens
  a `.base` path before shipping `view` rows (see §5.1).
- Follow the commit and CI loop. Subject:
  `feat(capture): open notes from the // Go To Notes picker`.

> [!decision] bases = no Nothing to confirm; `view` rows never appear.

## Phase `mac-note-polish`: no-blink swap, warm snapshot, accessibility, closeout

Make the `/` ⇄ `//` swap feel like one card changing its mind, not two cards.

- **Mounted swap.** Neither the continuation (`/` → `//`) nor the peel (`//` → `/`)
  tears down the card.
  - The scope capsule animates its token and caption.
  - Rows cross-fade, about 120 ms with an ease-out curve.
  - The panel height never changes (the existing "filtering never resizes the panel"
    budget holds across scopes).
  - Reduce Motion swaps instantly.
  - Until rows exist, the card shows a quiet "Loading notes…" placeholder, never a
    spinner. Distinguish "No notes match" from "Couldn't load notes".
- **Warm snapshot.** Add an app-memory `NoteGotoSnapshotStore`. It is never persisted
  and never logged.
  - It holds the last good `note_goto` rows, `vaultRoot`, `listed`, and the bob identity
    that produced them.
  - It fills from a background `capture-complete -c 2 -f json -- '//'` run with the same
    environment as other completion calls. Triggers:
    - a `/` Go To session opens while the store is empty or stale;
    - the panel shows while a store exists (stale-while-revalidate);
    - a filtered vault event arrives (visible `.md` changes, reusing the idle agenda's
      watcher filter), debounced.
  - It is trusted only when the response context was `note_goto`.
  - On continuation, the store's rows render immediately. The real `//` analysis then
    replaces them only if they differ, keeping the selection on the same `note_path`.
  - The `/` session's task rows are retained for the peel back.
- **Accessibility.** Announce scope changes ("Go To Notes, 184 notes" / "Go To Tasks")
  and filter result counts, matching existing picker announcements.
- **Tests:**
  - store fill, trust, and staleness rules;
  - no-blink swap state transitions, with the card identity preserved;
  - the placeholder and error states;
  - selection retention across a refresh;
  - Reduce Motion;
  - no `bob` spawn per keystroke.
- **Docs.** Update `task-pickers.md` and `capture-panel.md` for the swap and snapshot
  behavior.
- **Closeout.** Add a bead note with Bryan's Mac acceptance checklist:
  - the fast `//` race (type `//` faster than 350 ms);
  - `\` `/` and `\\`;
  - an open note keeps caret, scroll, and Reading mode in a split, a background tab, and
    Reading view;
  - `git status` on the Mac vault is clean after opening several notes;
  - paths with spaces, Unicode, `#`, and `&`;
  - duplicate basenames stay distinguishable;
  - the old-bob hint.

> [!decision] memory_note_goto_record When yes, read
> `sase memory read decisions:mac-capture-is-a-thin-client` for the strand shape, use
> `/sase_memory_write`, and add one strand under the `decisions` web, for example slug
> `note-goto-lists-living-notes`. Its claim: "Whole-draft `//` is Go To Notes: bob lists
> living notes only (the §5.1 tiers), dailies are date-keyed, and navigation never
> writes." Its evidence cites the research report, this epic, and the landed commits.
> Then run `sase memory init`. When no, make no memory edit and record a
> `PROPOSED FOLLOW-UP:` note only if a human did not review the decision.

- Follow the commit and CI loop. Subject:
  `feat(capture): keep Go To mounted across / and // with a warm notes snapshot`.

## Rollout and compatibility

- **Release.** bob-cli ships on the daily train. `cli-note-goto`'s `!` subject marks the
  `//…` capture break.
- **Mismatches.** Every combination is safe (§7 compatibility table):
  - the app opens Notes only when bob reports `note_goto`;
  - old apps ignore the new descriptor;
  - new bob refuses `//` captures for every client, so no client can file a junk inbox
    task.
- **No plugin release.** bob-open-notes already provides the landing.

## Out of scope (follow-ups, not v1)

- An **attic scope** behind ⌘X: `done/`, satellites, and `ref_history/` (⌘X is consumed
  now so it can be claimed later).
- A **Recent** group from capture-local open history.
- `//note#heading` landings, reusing `[[##` heading data.
- **`///` full-text search** once a content index exists.
- A cursor-restoring active-leaf lander (`bob-goto-note`) if far-right tab accumulation
  annoys.
- Option-↩ on ref rows opening the PDF via Bob Refs.
- Deciding whether `[[` completion adopts the same hidden tiers.
- `.base` views when decision `bases` is no.
