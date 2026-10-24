---
tier: epic
title: Pomodoro themes — remembered, hideable, renameable, and colored everywhere
goal: 'Every Pomodoro-name menu (Bob Mac Capture, zsh, and the Obsidian pickers) lists
  today''s names first and then previously used themes, and accepting a row submits
  exactly the name it shows. Ctrl+D hides a stale theme from suggestions in place,
  with undo. One bob transaction renames a theme across every daily note in a single
  revertable commit. Each theme wears a stable, learnable hue — in Obsidian Live Preview
  and Reading view, the plan block, Stats, Mac Capture, the CLI, tmux, and the menu
  bar — from one rule owned by bob.

  '
decisions:
  area_hues:
    ask: Should a theme named like a top-level area wear that area's hue?
    choices:
      area: JOB wears job's aqua, DEV dev's blue, LOVE love's yellow; pins still win,
        GTD stays muted
      hash_only: Hue is only a pin or the name hash, so JOB stays magenta beside the
        aqua job area
    default: area
    why: The same word gets the same color everywhere, and the BOB/JOB collision disappears
      unpinned
    answer: area
  cli_noun:
    ask: Should the new command group be spelled with the plural noun `bob themes`?
    choices:
      themes: bob themes list|hide|unhide|color|rename, a plural catalog noun like
        projects and plugins
      theme: bob theme list|hide|unhide|color|rename, singular like task, pomodoro,
        and ref
    default: themes
    why: It names a catalog, like projects and plugins; bob-cli-4b can add the other
      spelling as an alias
    answer: themes
  seed_pins:
    ask: Should rollout apply each collision pin bob suggests at that moment, once?
    default: true
    why: Recurring pairs (SASE and BLOG are both orange) look identical from day one
      otherwise
    answer: true
  memory_glossary:
    ask: Should the glossary gain a Pomodoro Theme strand, with the mac pom strand
      saying NAME, not THEME?
    memory:
    - glossary
    default: true
    requested: We should add a new glossary term for this.
    answer: true
  memory_decisions:
    ask: Should four decision records capture the theme identity, hide, color, and
      journal rules?
    memory:
    - decisions
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: true
phases:
- id: lossless
  title: Lossless name reuse and the exact-name literal
  depends_on: []
  size: medium
  description: 'lossless: fix bob-cli-bb so reusing a completed name keeps its spelling,
    and add the `#"NAME"` exact-name literal to every capture name position.'
- id: catalog
  title: Theme catalog engine, `bob themes list`, and the contract
  depends_on: []
  size: medium
  description: 'catalog: build the bob-cli theme catalog (scan, cache, journal fold,
    hue rule), ship the read-only list command, docs/pomodoro-themes.md, and the shared
    vectors.'
- id: prefs
  title: Hide, unhide, and color writers plus the vault journal rollout
  depends_on:
  - catalog
  size: medium
  description: 'prefs: add the journal writers, collision hints, the vault allowlist
    and union-merge rollout, and the one-time pin seeding.'
- id: suggest
  title: Recent themes in capture-complete
  depends_on:
  - lossless
  - catalog
  size: medium
  description: 'suggest: compose today rows, recent themes, and the create row in
    both Pomodoro-name completion contexts, with lossless replacements and additive
    JSON.'
- id: theme-json
  title: Theme hues across bob JSON and terminal output
  depends_on:
  - lossless
  - catalog
  size: medium
  description: 'theme-json: add theme refs to capture-parse spans, the agenda, plan,
    stats, and capture output, plus CLI and tmux swatches.'
- id: rename
  title: Global theme rename transaction
  depends_on:
  - prefs
  size: medium
  description: 'rename: implement the previewed, verified, single-commit rename modeled
    on `bob ref rename`, with refusals and reappearance detection.'
- id: mac-menus
  title: Mac Capture recent rows, Ctrl+D hide, and Name it suggestions
  depends_on:
  - suggest
  - prefs
  size: medium
  description: 'mac-menus: render Today/Recent rows with swatches, hide and undo with
    Ctrl+D, and suggest themes in the Name it prompt.'
- id: mac-colors
  title: Mac Capture theme tints, agenda pills, and preview swatches
  depends_on:
  - mac-menus
  - theme-json
  size: medium
  description: 'mac-colors: tint known names in the editor, wear pills in the agenda
    and previews, and refresh on theme journal changes.'
- id: obsidian-colors
  title: Obsidian api.themes and theme pills
  depends_on:
  - catalog
  size: medium
  description: 'obsidian-colors: add the ledger-tools themes API with JS mirrors,
    shared hue tokens, and pills in Live Preview, Reading view, the plan block, and
    Stats.'
- id: obsidian-pickers
  title: Recent themes and Ctrl+D in the Obsidian pickers
  depends_on:
  - obsidian-colors
  - suggest
  - prefs
  size: medium
  description: 'obsidian-pickers: compose recent rows, swatches, near-duplicate badges,
    and Ctrl+D hide in the link picker and nav Pomodoro pickers.'
- id: obsidian-manager
  title: Obsidian Pomodoro themes manager
  depends_on:
  - obsidian-pickers
  - rename
  size: medium
  description: 'obsidian-manager: add the Bob Pomodoro themes command to browse, hide,
    pin, inspect, and rename themes with a previewed rename.'
- id: menubar
  title: Menu-bar theme swatch
  depends_on:
  - catalog
  size: small
  description: 'menubar: show the running session''s theme swatch before the mac pom
    title in the chezmoi Hammerspoon indicator.'
- id: records
  title: Decision records, glossary, and end-to-end acceptance
  depends_on:
  - mac-colors
  - obsidian-manager
  - menubar
  size: medium
  description: 'records: write the glossary strand and decision records from what
    shipped, reconcile docs, and run the cross-surface acceptance.'
proposed_by: bbugyi200.apollo.research.2c.linker.w1
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-24 07:52:20
status: wip
bead_id: bob-cli-bc
---

- **PROMPT:** [prompts/202610/pomodoro_themes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/pomodoro_themes.md)
- **BEAD:** [bob-cli-bc](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-bc/README.md)

# Plan: Pomodoro themes

## Outcome

Bryan types `=bo` in Mac Capture at 8 a.m. on an empty day and sees:

```text
 =bo▏
 ┌──────────────────────────────────────────────────────┐
 │ RECENT                                               │
 │ ▪ BOB           yesterday · 26 days      +1 · 1/3    │
 │ ▪ BOOKS         Oct 21 · 3 days          +1 · 1/3    │
 │ + New Pomodoro BO                             New    │
 │                          ↩ use    ⌃D hide theme      │
 └──────────────────────────────────────────────────────┘
```

Return reuses BOB with its exact spelling. Ctrl+D on BOOKS turns the row into a ghost
(`BOOKS hidden · ⌃D undo`) and it is gone on the next keystroke. Later, the Obsidian
**Bob: Pomodoro themes** manager renames FIXES to FIX across two months of daily notes
after a preview, in one Git commit. Each theme wears the same soft tint in the daily
note, the plan block, Stats, the Mac panel, `bob plan`, tmux, and the menu bar.

