---
tier: epic
title: 'Bob command mode: press : in Obsidian to run bob'
goal: 'Pressing `:` in Obsidian''s Vim normal mode opens a fast, beautiful bob command
  sheet. The user filters a bob-owned catalog of every useful bob command, runs one
  with context-aware arguments and safe-write guards, and is notified when it finishes,
  with three honest outcome classes (completed, needs attention, failed). The full
  output is always one key away in a pager that doubles as run history. Vim''s own
  ex line stays reachable through `::`, Vim-shaped input, and `g:`.

  '
decisions:
  pomodoro_actions:
    ask: Add named Pomodoro session rows (start, stop, extend, shorten) that run bob
      capture operators?
    default: true
    why: The research listed them; starting a Pomodoro from any note is a common in-Obsidian
      need
    answer: true
  free_text_capture:
    ask: Also accept free-text capture TEXT… runs in command mode, previewed with
      a ⇧↵ dry run?
    default: false
    why: Keeps the research's exclusion; capture deserves live capture-parse spans,
      a separate feature
    answer: false
  decision_record:
    ask: Add a decisions record that `:` opens bob command mode and every vimrc map
      replaying `:` is nnoremap?
    memory:
    - decisions
    default: false
    answer: false
phases:
- id: vault-prep
  title: Make the colon key safe and keep the new plugin out of vault git
  depends_on: []
  size: small
  description: 'vault-prep: turn all 16 recursive `nmap … :bob_*<CR>` lines in the
    vault vimrc into `nnoremap`, add `nnoremap g: :`, and gitignore `.obsidian/plugins/bob-command-mode/`
    in the vault, without binding `:` yet.'
- id: catalog
  title: bob __catalog --protocol 1
  depends_on: []
  size: medium
  description: 'catalog: add the hidden, versioned JSON command catalog to bob-cli.
    A static table sits beside SUBCOMMANDS, positional args and about text come from
    the clap tree, coverage tests classify every public leaf, and a contract docs
    page documents it.'
- id: run-env
  title: Color, progress, and exit receipts for non-TTY runs
  depends_on:
  - catalog
  size: small
  description: 'run-env: bob honors CLICOLOR_FORCE, BOB_PROGRESS, and BOB_RUN_RECEIPT
    without a TTY, advertises them as catalog `features`, and documents them in the
    configuration reference.'
- id: completion-names
  title: Theme-name and relation-label completion providers
  depends_on: []
  size: small
  description: 'completion-names: complete theme names for `themes hide|unhide|color|rename
    OLD` through a read-only theme-catalog load, and complete relation labels for
    `relations hide|unhide`.'
- id: runner
  title: bob-command-mode plugin scaffold and run engine
  depends_on:
  - catalog
  - run-env
  size: medium
  description: 'runner: create the desktop-only bob-command-mode plugin with a bob
    resolver, child env, catalog cache, and detached file-backed runs. It also adds
    run records and receipts, writer queue and save-before-write, timeouts, generic
    outcome classification, api.commandMode v1, and Ctrl+P run commands, tested against
    a fake bob.'
- id: feedback
  title: Toasts, status chip, outcome adapters, and OS notifications
  depends_on:
  - runner
  size: medium
  description: 'feedback: replace the runner''s plain notices with the three-class
    toast cards and the live status-bar chip. It adds per-command outcome adapters
    and their action buttons, OS notifications for long unfocused runs, Rerun last
    command, and a Chromium preview harness for visual review.'
- id: pager
  title: Output pager and run history
  depends_on:
  - feedback
  size: medium
  description: 'pager: build the streaming output pager. It renders ANSI color, marks
    stderr, pretty-prints JSON, links vault references, and adds Vim-style keys, search,
    copy, rerun, stop, and history stepping. Show last output and the toasts'' Output
    action open it.'
- id: sheet
  title: The bottom command sheet
  depends_on:
  - pager
  size: medium
  description: 'sheet: build the `Bob: Command mode` bottom sheet. It has fuzzy ranking
    with keywords and frecency, Recent, For this note, and From clipboard sections,
    surface rows, preview-first and confirm flows, `?` help, and in-place growth into
    the pager for reports.'
- id: args-vim
  title: Argument mode, typed runs, Vim routing, and binding the colon key
  depends_on:
  - sheet
  - vault-prep
  - completion-names
  size: medium
  description: 'args-vim: add argument mode backed by `bob __complete`, a quote-aware
    tokenizer for typed bob argv with excluded-command refusal, and Vim routing (`::`,
    Vim-shaped input, fallback row, handleEx). It also adds the vimrc lint, then binds
    `:` in the vault vimrc.'
- id: integrations
  title: Move Snapshot to Highlights onto the runner and refresh ledger caches
  depends_on:
  - pager
  size: small
  description: 'integrations: bob-project-tasks'' Snapshot note to Highlights runs
    through api.commandMode when it is available, and bob-ledger-tools refreshes the
    caches a finished run invalidates.'
- id: docs-land
  title: Guide, reference polish, and end-to-end verification
  depends_on:
  - args-vim
  - integrations
  size: small
  description: 'docs-land: write the "Run bob from Obsidian" guide, finish the plugin
    reference page, run every repo''s gates and the end-to-end checklist, and add
    the decisions record if accepted.'
proposed_by: bbugyi200.athena.research.5c.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-26 10:35:51
status: done
bead_id: bob-cli-ce
---

- **PROMPT:** [prompts/202610/bob_colon_command_mode.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/bob_colon_command_mode.md)
- **BEAD:** [bob-cli-ce](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ce/README.md)

# Bob command mode: press `:` in Obsidian to run bob

## Outcome

In Vim normal mode, `:` opens a **bob command sheet** docked at the bottom of the active
pane. The user types a few letters (`sync`, `pdf`, `keep`), sees ranked bob commands
with one-line explanations, and presses `↵`. Reports such as `plan` or `freshness`
stream into an output pager in the same sheet. Actions such as `vault-sync run` or
`ref create` close the sheet, show a live status-bar chip, and end in a toast that says
**completed**, **needs attention**, or **failed**. The toast's **Output** button opens
the full stdout and stderr in the pager, which also serves as run history. Nothing makes
the user leave Obsidian, and nothing silently rewrites a note under the editor.

This plan implements the consolidated recommendation in
`research:202610/bob_colon_command_mode/bob_colon_command_mode.md`. Bryan agreed with
all of its requirements (R1–R9). Where this plan goes beyond the report, it says so. The
main departures are these:

- The whole v1 and v2 catalog ships at once, because catalog rows are cheap data and
  every behavior they need is generic.
- Ranking uses the plugin's own pure, unit-tested fuzzy scorer rather than Obsidian's
  `prepareFuzzySearch`, so ranking is deterministic in tests.
- bob writes an exit receipt (`BOB_RUN_RECEIPT`), so a run that outlives an Obsidian
  reload still reports its true exit code.
- The plugin learns which environment knobs bob supports from catalog `features`, so a
  newer plugin degrades gracefully on an older bob.

## Principles every phase follows

1. **bob owns the facts; the plugin owns presentation.** Command names, about text,
   arguments, effects, preview flags, confirm text, exit-code meanings, and exclusions
   come from `bob __catalog`. The plugin never hard-codes a bob command list. It owns
   only context mapping, surface rows (existing Obsidian UIs), ranking, keys, the sheet,
   the pager, toasts, the runner, and history. This applies
   `decisions:mac-capture-is-a-thin-client` by analogy.
2. **Writes are safe by construction.**
   - Save every open Markdown editor before any non-read run, and refuse the run if a
     save fails.
   - Run one writer at a time; reads run in parallel.
   - Preview first for broad rewrites, and show a confirm card for writers that have no
     dry run.
   - Never auto-kill a writer.
   - Never spawn a shell: argv goes straight to `spawn(bobPath, argv)`.
3. **Outcomes are honest.** Exit 0 is not proof of success, and a non-zero exit is not
   always a failure. Catalog exit-code hints and per-command adapters decide among
   _completed_, _needs attention_, and _failed_. A run whose fate is unknown says
   _unknown, log preserved_.
4. **Runs outlive the UI.** Children are detached, and their output goes to files.
   History is host-local and lives outside the vault.
5. **Beautiful means calm and native.**
   - Use only Obsidian theme variables.
   - Use monospace for anything you could type and the UI font for anything that
     explains it.
   - Give each meaning one accent color, and always pair color with an icon and text.
   - Keep motion small, respect `prefers-reduced-motion`, and add no new dependencies.

## The experience (shared spec for the plugin phases)

### Entry points

- **`:` in Vim normal mode.** The vault vimrc maps it with
  `exmap bob_command obcommand bob-command-mode:open` and `nnoremap : :bob_command<CR>`,
  bound in phase `args-vim`. Visual-mode `:` stays native, so `'<,'>` ranges keep
  working.
- **`g:`** opens Vim's native ex line, even when the plugin is disabled.
- **`Bob: Command mode`** (command id `open`) has no default hotkey. It covers Reading
  view, Canvas, and other non-editor views.
- **Ctrl+P commands:**
  - `Bob: Show last output` (`show-last-output`)
  - `Bob: Rerun last command` (`rerun-last`)
  - one `Bob: <title>` per catalog entry whose positionals are all optional and whose
    policy is `run` (id `run-<entry-id>`)

### Sheet layout

The sheet is an Obsidian `Modal` subclass with the class `bob-cmd-sheet`, docked at the
bottom centre of the active pane:

- **Width:** pane width − 48 px, at most 920 px.
- **Height:** up to 45 vh.
- **Background:** the note stays visible above, with the modal background dimmed to
  about 12%.
- **Motion:** slides up in 120 ms.

Copy the picker core's Obsidian ≥ 1.14 `isOpen` workaround (an own `pickerOpen` flag)
from `bob-navigation-hotkeys/src/200-filtered-picker-modals.js`, and copy the
`bob-cnp-*` row anatomy under a `bob-cmd-` prefix. Plugins never import each other's
code. A sketch, not a mockup:

```text
╭─ bob ─────────────────────────────────────────────── athena · bob 0.3.2 ─╮
│ ❯ sync▏                                                                  │
├──────────────────────────────────────────────────────────────────────────┤
│▌⟳  vault-sync run      Sync vault now                     syncs  ⇧↵  ~4 s│
│    Run one reconcile cycle: pull, commit, and push through Git           │
│  ✎  projects sync      Sync projects                      writes ⇧↵      │
│  ⓘ  vault-sync status  Vault sync status                  reads          │
├──────────────────────────────────────────────────────────────────────────┤
│ ↵ run  ⇥ args  ⇧↵ preview  ⌃↵ run now  ? help  :: vim  esc close         │
╰──────────────────────────────────────────────────────────────────────────╯
```

**Rows.**

- Rows are dense single lines of about 34 px, and only the selected row expands to show
  its `about` line.
- **Left:** a Lucide icon for the row kind:

  | Row kind    | Icon              |
  | ----------- | ----------------- |
  | `network`   | `globe`           |
  | `syncs`     | `refresh-cw`      |
  | `writes`    | `pencil-line`     |
  | `reads`     | `book-open`       |
  | Surface row | `app-window`      |
  | Vim row     | `terminal-square` |
  | Help        | `help-circle`     |
  | Recent row  | its outcome glyph |

- **Then:** the command path in monospace, with matched characters in `--text-accent` at
  weight 600. The human title follows in `--text-muted`.
- **Right-hand chips:**
  - the effect (`reads` muted, `writes` orange, `syncs` blue, `net` purple, `opens` for
    surface rows);
  - a `⇧↵` kbd chip when a preview exists;
  - a typical duration (`~41 s`, the median of the last 10 local runs, shown only above
    2 s);
  - an unavailable reason (`Mac only`, `Needs a newer bob`, `Enable bob-ledger-tools`).
    Unavailable rows stay visible, dimmed.
- **Selection:** a 3 px accent bar plus a tinted background, as in
  `.bob-cnp-row.is-selected`.
- **Section headers:** small caps, 0.06 em tracking.
- **Footer:** a `kbd` hint bar whose hints change with the stage.

**The header** reads `bob` on the left and `<short host> · bob <version>` on the right.
The version comes from the catalog.

**Empty query (just `:`).** Sections appear in this order:

1. **RECENT:** the last 4 distinct argv, each with a ✓/◐/✗/? glyph, age, and duration.
2. **FOR THIS NOTE · `<path>`:** context-filled rows, listed under _Context_ below.
3. **FROM CLIPBOARD · `<url>`:** shown only when the clipboard holds one http(s) URL.
4. **One section per catalog section**, in bob's order (Daily workflow, Tasks and
   projects, Vault, Integrations, Setup). Each section holds that section's entries and
   surface rows in catalog order.

**With a query.** The list is flat and ranked, best match first, with no section
headers. Two rows can be pinned above the matches:

- a typed-run row, when the query parses as bob argv (`args-vim`);
- a Vim row, when the query is Vim-shaped (`args-vim`).

When nothing matches, one explicit row offers "Run as Vim `:<query>`". `↵` never runs a
hidden best guess.

**Ranking** uses a pure scorer in the plugin, not Obsidian internals:

- **Matching:** fzf-style subsequence matching with bonuses for consecutive runs, word
  boundaries, and prefixes. Whitespace-separated query tokens must all match (AND).
- **Field weights:** command path 1.0, alias 0.9, title 0.8, keywords 0.6.
- **Bonuses:** an exact path-prefix match and an exact keyword match each add a bonus.
- **Ties:** frecency, kept per host, breaks ties, then catalog order. Frecency is the
  sum over each row's last 10 run timestamps of 100 / 70 / 50 / 30 / 10, for runs at
  most 4 / 14 / 31 / 90 days old and older.
- **Stability:**
  - A strong exact match always beats recency.
  - Results are recomputed only when the query changes.
  - Selection is kept by row key, so a background catalog refresh or late clipboard rows
    never move the cursor.

### Keys

**List stage:**

| Key                   | Action                                                                                                  |
| --------------------- | ------------------------------------------------------------------------------------------------------- |
| `↵`                   | Run the selected row, following its policy. If required args lack a value, enter argument mode instead. |
| `⇥`                   | Put the row's path in the input and enter argument mode; never runs.                                    |
| `⇧↵`                  | Dry run (the entry's `preview` args) into the pager.                                                    |
| `⌃↵`                  | Run now: skips preview-first and confirm, but never skips the save guard.                               |
| `↑ ↓` / `⌃N ⌃P`       | Move the selection (wraps) without changing the query.                                                  |
| `?` at query end      | `bob <path> --help` for the selected row in the pager; a bare `?` shows `bob --help`.                   |
| `::` at query start   | Switch to the Vim stage (VIM pill).                                                                     |
| `Esc`                 | Close.                                                                                                  |
| `⌫` on an empty query | Close, like Vim's own prompt.                                                                           |

Do not bind `j`, `k`, or `q` inside the input; they are search characters. Mouse click
on a row equals `↵`.

**Argument stage** (`args-vim`):

- The path, plus any fixed args, becomes a pill. Ghost placeholders show the missing
  positionals (`‹TARGET…›`). Completions list below the input.
- `⇥` accepts the top or selected completion. `↑ ↓` select a completion.
- `↵` accepts the arrowed completion when one is selected. Otherwise it runs, but only
  if every required positional has a value; if one is missing, an amber hint names it.
- `⇧↵` and `⌃↵` work as in the list stage.
- `Esc`, or `⌫` on an empty argument input, goes back to the list with the path as the
  query.

**Confirm card:** an amber-bordered card inside the sheet with the confirm text and the
argv. Only **`⌃↵`** or a click on **Run** confirms, so a reflexive second `↵` never
fires a writer. `Esc` returns to the list.

**Preview:** the dry run streams into the pager. The footer's primary action is
**`⌃↵ Apply`**, which runs the same argv without the preview args, plus any args an
adapter carried over. Apply is disabled if the preview failed.

**Pager** (phase `pager`):

| Key                        | Action                                                                        |
| -------------------------- | ----------------------------------------------------------------------------- |
| `j k d u g G`              | Scroll                                                                        |
| `/` then `n N`             | Search                                                                        |
| `w`                        | Wrap                                                                          |
| `y` / `Y`                  | Copy the ANSI-stripped output / the shell-quoted command                      |
| `r`                        | Rerun (re-resolves context and guards; never replays a stale preview's apply) |
| `[` / `]`                  | Older / newer run                                                             |
| `⌃C` with no text selected | Stop: reads at once; writers after a confirm                                  |
| `:`                        | Back to the command list                                                      |
| `Esc`                      | Back to the list when the pager was opened from it, otherwise close           |
| `q`                        | Close                                                                         |

### Presentation by kind

- **report:** the sheet grows upward in place to about 80 vh, full pane width at most
  1200 px, over 90 ms, and streams output. `Esc` returns to the list with the query
  intact, so `plan` → `freshness` → `ready` takes three short trips.
- **action:** the sheet closes at once, and focus and cursor return to the origin
  editor. A status chip appears for runs longer than 400 ms. A toast arrives on finish.
- **inline:** the sheet closes, and the toast carries the one-line result (for example
  `pomodoro status`).
- **No toast while watching:** a run whose pager is open and focused at finish gets no
  toast; its status pill updates instead.
- **OS notification:** a run longer than 15 s that finishes while Obsidian is unfocused
  raises one (a setting, on by default). Clicking it opens the run's pager.

### Toasts

A toast is a rich `Notice(DocumentFragment, duration)` card,
`bob-cmd-toast is-{completed|attention|failed|unknown|stopped}`. It mirrors
`bob-nh-notice`: a 4 px left border in the state color and a plain-text fallback when no
DOM is available.

- **Header:** a glyph (✓ green, ◐ amber, ✗ red, ? red, ■ muted), the entry title, and
  the duration on the right.
- **Body:**
  - a summary of at most 2 lines, from the adapter or the last meaningful line, with
    ANSI stripped;
  - the argv in muted monospace (`bob vault-sync run`).
- **Actions:** adapter actions first (for example **Open note**, **Reload 2 plugins**,
  **Scan intake**), then **Output**, **Retry**, and **Copy**.
- **Timing:**
  - completed hides after 5 s;
  - needs attention hides after 8 s;
  - failed and unknown stay until dismissed.
- **Replacement:** each entry keeps at most one sticky failure toast; a newer one
  replaces it.
- **Clicks:** clicking the card body equals **Output**.

### Status chip

`addStatusBarItem` with `bob-cmd-chip` shows a spinner and `ref create · 0:42`. With
several runs it shows `bob ×2 · 1:05`. A writer past its budget turns the chip amber
with "still running past ~3 min". Hovering shows the argv and the last progress line.
Clicking opens the newest active run's pager. The chip is hidden when idle.

### Context and the "For this note" rows

Context is captured once when the sheet opens: the origin leaf, file, editor, and
cursor. It is never retargeted silently. bob's catalog declares a `slot` for each
positional, and the plugin maps slots to context:

| Slot                                               | Context source                                                                                                 |
| -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `ref`                                              | Active note when its frontmatter `type` is `"[[ref]]"`                                                         |
| `area-note`                                        | Active note when `type` is `"[[area]]"` or `"[[project]]"`                                                     |
| `article-note`                                     | Active note when bob-articles treats it as an article; reuse bob-articles' detection rule, found in its source |
| `ref-target`                                       | Clipboard http(s) URL, else the active Markdown note                                                           |
| `url`, `ref-query`                                 | Clipboard http(s) URL; `url` values are validated as http(s) before any run                                    |
| `vault-note`                                       | Active Markdown note                                                                                           |
| `theme`, `relation-label`, `metric`, `pdf`, `text` | None; completion only                                                                                          |

The generated rows are complete argv rows. `↵` runs them under their policy.

- **FOR THIS NOTE**, by note type:
  - **ref note:** `ref today <note>` (Pull into today's Pomodoro), `ref show <note>`,
    and `ref rename <note>` (opens argument mode with REF filled).
  - **area or project note:** `ready <note>`.
  - **article note:** `articles outline <note>`, plus the _Format this article_ surface
    row.
  - **any other Markdown note (including area, project, and article notes):**
    `ref create <note>`, titled _Snapshot note to Highlights_.
- **FROM CLIPBOARD:** `capture <url>` (_Queue link for reading_, the fast background
  job), `ref create <url>` (_Create reference now_), and `ref find <url>` (_Is this in
  my library?_).

Optional positionals are never auto-filled on a plain row run. The context-filled
variant is its own visible row.

### Surface rows (plugin-owned)

Each surface row runs `app.commands.executeCommandById(id)` and is feature-detected
through `app.commands.commands[id]`. A missing command makes the row unavailable, with
an "Enable <plugin>" reason.

| Surface row              | Command id                                                        | Section |
| ------------------------ | ----------------------------------------------------------------- | ------- |
| Today's daily note       | `bob-ledger-tools:open-today-daily-note`                          | Daily   |
| Pomodoro themes manager  | `bob-navigation-hotkeys:open-pomodoro-themes-manager`             | Daily   |
| Stats dashboard          | `bob-ledger-tools:open-stats`                                     | Daily   |
| Dash tasks               | `bob-navigation-hotkeys:open-dash-tasks`                          | Tasks   |
| Relation labels manager  | `bob-navigation-hotkeys:open-relation-labels-manager`             | Tasks   |
| Reading Board            | `bob-ledger-tools:open-reading-board`                             | Vault   |
| Format this article      | `bob-articles:format-article`                                     | Vault   |
| Preview article adoption | bob-articles' adopt-preview command id (look it up in its source) | Vault   |

The plugin also owns three plain rows: **bob help** (`bob --help` into the pager),
**Show last output**, and **Run history** (the pager at the newest run).

## Contract: `bob __catalog --protocol 1`

The handshake mirrors `__complete`:

- **Request:** `bob __catalog --protocol N`, also accepting `--protocol=N`. A missing or
  malformed request prints usage to stderr and exits 2.
- **Supported protocol:** prints one JSON document to stdout and exits 0. It must never
  read the vault, the config, or caches, and never write anything.
- **Unsupported protocol:** prints
  `{"schema_version":1,"ok":false,"code":"unsupported_protocol","min":1,"max":1,"message":"…"}`
  and exits 2.

The success document looks like this; the comments are field rules:

```jsonc
{
  "schema_version": 1, "ok": true, "protocol": 1, "bob_version": "0.3.2",
  "features": [],                       // run-env adds: "clicolor-force", "progress-env", "run-receipt"
  "sections": [{"id": "daily", "title": "Daily workflow"}, …],  // daily, tasks, vault, integrations, setup
  "roots": ["articles", "capture", …],  // every public top-level name plus root alias names
  "aliases": [{"from": ["task-status-hooks"], "to": ["task", "reconcile"]},
              {"from": ["ref", "clip"], "to": ["ref", "create"]}, …],
  "entries": [{
    "id": "ref-create",                 // unique kebab-case
    "path": ["ref", "create"],          // canonical public leaf
    "fixed": [],                        // args always passed (for example ["--intake-only"], ["-f","json"], ["="])
    "title": "Create reference",        // human title (table)
    "about": "…",                       // the leaf's clap about, first line (derived)
    "section": "vault",                 // must equal the root's SUBCOMMANDS section (test)
    "keywords": ["pdf", "highlights", "clip", "snapshot", "url", "arxiv"],
    "effect": "writes",                 // reads | writes | syncs (writes plus Git commit/push)
    "network": true,
    "presentation": "action",           // report | action | inline
    "policy": "run",                    // run | preview-first | confirm
    "preview": ["--dry-run"],           // args inserted for a dry run, or null
    "confirm": null,                    // required text when policy = confirm
    "budget_secs": 900,
    "args": [{"name": "TARGET", "slot": "ref-target", "required": true, "repeat": true}],
    "exit_codes": [],                   // [{"code": 3, "outcome": "attention", "meaning": "…"}]
    "adapter": "ref-create",            // plugin adapter id, or null
    "hosts": [],                        // [] = every host; otherwise e.g. ["macos"]
    "invalidates": ["references", "tasks"]  // tasks | themes | relations | stats | references | plugins
  }, …],
  "excluded": [{"path": ["task", "move"], "reason": "Use Ctrl+Shift+M, which moves tasks through the editor"}, …]
}
```

**Argv assembly.** The plugin builds the final argv as
`path + (preview ? preview : []) + fixed + user args`. Preview args always come before
the fixed args, so `capture --dry-run -- -` works.

**Derived versus authored fields.**

- **Derived:** `args` (name, required, repeat) comes from the leaf's clap positionals,
  and `about` from its clap about. `aliases` comes from `ALIASES` and clap subcommand
  aliases. `roots` and `sections` come from `SUBCOMMANDS` and `SECTIONS`, excluding the
  Capture protocol section.
- **Authored:** every other field lives in the static table. The table authors only a
  positional's `slot`, defaulting to `text`.

**Compatibility.** Additive fields keep `schema_version` 1. Clients ignore unknown
fields. An unknown `effect` is treated as `writes`, an unknown `presentation` as
`action`, an unknown `policy` as `confirm`, and an unknown `slot` as `text`.
Incompatible changes bump the protocol.

**Raw refusal.** Raw typed argv on an `excluded` path is refused with its reason.
Entries may still exist on an excluded leaf, but only through their own rows: the
capture rows and `ref-queue`. A typed argv whose longest path prefix is an entry
inherits that entry's metadata. That is how "any bob argv" runs under the same guards.

## Catalog v1 contents

Every useful command ships now. Effects: `R` reads, `W` writes, `S` syncs, `+net`
network. Presentation: `rep` report, `act` action, `inl` inline. Budgets are in seconds.
Reads default to 60 and writers to 180. `D` marks the `--dry-run` preview.

**Daily workflow**

| id               | argv (path · fixed · ARGS)              | title                  | eff | pres | policy        | prev | budget | notes                              |
| ---------------- | --------------------------------------- | ---------------------- | --- | ---- | ------------- | ---- | ------ | ---------------------------------- |
| plan             | `plan`                                  | Today's plan           | R   | rep  | run           | –    | 60     |                                    |
| freshness        | `freshness list`                        | Review queue           | R   | rep  | run           | –    | 60     |                                    |
| ready            | `ready [NOTE]`                          | Ready lanes            | R   | rep  | run           | –    | 60     | NOTE slot `area-note`              |
| pomodoro-status  | `pomodoro status`                       | Pomodoro status        | R   | inl  | run           | –    | 30     | adapter `pomodoro-status`          |
| pomodoro-start   | `capture` · `=`                         | Start next Pomodoro    | W   | act  | run           | D    | 60     | adapter `capture-session`          |
| pomodoro-stop    | `capture` · `x`                         | Stop running Pomodoro  | W   | act  | run           | D    | 60     | adapter `capture-session`          |
| pomodoro-extend  | `capture` · `+`                         | Extend Pomodoro 5 min  | W   | act  | run           | D    | 60     | adapter `capture-session`          |
| pomodoro-shorten | `capture` · `-- -`                      | Shorten Pomodoro 5 min | W   | act  | run           | D    | 60     | adapter `capture-session`          |
| stats            | `stats overview`                        | Stats glance           | R   | rep  | run           | –    | 60     |                                    |
| stats-show       | `stats show [METRIC]…`                  | Stats chart            | R   | rep  | run           | –    | 60     | slot `metric`                      |
| stats-metrics    | `stats metrics`                         | Stats metric catalog   | R   | rep  | run           | –    | 60     |                                    |
| themes-list      | `themes list`                           | List Pomodoro themes   | R   | rep  | run           | –    | 60     |                                    |
| themes-hide      | `themes hide` · `-f json` · `NAME…`     | Hide themes            | W   | act  | run           | –    | 60     | slot `theme`; adapter `json-write` |
| themes-unhide    | `themes unhide` · `-f json` · `NAME…`   | Unhide themes          | W   | act  | run           | –    | 60     | same                               |
| themes-color     | `themes color` · `-f json` · `NAME HUE` | Pin a theme's hue      | W   | act  | run           | –    | 60     | same                               |
| themes-rename    | `themes rename OLD NEW`                 | Rename a theme         | S   | act  | preview-first | D    | 180    | OLD slot `theme`                   |

> [!decision] pomodoro_actions The four `pomodoro-*` capture rows ship only when this is
> yes. With no, the `capture` leaf has only the `ref-queue` entry. The `capture-session`
> adapter reads bob's human result line ("Started …", "finish the current Pomodoro
> first", and so on) for the toast summary. Run `bob capture --help` for the exact
> grammar before authoring these rows, and test each against a fixture vault.

**Tasks and projects**

| id               | argv                                      | title                         | eff | pres | policy        | prev | budget | notes                                                 |
| ---------------- | ----------------------------------------- | ----------------------------- | --- | ---- | ------------- | ---- | ------ | ----------------------------------------------------- |
| projects-list    | `projects list`                           | Projects                      | R   | rep  | run           | –    | 60     |                                                       |
| projects-sync    | `projects sync`                           | Sync projects                 | W   | act  | run           | D    | 180    | toast offers **Reconcile tasks**                      |
| relations-list   | `relations list`                          | Relation labels               | R   | rep  | run           | –    | 60     |                                                       |
| relations-hide   | `relations hide` · `-f json` · `LABEL…`   | Hide relation labels          | W   | act  | run           | –    | 60     | slot `relation-label`; adapter `json-write`           |
| relations-unhide | `relations unhide` · `-f json` · `LABEL…` | Unhide relation labels        | W   | act  | run           | –    | 60     | same                                                  |
| relations-rename | `relations rename OLD NEW`                | Rename a relation label       | S   | act  | preview-first | D    | 180    | OLD slot `relation-label`                             |
| task-reconcile   | `task reconcile`                          | Reconcile task statuses       | W   | act  | run           | D    | 180    | outlasts its 120 s retry                              |
| task-reroll      | `task reroll`                             | Re-roll due prioritized tasks | W   | act  | preview-first | D    | 180    | adapter `task-reroll` applies with the preview's seed |
| task-archive     | `task archive`                            | Archive done tasks            | W   | act  | confirm       | –    | 300    | no dry run                                            |

**Vault**

| id                | argv                            | title                              | eff   | pres | policy        | prev | budget | notes                                                  |
| ----------------- | ------------------------------- | ---------------------------------- | ----- | ---- | ------------- | ---- | ------ | ------------------------------------------------------ |
| articles-list     | `articles list`                 | Articles                           | R     | rep  | run           | –    | 60     |                                                        |
| articles-outline  | `articles outline NOTE`         | Article outline                    | R     | rep  | run           | –    | 60     | slot `article-note`                                    |
| nightly           | `nightly`                       | Nightly maintenance                | S     | act  | confirm       | –    | 1800   | confirm text warns that athena's timer already runs it |
| query-source      | `query` · `--source` · `SOURCE` | Find notes by Dataview source      | R     | rep  | run           | –    | 60     |                                                        |
| query-tasks       | `query` · `--tasks` · `QUERY`   | Run a Tasks query                  | R     | rep  | run           | –    | 60     |                                                        |
| query-dql         | `query` · `--query` · `DQL`     | Run a Dataview query               | R     | rep  | run           | –    | 60     |                                                        |
| ref-queue         | `capture URL`                   | Queue link for reading             | W     | act  | run           | D    | 60     | slot `url`; on the excluded `capture` leaf             |
| ref-create        | `ref create TARGET…`            | Create reference                   | W+net | act  | run           | D    | 900    | slot `ref-target`; adapter `ref-create`                |
| ref-find          | `ref find QUERY…`               | Find in library                    | R     | rep  | run           | –    | 60     | slot `ref-query`                                       |
| ref-list          | `ref list`                      | Reading list                       | R     | rep  | run           | –    | 60     |                                                        |
| ref-show          | `ref show REF…`                 | Show reference                     | R     | rep  | run           | –    | 60     | slot `ref`                                             |
| ref-today         | `ref today REF`                 | Pull reading into today's Pomodoro | W     | act  | run           | D    | 60     | slot `ref`                                             |
| ref-rename        | `ref rename REF NAME`           | Rename a reference                 | S     | act  | preview-first | D    | 300    | slot `ref`, `text`                                     |
| ref-doctor        | `ref doctor`                    | Library health check               | R     | rep  | run           | –    | 120    |                                                        |
| ref-scan          | `ref scan`                      | Scan Highlights library            | W     | act  | run           | D    | 1800   | adapter `ref-scan`                                     |
| ref-scan-intake   | `ref scan` · `--intake-only`    | File queued intake                 | W     | act  | run           | D    | 900    | adapter `ref-scan`                                     |
| ref-jobs          | `ref jobs list`                 | Reference jobs                     | R     | rep  | run           | –    | 60     |                                                        |
| ref-jobs-run      | `ref jobs run`                  | Run queued reference jobs          | W+net | act  | run           | –    | 1800   |                                                        |
| ref-sync          | `ref sync PDF`                  | Sync one PDF marker                | W     | act  | run           | D    | 300    | slot `pdf`                                             |
| ref-marker        | `ref marker PDF`                | Inspect a PDF marker               | R     | rep  | run           | –    | 60     | slot `pdf`                                             |
| vault-sync        | `vault-sync run`                | Sync vault now                     | S     | act  | run           | D    | 300    | adapter `vault-sync`                                   |
| vault-sync-status | `vault-sync status`             | Vault sync status                  | R     | rep  | run           | –    | 30     |                                                        |

**Integrations**

| id           | argv           | title            | eff   | pres | policy        | prev | budget | notes                |
| ------------ | -------------- | ---------------- | ----- | ---- | ------------- | ---- | ------ | -------------------- |
| gkeep-list   | `gkeep list`   | Keep inbox       | R+net | rep  | run           | –    | 60     |                      |
| gkeep-pull   | `gkeep pull`   | Drain Keep inbox | W+net | act  | preview-first | D    | 300    | adapter `gkeep-pull` |
| gkeep-doctor | `gkeep doctor` | Keep setup check | R+net | rep  | run           | –    | 60     |                      |

**Setup**

| id           | argv                       | title           | eff   | pres | policy | prev | budget | notes                             |
| ------------ | -------------------------- | --------------- | ----- | ---- | ------ | ---- | ------ | --------------------------------- |
| plugins-list | `plugins list`             | Plugin versions | R+net | rep  | run    | –    | 120    | pulls the plugins repo by default |
| plugins-sync | `plugins sync` · `-f json` | Deploy plugins  | W     | act  | run    | D    | 180    | adapter `plugins-sync`            |

**Excluded** (raw argv refused with the reason):

- `capture` (free text): "Capture text from Bob Mac Capture or a terminal; command mode
  runs its named capture rows".
- Every `capture-*` protocol leaf and every `capture-templates` member: "Capture
  protocol for capture clients".
- Every `completion` member: "Shell completion; run it in a terminal".
- `pomodoro notify`: "Long-running notifier that never exits".
- `pomodoro tmux`: "Prints a tmux status line".
- `gkeep login`: "Needs an interactive terminal and handles credentials".
- `gkeep migrate-markers`, `gkeep migrate-tasks`, `ref migrate-tasks`, and
  `ref migrate-zorg`: "One-time migration; run it from a terminal".
- `task move`: "Use Ctrl+Shift+M, which moves tasks through the editor".
- `articles format`: "Use Format this article (bob-articles formats through the
  editor)".
- `articles adopt`: "Use bob-articles' adoption preview".

Hidden leaves such as `freshness seed` are neither entries nor excluded.

> [!decision] free_text_capture = no `capture` stays in `excluded`. A typed
> `capture buy milk` shows the refusal plus the capture rows as suggestions.

> [!decision] free_text_capture Add an entry `capture` with path `["capture"]`, title
> "Capture", effect W, action, policy run, preview D, one `TEXT…` slot `text`, and
> adapter `capture-session`. Remove `capture` from `excluded`. Typed `capture …` then
> runs with the guards, and `⇧↵` previews it.

**Keywords.** Choose keywords generously, all lowercase. Acceptance requires these
lookups to find these rows:

| Query                           | Expected rows                       |
| ------------------------------- | ----------------------------------- |
| `push`, `pull`, `git`, `commit` | `vault-sync`                        |
| `keep`, `inbox`                 | `gkeep-*`                           |
| `pdf`                           | `ref-create` and `ref-scan`         |
| `deploy`, `install`             | `plugins-sync`                      |
| `pom`, `tomato`                 | Pomodoro rows                       |
| `reading`                       | `ref-list` and `ref-queue`          |
| `clip`                          | `ref-create`                        |
| `task-status-hooks`             | `task-reconcile`, through `aliases` |

**Hosts.** Leave `hosts` empty unless bob's own docs or code restrict a command to one
OS. The research suggested `ref scan` is Mac-only; confirm that against
`docs/highlights-ref-sync.md` before marking it.

## Run engine (spec for `runner`, extended by later phases)

1. **Resolve bob.** Try these in order, recording the source and every path tried:
   1. the plugin setting `bobPath`;
   2. `app.plugins.plugins["bob-ledger-tools"]?.api?.stats?.bobPath()`
      (feature-detected);
   3. the candidates `~/.cargo/bin/bob`, `~/.local/bin/bob`, `/opt/homebrew/bin/bob`,
      and `/usr/local/bin/bob`;
   4. `bob` on `PATH`.
2. **Catalog cache.**
   - **At layout-ready:** take `fs.stat` of the bob path (path, mtimeMs, size) as a
     fingerprint, and load the persisted `{fingerprint, catalog}` from plugin data. If
     the fingerprint differs or is missing, run `bob __catalog --protocol 1`
     (`execFile`, 5 s, 8 MiB), validate it, and persist it.
   - **At sheet open:** synchronously re-stat. On a change, keep showing the cached
     catalog, refresh in the background, and re-render while keeping the selection.
     Opening the sheet never spawns a process.
   - **States:** `ready`, `missing-bob` (the paths tried plus **Open settings**),
     `old-bob` (no `__catalog`, so one "Update bob" row; surface rows still work), and
     `protocol-skew` (the side to update is named).
3. **Run files.** Each run keeps its files under
   `${XDG_STATE_HOME:-$HOME/.local/state}/bob-cli/command-mode/runs/`, in a directory
   with mode 0700 and files with mode 0600. The plugin is the only writer. `<id>` is
   `YYYYMMDD-HHMMSS-<4 random>`.

   | File                    | Contents                                             |
   | ----------------------- | ---------------------------------------------------- |
   | `<id>.out` / `<id>.err` | stdout / stderr                                      |
   | `<id>.json`             | the run record, written atomically (tmp + rename)    |
   | `<id>.rc`               | bob's receipt, when the `run-receipt` feature exists |

   The record holds `schema`, `id`, `argv`, `entry_id`, `title`, `effect`,
   `presentation`, `preview`, `trigger`, `bob_path`, `bob_version`, `host`,
   `started_at`, `finished_at`, `duration_ms`, `pid`, `exit_code`, `signal`, `outcome`,
   `summary`, `interleave`, and `truncated`.
   - `trigger` is one of `sheet`, `command`, `api`, or `retry`.
   - `outcome` is one of `queued`, `running`, `completed`, `attention`, `failed`,
     `unknown`, or `stopped`.
   - `interleave` is a list of `[stream, bytes]` chunk pairs, so a reopened run renders
     in arrival order.

   Retention keeps the newest 50 runs and anything younger than 14 days, never deleting
   an unfinished run. Both limits are settings.

4. **Spawn.** Call
   `spawn(bobPath, argv, { cwd: vaultBase, detached: true, stdio: ['ignore', outFd, errFd], env })`
   and `unref()` it, then close the fds in the parent. POSIX only: on `win32`, show "Bob
   command mode supports Linux and macOS". A non-filesystem vault refuses with "Command
   mode needs a local vault".
5. **Environment.**
   - Start from `process.env`.
   - Set `BOB_DIR` to the vault base path. Never fall back to `~/bob`.
   - Prepend the existing candidate directories to `PATH` (`~/.cargo/bin`,
     `~/.local/bin`, `/opt/homebrew/bin`, `/usr/local/bin`) so the Mac GUI finds `pass`
     and Homebrew tools.
   - Set `COLUMNS` to the pager width in monospace glyphs (default 100).
   - Drop `BOB_NOW`, `BOB_DAY_FILE`, and `BOB_COMPLETE_*`.
   - Keep `XDG_RUNTIME_DIR` (lock parity).
   - When the catalog advertises the features, set `CLICOLOR_FORCE=1`, `BOB_PROGRESS=1`,
     and `BOB_RUN_RECEIPT=<id>.rc`. An inherited `NO_COLOR` still wins. Without the
     color feature, set `NO_COLOR=1`.
6. **Tail.** Poll `fs.read` at offsets every 150 ms while a run is live. Decode each
   stream with its own streaming `TextDecoder`, so multi-byte characters split across
   chunks survive. Record `interleave`. Keep at most 4 MiB per stream in memory; the
   files keep everything, and the pager links to the full log.
7. **Writer guards.**
   - Any non-`reads` run passes these guards.
   - **Duplicate guard:** a duplicate argv (same argv and preview flag) that is already
     running or queued focuses the existing run instead of starting another. This makes
     a double `↵` one run.
   - **Writer queue:** writers queue FIFO, showing "queued behind `task reconcile`".
     Reads run in parallel, up to 4.
   - **Save guard:** right before a writer spawns, `await view.save()` every
     `MarkdownView` across all leaves (`iterateAllLeaves`, including pop-outs). A failed
     save refuses the run with "Could not save `<path>`; run refused". This follows the
     theme-rename precedent in `bob-navigation-hotkeys/src/203-…`.
   - Never inspect or take bob's lock; bob arbitrates it.
8. **Budgets.**
   - **Reads:** at `budget_secs`, send SIGTERM to the process group (`kill(-pid)`), then
     SIGKILL after 3 s. The outcome is failed, "timed out after N s".
   - **Writers and syncs:** never killed automatically. Past budget the run is marked
     `overdue`.
   - **Stop:** SIGTERM to the group, then SIGKILL after 5 s, for an outcome of
     `stopped`. Stop is offered only for runs this instance spawned.
9. **Finalization survives plugin reloads.**
   - The exit handler is self-contained. It reads the receipt, then the in-memory
     result, classifies the run, writes the record, and only then notifies the UI if the
     instance is still loaded. A plugin reload mid-run therefore still finalizes the
     record from the old closure.
   - On load, the instance adopts unfinished records. It polls each one's record and
     receipt every 1 s, and finalizes from the receipt when one appears, classifying
     from the files.
   - Without a receipt, a run becomes `unknown — log preserved` at
     `started_at + 2 × budget`.
   - PIDs are never trusted after a reload.
   - Runs found already finished at load are finalized silently. A run that failed while
     Obsidian was closed gets one summary notice.
10. **Generic classification** is a pure function, applied in this order:
    1. A spawn ENOENT is failed: "bob not found", with the paths tried.
    2. A run stopped by the user is stopped.
    3. A timeout is failed.
    4. A signal is failed.
    5. bob's lock phrase "another Bob vault maintenance run is already active" means
       attention on exit 0 and failed on non-zero, worded "Another bob maintenance run
       holds the lock", with **Retry**.
    6. A matching catalog `exit_codes` hint applies its outcome and meaning.
    7. The adapter, which arrives in `feedback`, can adjust the result.
    8. Otherwise exit 0 is completed, summarized by the last non-empty stdout line or
       "Done".
    9. Otherwise the run failed. Its summary is the first stderr line that starts with
       `error:`, else the first non-empty stderr line, else the last stdout line.

    ANSI is always stripped before classification.

11. **API.** The plugin exposes
    `api = { version: 1, commandMode: Object.freeze({ version: 1, run(argv, opts), openOutput(runId), onRunFinished(cb), catalog(), lastRun() }) }`.
    - `run` settles to `{ok, runId, outcome, exitCode, reason?}` and never rejects.
    - `opts` holds `presentation`, `title`, `entryId`, and `trigger`.
    - Every finish also fires
      `app.workspace.trigger("bob-command-mode:run-finished", {runId, argv, entryId, outcome, effect, invalidates})`.
      Other plugins can subscribe with `registerEvent`, whatever the load order.

## Phase details

Each phase lists its scope, key files, tests, and acceptance criteria. The _Repository
conventions_ section below applies to every phase.

### vault-prep

Edit `~/bob/obsidian_vimrc.md`, Bryan's vault vimrc:

- Change all 16 `nmap X :bob_*<CR>` lines to `nnoremap`: `-`, `[[`, `]]`, `!`,
  `[<Space>`, `]<Space>`, `<C-j>`, `<C-k>`, `\<`, `\>`, `\\`, `]s`, `[s`, `[S`, `]S`,
  and `Z`. Leave every `exmap` and `set` line unchanged.
- Add a header comment: "Every mapping that replays `:` must be nnoremap — `:` will open
  bob command mode and a recursive map would replay into it."
- Add `" g: always opens Vim's native ex line, even with bob command mode disabled.`
  followed by `nnoremap g: :`.
- **Do not bind `:` yet.**

Before relying on this, read Obsidian's bundled CodeMirror Vim if it is reachable
(`lib/codemirror/vim.js` inside `obsidian.asar`). Confirm that `nnoremap` is a built-in
ex command and that obsidian-vimrc-support passes the lines through unchanged. This is
findings F1–F3 of the research.

In the vault `.gitignore` custom-plugin block, add `.obsidian/plugins/bob-command-mode/`
so the per-host deploy, `data.json`, and frecency never sync. Verify with
`git check-ignore -v .obsidian/plugins/bob-command-mode/data.json`. The `bob-articles`
omission in that block is tracked separately as bead `bob-cli-cd`; do not fix it here.

Deliver both vault edits through the git vault sync workflow in
`docs/vault-git-sync.md`. Preserve unrelated vault changes.

**Acceptance:**

- The diff touches exactly the 16 mapping lines plus the additions.
- A manual smoke list is reported for Bryan: `-`, `]s`, `[s`, `[S`, `]S`, `Z`, `!`, and
  `\\` behave as before after a vimrc reload, and `g:` opens the ex line.
- Report that agents cannot drive the Obsidian UI; do not claim a UI pass.

### catalog

Work in bob-cli:

- Add a new module, `src/native/catalog/`, with the request parsing, the static table,
  and the JSON writer.
- Intercept the command in `run_bob_inner` (`src/runner.rs`) right beside `__complete`,
  before alias rewriting and clap. It is therefore absent from help, the docs catalog,
  and completion.
- Reuse the completion protocol's request-parsing style
  (`src/native/completion/protocol.rs`).
- Derive the leaf data from `crate::native::completion::tree()`. Call `.build()` first,
  skip hidden subcommands and `help`, and take positionals from the clap args.
- Script-backed leaves (`task archive`, `nightly`, `pomodoro status`) must resolve the
  same way. Check how `tree()` models them.
- Do not modify `completion/kinds.rs`; `completion-names` edits it in parallel.

**Unit tests:**

- Every `path` resolves to a public leaf.
- Every `preview` and `fixed` option exists on that leaf, as a long or short flag.
- Every public leaf outside the Capture protocol section is an entry or excluded.
  Capture-protocol leaves are always excluded.
- A path appears in both lists only for entries with non-empty `fixed` or a `url` slot
  (the capture rows).
- Ids are unique kebab-case, and keywords are lowercase.
- `confirm` policy implies confirm text, and `preview-first` implies `preview`.
- `reads` implies policy `run` and no preview.
- Each `section` equals its root's `SUBCOMMANDS` section.
- Every slot is from the known set.
- `aliases` resolve to entries or excluded paths.

**Integration tests** go in `tests/cli/catalog.rs`, through `support::bob_command()`:

- The output parses and has `schema_version` 1.
- `--protocol=1` and `--protocol 1` both work.
- `--protocol 2` gives the JSON error and exit 2; a missing `--protocol` exits 2.
- The output has no ANSI.
- The run succeeds with `BOB_DIR` pointing nowhere and leaves the XDG cache and state
  untouched, mirroring `completion_is_read_only`.
- It finishes well under 100 ms in a release build. Assert loosely or just measure and
  log.

**Docs:**

- Add a new contract page, `docs/command-catalog.md`, titled "Command catalog protocol".
  It covers the handshake, every field, the compatibility rules, the inclusion rule, and
  the coverage guarantee.
- Register it in `docs/_meta/contracts.yml` and add a one-line cross-link from
  `docs/completion.md` § Protocol 1.
- Run `just docs-gen` and `just docs-check`.

**Acceptance:**

- `bob __catalog --protocol 1 | jq '.entries | length'` reports the table above, minus
  the Pomodoro rows if `pomodoro_actions` is no.
- `just all` passes.

### run-env

Work in bob-cli:

- **`CLICOLOR_FORCE`.** A non-empty value other than `0` enables color without a TTY;
  any `NO_COLOR` still wins. Centralize the rule in one helper in `src/native/style.rs`.
  Route through it `Styler::detect`, root help (`runner.rs` near line 1224),
  `highlights_ref/cli.rs` near line 220, `nightly.rs` near line 312, and any other
  `is_terminal() && NO_COLOR` copy found by grep. Script-backed commands with their own
  color logic follow the same rule where feasible; otherwise note the gap in the docs.
- **`BOB_PROGRESS=1`.** Progress lines print on non-TTY stderr at every site gated on
  `stderr().is_terminal()`:
  - `highlights_ref/clip.rs` (two sites);
  - `highlights_ref/create.rs`;
  - `highlights_ref/collection.rs` (four sites);
  - `gkeep/ui.rs`.

  Progress must be line-oriented, with no `\r` redraws, when stderr is not a TTY.

- **`BOB_RUN_RECEIPT=<path>`.** When the top-level runner returns an exit code
  (including the quiet-EPIPE 0), bob writes
  `{"schema_version":1,"exit_code":N,"finished_at":"<RFC3339>"}` atomically: a temp file
  in the same directory, then a rename.
  - Write it in one place, `src/main.rs` or `run_bob`, so every command is covered,
    including script-backed ones.
  - A failed write prints one stderr warning and never changes the exit code.
  - There is no receipt on panic or signal; that absence is what makes "unknown" honest.
- Add `"clicolor-force"`, `"progress-env"`, and `"run-receipt"` to the catalog
  `features`.
- Document all three variables in `docs/reference/configuration.md`; the env lint
  requires it. Describe `features` in `docs/command-catalog.md`.

**Tests:**

- With non-TTY output and `CLICOLOR_FORCE=1`, a styled command emits ANSI. With
  `NO_COLOR` also set, it emits none. `CLICOLOR_FORCE=0` emits none.
- `BOB_PROGRESS=1` emits a progress line on piped stderr for one offline-reachable
  progress site. Pick a fixture path that touches no network, such as a local Markdown
  `ref create --dry-run`, if it reports progress.
- A receipt is written with exit 0, a usage error (2), and a failing command (1).
- An unwritable receipt path gives a warning and an unchanged exit code.

**Acceptance:** `just all` and `just docs-check` pass.

### completion-names

Work in bob-cli:

- Add `Kind::ThemeName`, wired in `kinds.rs`, `providers.rs`, and the vault-kind arm in
  `present.rs`, for `themes hide|unhide|color NAME` and `themes rename OLD`. `NEW` and
  `HUE` stay free text.
  - The value is `record.name`, the description is
    `N days · last used YYYY-MM-DD[ · hidden]`, and the group is `themes`.
  - `unhide` offers hidden themes only. The others offer non-hidden themes first.
- **Read-only requirement.** `pomodoro_themes::load_records` writes caches under
  `$XDG_CACHE_HOME/bob-cli/themes/` (`cache::store_known_keys`, `cache::store`, and
  `cache::store_completion`). Thread a no-store mode through the load path, so that
  completion may _read_ those caches but never writes them.
  - `tests/cli/completion/vault.rs::completion_is_read_only` must keep passing. Extend
    it to cover a theme completion.
  - Stay inside the 150 ms completion deadline. A cold cache that times out returns no
    candidates, as today.
- Wire `relations hide|unhide LABEL` to the existing `Kind::RelationLabel` provider.
  `unhide` offers hidden labels first.
- Update `docs/completion.md` "What completes". Remove the claim that theme names and
  relation hide/unhide arguments remain free text.

**Tests:** completion rows for each new slot, including the hidden filtering.

**Acceptance:** `just all` passes.

### runner

Create `plugins/bob-command-mode/` in bob-plugins as a fragments plugin, using
bob-articles as the template:

- **`manifest.json`:** `id` `bob-command-mode`, `name` "Bob Command Mode", `version`
  0.1.0, `minAppVersion` 1.8.7, `isDesktopOnly: true`, `author` "Bryan M Bugyi", and a
  one-line description.
- Add `src/fragments.json`, `styles.css`, the generated `main.js`, and
  `docs/ bob-command-mode.md` (created by `npm run docs:gen`).
- Add `PURPOSES` and `PAGE_DESCRIPTIONS` entries in `scripts/docs-inventory.mjs`, and
  the `docs/nav.yml` entry.
- Change the "seven plugins" prose to eight in `README.md`,
  `docs/develop/architecture.md`, and `docs/develop/deployment.md`.

Suggested fragments, each at most 1000 lines:

| Fragment        | Contents                                         |
| --------------- | ------------------------------------------------ |
| `010-load`      | requires and constants                           |
| `020-util`      | quote-aware formatting, ANSI strip, durations    |
| `030-bob-env`   | resolver and env                                 |
| `040-catalog`   | catalog cache                                    |
| `050-run-store` | records, receipts, retention, adoption           |
| `060-runner`    | spawn, tail, queue, guards, budgets, stop        |
| `070-outcomes`  | the generic classifier and adapter hook registry |
| `080-api`       | the API                                          |
| `090-plugin`    | lifecycle, commands, settings tab                |
| `990-exports`   | `module.exports` and `helpers` for tests         |

**Settings** (minimal):

- bob path override (empty means automatic);
- OS notifications (on);
- history: keep N runs (50) and D days (14).

Frecency also lives in `data.json`, which is host-local because `vault-prep` gitignores
the folder.

**Ctrl+P commands.** Register the `run-<entry-id>` commands from the catalog at load
(persisted catalog first, then refreshed). Their results go through a minimal notifier
interface, `notify(run)`, rendered as a plain `Notice` that `feedback` replaces.

Dynamically registered commands cannot be read by `scripts/docs-inventory.mjs`, which
requires literal ids. Register them through a helper call shape the scanner ignores, and
describe them in prose on the plugin page. Check that `npm run docs:inventory:check`
stays green.

**Tests:**

- Add `scripts/command-mode-harness.cjs`, a fake `obsidian` reusing
  `scripts/modal-harness.cjs` stubs.
- Add `scripts/fixtures/fake-bob.cjs`, an executable Node script (`#!/usr/bin/env node`,
  mode 755). It serves a fixture catalog and, driven by argv/env, sleeps, emits stdout
  and stderr (including a multi-byte character split across writes and more than 1 MiB
  of output), exits with any code or signal, writes `BOB_RUN_RECEIPT`, and echoes its
  env and argv for assertions.
- Add the `scripts/test-command-mode-*.cjs` files to the `package.json` `test` list.

Cover:

- resolver order;
- env assembly, including the feature gating and `NO_COLOR` precedence;
- the catalog cache states and the fingerprint refresh;
- argv assembly, including preview before fixed;
- literal args with spaces, quotes, `$()`, and backticks;
- detached file-backed runs;
- the interleave index;
- the duplicate guard (a double `↵` is one run);
- writer FIFO with parallel reads;
- the save-guard refusal;
- read timeout kills versus writer `overdue`;
- stop;
- receipt-based finalization after a simulated reload (a new instance adopting an
  unfinished record);
- `unknown` without a receipt;
- retention;
- every generic classification branch;
- API settlement (never rejects);
- the workspace event.

**Deploy:**

- Run `npm run build`, `npm test`, `npm run validate`, and `npm run docs:check`.
- Run
  `bob plugins sync --repo <bob-plugins path> --no-pull --plugin bob-command-mode --dry-run`,
  then the same command without `--dry-run`.
- Report that Bryan must enable **Bob Command Mode** once under Settings → Community
  plugins, and run `bob plugins sync` on his other hosts.

### feedback

- Add the toast card and status chip from the spec, as fragments `110-toasts` and
  `120-status-chip`, plus their CSS.
- Swap the runner's plain notifier for them.
- Add OS notifications: `new Notification(title, {body})` when the run took more than 15
  s and `document.hasFocus()` is false. Clicking one focuses the window and opens the
  run once the pager exists. Until `pager` lands, Output opens the `.out` file through
  Electron `shell.openPath`.
- Add `Bob: Rerun last command`.

**Adapters** live in fragment `130-adapters`. Each is a pure function
`(run, text) → {outcome?, summary?, actions?, carry?}`, registered by catalog `adapter`
id. Unknown ids are ignored.

| Adapter           | Behavior                                                                                                                                                                                                                                                                                                                                                                |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ref-create`      | A `queued: <path> (<reason>…)` line means attention with the reason, plus a **Scan intake** action that runs `ref-scan-intake`. An `ingested: … → <note>` line means completed, plus an **Open note** action.                                                                                                                                                           |
| `ref-scan`        | A human summary with `N held` means attention ("N PDF writes held"). For `--intake-only`, the "still queued" exit-1 result is attention; real errors stay failed. Verify both texts against bob's output.                                                                                                                                                               |
| `vault-sync`      | Exit 0 with empty stdout and stderr means attention: "Skipped — another bob maintenance run holds the lock". bob's `quiet_if_held` path is silent; confirm a normal run always logs. Conflict lines mean attention, plus **Open conflict log** (`_conflicts/sync_conflicts.md`).                                                                                        |
| `json-write`      | Parses `{ok, code, message}`. `unchanged` is attention, `lock_busy` is failed with **Retry**, `unknown_theme` is failed. A success summary names the result (for example "Hid DEEP WORK").                                                                                                                                                                              |
| `plugins-sync`    | Parses the JSON. Changed ids are those with any file action `created`, `updated`, or `forced`. A summary such as "Deployed 2: bob-ledger-tools, bob-command-mode" adds a **Reload N plugins** action: `app.plugins.disablePlugin(id)` then `enablePlugin(id)` for each, reloading `bob-command-mode` itself last, after its record is saved. Dirty skips are attention. |
| `task-reroll`     | On a preview, it extracts the seed and returns `carry: ["--seed", seed]` so Apply reproduces the preview. Verify where the dry run prints the seed (human or `-f json`).                                                                                                                                                                                                |
| `pomodoro-status` | Uses the single line as the summary. Empty output is completed with "No Pomodoro running".                                                                                                                                                                                                                                                                              |
| `capture-session` | Uses bob's first result line as the summary. "finish the current Pomodoro first" and other refusals stay failed, with bob's wording.                                                                                                                                                                                                                                    |
| `gkeep-pull`      | Zero notes pulled is completed with "Keep inbox already empty".                                                                                                                                                                                                                                                                                                         |

Projects-sync success also offers **Reconcile tasks**.

**Known-failure wording:**

- A dirty-paths error becomes "Commit pending edits first", plus **Sync vault, then
  retry**.
- `bob-not-found` lists the paths tried, plus **Open settings**.

**Visual review harness.** Add `scripts/preview-command-mode.cjs` and a template,
modeled on `scripts/preview-day-strip.cjs`. It renders the production DOM and
`styles.css` in headless Chrome (`google-chrome`) under an Obsidian baseline stylesheet,
in both light and dark themes. It covers:

- each toast class: completed, attention with an action, failed with stderr, unknown,
  and stopped;
- the chip: one run, two runs, and overdue.

Write PNGs to a temp directory, look at them, and iterate until they are calm,
consistent, and readable.

**Tests:** every adapter on fixture outputs, toast DOM and timing per class,
single-sticky-failure replacement, the chip lifecycle and 400 ms appearance, and the OS
notification conditions.

### pager

Build the output view as one component that renders inside the sheet's output stage and
can also open standalone. Until `sheet` exists, it is a sheet-shaped modal with only the
output stage. Suggested fragments are `140-ansi-and-links` and `150-pager`.

**Header:**

- a status pill (`● running 0:12`, `✓ completed · 1.2 s`, `◐ needs attention · exit 0`,
  `✗ failed · exit 1`, `? unknown`, `■ stopped`, or `⧗ queued behind …`);
- the argv as copyable monospace;
- the start time;
- a **Stop** button while running; a writer needs confirmation.

**Body:**

- `--font-monospace` at 13 px, line-height 1.45, `white-space: pre` with horizontal
  scroll, or wrapped with `w`.
- stderr chunks carry a faint red gutter tick.
- The view streams while live and replays `interleave` for finished runs.
- A run larger than the in-memory cap shows "View truncated — open the full log".
- A body that is a single JSON document is pretty-printed, with muted keys.

**ANSI SGR.** Support reset, bold, dim, italic, underline, 30–37 and 90–97 foreground,
40–47 background, `38;5;n`, and `38;2;r;g;b`. Map colors to theme variables
(`--color-red` and the others) so light and dark themes both work. Strip every other
escape sequence. Render text nodes and spans only: never `innerHTML`, and never
Markdown.

**Linkifier.** Link these forms:

- vault-relative `*.md`, optionally followed by `:LINE`, `#^id`, or `#heading`;
- `[[wikilink]]`, `[[wikilink|alias]]`, and `[[note#^id]]`;
- absolute paths inside the vault base, relativized.

Link only targets that resolve (`getAbstractFileByPath` or
`metadataCache.getFirstLinkpathDest`). A click closes the sheet and opens the target
with `openLinkText(link, "", newLeaf, {eState: {line}})`; Mod-click opens a new tab.

**Keys and wiring:**

- Implement the pager keys table.
- `[`/`]` step through history newest-first, and the header shows "3 of 50".
- The toasts' **Output**, the chip click, the OS notification click,
  `api.commandMode.openOutput`, and the new `Bob: Show last output` command all open
  this view.
- Extend the preview harness with running, completed (with ANSI and links), failed, and
  JSON pages.

**Tests:**

- ANSI parsing, including a sequence split across chunks;
- linkify positives and negatives, including non-existent paths;
- JSON pretty-printing;
- key handling;
- history stepping;
- copy formats: shell-quoted `Y`, ANSI-stripped `y`;
- stop confirmation for writers.

### sheet

Add the `Bob: Command mode` command (`open`) and the bottom sheet, as the list-stage,
confirm-card, and preview flows in the spec, with in-place growth into the pager for
reports and previews. Suggested fragments are `160-sheet-core`, `170-rank`,
`180-rows-and-sections`, `190-context`, and `200-flows`.

**Behavior:**

- **Positioning:** read the active leaf's container rect at open and set CSS variables.
  Lighten `.modal-bg` for this sheet only.
- **Context:** capture origin context, generate the For this note and From clipboard
  rows, and read the clipboard asynchronously. Clipboard rows are inserted after For
  this note without moving the selection.
- **Recent** comes from history. Durations come from history medians.
- Frecency updates are written after the sheet closes, so the list never reorders
  mid-session.
- Show the surface rows and the help row.
- Apply the row policy:
  - preview-first runs the preview into the pager, with `⌃↵ Apply`;
  - confirm shows the card;
  - `⌃↵` skips both.
- **Unavailable rows:** `↵` shows the reason in the footer message line.
- **Closing:** focus returns to the origin editor and cursor, deferred with `setTimeout`
  as in the picker core.
- **Missing-bob, old-bob, and skew states** render as single explanatory rows above the
  surface rows.

**Accessibility:**

- The input has `role="combobox"` with `aria-expanded`, `aria-controls`, and
  `aria-activedescendant`.
- The list has `role="listbox"`, and its rows have `role="option"` and `aria-selected`.
- A polite live region announces stage changes and final outcomes, never streamed lines.
- Focus is visible, and text scales.

**Performance:** the sheet opens in under 100 ms warm, filtering takes under 30 ms for
the whole catalog, and no process is spawned per keystroke.

**Tests:**

- ranking: exact beats recency, alias and keyword lookups (the keyword acceptance table
  above), AND across tokens, and stable ties;
- section assembly for each note type and clipboard state;
- the policy flows;
- `⌫` on empty closes, and `Esc` behavior per stage;
- the double-`↵` guard;
- the report → pager → `Esc` → list round trip with the query intact;
- unavailable rows.

Extend the preview harness with:

- the empty state on a ref note with a clipboard URL;
- a filtered list;
- the confirm card;
- the preview with Apply;
- missing bob.

Review the screenshots in light and dark and iterate on spacing, contrast, and
hierarchy.

### args-vim

**Argument stage.** Build the stage from the spec. Completions come from
`bob __complete bash --protocol 1 -- bob <path> <fixed> <tokens…>`, run with `execFile`
and a 1 s timeout, debounced by 60 ms, with stale replies discarded. Split rows on TAB
into `value⇥description⇥group⇥space`, and group them by the `group` column. Handle these
directives:

- `!files-in root glob`: vault files filtered by the glob, active note first;
- `!files` and `!dirs`: vault-relative files or folders;
- `!message`: a hint line under the input;
- `!prefix`: honored as in the zsh adapter.

The fixed args are passed to `__complete` too, so the protocol sees the real line.

**Tokenizer.** A pure function, `tokenize(text) → {tokens, partial, error}`:

- whitespace separates tokens;
- `'…'` is literal;
- `"…"` honors `\"` and `\\`;
- a backslash outside quotes escapes the next character.

There is no expansion of `$`, backticks, `~`, or globs. An unbalanced quote is an
`error` shown inline, and it blocks the run.

**Typed runs.** Tokenize the list-stage query, then apply `aliases`.

- When the longest prefix of tokens matching an entry path leaves more tokens (or
  flags), pin a row "Run `bob <argv>`". It inherits that entry's metadata, policy, and
  guards.
- An `excluded` path pins a disabled row that shows the reason and suggests the entries
  on that leaf.
- A first token in `roots` with an incomplete path shows only the fuzzy rows.

**Vim routing:**

- `::` switches to the VIM stage.
- Vim-shaped input pins "Run as Vim `:<query>`" first. The research defines Vim-shaped
  as input matching any of these:
  - it starts with a range or punctuation character (`% . $ ' , ; + - / ? & ~ ! <`) or a
    digit;
  - it matches `^(s|g|v|substitute|global|vglobal)\W`;
  - its first word is exactly a default CodeMirror Vim ex name
    (`w q wq x noh nohlsearch sort set normal reg registers marks undo redo delmarks`
    and so on), `obcommand`, `exmap`, or an `exmap` name read from the vimrc.
- In the VIM stage, the vimrc's `exmap` names appear as suggestion rows.
- `↵` closes the sheet, restores the origin editor, and calls `Vim.handleEx(cm, text)`.
  Get `cm` the way `bob-navigation-hotkeys/src/610-plugin-jumps-and-counted-keys.js`
  does for its `nohlsearch` call. With no origin editor, the Vim row is unavailable
  ("Open a note first").
- A count before `:` (`5:`) is lost; this is accepted.

**Vimrc lint.** On load, and when the vimrc file changes:

1. Read `vimrcFileName` from `<configDir>/plugins/obsidian-vimrc-support/data.json`
   through the vault adapter.
2. Read the vimrc and parse it for recursive `map` or `nmap` lines whose right-hand side
   contains `:`.
3. Show one notice per session naming the keys, for example: "`-` and 15 others replay
   `:`; change them to nnoremap so `:` command mode cannot capture them."

The `exmap` names from the same parse feed the Vim-shape rule.

**Bind `:`** in `~/bob/obsidian_vimrc.md`, after the `g:` line:

```vim
" `:` opens bob command mode; `::` inside it switches to Vim's ex line.
exmap bob_command obcommand bob-command-mode:open
nnoremap : :bob_command<CR>
```

Deliver it through vault git sync. If `bob-command-mode` is not in
`.obsidian/community-plugins.json`, keep the binding, since `g:` still works, but say so
prominently in the report.

**Tests:**

- the tokenizer (quotes, escapes, unbalanced input, literal `$()` and backticks);
- typed-run resolution (aliases, longest prefix, excluded refusal, inherited metadata);
- the argument stage with a fake `__complete` (grouping, each directive, debounce and
  stale drop, the required-arg hint);
- Vim-shape positives and negatives (`%s/a/b/g`, `42`, `noh`, `sort`, an exmap name,
  versus `sync`, `ref create`, `stats`);
- `handleEx` dispatch on a fake Vim;
- the lint parser.

Extend the preview harness with the argument stage (completions plus a ghost
placeholder) and the VIM stage.

### integrations

**bob-project-tasks.** In `snapshotActiveNote` (`plugins/bob-project-tasks/main.js`,
hand-edited), when
`app.plugins.plugins["bob-command-mode"]?.api?.commandMode?.version >= 1`, call
`run(["ref","create", path, ...(closed ? ["-C"] : [])], {presentation: "action", title: "Snapshot to Highlights", entryId: "ref-create", trigger: "api"})`
and return. Otherwise keep the legacy path byte for byte. Extend
`scripts/test-project-tasks-snapshot.cjs` for both paths and bump the version.

**bob-ledger-tools.**
`registerEvent(app.workspace.on("bob-command-mode:run-finished", …))`. Map `invalidates`
tags to the existing refresh or invalidate functions for these caches:

| Tag          | Ledger cache                                             |
| ------------ | -------------------------------------------------------- |
| `themes`     | theme catalog cache                                      |
| `relations`  | relation labels and relation index                       |
| `stats`      | stats data                                               |
| `references` | Reading Board                                            |
| `tasks`      | plan, freshness, and Today views (whatever caches exist) |

Ignore unknown tags. Add tests, bump the version, deploy both plugins with scoped
`bob plugins sync`, and update both plugin docs pages where behavior is visible.

### docs-land

**bob-cli:**

- Add `docs/guides/run-bob-from-obsidian.md`, a goal guide. It covers pressing `:`,
  filtering, running, reading output, preview and confirm, typed argv, Vim ex through
  `::` and `g:`, history, and troubleshooting:
  - bob not found or old bob;
  - `:` does nothing because the plugin is not enabled;
  - Mac `PATH`.
- Add it to the mkdocs nav and `docs/guides/index.md`. Cross-link it from
  `docs/start/obsidian.md` where keys are listed.

**bob-plugins:**

- Finish `docs/bob-command-mode.md`: keys per stage, settings, run-history location,
  `api.commandMode`, the workspace event, and the vimrc lines.
- Add the API row in `docs/develop/plugin-apis.md`.

**End-to-end check.** Run every repo's gates, then this checklist on a synthetic vault
with the real bob built from master. The UI items go to Bryan as a manual list.

- **Keyboard:**
  - insert-mode `:` stays text;
  - normal-mode `:` opens the sheet once;
  - `-`, `]s`, and `Z` still work;
  - `:%s` and `:42` work via routing;
  - visual `:` stays native;
  - `g:` opens the ex line.
- **Discovery:** every keyword in the acceptance table finds its rows, and no unlisted
  writer runs just because its name matched.
- **Execution:**
  - paths with spaces, quotes, `$()`, and backticks stay literal;
  - output over 1 MiB and split multi-byte characters stay intact.
- **Outcomes:** each of these is distinct and truthful:
  - exit 0 with held or queued work;
  - the check and no-op codes;
  - malformed JSON;
  - ENOENT;
  - a signal;
  - empty successful output;
  - a reload with no receipt (unknown).
- **Lifecycle:**
  - closing and reopening keeps a run;
  - a double `↵` is one run;
  - `plugins sync` with Reload never loses its own record.
- **Safety:** a save failure refuses the run; edits during preview and apply are
  handled; lock contention and a concurrent timer sync both behave.

> [!decision] decision_record When this is yes, add the decisions strand
> `colon-opens-bob-command-mode` through `/sase_memory_write`, then run
> `sase memory init`. It applies to the Bob vault and bob-plugins, and records that:
>
> - `:` in Vim normal mode opens bob command mode;
> - Vim's ex line stays reachable via `::`, Vim-shaped routing, and `g:`;
> - every vault vimrc mapping that replays `:` must be `nnoremap`.
>
> Cite the research report and this plan. Its rejected alternatives are a plugin
> `mapCommand` (the same recursion), the `\b` trigger, and a centred modal.

## Repository conventions (all phases)

- **Other repos.** Open bob-plugins with `sase repo open bob-plugins` and work only at
  the printed path.
- **bob-plugins build:**
  - Edit `src/` fragments only, never a generated `main.js`.
  - Run `npm run build`, `npm test`, `npm run validate`, and `npm run docs:check`.
  - Register each new test file in `package.json`.
  - Bump the touched plugin's manifest version.
  - Deploy with `bob plugins sync --repo <that path> --no-pull --plugin <id>` after a
    `--dry-run`, never with `--force`.
- **bob-cli:**
  - Run `just all`, plus `just docs-gen` and `just docs-check` when docs or help change.
  - Use Conventional Commits.
  - Never edit the `Cargo.toml` version or `CHANGELOG.md` (the daily release train).
  - Every change here is additive, so there is no `!`.
- **Vault edits.** Go through the git vault sync workflow (`docs/vault-git-sync.md`).
  Never re-enable Obsidian Sync.
- **Obsidian UI.** Agents cannot drive it. Report what was verified automatically and
  list the manual smoke steps; never claim a UI pass.
- **Help text.** CLI help changes follow the `cli_rules.md` memory note. `__catalog` is
  hidden like `__complete`, so it adds no public help.

## Risks

- **A `:` that does nothing.** If the plugin is disabled or not yet deployed on a host,
  `:` runs an `obcommand` that does nothing. `g:` is the safety valve, and the guide
  covers it.
- **Electron behavior differences** for detached children: the record-writing exit
  handler and receipt adoption cover reloads, and the `unknown` state covers the rest.
- **Private plugin reload API** (`disablePlugin`/`enablePlugin`): it is used only behind
  the explicit **Reload** button and is feature-detected.
- **Adapter text drift.** bob's human lines can change. Adapters key on stable
  substrings and JSON codes, and fall back to the generic classification.