## Context

- **Research (accepted in full by Bryan):**
  `research:202610/pomodoro_theme_catalog/pomodoro_theme_catalog.md`. Read it with
  `sase artifact read`. Its adjustments ADJ-1 to ADJ-10 are requirements of this epic.
  This plan settles the open questions that research left and adds the same-named area
  hue rule (`area_hues`).
- **Vault facts (read-only scan, 2026-10-24):**
  - 81 themes since 2026-08-26, 40 of them used on only one day.
  - Drift pairs: FIX/FIXES, TOOL/TOOLS, SASE V18/SASE-V18.
  - October entry counts by name: GTD 71, BOB 49, JOB 48, BLOG 42, SASE 21, SASE V18 12,
    READ 11.
  - Top-level area colors: job aqua, dev blue, love yellow, gtd orange, body green, cash
    magenta.
  - The name hash alone gives BOB, JOB, and DEV magenta, and SASE and BLOG orange.
- **Today's behavior:**
  - Completion is today-only: `src/native/capture_complete/pomodoros.rs`.
  - Selection is exact-then-prefix: `select_named` / `select_by_slug` in
    `src/native/capture_pomodoros.rs`.
  - Components and keys come from `plan_budget::split_components` and
    `normalize_component`.
  - Hues come from `pomodoro_homes/palette.rs` (`Hue`, `fallback_hue`, `fnv1a32`).
- **Rename precedent:** `bob ref rename` (`src/native/highlights_ref/rename/`).
- **Journal precedent:** the per-host, union-merged `.bob/stats/review/*.jsonl`
  (`docs/stats.md`, `docs/vault-git-sync.md`).
- **Related beads:**
  - bob-cli-bb: lossy reuse of a completed name. The `lossless` phase fixes it.
  - bob-cli-4b: noun policy, which bears on `cli_noun`.
  - bob-cli-a9: Mac short start tokens. Adjacent to this work, but not part of it.

## Design

`docs/pomodoro-themes.md` (written by `catalog`) is the normative contract. Its rule IDs
PT1–PT10 mirror the sections below. Later phases extend it rather than restating rules
elsewhere.

### PT1 Vocabulary and identity

- **Theme:** one `+`-separated component of a Pomodoro entry's ` — NAME` tail (ADJ-1).
  - Split with `split_components`.
  - The key is `normalize_component`: whitespace collapsed, lowercased.
  - The display name is the most recent authored spelling for that key.
- **Pomodoro name:** the whole tail. `— BOB + DECKS` is one name with two themes.
- **Not themes:** an unnamed entry, the budget's `(unnamed)` placeholder, a Pomodoro
  home, or an Obsidian appearance theme.
- **Naming in code:** use `pomodoro_themes` (Rust module), `bob-pomo-theme-*` (CSS),
  `PomodoroTheme*` (Swift), and `api.themes` (JS) so nothing collides with Obsidian
  themes.
- **User-facing words:** **theme**, **Recent**, **Hide**/**Unhide**, **Pin color**, and
  **Rename theme**. Never "disable" or "retire": hiding changes suggestions only
  (ADJ-2).

### PT2 Catalog

- **Source:** every canonical daily note, `YYYY/YYYYMMDD.md` and nothing else.
  - Not `_templates/`, `done/`, `_conflicts/`, or `*_done.md` / `*_poms.md` siblings.
  - Read the column-0 entries in `## Pomodoros`, open and closed, through
    `capture_pomodoros::scan`. Fenced lines are skipped.
- **Per-theme fields:** `key`, `name`, `first_used`, `last_used`, `days_used`, and
  `entries`.
  - Only days up to and including today count. Future-dated notes feed rename, not
    recency.
  - `today` is `open`, `closed`, or absent, for the theme's presence in today's ledger.
- **History is derived only from ledger entries.** Never infer it from task text or
  homes.

### PT3 Cache

- **Location:** `bob_cli_cache_dir()/themes/catalog-v1.json`, keyed by vault root. It is
  machine-local, never synced, and disposable.
- **Contents:** a per-file `(mtime_ns, size)` fingerprint and that file's theme summary.
- **Today's note is always read live.**
- **Stat sweep:** a full sweep of past days runs at most once per 60 s, or when a year
  directory's mtime changes. A past-day hand edit therefore shows up within a minute.
- **Rebuild** on a corrupt or missing cache, after a rename, and with `--fresh`. The
  rename preview always rescans from source.
- **Writes** are atomic (temp file + rename).
- **Budget:** warm `capture-complete` and `capture-parse` stay within +5 ms p95 of
  today's baseline on apollo against the live vault, measured read-only with
  `hyperfine`, 50 runs.

### PT4 Preferences journal

- **Location:** `.bob/themes/<host>.jsonl`, one append-only file per host, inside the
  vault so it syncs (ADJ-2, rejected alternative: `config.yml` is chezmoi-managed).
- **`host`:** the same host label vault-sync uses in its commit subjects.
- **Events (`v: 1`):**

  ```json
  {"v":1,"ts":"2026-10-24T09:41:07-04:00","host":"mac","kind":"theme.hide","key":"fixes","name":"FIXES","via":"bob-mac-capture"}
  {"v":1,"ts":"2026-10-24T09:43:00-04:00","host":"apollo","kind":"theme.color","key":"blog","name":"BLOG","hue":"red","via":"bob-cli"}
  {"v":1,"ts":"2026-10-24T10:02:11-04:00","host":"apollo","kind":"theme.rename","from":"fixes","to":"fix","from_name":"FIXES","to_name":"FIX","via":"bob-cli"}
  ```

  - The kinds are `theme.hide`, `theme.unhide`, `theme.color` (`hue` is a slot name, or
    `null` to unpin), and `theme.rename`.
  - `via` names the client that asked: `bob-cli`, `bob-mac-capture`, or a plugin id.

- **Fold:**
  - Read every `*.jsonl` file in the directory.
  - Skip malformed lines with a bounded warning.
  - Sort by `(ts instant, host, line number)`.
  - Last writer wins per key for the hidden state and for the pin.
  - A rename moves the source key's explicit state to the target, unless the target
    already has explicit state of its own.
  - Events after the rename on the old key apply to the old key.
- **Writers:** bob is the only writer. Clients ask bob; the JS mirror only reads.

### PT5 Hue rule

> [!decision] area_hues = area In order:
>
> 1. A journal pin wins.
> 2. Else, a key in normalized `plan.exempt` is muted: `hue: null`,
>    `hue_source: "exempt"` (ADJ-4: GTD).
> 3. Else, a key equal to a **top-level area note's stem** (an area whose `parent` is
>    `[[area]]`) wears that area's resolved hue (`hue_source: "area"`). This is the same
>    resolution the home chips use: a frontmatter `color:` slot, else the stem hash.
> 4. Else, `Hue::ALL[fnv1a32(key) % 8]` (`hue_source: "hash"`).
>
> Results: JOB → aqua, DEV → blue, LOVE → yellow, GTD → muted. Reuse the existing area
> resolver (`pomodoro_homes`) in Rust and the home-chip area hue code in JS.

> [!decision] area_hues = hash_only In order:
>
> 1. A pin.
> 2. Else, exempt themes are muted.
> 3. Else, `fnv1a32(key) % 8` (`hue_source` is `pin`, `exempt`, or `hash`).
>
> No area lookup happens, and the vectors carry no `areas` input.

- **Palette:** the eight existing slots (blue, orange, aqua, yellow, magenta, green,
  violet, red), with their light/dark hex values and xterm-256 codes. No new palette.
- **Stable, not unique (ADJ-3):** pins fix recurring collisions.
- **Single source:** Rust computes the hue. JS mirrors it under shared vectors. Swift
  and Lua only consume the `hue` bob reports.

### PT6 Visual language

Names wear a tint; places wear a dot.

- **Pill** (Obsidian, Mac): a soft tint of the hue behind normal-colored text.
  - The tint is 16 % in light mode and 24 % in dark mode, tuned once by eye and
    documented.
  - Corner radius 4 px, horizontal padding about 0.3 em, no border, and
    `box-decoration-break: clone`.
  - On hover, the tint rises to about 24 % / 32 %.
  - The hue is **never** used as the text color.
- **Muted pill:** used for exempt themes such as GTD. Hover background, muted text.
- **Swatch:** in dense rows, an 8 pt rounded square (radius 2) before the name. Area
  homes keep their ● circle, so a theme never looks like a home.
- **Composites:** each component gets its own pill, and the `+` stays neutral.
- **CLI:** an xterm-256 `■` before a default-color name, only when color is enabled (TTY
  and no `NO_COLOR`). An exempt theme gets `□`.
- **Typing:** a name the catalog does not know stays neutral while you type it, so it
  does not flicker through hues. It takes its hue once it is committed or the cursor
  leaves the line.
- **Color is never the only signal.** The name text, the Today/Recent labels, and the
  `hidden`, `pinned`, and `exempt` badges carry the meaning under `NO_COLOR`, Increase
  Contrast, and VoiceOver. Pills get a tooltip or accessibility label such as
  `BOB · magenta · 26 days`.

### PT7 Menu composition

This rule applies to every Pomodoro-name menu: `capture-complete` contexts
`pomodoro_name` (`@route:id#…` and its siblings) and `pomodoro_start_name` (`=#…`,
`=<X>#…`, bare `=name`), and the Obsidian pickers. It is filter first, then group:

1. **A: Today rows that match the query.** These are today's existing rows, with today's
   ranking and order unchanged. In the start context that means planned placeholders
   (next-up first) and "again" rows; in the link context, open named entries.
2. **B: Recent exact and prefix matches.** History themes whose slug equals or starts
   with the query slug. Sort by exact first, then `last_used` descending, then
   `days_used` descending, then name.
3. **C: The create row.** It appears only when the query is creatable and no A or B row
   is an exact match. An exact B match _is_ the create row: it carries
   `creates_pomodoro`, `plan_themes_after`, and `plan_themes_cap`. This moves the create
   row below the B rows (ADJ-9).
4. **D: Recent substring matches**, in B's order.
5. **E: Today's query-independent utility rows.** These are the unnamed "Name it" rows
   and, in the start context, the running row. They stay after the create row, as now,
   so `=#zzz` followed by Return still creates.

**Filters and caps:**

- Recent rows exclude hidden themes.
- Recent rows exclude any theme that is a component of a named Today row in the same
  menu. That check runs before the query filter. A theme completed today is therefore
  Recent ("today") in the link context, but an "again" row in the start context.
- An empty query shows at most 5 Recent rows. A non-empty query shows at most 12. The
  full list lives in `bob themes list` and the manager.
- Matching is ASCII case-insensitive on slugs. A partial quoted literal (`#"SASE V`)
  matches on the normalized key prefix.

**Near-duplicate heads-up (ADJ-8):**

- A create row gets `similar_to`: at most 2 visible known names, most recent first. A
  name qualifies when its key:
  - equals the query modulo space, hyphen, and punctuation (SASE V18 / SASE-V18);
  - is its plural or singular (`s`, `es`); or
  - is within Levenshtein distance 1 when both keys have at least 5 characters.
- It needs a query key of at least 3 characters.
- It never reorders rows or changes Return. Clients render it as a quiet `like FIX`
  badge.

**No silent writes.** Fetching, filtering, and accepting suggestions never touch the
journal. A history row hits the same missing-section, multiple-running, strict-budget,
and placement refusals that typing the name would.

### PT8 Lossless acceptance (ADJ-7)

- **Plain slug only when it round-trips.** A history row, and an "again" row too, emits
  its plain slug (with the existing bare/`#` spelling rule) only when both hold:
  - `canonicalize_pomodoro_name(slug)` equals the displayed name.
  - `select_named(today, slug)` is `Missing`, or resolves to an entry with the same key.
- **Otherwise it emits the exact-name literal:** `=#"DEEP WORK"` or
  `@route:id#"SASE V18"`.
- **The literal follows the quote precedent of `&"Shopping List":bar`:**
  - Straight `"…"` and curly `“…”` quotes are both accepted; bob always emits straight
    quotes.
  - `\"` and `\\` are the only escapes. Valid names cannot contain either character, but
    the decoder still honors the escapes.
  - The token extent runs through the closing quote. Normal terminators apply after it.
  - A literal without its closing quote is an incomplete editing state, never a mistake.
- **Semantics:**
  - Normalize the name (collapse whitespace, uppercase, validate with
    `is_pomodoro_name`) and compare by **key equality**, never by slug or prefix. SASE
    V18 and SASE-V18 stay distinct.
  - An open entry with an equal key is `Found`.
  - Else, a completed entry with an equal key is `CompletedOnly`, which creates a new
    entry with that entry's spelling.
  - Else, create with the literal's canonical name.
- **Where:** every position that takes a Pomodoro name after `#`, including select-only
  and close positions, so one rule holds everywhere. The bare `=name` spelling never
  takes quotes.

### PT9 Hide

- **Effect:** hiding removes a theme from **Recent** rows only. Unaffected:
  - today's rows;
  - typing the name, which still creates;
  - history, Stats, and colors.
- **Using a hidden name never unhides it** (no state change behind your back).
- **Ctrl+D**, while a _Pomodoro-name_ menu is visible in Mac Capture or any Obsidian
  Pomodoro picker:
  - **Recent row:** hide. The row stays in place as a dimmed ghost,
    `BOOKS hidden · ⌃D undo`, until the next query. Ctrl+D on the ghost unhides.
  - **Create row whose exact name is a hidden theme** (`hidden: true`, shown as
    `FIXES · hidden`): unhide.
  - **Today row or any other create row:** consumed. The status hint reads "BOB is on
    today's ledger — nothing to hide" or "Nothing to hide".
- **Ctrl+D consistency:** the key is always consumed in those menus, so it never means
  hide on one row and delete-forward on the next. In every other menu, and with no menu,
  Ctrl+D keeps its current meaning.
- **Return on a ghost row** accepts it like any row, because hidden names stay usable.
- **Writes are optimistic.** The client marks the row first, then runs bob in a
  dedicated, serial, non-cancelling lane. On failure it restores the row and reports the
  error.
- **Other ways in:** the CLI and the manager.

### PT10 Global rename (ADJ-6: never from a completion menu)

```text
bob themes rename OLD NEW [-d/--dry-run] [-f/--format human|json] [-m/--merge] [-o/--offline]
```

- **Scope:** the theme component in ` — NAME` tails of Pomodoro entries:
  - every canonical daily note — past, today, and future, open and closed;
  - plus `_templates/daily.md`.
- **Byte preservation:** separators keep their `+` spelling. A resulting duplicate
  collapses (`FIX + FIXES` → `FIX`). Every other byte is preserved. Prose, task text,
  and fenced examples are never touched.
- **NEW** must be a valid single theme. A NEW containing `+` is `invalid_name`.
- **Flow (copy `bob ref rename`):**
  1. Take `bob_sync.lock`, then pre-sync unless `--offline`, then replan from the synced
     files.
  2. Snapshot each target and compare it again right before writing. A change since
     planning aborts with `stale_note`.
  3. Write each file atomically, then append `theme.rename`.
  4. Verify by rescanning: OLD is absent, entry counts are unchanged, and only name
     tails differ. On failure, restore the before-images.
  5. Make one commit, `bob themes rename: FIXES → FIX`, then post-sync. Undo is
     `git revert`, documented as a runbook.
- **Refusal codes:**
  - `unknown_theme`, `invalid_name`, `unchanged`.
  - `theme_exists`: NEW is already used and `--merge` is absent.
  - `config_reference`: OLD appears in `plan.exempt`, `plan.inventory_labels`, or a
    Stats metric filter, and NEW does not. Edit the chezmoi config to add NEW first.
  - The `ref rename` set: `lock_busy`, `dirty_paths`, `not_a_worktree`, `sync_failed`,
    `verify_failed`, plus `stale_note`.
- **Color and state:** the color follows the new name, and the preview says so. Hidden
  state and pins move with the rename per PT4.
- **Reappearance:** if OLD's spelling reappears after a rename event (an offline host,
  muscle memory), `bob themes list` reports it in `renamed_reappeared`. It only reports;
  it never rewrites on its own.

### JSON contract (additive, `schema_version` stays 1)

- **`ThemeRef`:** `{key, name, hue, hue_source}`, plus `known: false` only for names the
  catalog does not know. Where offsets matter, it also carries `start` / `end` byte
  offsets into the draft.
- **`PomodoroNameCandidate` additions:**
  - `themes: [ThemeRef]` on every named row.
  - `origin: "history"` on Recent rows. Absent means today.
  - `last_used` and `days_used` on Recent rows.
  - `hidden: true` on a create row whose exact name is hidden.
  - `similar_to: ["FIX"]` on a create row.
- **Recent rows:** `creates_pomodoro: true`, the plan-theme preview fields, and no `ref`
  or `line`. An old Mac build shows them as "New X" rows, which is still the correct
  action.
- **`bob themes list -f json`:**

  ```json
  {
    "schema_version": 1,
    "today": "2026-10-24",
    "window_days": 90,
    "counts": { "known": 81, "listed": 40, "today": 4, "hidden": 2 },
    "themes": [
      {
        "key": "bob",
        "name": "BOB",
        "hue": "magenta",
        "hue_source": "hash",
        "pinned": false,
        "hidden": false,
        "exempt": false,
        "first_used": "2026-08-26",
        "last_used": "2026-10-24",
        "days_used": 26,
        "entries": 49,
        "today": "open"
      }
    ],
    "hints": [
      {
        "code": "theme_hue_collision",
        "themes": ["SASE", "BLOG"],
        "hue": "orange",
        "days_together": 6,
        "suggest": { "name": "BLOG", "hue": "red" }
      }
    ],
    "renamed_reappeared": [],
    "warnings": []
  }
  ```

- **Writers:** `hide`, `unhide`, `color`, and `rename` emit `{schema_version, ok, ...}`.
  On refusal they emit `{ok:false, code, message}`, even on a nonzero exit, following
  the `capture-templates delete` pattern the Mac already decodes.

### `bob themes` CLI

> [!decision] cli_noun = themes The group is `bob themes`, spelled that way everywhere
> in this plan.

> [!decision] cli_noun = theme The group is `bob theme`: substitute it for `bob themes`
> everywhere in this plan, with no alias in this epic.

- **Placement:** a new top-level catalog noun in **Daily workflow**, after `stats`.
  - Not under `bob task`, which is reserved for task-line rewrites.
  - Not under `bob pomodoro`, whose bare default takes flags.
  - Never a `capture-*` sibling.
- **Members,** alphabetical in help:
  - `color NAME HUE|auto`
  - `hide NAME…`
  - `list [-a/--all] [-F/--fresh] [-f/--format human|json] [-H/--hidden] [-n/--limit N] [-q/--query Q]`
  - `rename OLD NEW …`
  - `unhide NAME…`
- **Bare `bob themes` is `list`.** This is allowed because it is a flag-only, read-only
  default.
- **List defaults:** visible themes used in the last 90 days or today, newest first.
  - `-a` shows every theme and flags hidden ones.
  - `-H` shows only hidden themes.
  - `-q` ranks per PT7's B/D groups.
- **Help and completion:** every option has a short alias. `--help` is excellent and has
  examples. `bob <TAB>` completion places `themes` in the Daily workflow group. NAME
  arguments complete theme names where the completion system supports dynamic values.
- **Human list:**

  ```text
  THEMES  81 known · 4 today · 2 hidden · last 90 days
    ■ BOB        today     26 days   magenta
    ■ JOB        Oct 23    24 days   aqua · area
    ■ BLOG       Oct 23    14 days   red · pinned
    □ GTD        today     51 days   muted · exempt
    hint  SASE and BLOG share orange and met on 6 days → bob themes color BLOG red
  ```

### Clients

- **Bob Mac Capture stays a thin client** (`decisions:mac-capture-is-a-thin-client`).
  - It renders and filters what bob returns, and never computes a hue, ranks history, or
    scans the vault.
  - It decodes with `decodeIfPresent`. With an older bob it shows no Recent rows and no
    swatches, and Ctrl+D reports "Update bob to hide themes".
- **Obsidian uses one `api.themes` namespace in bob-ledger-tools.** Rules for it:
  - Every picker uses it. No plugin keeps its own hidden list or history scanner.
  - Today's rows still come from the live editor buffer.
  - History and writes need the desktop app. On mobile, pickers stay today-only, but
    colors (hash, area, and journal pins) still render.

## Cross-cutting rules for every phase

- **Contracts:**
  - New behavior lands in bob-cli and its docs first. JSON grows additively at
    `schema_version` 1.
  - Any phase that changes a rule updates `docs/pomodoro-themes.md` and the matching
    reference page in the same phase.
- **Shared vectors:**
  - `tests/fixtures/pomodoro_themes/vectors.json` in bob-cli is the single source.
    Sections: `components`, `hue`, `fold`, `rank`, `menu`, `similar`, `literal`.
  - bob-plugins copies it byte-for-byte to
    `plugins/bob-ledger-tools/scripts/fixtures/pomodoro_themes/` with a README that
    records the bob-cli base commit, like the `pomodoro_homes` fixture.
  - A JS case marked `rust_only` is skipped there.
- **CLI and docs rules:** read `cli_rules.md` and `docs.md` with `/sase_memory_read`
  before touching commands, help, or docs.
  - Run `just docs-gen` for new flags, then `just docs-check`.
  - Root behavior contracts live in `docs/*.md`. The user glossary is
    `docs/reference/glossary.md`.
- **Release train:**
  - Never edit `Cargo.toml` versions or `CHANGELOG.md`.
  - Use conventional commits (`feat(themes): …`, `feat(capture): …`). Mark only a true
    CLI or JSON contract break with `!`.
  - bob-plugins and bob-mac-capture follow their own versioning conventions (plugin
    manifests are semver).
- **Cross-repo work:** open bob-plugins, bob-mac-capture, chezmoi, and the research
  sidecar only through `/sase_repo`. Read each repo's `AGENTS.md`.
  - Plugin phases deploy with `bob plugins sync` once their tests pass. Deployment needs
    an installed bob that already has `bob themes`, so install bob-cli master first if
    needed.
- **Mac builds:**
  - `just portable-check` covers CaptureCore on Linux. Use athena over Tailscale SSH if
    the local host lacks Swift (`tailnet.md`).
  - The `BobMacCapture` target builds only on macOS CI. Confirm the CI run is green
    before you close the phase.
- **Vault safety:**
  - Tests use scratch vaults with `BOB_DIR`, `BOB_DAY_FILE`, `BOB_NOW`, and
    `BOB_CONFIG_FILE`.
  - Against `~/bob`, only read and measure, except the documented rollout steps in
    `prefs`. Never write into `~/bob/.obsidian/plugins` by hand.
- **Baselines:** compare test failures against a clean-base run. bob-plugins has had
  pre-existing failures (bob-cli-6v, bob-cli-6w, bob-cli-7i); check their status first.
  Record new discoveries as `PROPOSED FOLLOW-UP:` notes on your phase bead. Phase
  workers create no beads.

## Phases

### lossless

**Repo:** bob-cli. Fixes bob-cli-bb and adds PT8.

1. **bob-cli-bb.**
   - If bob-cli-bb is already closed, verify the fix and skip to step 2.
   - Otherwise, when the selection is `CompletedOnly(entry)`, every creating path names
     the new entry with the completed entry's canonical name:
     - `capture_task_toggle/{ledger,relocation}.rs` (`insert_named_placeholder`);
     - `capture/pomodoro_insert.rs` (`insert_named_pomodoro_child_block`);
     - the `capture/pomodoro_link.rs` CompletedOnly arm;
     - `capture/pomodoro_start.rs`, which is already correct (keep it under test).
   - Add a regression test per path with `DEEP WORK`.
   - Close the bead: `sase bead close bob-cli-bb --note "<verification>"`.
2. **Exact-name literal (PT8).**
   - Inventory every `#name` position in `capture_language`: `bare_name.rs`
     (`start_name_extent`), `tokens.rs`, `editor_pomodoro.rs`, `start_selection.rs`,
     `close_selection.rs`, link suffixes, and `^route:id#`.
   - Make each one lex `#"…"` and `#“…”`, plus the unterminated editing state.
   - Add `NamedSelection` resolution by key equality, alongside `select_named` /
     `select_named_exact`. Reuse the internal `PomodoroLinkTarget::ExactName` path where
     it fits, but compare keys, not slugs.
   - Wire the new resolution into start, link, toggle, insert, and close execution.
3. **Spans and completion.**
   - `capture-parse` emits `pomodoro_name` spans covering the whole literal, including
     its quotes.
   - `capture-complete` treats a partial literal as the Pomodoro-name context: the query
     is the literal's text, and the replacement range covers the whole partial literal.
   - `capture-rewrite` must keep literals intact.
4. **Docs.**
   - `docs/capture.md`: grammar at a glance, the named-start and link sections, the
     capture-parse table, and conformance examples.
   - Add round-trip fixtures for spaces, punctuation (`SASE.MD`, `FIX-APOLLO`), digits,
     `+` composites, curly quotes, and an unterminated literal.
   - Add a zsh shell-quoting check for the completion replacement.

**Acceptance:**

- In a scratch vault, `bob capture -d '=#"DEEP WORK"'` and
  `bob capture -d '@inbox_t:foo1#"DEEP WORK"'` both keep `DEEP WORK`.
- With `BOB BUILD` open, `=#"BOB"` creates BOB instead of selecting BOB BUILD.
- `just check` (or the repo's full gate) is green.

### catalog

**Repo:** bob-cli. Covers PT1–PT5, the read side of the JSON contract, and the CLI's
`list`.

1. **The `src/native/pomodoro_themes/` module,** with these parts:
   - `scan`: PT2.
   - `cache`: PT3.
   - `journal`: PT4, a reader and the fold.
   - `hue`: PT5.
   - `rank`: PT7 groups B and D, plus the filters.
   - `ThemeRef` serialization.
   - A `Catalog` API that later phases call. Treat its shape as a contract: fetch the
     catalog for a vault and "today", look up a key, compute ThemeRefs for a name, and
     rank history for a query and exclusions.
2. **`bob themes` group and `bob themes list`,** with human and JSON output per the
   design.
   - Register the group in `src/runner.rs` (SUBCOMMANDS, Daily workflow) and in the
     completion kinds.
   - Emit the `hints` array empty; `prefs` fills it.
3. **`docs/pomodoro-themes.md`:** a new root contract with rules PT1–PT10, the JSON
   shapes, the rendering rules (PT6), and the degraded modes.
   - Rules not built yet are marked `planned (phase <id>)` until their phase lands.
   - Link it from `docs/plan.md`'s Themes section and from `docs/stats.md`'s theme
     dimension, and add it to the mkdocs nav.
   - Add a **Pomodoro theme** entry to `docs/reference/glossary.md`.
4. **Vectors:** `tests/fixtures/pomodoro_themes/vectors.json` with the `components`,
   `hue` (including the `areas` input under `area_hues = area`), `fold`, and `rank`
   sections. Rust tests consume every section.
5. **Measure** the PT3 budget on apollo, using a throwaway invocation over the live
   vault, and record the numbers in the phase note.

**Acceptance:**

- `bob themes` on the live vault lists BOB, JOB, BLOG, and GTD with the expected hues.
- `bob themes -q fix -f json` ranks FIX before FIXES.
- `--fresh` and cached runs agree.
- The full gate is green.

### prefs

**Repo:** bob-cli, plus the vault rollout. Covers the PT4 writers and PT9 hide via the
CLI.

1. **`hide`, `unhide`, and `color`.**
   - Append under `bob_sync.lock` with a bounded wait of at most 3 s. A busy lock is
     `lock_busy`.
   - Write one `O_APPEND` write per line and create the directory on first use.
   - Refusals: `unknown_theme`, `invalid_hue`, and an `unchanged` no-op that writes
     nothing.
   - Output: JSON and human, with `via` set from a hidden internal `--via` argument that
     clients pass.
2. **Collision hints** (`theme_hue_collision`).
   - **Which pairs:** unpinned, non-exempt pairs that share a hue and met on at least 2
     of the last 30 days.
   - **Which theme to move:** the one with fewer `days_used`.
   - **Suggested hue:** the slot least used by the last 30 days' themes, excluding hues
     of themes that met the moved theme in that window. Ties go to slot order.
   - **Where:** in `bob themes list` (human and JSON), then add the `similar` vectors.
3. **Vault rollout.**
   - Add `!/.bob/themes/*.jsonl` to `~/bob/.gitignore` and
     `.bob/themes/*.jsonl merge=union` to `.gitattributes`.
   - Check both with `git check-ignore -v` and `git check-attr merge`.
   - Commit in the vault (`chore(vault): allow Pomodoro theme journal`) and run
     `bob vault-sync`.
   - Document it in `docs/vault-git-sync.md` next to the Stats journal, including that a
     host which has not synced the allowlist keeps its hides local.

> [!decision] seed_pins 4. **Seeding.** After the rollout, run `bob themes list -f json`
> on the live vault and apply each `theme_hue_collision` suggestion exactly once with
> `bob themes color`. Record the pins written, and how to undo each
> (`bob themes color NAME auto`), in the phase note.

> [!decision] seed_pins = no 4. **No seeding.** Write no pins. The hints stay visible in
> `bob themes list` and in the manager.

**Acceptance:**

- A scratch-vault hide/unhide/color round trip folds correctly across two host files.
- Concurrent appends from two processes both survive.
- On the live vault, `git check-attr` reports `merge: union`.

### suggest

**Repo:** bob-cli. Covers PT7, the PT8 replacements, and the completion JSON additions.

1. **Compose rows in `capture_complete/pomodoros.rs`** for both contexts, through the
   `Catalog` API:
   - Today rows (A and E) come from the existing builders.
   - Recent rows (B and D) are new.
   - Move the create row per PT7 by replacing `insert_pomodoro_creation_candidate`'s
     prefix-based insertion.
   - Dedupe, filter hidden themes, apply the caps, and fold an exact B match into the
     create row.
2. **Replacements per PT8** for Recent and "again" rows. The prefix-shadowed "again" row
   is the same V2 bug, so fix it here too.
3. **Additive candidate fields** (`themes`, `origin`, `last_used`, `days_used`,
   `hidden`, `similar_to`) in `capture_complete/model.rs`. Render them in zsh
   (`capture_complete/shell.rs`) with a "recent · Oct 21 · 3 days" description.
4. **The `menu` and `literal` vector sections,** covering:
   - prefix-shadowing (SASE vs SASE V18 open);
   - a hidden exact match;
   - a theme completed today in each context;
   - the empty-query cap;
   - `=#zzz` with name-it rows.
5. **Docs:** update the completion sections of `docs/capture.md` and PT7/PT8 in
   `docs/pomodoro-themes.md`.

**Acceptance:**

- `capture-complete` for `=#bo` on a scratch vault with history returns BOB (B), the
  create row, then substring rows, all with the expected replacements.
- Today rows keep their prior order.
- The PT3 budget holds, with numbers recorded.
- The full gate is green.

### theme-json

**Repo:** bob-cli. Covers PT6 for the terminal, and theme refs wherever bob already
reports a Pomodoro or theme name.

1. **`capture-parse`:** each `pomodoro_name` and `task_toggle_pomodoro_name` span gains
   `themes` with component offsets. Quotes and `+` are excluded. Names the catalog does
   not know carry `known: false`.
2. **`capture-pomodoros`,** plain and `--tasks` (the agenda): every entry gains
   `themes`.
3. **Sibling ThemeRef arrays** wherever JSON already lists theme names, named
   consistently and documented in `docs/pomodoro-themes.md`:
   - plan budgets (`theme_names`, `added_themes`) in `bob plan`, `bob capture`, and the
     task-reconcile JSON;
   - the Stats theme dimension (a theme hue map beside `homes`).
4. **Terminal swatches:** in the `bob plan` theme line (`★ ■BOB · ■DECKS`), in
   `bob capture` human destination lines, and in the Stats human theme dimension.
5. **`bob pomodoro tmux`:** a `#[fg=colourN]■#[default]` swatch before each non-exempt
   component.
   - `bob pomodoro status` text stays byte-identical, because the mac pom parses it.
6. **Docs:** update `docs/capture.md`, `docs/plan.md`, `docs/stats.md`, and the tmux
   help.

**Acceptance:**

- JSON snapshots cover each new field.
- The `NO_COLOR` output is unchanged.
- `bob pomodoro status` output is byte-identical.
- The PT3 budget holds for `capture-parse`.

### rename

**Repo:** bob-cli. Covers PT10.

1. **`bob themes rename`.** Mirror `src/native/highlights_ref/rename/` (plan, verify,
   report, and the commit and sync flow) and reuse its lock, sync, and dirty-path
   helpers. Produce:
   - the human dry run: one line per file with the before/after tails, plus the state
     and color lines;
   - JSON for the Obsidian manager.
2. **Journal and cache:** append the `theme.rename` event, invalidate the cache, and add
   `renamed_reappeared` detection to `list`.
3. **Docs:** the PT10 contract, the refusal table, and a `git revert` runbook in
   `docs/pomodoro-themes.md`. Mention the command in `docs/vault-git-sync.md`'s
   operations.

**Acceptance:**

- Scratch-vault tests cover:
  - a single theme, a composite collapse, and a template rewrite;
  - fenced text and prose left untouched;
  - each refusal code;
  - stale-note abort and restore;
  - one commit;
  - `git revert` restoring every byte.
- No live-vault rename is run.

### mac-menus

**Repo:** bob-mac-capture.

1. **Decode** the new candidate fields in `Sources/CaptureCore/CaptureModels.swift`.
2. **Present** in `CompletionRowContent.swift` and `CapturePanelView.swift`
   (`CompletionList`, `CompletionRow`):
   - a quiet `RECENT` label above the first Recent row, and a `TODAY` label above the
     first row only when the menu also holds Recent rows;
   - the 8 pt theme swatch, using the existing light/dark hex of `PomodoroHomeHue`;
   - the `Oct 21 · 3 days` meta;
   - a `like FIX` badge and a `hidden` badge;
   - a contextual hint line, `↩ use · ⌃D hide theme`.
3. **Ctrl+D (PT9).**
   - Route it in `CaptureKeyCommandRouter.swift`, only for the `pomodoro_name` and
     `pomodoro_start_name` contexts.
   - Follow the ghost-row state and restore-on-failure pattern of the template delete
     (`CapturePanelModel`).
   - Run `bob themes hide|unhide NAME --via bob-mac-capture -f json` on a serial,
     non-cancelling `theme-pref` lane in `BobProcessClient.swift`.
   - An older bob maps to `unsupportedCommand`, which shows "Update bob to hide themes".
4. **Selection:** a refresh that answers the same draft and cursor keeps the selected
   row by candidate id. A new query still selects the first row.
5. **Name it prompt** (`PomodoroNamePromptCard`):
   - Show up to 5 suggestions from `bob themes list -q Q -n 6 -f json`, run on a
     cancellable `theme-suggest` lane with the existing 50 ms debounce. Drop names that
     are `today: open`.
   - ↓/↑ and Ctrl-N/P select a suggestion. Tab fills it into the field.
   - Return submits the selected suggestion, or the typed text when nothing is selected.
     Nothing is selected by default.
6. **Docs and tests:** update `docs/keyboard.md` and the `docs/protocol.md`
   compatibility rows. Add CaptureCore tests for decoding, presentation, and the ghost
   state, with fixtures from bob's real output.

**Acceptance:** `just portable-check` is green, macOS CI is green, and the keyboard docs
match the router.

### mac-colors

**Repo:** bob-mac-capture.

1. **Shared views:** one `PomodoroThemeSwatch` and one `PomodoroThemePill` (PT6 tints),
   both reused everywhere below.
2. **Editor:** tint the background behind known `themes` offsets of `pomodoro_name`
   spans in `applyHighlighting`. Unknown names stay neutral. The syntax color is
   unchanged.
3. **Agenda:** in `CaptureAgendaPresentation.swift` and `CaptureAgendaView.swift`, named
   entries wear pills (or swatches in the one-row fold). Home dots stay as they are.
4. **Preview:**
   - destination rows and the `+1 BOB` delta chip in
     `CapturePlanBudgetPresentation.swift` and `CapturePlanBudgetMeterRow.swift`;
   - start, toggle, and project-note presentations that show a Pomodoro name;
   - the Refs Today pill.
5. **Refresh:** add a narrow exception for `.bob/themes/*.jsonl` in
   `CaptureAgendaRefreshFilter.swift`, so hides and pins from any host refresh the
   agenda. Also refresh after the app's own theme writes.
6. **Docs and tests:** update `docs/protocol.md`, and add presentation tests.

**Acceptance:** `just portable-check` is green, macOS CI is green, and screenshots of
light and dark agenda and completion are attached to the phase note if CI produces them.

### obsidian-colors

**Repo:** bob-plugins.

1. **`api.themes` v1 in bob-ledger-tools.**
   - **Data:** a cached `bob themes list -a -f json`, served stale-while-revalidate with
     single flight, through `bobSpawnJson` and the `apiStatsFetch` pattern in
     `236-plugin-stats-data.js`. On mobile it is `available: false`.
   - **Synchronous:** `hueFor`, `componentsOf` (with offsets), `historyRows`, and
     `similarTo`.
   - **Writes:** `hide`, `unhide`, `color`, and `rename`, which spawn bob with
     `--via bob-ledger-tools`, then refresh.
   - **`onChange`.**
   - **JS mirrors** of the hue rule, the journal fold, and the PT7 B/D ranking:
     - The fold reads `.bob/themes/*.jsonl` through the vault adapter on load, on
       refresh, and after its own writes. This works on mobile.
     - The vector sections `components`, `hue`, `fold`, and `rank` must pass.
   - **Docs:** `docs/develop/plugin-apis.md`.
2. **Hue tokens:** `--bob-hue-<slot>`, with the light/dark values of today's
   `--bob-area-*`, which become aliases.
   - Keep one source fragment that the build injects into every plugin stylesheet that
     draws theme color, so the nav and block-id-prompt pickers do not depend on
     ledger-tools CSS.
   - Add the `.bob-pomo-theme` pill, muted, and swatch classes.
3. **Live Preview.**
   - Add `Decoration.mark` per component of the name tail on column-0 `## Pomodoros`
     entries of canonical daily notes.
   - Reuse the home-chip line scan and refresh effect
     (`273-plugin-pomodoro-home-chips.js`).
   - Follow the PT6 typing rule: an unknown name on the cursor line stays neutral.
   - Add a toggle command, **Toggle Pomodoro theme colors**, default on, gated on a body
     class.
4. **Reading view:** the post-processor wraps the components, scoped like
   `274-…-reading.js`.
5. **Plan block:** the `★` theme line renders pills.
6. **Stats:** Themes panel bars take the theme hue. They prefer bob's report hue when
   present, else `hueFor`. Exempt themes and "other" are gray.
7. **Docs and tests:** update `docs/bob-ledger-tools.md`. Add tests in a new
   `scripts/test-*.cjs` that is listed in `package.json`. Run `npm test`, then
   `bob plugins sync -p bob-ledger-tools`.

**Acceptance:** `npm test` passes, with only the known baseline failures. A daily note
in Live Preview shows pills that match `bob themes`.

### obsidian-pickers

**Repo:** bob-plugins.

1. **Link picker** (block-id-prompt, `075-pomodoro-link-targets.js` and
   `085-pomodoro-link-picker-modal.js`):
   - Compose rows per PT7. A Recent row resolves to `{kind:"new", name}` with the exact
     name, so the `pick()` API is unchanged.
   - Swatches on Today and Recent rows; `Recent` and `Today` labels with the Mac's rule
     (mac-menus step 2); `like X` and `hidden` badges.
   - Ctrl+D per PT9: ghost row, Notice and restore on failure, and a footer hint.
   - Feature-detect `api.themes` (v1). When it is absent, the picker stays today-only.
2. **Nav Pomodoro pickers** (`160-pomodoro-reorder.js`, `200-filtered-picker-modals.js`,
   `640-plugin-notes-and-task-move.js`):
   - Recent rows become `new` rows in bullet mode, or `rename` rows in entry mode. This
     single-session rename keeps its scope (ADJ-6). Ctrl+X merge stays today-only.
   - The same swatches, labels, and Ctrl+D behavior as the link picker.
   - The successor picker inherits through `api.pomodoroLinkPicker`.
3. **Vectors:** add the `menu` and `similar` vector sections to the JS tests.
4. **Docs and deploy:** update `docs/block-id-prompt.md` and
   `docs/bob-navigation-hotkeys.md` (keymaps), run `npm test`, then deploy both plugins.

**Acceptance:** `npm test` passes, with only the known baseline failures. Ctrl+D hides
and undoes in each picker on desktop.

### obsidian-manager

**Repo:** bob-plugins (nav).

1. **Command:** **Bob: Pomodoro themes**, a `FilteredPickerModal` subclass with no
   default hotkey.
   - **Views:** Active, Hidden, Possible duplicates (the PT7 similarity rule across the
     catalog), and All. Tab cycles them.
   - **Rows:** a pill, last used (relative), days used, and `today`, `hidden`, `pinned`,
     and `exempt` badges.
   - **Collision hints** appear as quiet rows at the top.
   - **Renames:** `renamed_reappeared` names are flagged.
2. **Keys:**
   - Ctrl+D toggles hide in place.
   - Enter opens actions: `r` rename, `c` color, `h` hide/unhide, `o` occurrences.
3. **Rename:**
   - Validated input, then a preview built from `rename -d -f json`: files, entries,
     before/after tails, and the state and color change.
   - Confirm, adding `--merge` when needed. Before applying, save the open editors, and
     refuse if an open editor has unsaved changes to a target note.
   - Finish with a Notice: "Renamed FIXES → FIX in 3 notes · one commit".
4. **Color:** an 8-swatch strip plus Auto (keys `1`–`8`, `a`). Occurrences lists days;
   Enter opens that daily note at the entry.
5. **Mobile:** a Notice that the manager needs the desktop app.
6. **Docs and deploy:** update `docs/bob-navigation-hotkeys.md`, add tests, run
   `npm test`, then deploy.

**Acceptance:** a scratch-vault rename through the manager flow test produces one
commit. Hide, color, and occurrences work against a stubbed bob.

### menubar

**Repo:** chezmoi (`home/dot_hammerspoon/`).

- **Swatch:** the mac pom prepends a colored `■`, an `hs.styledtext` using the
  light/dark hex for the menu-bar appearance, for each non-exempt theme of the running
  session.
- **Hue source:** `bob themes list -a -f json`, fetched on the indicator's existing
  re-sync triggers and cached until the next re-sync.
- **The title text is never tinted,** because red and green already mean overdue and
  worked.
- **Fallbacks:** an exempt or unknown theme, or a failed bob call, means no swatch and
  an unchanged title.
- **Docs:** update the chezmoi README's "Pomodoro menu bar" spec.
- **Verification:** verify on the MacBook if it is reachable over Tailscale SSH
  (`tailnet.md`). Otherwise leave exact manual-check steps in the phase note.

### records

**Repos:** bob-cli memory and docs.

> [!decision] memory_glossary Through `/sase_memory_write`, add the strand
> `glossary:pomodoro-theme` (Pomodoro Theme; aka theme, Pomodoro name). Write it from
> what shipped, using this outline:
>
> - one `+` component of a [[pomodoro]]'s ` — NAME` suffix, keyed like the plan budget's
>   components and displayed ALL CAPS, with no registry;
> - what `plan.max_themes` counts (exempt themes such as GTD excluded) and what the
>   Stats `theme` dimension and `bob themes` list, hide, color, and rename;
> - a catalog derived from the daily notes plus `.bob/themes/<host>.jsonl`;
> - hiding affects history suggestions only;
> - the hue order of PT5, drawn as a tint or rounded swatch, never an area's ● dot;
> - not a Pomodoro home, an Obsidian theme, or `(unnamed)`.
>
> In `glossary:mac-menu-bar-pomodoro-indicator`, change "`THEME (50m) · 🍅 12:34`, where
> THEME is the Pomodoro's ` — NAME`" so it names the whole name (NAME). Mention the
> theme swatch the `menubar` phase added. Run `sase memory init`.

> [!decision] memory_decisions Through `/sase_memory_write`, add four `decisions`
> records. Each gives where it applies, the claim, why, the rejected alternatives (from
> the research's Rejected alternatives), the cost, what reopens it, and evidence: this
> plan, the research ref, and the landed commits.
>
> 1. `pomodoro-themes-are-name-components`: a theme is a `+` component of a Pomodoro
>    name. Daily notes are its only store. A global rename is one previewed bob
>    transaction ending in one revertable commit.
> 2. `hidden-themes-leave-suggestions-only`: hide affects Recent rows only, today's rows
>    are never hidden, and use never unhides. Ctrl+D works in Pomodoro-name menus.
> 3. `theme-colors-are-hashed-area-hues-with-pins`: the PT5 order as shipped, the pill
>    and swatch form, and never text color.
> 4. `theme-state-is-a-union-merged-vault-journal`: per-host JSONL under `.bob/themes/`,
>    union-merged, with bob as the only writer.
>
> Run `sase memory init`.

**Also:**

- Drop every `planned (phase …)` marker from `docs/pomodoro-themes.md`, and check the
  cross-links from `docs/capture.md`, `docs/plan.md`, `docs/stats.md`, the Mac docs, and
  the plugin docs. Then run `just docs-check`.
- Run the end-to-end acceptance below and record the results.

## End-to-end acceptance

1. **Suggestions:** in a scratch vault with two months of synthetic history:
   - `=#bo` and `@inbox_t:x#deep` list Today rows, then Recent rows, then the create
     row, then substring rows.
   - Accepting each row through `bob capture -d` reports the shown name, byte for byte,
     for SASE V18, DEEP WORK, and BOB while BOB BUILD is open.
2. **Hide:** `bob themes hide BOOKS` removes BOOKS from Recent everywhere (CLI JSON and
   the vectors-driven JS menu). Typing `=#books` still creates and shows `hidden` on the
   create row. `unhide` restores it.
3. **Rename:** a scratch rename of FIXES → FIX passes the dry run, apply, verify, one
   commit, and `git revert`.
4. **Colors:** on the live vault (read-only), these agree for BOB, JOB, BLOG, GTD, and a
   composite: `bob themes -f json`, `bob plan` swatches, `capture-pomodoros --tasks`
   themes, the JS `hueFor` against vectors, and the Stats JSON.
5. **Performance:** the PT3 budgets are met, with numbers.
6. **Docs:** `just docs-check` is green, and the CLI reference shows `bob themes` in
   Daily workflow.

## Risks

- **Keystroke latency on the Mac:** keep the catalog warm and bound the sweep (PT3).
  Measure on apollo in each bob-cli phase that touches the hot path. If the Mac
  regresses, the escalation path is a longer sweep interval, not an in-app index.
- **New lexer surface:** the exact-name literal is covered by round-trip fixtures, and
  the slug stays the common case.
- **JS/Rust drift:** one vectors file gates the hue, the fold, the ranking, and the
  menu.
- **Rename vs open editors:** compare-before-write, the manager's save-and-refuse step,
  and Obsidian's reload of clean buffers bound the risk.
- **Journal allowlist lag:** a host that has not synced `.gitignore` keeps its hides
  local until it does. This is documented.
- **Clock skew:** last writer wins across hosts and trusts the clocks. That is an
  accepted risk for one user.
