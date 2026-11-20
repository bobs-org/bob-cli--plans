---
tier: epic
title: "Command chips: one-gesture bob runs from Obsidian notes"
goal:
  "Typing `run: bob …` in an inline code span, or a `bob-run` fence, gives a quiet ▶
  chip. One click or ⌘↵ runs it through Bob Command Mode the way the sheet's Ctrl+Enter
  would. A chip holds one command or a whole `&&` sequence, and its run state shows in
  place. The copy-chip plugin becomes `bob-chips` and owns both kinds, and the GTD rows
  that hold bob commands become command chips."
decisions:
  gtd_ack_copy_chip:
    ask: Turn the Ack row's terminal-only `bob gcal ack -n 7` into a copy chip?
    default: true
    why:
      It needs a TTY, so it can't run as a chip; a ⧉ chip makes the terminal step one
      gesture
    answer: true
  memory_glossary_command_chip:
    ask: Add the glossary:command-chip strand drafted in the adopt phase?
    default: true
    memory:
      - glossary:command-chip
    requested:
      As a part of this change, we should add a new glossary term for "command chip".
    answer: true
  memory_glossary_copy_chip:
    ask: Correct glossary:copy-chip for the plugin rename and point it at Command Chip?
    default: true
    memory:
      - glossary:copy-chip
    corrects:
      The `bob-copy-chips` plugin renders the span as a quiet pill and the fence as a
      titled card.
    answer: true
  memory_decision_record:
    ask:
      Record the command-chip safety policy as
      decisions:command-chips-run-through-command-mode?
    default: false
    memory:
      - decisions:command-chips-run-through-command-mode
    answer: false
phases:
  - id: rename
    title: Rename bob-copy-chips to bob-chips with no behavior change
    depends_on: []
    size: medium
    description:
      "rename: move the copy-chip plugin to the id bob-chips. The phase includes a
      dual-listed transition and a double-load guard. It updates every touchpoint: the
      navigation C-row lookup, docs, chezmoi allowlist, vault enablement, the vimrc
      obcommand ids, and a bob-cli docs redirect. Copy-chip behavior and api.copyChips
      v2 stay byte-stable."
  - id: engine
    title: Command Mode chip engine (resolve, runText, openText, provenance)
    depends_on: []
    size: medium
    description:
      "engine: add feature-detected api.commandMode members. resolve(text) parses a chip
      payload into catalog-legal steps. runText runs it with sheet Ctrl+Enter semantics
      plus the confirm floor, and openText is the guarded path. Also add activeRun and
      onCatalogChanged, plus trigger/source provenance on run records. One activation
      planner is shared with the sheet so they can never drift."
  - id: grammar
    title: run:/bob-run grammar, command-chip cursor rule, and vectors
    depends_on:
      - rename
    size: medium
    description:
      "grammar: build the pure run: and bob-run grammar on the shared scanner, with a
      serializer, link spans, and the command-chip chipAt rule (under cursor, link,
      line, task). Publish the pure members of api.commandChips v1, pinned by JSON
      vectors and a round-trip test."
  - id: inline
    title: Inline command chips in Live Preview and Reading view
    depends_on:
      - grammar
      - engine
    size: large
    description:
      "inline: render run: spans as ▶ pills in Live Preview and Reading view. Click
      runs through runText and Shift+Click opens Command Mode. Live state comes from a
      store keyed by signature. Unavailable and copy-only faces cover no-run folders,
      mobile, and a missing Command Mode. Includes visual previews, perf, and stub-bob
      e2e."
  - id: fence
    title: The bob-run sequence card
    depends_on:
      - inline
    size: medium
    description:
      "fence: render bob-run fences as titled sequence cards. Each card has numbered
      monospace steps, a Run button, per-step live marks, unavailable and copy-only
      variants, and an armed state."
  - id: keymap
    title: ⌘↵ runs the command chip at the cursor, with an armed chip and keycap hint
    depends_on:
      - inline
    size: medium
    description:
      "keymap: compose a command-chip step into the existing Meta+Enter dispatcher in
      bob-navigation-hotkeys, add targetAt/runAtCursor and a palette command, and paint
      the armed chip and a muted ⌘↵ Run keycap from the live binding. The Gmail and
      Highlights hints yield, and every old ⌘↵ target is regression-tested."
  - id: author
    title: Mod+Shift+Y makes and edits command chips
    depends_on:
      - fence
    size: medium
    description:
      "author: extend the make-or-edit card with a Copy | Run kind row. Run drafts get
      live per-step validation from resolve, an Add-bob fix, and inline-or-fence
      serialization. Conversion defaults are predictable, and Alt+Click edits a command
      chip."
  - id: adopt
    title: GTD rows, guide, glossary, and coordination
    depends_on:
      - keymap
      - author
    size: medium
    description:
      "adopt: convert the open gtd_daily.md rows that hold bob commands (the Bank row
      only once the Mac's bob has -D), add the vimrc comment and the bob-cli guide
      section, apply the accepted memory decisions, and leave coordination notes on
      bob-cli-nc and bob-cli-nn."
proposed_by: bbugyi200.apollo.research.5k.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-20 05:59:31
status: wip
---

- **PROMPT:**
  [prompts/202611/command_chips.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/command_chips.md)

# Plan: Command chips

## Context

Bryan asked for a bob **command chip** in Obsidian. It should take its cues from the
copy chip (epic `bob-cli-ni`, `plan:202611/copy_chips.md`) and run `bob` commands,
including bulk commands, the way Command Mode's `Ctrl+Enter` runs them. `⌘↵` should run
the single command chip on the current line or, as a fallback, in the current task, and
it must be clear when that key is live. Existing recurring tasks that hold bob commands
should adopt chips, and the glossary gains a "command chip" term.

The consolidated research report designed this feature. Every phase worker reads it
first:
`sase artifact read research:202611/obsidian_command_chips/obsidian_command_chips.md "<why>"`.
**Bryan accepted every requirement in that report.** Its memory edits are the exception:
they happen only through the memory decisions above, except the command-chip glossary
term, which Bryan asked for. This plan carries out the report's design. Where it goes
further, the section "Design choices beyond the research" says so. Where this plan and
the report disagree, this plan wins.

Where the work lands:

- **bob-plugins** (open with `/sase_repo`; read its `AGENTS.md`):
  - `plugins/bob-copy-chips/` becomes `plugins/bob-chips/` (rename), which gains the
    command-chip kind (grammar, inline, fence, keymap, author);
  - `bob-command-mode` gains the chip engine (engine);
  - `bob-navigation-hotkeys` composes the chip step into `⌘↵`, and `bob-ledger-tools`'
    `⌘↵ Gmail` hint learns to yield (keymap).
- **chezmoi** (open with `/sase_repo`; read its `AGENTS.md`): the `plugins.sync`
  allowlist in `home/dot_config/bob/config.yml` (rename).
- **The Bob vault `~/bob`**: `.gitignore`, `.obsidian/community-plugins.json`,
  `obsidian_vimrc.md` (rename), and the open rows of `gtd_daily.md` (adopt). Follow the
  vault `AGENTS.md`: inspect `git status` first, and commit only your own files with
  `git add <files> && git commit`, never `/sase_git_commit`.
- **bob-cli** (this repo): a `mkdocs.yml` redirect (rename), the
  `docs/guides/run-bob-from-obsidian.md` section and glossary memory (adopt). bob-cli
  code does not change.

Out of scope: any bob CLI or catalog change, Bob Mac Capture
(`decisions:mac-capture-is-a-thin-client`), and the sheet accepting typed `a && b`
input.

## The product

### Syntax

| You type                                                                                       | You see (Live Preview, cursor elsewhere)                                                    |
| ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| `` `run: bob task reconcile` ``                                                                | ▶ `bob task reconcile` (monospace pill)                                                    |
| `` `run[Sync calendar and inboxes]: bob gcal sync -n 2 && bob gkeep pull && bob gmail sync` `` | ▶ Sync calendar and inboxes · 3 (text-font pill, faint step count)                         |
| `` `run: bob gcal sync -n 2 && bob gkeep pull` ``                                              | ▶ `bob gcal sync -n 2 › gkeep pull` (each later step's leading `bob` drawn as a faint `›`) |
| a ` ```bob-run Morning sync ` fence, one `bob …` per line                                      | a titled sequence card with numbered steps and a Run button                                 |
| `` `run: bob gcal synk` ``                                                                     | ⊘ dimmed, dashed: "bob has no command `gcal synk`"                                          |
| `` `run: ls -la` `` / `` `run: bob plan \| less` ``                                            | ⊘ "command chips run only bob" / "command chips run bob, not a shell — remove `\|`"         |

The inline grammar mirrors the copy chip's exactly. It is
`run [ "[" LABEL "]" ] ":" [ " " ] TEXT` inside a CommonMark code span, after the span's
own normalization: lowercase `run` comes first, labels are one line with `\]` and `\\`
escapes, longer backtick runs hold text that contains backticks, and `\|` stands for a
pipe in table cells. Every step starts with the literal word `bob`. The payload is
byte-for-byte what Command Mode's sequence **Copy** produces and what a terminal
accepts. Plain `` `bob …` `` spans (about 2,400 in the vault) stay inert forever: the
`run:` prefix is the explicit authoring act. With the plugin off, every chip still reads
as inline code or a fence.

Bulk means one chip with many steps: a quote-aware `&&` inline, or one step per
non-blank line in a `bob-run` fence. Three separate chips on a line are three actions,
not one bulk action.

### What a run does

| Chip contents                                     | Click or `⌘↵`                                                                                                                                                                                                           | Shift+Click                                                                      |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| One `report` command (reads)                      | Opens the sheet and streams its output, as `Ctrl+Enter` on that row does                                                                                                                                                | Same                                                                             |
| One `run` or `preview-first` writer               | Runs now (preview skipped): the usual outcome toast, status chip, and live chip state                                                                                                                                   | `preview-first`: the sheet opens at its preview. A `run` writer: the review card |
| One `confirm` command (`task archive`, `nightly`) | The sheet's confirm card: the **confirm floor**                                                                                                                                                                         | Same                                                                             |
| Several steps                                     | One Command Mode sequence: status chip `N/M`, halts on failed / unknown / stopped, continues on needs-attention, one summary card (Output / Copy / Rerun / Resume from k). A `confirm` step opens the review card first | The review card with every step and its guard notes                              |
| Already running                                   | Opens that run's live output; never a second run                                                                                                                                                                        | —                                                                                |

Runs keep every engine guard: save-all before writers, the single-writer queue, dedup,
host-local records, and toasts. Running a chip never completes, freshens, or advances
its task or the `]s` walk. `⌃↵` still completes the row.

### Where chips run, copy, or refuse

- **Run**: Live Preview, Source mode (via `⌘↵`), Reading view, embeds, hover previews,
  and Tasks / Dataview query results whose source note is known.
- **Copy only** (copy glyph, click copies the payload byte-exact, and the tooltip says
  why):
  - the source note is under `mail/`, `ref/`, `ref_history/`, `.bob/`, or `_conflicts/`.
    These hold text Bryan did not author, and this feature's own research report becomes
    a `ref/` note full of `run:` examples once ingested;
  - a query result whose source note cannot be resolved;
  - mobile, or Command Mode absent or disabled.
- **Refuse** (⊘, the click shows bob's reason, nothing runs): syntax errors, non-`bob`
  steps, shell operators, unknown or excluded commands, missing required arguments, and
  commands unavailable on this host.
- Rendering, hovering, scrolling, parsing, query rendering, and export never spawn bob.

### What `⌘↵` targets

`⌘↵` is the vault's existing `Meta+Enter` binding on
`bob-navigation-hotkeys:open-reference-pdf-in-highlights`, a dispatcher that already
chains Highlights → Gmail → open link in new tab. The editor's `Ctrl+Enter` (task
completion) is untouched. One pure rule decides the command-chip step. The dispatcher
and the hint both use it, so they never disagree:

1. The cursor is inside a `bob-run` fence: that fence.
2. A command chip under the cursor: that chip.
3. The cursor is on a link (Markdown link, wikilink or embed, autolink, bare URL): no
   chip. The existing chain opens the link.
4. One command chip on the cursor line: that chip. Two or more is ambiguous: "2 command
   chips on this line · move onto one". It refuses and never falls through.
5. The cursor line holds any link: no chip. The line belongs to the existing chain
   (Highlights, Gmail, link).
6. The only command chip in the cursor's task (its owned lines, never a nested
   subtask's, with a touched `bob-run` fence counting once): that chip. Two or more is
   ambiguous: "N command chips in this task · move onto one".
7. Otherwise the existing chain runs unchanged.

A target that resolves invalid or unavailable refuses with bob's reason and never falls
through. Copy chips never count, and `gy` never sees command chips: kinds never compete.
In a no-run context the whole step is skipped and `⌘↵` behaves exactly as today.

**Discoverability.** When `⌘↵` would run a chip, two coordinated cues appear:

- the targeted chip is **armed**: an accent border and an accent ▶, with no size
  change;
- a muted **`⌘↵ Run`** keycap capsule sits at the end of the cursor line. It reads
  `⌘↵ Run · <label>` when the target sits on another line of the task, and `⌘↵ Output`
  while the target runs.

The keycap uses the same component family and keycap convention as the shipped
`⌘↵ Gmail` hint, and its glyphs come from the live binding: a rebound or missing hotkey
never advertises a false key. Ambiguous, refused, and copy-only targets show no keycap;
a refusal is never advertised. Only one `⌘↵` hint shows at a time: the Gmail capsule and
the Highlights status hint yield whenever the command-chip rule claims the line. Chip
tooltips also say "Click or ⌘↵ to run".

### Look

Command chips share the copy chip's geometry: a hairline pill, padding `0 .45em`, font
size `.92em`, inherited line height, `--radius-s`, `box-decoration-break: clone`, and a
fixed-width glyph slot. Like copy chips, they wrap and never truncate. Theme variables
only: no hex values and no per-chip colors. Two differences mark a verb rather than a
noun:

- a faint accent wash at rest:
  - background
    `color-mix(in srgb, var(--interactive-accent) 7%, var(--background-secondary))`;
  - border
    `color-mix(in srgb, var(--interactive-accent) 25%, var(--background-modifier-border))`;
- a Lucide `play` glyph in `--text-faint`, accent on hover, with `cursor: pointer`.

**The glyph always tells you what a click will do.** No state change ever reflows the
line.

| State                          | Glyph and ink                                                                                                                                                                                 | Click                         |
| ------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------- |
| Ready                          | ▶ faint, accent on hover                                                                                                                                                                     | Run                           |
| Armed (`⌘↵` target)            | ▶ accent, accent border                                                                                                                                                                      | Run                           |
| Queued (writer waits its turn) | `hourglass`, dashed accent border                                                                                                                                                             | Open live output              |
| Running                        | `loader-circle` spinning; sequences add a 2 px segmented underline (done green, current accent, upcoming faint) drawn as a background gradient, and the unlabeled face tints the current step | Open live output              |
| Completed                      | green `check` for about 1.6 s plus one `--bob-yank-flash-bg` pulse, then Ready                                                                                                                | Run                           |
| Needs attention                | ▶ in `--color-orange`, orange border, kept until the next run in this session                                                                                                                | Run again                     |
| Failed                         | ▶ in `--text-error`, error border, kept until the next run; the tooltip carries the first error line                                                                                         | Run again                     |
| Unavailable                    | `ban`, `--text-muted` text, dashed border                                                                                                                                                     | Notice with bob's reason      |
| Copy only                      | `copy`                                                                                                                                                                                        | Copy, with copy-chip feedback |

Reduced motion keeps the glyph changes and drops the spin and pulse. A shared polite
live region announces "Running …", "… finished", and "… failed". Reading controls are
real `<button type="button">`s named "Run" plus the face via `aria-labelledby`. Print
and PDF export show plain inline code. The tooltip, as DOM text, gives the full command
line(s), then one policy line ("Runs now · skips its preview", "Asks first: <confirm
text>", or "Opens its output"), then "Click or ⌘↵ to run · ⇧Click opens Command Mode"
(the author phase appends "· ⌥Click edits").

The walk row, cursor on it, in Live Preview:

```text
☐ Run ▶ Sync calendar and inboxes · 3                              ⌘↵ Run
       ╰ armed: accent border + accent ▶            muted keycap at line end ╯
```

The running chip, step 2 of 3:

```text
☐ Run ⟳ Sync calendar and inboxes · 3
        ▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔▔
        done    current    upcoming   (2 px underline, no reflow)
```

The `bob-run` card (fence phase):

```text
╭──────────────────────────────────────────────────────────────╮
│ ▶  Morning sync                    3 steps · writes  [ Run ] │
├──────────────────────────────────────────────────────────────┤
│  1  bob gcal sync -n 2                                       │
│  2  bob gkeep pull                                           │
│  3  bob gmail sync                                           │
╰──────────────────────────────────────────────────────────────╯
  running:  ⟳ header · "2 of 3" · [ Output ]   ✓ 1 · ⟳ 2 · ○ 3
  failed:   "stopped at 2 · failed"            ✓ 1 · ✗ 2 <reason> · Output · – 3 skipped
```

The make-or-edit card in Run kind (author phase):

```text
╭─ ▶ COMMAND CHIP ─────────────────────────────────────────────────╮
│ Kind      [ Copy | ▶ Run ]                                       │
│ Label     [ Sync calendar and inboxes                          ] │
│ Command   [ bob gcal sync -n 2 && bob gkeep pull && bob gmail… ] │
│ Steps     ✓ Sync Google Calendar          writes · preview skipped│
│           ✓ Pull Google Keep              writes · preview skipped│
│           ✓ Sync Gmail                    writes · preview skipped│
│ Preview   ▶ Sync calendar and inboxes · 3            inline chip  │
│ Markdown  `run[Sync calendar and inboxes]: bob gcal sync -n 2 …`  │
╰─ ⏎ save · ⇧⏎ new step · Tab next field · Esc cancel ─────────────╯
```

### Every way in

| Gesture                                                            | Where                                                             | Effect                                                                             |
| ------------------------------------------------------------------ | ----------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Click / tap / Enter / Space on a chip                              | Live Preview, Reading view, embeds, hover previews, query results | Run (or copy, in copy-only contexts)                                               |
| Shift+Click / Shift+Enter                                          | Same                                                              | Open in Command Mode (guarded path)                                                |
| `⌘↵` (Meta+Enter)                                                  | Editor, Live Preview or Source mode, any Vim mode                 | Run the targeted command chip, else the existing chain                             |
| Palette **Run command chip at cursor** (`bob-chips:run-at-cursor`) | Editor                                                            | Same target rule. No default hotkey: it covers hosts where Super+Enter is captured |
| Run button                                                         | `bob-run` card                                                    | Run the sequence; Shift+Click opens Command Mode                                   |
| `Mod+Shift+Y` (**Make or edit chip**) / Alt+Click                  | Editor / Live Preview chip                                        | Make, convert, or edit either kind                                                 |

## Design choices beyond the research

1. **Rename now, dual-listed.** The id becomes `bob-chips`, as the research's lead call
   recommends and as Bryan leaned. Plugin folders are per-host and gitignored, and the
   Mac is reachable only on a best-effort basis. So the vault enables **both** ids
   during the transition, and `bob-chips` runtime-disables a loaded `bob-copy-chips`
   (never saving the enabled list). No host loses chips before its next
   `bob plugins sync`. Retiring the old id is a recorded follow-up.
2. **A link rule in the target resolver** (rules 3 and 5). The research ordered "cursor
   on a link" first but let the task fallback beat the line's own Highlights or Gmail
   target. Rule 5 keeps the task fallback off any line that has a link, so a line's own
   target always wins. The rule stays pure and synchronous, so the hint can be exactly
   truthful.
3. **One activation planner.** The engine extracts the sheet's row-activation branch
   (`activateRow` with `force`) into one pure function. The sheet and `runText` both
   call it, so chip runs cannot drift from `Ctrl+Enter`. The only intended difference is
   the confirm floor.
4. **Run state is session-scoped and chip-started.** The store mirrors runs started
   through `runText`, keyed by the resolved signature, and it outlives widget rebuilds.
   Sheet runs of the same argv are still deduped by the engine. Needs-attention and
   failed ink last until the next run or reload, and nothing reruns by itself.
5. **Query results check the real source note.** A chip rendered inside a Tasks or
   Dataview query runs only when its source note resolves outside the no-run folders. An
   unknown source is copy-only. This closes the gap where a dashboard could turn
   untrusted `ref/` text into a button.
6. **A faint accent wash and a segmented underline** mark command chips as actions
   without loud buttons. Sequence progress reads at a glance with zero reflow.
7. **No new Vim mapping.** `⌘↵` already works in every Vim mode, and the palette command
   covers hosts that capture Super+Enter. Writers stay behind a deliberate chord. The
   vimrc gains a comment line only.
8. **The sync row is labeled.** "Sync calendar and inboxes · 3" keeps the walk row
   short, and hovering shows all three commands. `task reconcile` stays unlabeled,
   because its literal face is short and clear.

## Reliability contract

Copy-chip rules R1–R14 carry over (payloads never in `innerHTML` or attributes, a drag
never activates, rendering never writes, and so on). Command chips add:

- **C1** Only `bob-command-mode` tokenizes argv and spawns bob. There is no shell, and
  the program is always the resolved bob binary, never a word from the note.
- **C2** The whole chip is validated before step 1 and re-resolved at activation. A
  widget whose source text no longer matches refreshes and needs a fresh gesture. An
  accepted run executes frozen argv, even if step 1 rewrites the note.
- **C3** Authored text can never choose `entryId`, `effect`, `policy`, `presentation`,
  `skipGuards`, `preview`, or a binary path. `skipGuards` comes only from `runText`'s
  documented `Ctrl+Enter` semantics. `confirm` commands always ask.
- **C4** Run state lives in a store keyed by signature, not in widgets, so it survives
  decoration rebuilds and notes that rewrite themselves mid-run.
- **C5** Duplicate activation opens the live run. A synchronous latch (≥ 400 ms) plus
  the engine's dedup swallows double clicks and key repeats.
- **C6** Partial success is honest: completed steps stay committed, and Resume and Rerun
  stay explicit. Nothing auto-retries, including after a reload.
- **C7** No-run folders are judged by the source note's path, including embeds, hover
  previews, and query results. An unknown source is copy-only.
- **C8** Hints appear only when `⌘↵` would run that chip, with keycaps from the live
  binding, and one `⌘↵` hint shows at a time.
- **C9** Rendering, parsing, hovering, query rendering, and export never spawn bob or
  write the vault.
- **C10** Running never touches the chip's task, its status, freshness, or the walk
  (`decisions:task-status-is-derived`, `decisions:answering-advances-the-walk`).
- **C11** Copy chips' behavior and `api.copyChips` v2 stay byte-stable. Kinds never
  compete.
- **C12** The docs say plainly that vault text is an attack surface. They list the
  bounds: explicit prefix, catalog validation, confirm floor, no shell, no-run folders,
  and desktop-only execution.

## Rules for every bob-plugins phase

- **Build.** Edit only `src/` fragments and keep each at or under 1000 lines (split
  before you cross it). Run `npm run build`, `npm test`, `npm run validate`,
  `npm run docs:gen`, and `npm run docs:check`. Never hand-edit a generated `main.js`.
- **Versions.** Bump the manifest version of every plugin you change: a minor bump for
  new behavior. `bob-chips` starts at `0.6.0` in rename.
- **Parallel phases.** `fence` and `keymap` (and then `author` and `keymap`) may land
  concurrently in `bob-chips`. After rebasing, regenerate generated files (`main.js`,
  the docs inventory) rather than hand-merging them. Bump the version relative to what
  master holds when you land.
- **Tests.** Add every new `node --test` script to `test:unit` or `test:perf` in
  `package.json`. Pin pure behavior with JSON vectors under `scripts/fixtures/`.
- **Deploy.** Commit in bob-plugins, then run `bob plugins sync`. On a refusal, follow
  bob-plugins `AGENTS.md`; never pass `-F` on your own. Ask Bryan to reload Obsidian.
- **Real-Obsidian e2e.** Follow the CDP harness pattern of
  `scripts/obsidian-copy-chips-e2e.cjs` (its `-lib.cjs` and offline self-test wrapper)
  and `scripts/obsidian-command-mode-tab-e2e.cjs`. Run it in an isolated scratch vault
  under `xvfb-run -a`, with a stub `bob` on `PATH` that answers `__catalog` from a
  fixture and records its argv. Never run it against the live vault. Run it on athena
  when this host lacks Obsidian or xvfb, and if neither host can, say so in the phase
  note.
- **Visual review.** UI phases extend a Chromium preview script (precedent:
  `scripts/preview-command-mode.cjs`) that renders every state from production fragments
  and styles in light and dark themes. Attach the PNGs with `/sase_artifact`.
- **Decision records to honor**: `decisions:q-and-ctrl-bracket-mirror-esc` (Esc, `q`,
  and `Ctrl+[` close cards; `q` types inside text fields),
  `decisions:task-status-is-derived`, and `decisions:answering-advances-the-walk`.

## Phase: rename — `bob-copy-chips` → `bob-chips`, no behavior change

1. **bob-plugins.**
   - `git mv plugins/bob-copy-chips plugins/bob-chips`. Set the manifest to id
     `bob-chips`, name "Bob Chips", and version `0.6.0`, and keep the description about
     copy chips for now. Internal names (`copyChip*`, `.bob-copy-chip*` classes, test
     file names) stay, so this phase stays behavior-neutral.
   - **Double-load guard.** On `workspace.onLayoutReady`, if
     `app.plugins.plugins["bob-copy-chips"]` is loaded, call
     `app.plugins.disablePlugin("bob-copy-chips")`. Never call `disablePluginAndSave`,
     never write `community-plugins.json`, and log one `console.info`. Unit-test it with
     a fake `app.plugins`.
   - `bob-navigation-hotkeys` (`641-plugin-copy-card.js` and anything else the grep
     finds): look up `plugins["bob-chips"] ?? plugins["bob-copy-chips"]`. Bump its
     version.
   - Repoint every script, helper, perf test, and e2e harness from
     `plugins/bob-copy-chips` to `plugins/bob-chips`
     (`rg -l bob-copy-chips scripts plugins docs package.json`).
   - Docs: `git mv docs/bob-copy-chips.md docs/bob-chips.md` and retitle it "Bob Chips".
     Update `docs/nav.yml`, the generated `docs/index.md`, and every cross-link
     (`copy-card.md`, `bob-navigation-hotkeys.md`, `develop/plugin-apis.md`). The API
     section becomes "Bob Chips", shows `plugins["bob-chips"]`, and notes the retired id
     with the dual lookup.
2. **chezmoi.** In `home/dot_config/bob/config.yml` `plugins.sync`, replace
   `bob-copy-chips` with `bob-chips`. Apply it on this host the way the chezmoi repo's
   `AGENTS.md` prescribes, so `~/.config/bob/config.yml` matches before you sync.
3. **Vault.**
   - `.gitignore`: add `.obsidian/plugins/bob-chips/` beside the existing
     `bob-copy-chips` line, and keep that line, because stale folders must stay ignored.
   - `.obsidian/community-plugins.json`: add `"bob-chips"` right after
     `"bob-copy-chips"`. Keep both (dual listing, design choice 1).
   - `obsidian_vimrc.md`: change the two `obcommand bob-copy-chips:` ids to `bob-chips:`
     and leave the comments as they are.
4. **bob-cli.** Add
   `reference/plugins/bob-copy-chips.md: reference/plugins/bob-chips.md` to `mkdocs.yml`
   `redirect_maps`, then run `just docs-check`.
5. **Memory** (accepted decisions only).

   > [!decision] memory_glossary_copy_chip Run `/sase_memory_write`, then in
   > `glossary:copy-chip` change "The `bob-copy-chips` plugin" to "The `bob-chips`
   > plugin" and "See `docs/bob-copy-chips.md`." to "See `docs/bob-chips.md`.". Change
   > nothing else in this phase; adopt adds the Command Chip cross-link. Run
   > `sase memory init` and `just docs-gen`, and commit the regenerated
   > `docs/reference/glossary.md`.

6. **Verify and deploy.** Commit bob-plugins, then run `bob plugins sync`.
   `bob plugins list` must show `bob-chips` synced and enabled. The full copy-chips
   unit, perf, and e2e suites pass unchanged against the new id. Without memory
   acceptance, a `rg bob-copy-chips` over bob-plugins, the vault config files, and
   bob-cli `docs/` and `mkdocs.yml` leaves only the dual lookup, the guard, the
   redirect, the `.gitignore` line, the enabled-list entry, and the API note. With the
   copy-chip memory decision accepted, the glossary hits are gone too.
7. **Report.** Ask Bryan to reload Obsidian. Each other desktop host renders chips
   through the old folder until it runs `chezmoi update` and `bob plugins sync`. Record
   a `PROPOSED FOLLOW-UP:` note: `bob plugins sync` has no notion of retired ids, so the
   stale `bob-copy-chips` folders and enabled-list entry should be retired once every
   host deploys `bob-chips`.

## Phase: engine — `api.commandMode` chip members

All work is in `plugins/bob-command-mode`. `api.commandMode` keeps `version: 1`. New
members are feature-detected with `typeof`, following the `preview` precedent. Every
member is frozen and total: it never throws and never rejects.

1. **`resolve(text)`**, pure and synchronous over the cached catalog:
   - Split into steps at unquoted `&&` and at line breaks, skipping blank lines. Quotes
     and escapes follow `cmTokenizeBobArgv` exactly.
   - Tokenize each step with `cmTokenizeBobArgv`. The first word must be exactly `bob`
     (not `/usr/bin/bob`, `./bob`, `sudo bob`, or `X=1 bob`); strip it. Apply
     `cmApplyCatalogAliases`, then check exclusions (`cmExcludedReason`, reason
     verbatim), the entry (`cmFindEntryForArgv`), required positionals, and host
     availability (`cmSheetEntryUnavailable`). An exact catalog path with no arguments
     (`bob task reconcile`) resolves too, unlike the sheet's typed pin.
   - Refuse unquoted shell syntax with "command chips run bob, not a shell — remove
     `<op>`": `;`, `|`, `||`, a lone `&`, `>`, `>>`, `<`, `$(`, `${`, `$NAME`, and a
     backtick. Also refuse empty steps and leading or trailing `&&`.
   - Never expand `~` or globs; bob does that itself where it chooses (`bank import`).
   - Return frozen `{ state, signature, steps, reason, step }`:
     - `state` is `"ready"`, `"invalid"` (syntax, shell, not bob, unknown, excluded, or
       a missing argument), `"unavailable"` (this host), or `"loading"` (no catalog
       yet).
     - Each step is
       `{ index, text, argv, entryId, title, effect, policy, presentation, hasPreview, confirm }`,
       where `text` is the trimmed source substring used for faces.
     - `signature` is `cmSequenceSignature` over the steps, for any length.
     - `reason` and `step` name the first failure.
   - Messages: "bob has no command `gcal synk`", "needs `<arg>`", "empty step after
     `&&`", "command chips run only bob", "not available on this host: <reason>",
     "Command Mode is still loading bob's catalog".
2. **One activation planner.** Extract the decision `activateRow(row, { force: true })`
   makes into a pure `cmActivationPlan(row, { force, confirmFloor })` that returns
   `confirm | report | inline | run`. Use it in the sheet, and keep the sheet's behavior
   identical: the existing sheet suites pass unchanged.
3. **`runText(text, opts)`**, with `opts` = `{ source: { path, line }, onState }`. It
   ignores every other option (C3), and `trigger` is always `"chip"`.
   - Re-resolve now. If the catalog is still loading, wait up to 5 s for the in-flight
     load. A result that is not ready settles
     `{ ok: false, outcome: "refused", reason }` without spawning.
   - If the signature is active (`activeRun`), open that run's live output and settle
     `{ ok: true, stage: "output", duplicate: true, runId }`.
   - One step, planned by `cmActivationPlan` with the confirm floor:
     - `confirm` opens the sheet's confirm card (`stage: "confirm"`);
     - `report` opens the sheet and streams the output stage (`stage: "output"`);
     - otherwise call
       `runner.run(argv, { entryId, title, trigger: "chip", source, skipGuards: true })`,
       with the same argv and guards the sheet's `Ctrl+Enter` passes.
   - Several steps:
     - any `confirm` step opens the sheet's review card (`cmSheetSequenceShowReview`
       with `cmSequenceGuards`; `stage: "review"`);
     - otherwise call `commandModeRunSequence(steps, { trigger: "chip", source, … })`.
   - `onState` receives
     `{ state: "queued" | "running" | "step" | "finished", runId, step, total, outcome }`.
     Add an internal `onState` hook to `runner.run` for the queued → running transition.
     Wrap every callback; a throwing callback never affects the run.
   - Settle `{ ok, outcome, stage, runId, sequenceId, reason }`.
4. **`openText(text, opts)`**, the guarded path. One step with a preview opens the sheet
   at that preview (the existing `preview` path). One report runs as in `runText`.
   Anything else opens the review card listing every step with its effect and guard
   notes; Enter runs it and Esc backs out.
5. **`activeRun(signature)`** returns frozen
   `{ kind: "run" | "sequence", state, runId, sequenceId, step, total }` or `null`. It
   covers `runText` runs and sheet sequences (`commandModeSequenceIsActive`).
   **`onCatalogChanged(callback)`** fires after the first load and on every fingerprint
   change, and returns an unsubscribe function.
6. **Provenance.** Run records and sequence stamps gain `trigger: "chip"` and
   `source: { path, line }`. Keep `path` only if it is a vault-relative string of at
   most 512 characters, and `line` only if it is a non-negative integer; otherwise drop
   the field. The run-finished event carries both. The pager header and the RECENT row's
   meta line show `▶ <note basename>` for chip runs.
7. **Tests.**
   - `scripts/fixtures/command-mode-resolve-vectors.json` over a pinned catalog fixture
     must cover: quote-aware `&&` (`bob task add "a && b"` is one step), escapes,
     newlines, every shell refusal, the bob rules, aliases, exclusions, unknown
     commands, missing positionals, host-unavailable commands, `loading`, and signature
     equality with the sheet's sequences.
   - Parity: for `run`, `preview-first`, `report`, and `inline` rows, `runText` reaches
     `runner.run` or the sheet with exactly the options the sheet's `Ctrl+Enter` uses,
     plus `trigger` and `source`.
   - A `confirm` command or a sequence with a `confirm` step never spawns before
     confirmation.
   - Duplicate activation opens output and never spawns twice. A refusal never spawns. A
     throwing `onState` never breaks the run.
   - Both `openText` branches.
   - Bad provenance is dropped.
8. **Docs.** `docs/bob-command-mode.md` gains "Runs from command chips", including the
   confirm floor and the attack-surface note (C12). `docs/develop/plugin-apis.md`
   documents the new members and the existing `run` options `trigger` and `skipGuards`.
   Bump the version to `0.3.0`.

## Phase: grammar — `run:` / `bob-run` grammar and the command-chip cursor rule

All work is pure code in `plugins/bob-chips` with no Obsidian imports and no engine
calls.

1. **Parser.** `runChipParseContent(content)` mirrors `copyChipParseContent`, with `run`
   as the lowercase prefix. It returns `{ label, text }` or `null`.
2. **One scanning pass, two kinds.** Add `chipScanLine(line, opts)`, which returns
   kind-tagged `{ kind: "copy" | "run", label, text, from, to }`. It shares the
   code-span finder and every exclusion: links, wikilinks, math, comments, physical
   multiline spans, and table pipes. Reimplement `copyChipScanLine` as the copy-kind
   filter, and keep its output byte-identical (the existing copy vectors must pass
   untouched).
3. **Fences.** `runChipParseFenceInfo(info)` accepts the language token `bob-run` plus
   an optional label, mirroring `bob-copy`. The body is LF-joined and verbatim. Fence
   detection reuses the shared list-nested backtick and tilde handling.
4. **Link spans.** `chipLinkSpans(line)` returns the `[from, to)` ranges of Markdown
   links, wikilinks and embeds, autolinks, and bare URLs, outside code spans. It backs
   resolver rules 3 and 5.
5. **`commandChipAt({ lines, line, ch, scope? })`** implements the `⌘↵` rule (rules 1–7
   above). The task step takes the same opt-in `{ kind: "task", lines }` scope copy
   chips use, from `bob-navigation-hotkeys` `api.taskScope`. It returns frozen results:
   - `{ state: "none", count: 0 }`;
   - `{ state: "ambiguous", count, via: "line" | "task", message }`;
   - `{ state: "ready", count, via: "fence" | "cursor" | "line" | "task", chip }`, where
     `chip` is `{ form: "inline" | "block", label, text, line, from, to }`.

   Copy chips never count, and `copyChipAt` stays unchanged.

6. **Serializer.** `runChipSerialize({ label, text }, { table })`: text with a line
   break becomes a `bob-run` fence; otherwise an inline span with the shortest safe
   backtick run, label escapes, and `\|` in tables. A block in a table cell is refused.
7. **`api.commandChips` v1** (frozen, never throws): `version`, `parse`, `scanLine` (run
   kind only), `parseFenceInfo`, `fencePayload`, `serialize`, `chipAt`. Keymap later
   adds `targetAt` and `runAtCursor`, detected with `typeof`. `api.copyChips` stays v2.
8. **Tests.**
   - `scripts/fixtures/command-chips-vectors.json` mirrors the copy vectors for `run`
     and adds mixed-kind lines.
   - `chipAt` vectors cover each rule: link under the cursor, a line with a link, a task
     fallback blocked by a link, fences, nested tasks, frontmatter, and ambiguous lines
     and tasks.
   - Add a serialize → parse round-trip property test and a check that copy vectors are
     unchanged. Document the grammar in `docs/bob-chips.md`, and the API in
     `docs/develop/plugin-apis.md`.

## Phase: inline — Live Preview and Reading view command chips

1. **Rendering.**
   - Live Preview: extend the copy-chip decoration builder so the one scanning pass also
     emits command-chip widgets (`.bob-run-chip`). Widgets reveal source when the cursor
     enters the span, like copy chips.
   - Reading view: the post-processor turns command spans into real `<button>`s. This
     covers embeds, hover previews, and query results.
   - Faces follow "Syntax". Unlabeled faces use `resolve` step texts with faint `›`
     separators; without a resolve result (copy-only or loading) the payload shows
     verbatim. Labeled faces add a faint tabular `· N` when N > 1.
   - Update the manifest description to name both kinds.
2. **The run path** `commandChipActivate(chip, ctx)`, where ctx holds the source path,
   line, and view or element:
   - check the no-run folders against the source path (C7);
   - check that Command Mode is present (`typeof api.commandMode.runText`);
   - stale-check the source text: in Live Preview, `sliceDoc` at the widget range; in
     Reading view, section info where available;
   - take the latch;
   - call `runText` with `source` and an `onState` that feeds the store.

   A `confirm`, `review`, or `output` stage ends without a running state. Shift+Click
   and Shift+Enter call `openText`. Alt+Click does nothing here; the author phase wires
   it.

3. **State store** `CommandChipRunStore`: keyed by signature, holding state, `runId`,
   step, total, and outcome. Mounted widgets subscribe and repaint classes in place, so
   the spinner never restarts. A rebuilt widget reads the store, so a mid-run rebuild
   still shows running. Completed returns to Ready after about 1.6 s; attention and
   failed persist for the session. It also holds the `armed` target that keymap sets and
   that widgets render as `.is-armed`.
4. **Faces for every state** in the "Look" table, plus Loading (▶ faint;
   `onCatalogChanged` re-renders). Unavailable clicks show one Notice with the reason.
   Copy-only clicks reuse `copyChipCopyText` and the copy feedback.
5. **Query results.** Resolve the real source note from the rendered result: Tasks links
   each task to its file, and Dataview passes the file. Run only when it resolves
   outside the no-run folders; otherwise render copy-only with "source note unknown".
   Verify the exact DOM hooks against the installed Tasks and Dataview versions.
6. **Tests and evidence.**
   - DOM tests for each state, both kinds on one line, Reading buttons and accessible
     names, drag and secondary-button suppression, the latch (a double click within 400
     ms makes one `runText` call), and Shift+Click.
   - No-run folders through each path: note, embed, hover, and query. Copy-only when
     Command Mode is absent. An unavailable click never calls `runText`.
   - A store test for the self-rewriting note (the widget is rebuilt mid-run).
   - Extend the Live Preview perf test: a 2,000-line note with mixed chips stays within
     the existing copy-chip budget, and `resolve` results are cached per catalog
     fingerprint and text.
   - Stub-bob e2e: a click runs and the stub records exact argv, a 3-step sequence
     reaches the summary card, a failing step halts and leaves failed ink, a
     `bob-copy`-adjacent copy chip still copies, and a `ref/` chip copies instead of
     running.
   - Add `scripts/preview-command-chips.cjs` covering every state, light and dark, and
     attach the PNGs.
7. **Docs.** `docs/bob-chips.md` gains "Command chips": syntax, what a run does, where
   chips run, copy, or refuse, look and states, gestures, and the attack-surface note.

## Phase: fence — the `bob-run` sequence card

1. Render `bob-run` fences in Live Preview and Reading view as the card in "Look":
   - header with the ▶ glyph, the label (unlabeled: "Run · N steps"), meta, and a Run
     button;
   - meta is "N steps · <effect>", where the effect is the strongest step effect (reads
     < writes < syncs), plus "· asks first" when any step is `confirm`;
   - numbered monospace steps below.

   Clicking the body selects text and never runs. The Run button runs (Shift+Click opens
   Command Mode). The button has a fixed minimum width, so "Run", "Output", and "Run"
   never reflow.

2. **Live marks** from the store: ✓ done (with its duration), a spinner on the current
   step, ○ upcoming, ✗ failed with bob's first error line and an Output link, and `–`
   skipped. The header reads "2 of 3" while running and "stopped at k · <outcome>"
   afterwards. An armed card takes an accent header border.
3. **Variants.**
   - Unavailable: a ⊘ header with the reason, and the failing step marked ✗ with its
     reason.
   - Copy-only: a copy glyph, and the button reads Copy, copying the body byte-exact.
   - Print: a plain code listing.
4. **Resolver.** Fence block chips already resolve through `commandChipAt`. Make sure
   the card's Run and `⌘↵` share `commandChipActivate`.
5. **Tests.** DOM tests per state and variant, e2e (the card runs 3 steps and shows
   marks; a failure leaves the ✗ row), preview PNGs, and docs.

## Phase: keymap — `⌘↵`, the armed chip, and the keycap

1. **bob-chips.**
   - `api.commandChips.targetAt(view)` is editor-bound. It reads lines and the cursor,
     gets the task scope from `bob-navigation-hotkeys` `api.taskScope`, applies the
     no-run, Command Mode, and resolve checks, and returns:
     - `none`;
     - `ambiguous` (with its message);
     - `blocked` (with bob's reason);
     - `ready` (with the chip, `via`, and `running`).
   - `runAtCursor(view)` resolves to `{ handled, outcome, stage, reason }`:
     - `none` → `{ handled: false }`;
     - `ambiguous` or `blocked` → one Notice and `handled: true`;
     - `ready` → `commandChipActivate` plus a brief armed pulse.
   - Register **Run command chip at cursor** (`run-at-cursor`, no default hotkey).
2. **bob-navigation-hotkeys.**
   - At the top of `openReferencePdfInHighlightsOrLink`, once the editor context
     resolves (Live Preview or Source mode), call `runAtCursor` when it exists. If
     `handled` comes back, stop. Otherwise the existing chain runs unchanged.
   - Set the capability flag `this.commandChipDispatch = 1`.
   - Make the Highlights status hint and rail yield when `targetAt` is not `none`.
   - Rename the command's display name to cover chips, and keep its id. Bump the
     version.
3. **The hint.**
   - A `bob-chips` CM6 view plugin paints the end-of-line keycap capsule
     (`.bob-run-chip-hint`, styled like `.bob-gmail-open-hint`), and sets the store's
     armed target.
   - It shows only when all of these hold:
     - `targetAt` is `ready`;
     - the capability flag is `1`;
     - `app.hotkeyManager` has a binding for
       `bob-navigation-hotkeys:open-reference-pdf-in-highlights`;
     - the editor has one empty selection.
   - The keycap is formatted with the `⌘↵ Gmail` hint's platform convention, from the
     live binding. Its `aria-label` reads "Cmd+Enter runs <face>". It updates on
     selection, doc, focus, hotkey, and catalog changes, with debounced rebuilds.
4. **bob-ledger-tools.** The `⌘↵ Gmail` capsule hides whenever `targetAt` is not `none`.
   Bump the version.
5. **Tests.**
   - The dispatcher order for every rule, including an ambiguous refusal that never
     falls through.
   - Regression for every old `⌘↵` target: a Highlights reading task, a highlight block
     link, a Gmail mail link, and a plain link in a new tab.
   - No-run notes and an absent Command Mode keep today's behavior.
   - The keycap tells the truth: a rebound key shows the new keycap, and no binding
     shows no keycap.
   - One hint at a time on lines with both a chip and a mail link.
   - e2e: `⌘↵` on the GTD-shaped fixture rows (line chip, task fallback, Bank-shaped row
     with links).
   - Preview PNGs of the armed chip and keycap.
6. **Docs.** `bob-chips.md` ("⌘↵ and the hint"), `bob-navigation-hotkeys.md` (the new
   chain order), and `plugin-apis.md`.

## Phase: author — make or edit command chips

1. Rename the command's display name to **Make or edit chip** and keep the id
   `make-or-edit`. Add a **Kind** row: a two-button segmented control in the Tab cycle,
   switched with ←/→ or Space. The title reads "⧉ COPY CHIP" or "▶ COMMAND CHIP", and
   the Text field is labeled "Command" in Run.
2. **Defaults are predictable and never silently switched.**
   - Editing a chip keeps its kind.
   - A selection, a plain code span, or a clipboard prefill whose text resolves `ready`
     as a bob command line defaults to Run. Text that would resolve after a `bob `
     prefix also defaults to Run and shows the Add-bob fix.
   - Everything else defaults to Copy.
3. **Run validation is live and per step**, through `resolve`:
   - each step row shows ✓ or ✗, the catalog title, the effect, and the policy note
     ("preview skipped", "asks first", "opens its output");
   - invalid steps block Save with bob's reason;
   - a missing `bob` offers **Add bob** (a button that is also in the Tab cycle);
   - host-unavailable commands warn but allow Save, because a Mac-only chip may be
     authored on athena;
   - without Command Mode, the row says the commands can't be checked here and Save is
     allowed.
4. **Serialization.** `runChipSerialize` picks the inline span or a `bob-run` fence (⇧⏎
   adds a step line). The preview shows the live chip or card plus the Markdown bytes.
   Save is one undo step with the existing stale-write guard.
5. Alt+Click on a command chip opens the card for that chip, and the tooltip gains "·
   ⌥Click edits".
6. **Tests.** Model tests for defaults, validation, Add bob, and kind switching that
   preserves Label and Text. Modal key tests (`q` types inside fields; Esc, `q`, and
   `Ctrl+[` close). Serializer cases. e2e: convert `` `bob task reconcile` `` with
   Enter. Preview PNGs and docs.

## Phase: adopt — GTD rows, guide, memory, coordination

1. **Preflight.** `bob plugins list` shows `bob-chips` (with keymap and author) synced
   and enabled, and `bob-command-mode` carries the engine.
2. **Census.** Re-read `gtd_daily.md` fresh. Re-run the census of open recurring tasks
   that hold bob commands, outside the no-run folders and `done/`:

   ```bash
   rg -n --glob '!{mail,ref,ref_history,.bob,_conflicts,done}/**' '^\s*- \[[^xX-]\] .*repeat::' ~/bob \
     | rg '`[^`]*\b(bob|gcal|gkeep|gmail|task reconcile|tick)\b'
   ```

   At planning time, the only hits outside copy-chip prompts were the four rows below.
   Convert any other open recurring row that runs a catalog-legal bob command the same
   way, and report it.

3. **Rows.** Edit only the open instances, never the Done & Canceled history; the
   recurrence regenerates from the current text. Change nothing else in a row: no
   status, fields, dates, or other words, except where shown.
   - `#sync`: `` Run `gcal sync -n 2` + `gkeep pull` + `gmail sync` `` →
     `` Run `run[Sync calendar and inboxes]: bob gcal sync -n 2 && bob gkeep pull && bob gmail sync` ``
   - `#init`: `` Run `task reconcile` `` → `` Run `run: bob task reconcile` ``
   - **Bank**, gated:
     - Convert only if `ssh mac 'bob bank import --help'` lists `-D, --delete`:
       `` `bob bank import ~/Downloads/*.qfx && rm ~/Downloads/*.qfx` `` →
       `` `run[Import bank files]: bob bank import -D ~/Downloads/*.qfx` ``.
     - `-D`'s digest-gated delete replaces the `rm`, and `rm` never runs through chips.
     - If the Mac is unreachable or lacks `-D`, leave the row unchanged and record a
       `PROPOSED FOLLOW-UP:` note with the exact replacement.
   - **Ack**:

     > [!decision] gtd_ack_copy_chip Replace `` run `bob gcal ack -n 7` `` with
     > ``run `copy: bob gcal ack -n 7` in a terminal``. It can never be a command chip,
     > because its explicit walk needs a TTY.

     > [!decision] gtd_ack_copy_chip = no Leave the Ack row unchanged.

   - The six period-review prompts stay copy chips.

4. **Vimrc.** Under the `gy` comment lines, add exactly one comment line and no mapping:
   `" ⌘↵ (Meta+Enter) runs the command chip at the cursor — the one under it, the line's only one, or the task's only one — through Command Mode.`
5. **Verify.**
   - A Node one-liner that loads `plugins/bob-chips/main.js` `helpers` and
     `bob-command-mode` helpers with `bob __catalog --protocol 1` output must parse
     every converted chip and resolve it `ready`. The payloads must equal the intended
     command lines.
   - `bob task reconcile` leaves the converted rows' text unchanged.
   - Commit only `gtd_daily.md` and `obsidian_vimrc.md` in the vault.
6. **bob-cli guide.** `docs/guides/run-bob-from-obsidian.md` gains "Run a command from a
   note". It covers how to write `run: bob …`, click or `⌘↵` and the keycap,
   Shift+Click, sequences with `&&` or a `bob-run` fence, the confirm floor, copy-only
   notes, and the rule that running never completes the task, and it links to the Bob
   Chips reference. Change only guide steps this makes incorrect, and run
   `just docs-check`.
7. **Memory** (accepted decisions only). Run `/sase_memory_write` first, then
   `sase memory init` and `just docs-gen`, and commit the regenerated glossary page.

   > [!decision] memory_glossary_command_chip Add the `glossary:command-chip` strand
   > ("Command Chip"; aliases `run chip`, `bob-run`; never claim `chip`, which Copy Chip
   > owns). Body: An authored one-gesture bob run in an Obsidian note. It is an inline
   > code span that starts with `run:` (or `run[LABEL]:`) whose payload is one `bob`
   > command line, or several joined by `&&`, or a `bob-run` fence with one command per
   > line; every step starts with the word `bob`. The `bob-chips` plugin renders the
   > span as a quiet pill with a ▶ glyph and the fence as a titled sequence card. A
   > click runs it through [[command-mode]]'s engine the way `Ctrl+Enter` runs a sheet
   > row. So does `⌘↵` (Meta+Enter), when the chip is under the cursor, is the cursor
   > line's only command chip, or, on a line with no link, is its task's only one.
   > Previews are skipped, the binary's `confirm` commands still ask, and several steps
   > run as one sequence that halts at the first failure. A muted `⌘↵ Run` keycap at the
   > end of the cursor line marks the chip `⌘↵` would run. Shift+Click opens it in
   > Command Mode instead, and `Mod+Shift+Y` makes or edits one. Payloads are literal
   > and validated against bob's catalog: no shell, no variables, only bob. Unknown or
   > excluded commands render dimmed with bob's reason. In `mail/`, `ref/`,
   > `ref_history/`, `.bob/`, and `_conflicts/` notes, on mobile, or without Command
   > Mode, a command chip only copies its text. Running a chip never completes its task
   > or advances the walk. Not a [[copy-chip]], a Command Mode RECENT row, or a shell
   > snippet. See `docs/bob-chips.md`.

   > [!decision] memory_glossary_copy_chip Rename already corrected the plugin name and
   > docs path. Here, extend the strand's closing "Not a …" sentence to name
   > [[command-chip]] (`run:` spans, which `gy` never copies).

   > [!decision] memory_decision_record Add
   > `sase/memory/decisions/command-chips-run-through-command-mode.md` in the decisions
   > web's record shape (Applies to / Claim / Why / Cost / Reopens when / Evidence).
   > Claim: executable chips need the explicit `run:` / `bob-run` prefix. They run only
   > catalog-legal `bob` argv, only through Command Mode, with `Ctrl+Enter` semantics
   > except that `confirm` commands still ask. They never use a shell. They only copy in
   > `mail/`, `ref/`, `ref_history/`, `.bob/`, and `_conflicts/` notes, and running
   > never completes a task. Rejected: bare `bob …` spans as buttons, `obsidian://`
   > links, button frameworks, a separate plugin, and a preview-first detour. Evidence:
   > the research ref and the shipped commits.

   For each memory decision left off, record a `PROPOSED FOLLOW-UP:` note on this phase
   bead, unless a human reviewer turned it off.

8. **Coordination notes** (bead notes, not memory):
   - `bob-cli-nc`: the `#sync` row is now a command chip. When it switches to
     `bob tick -a`, write `` `run: bob tick -a` `` and keep the chip form.
   - `bob-cli-nn`: plain-text surfaces also show raw `run:` / `run[…]:` source. The face
     helper should cover command chips (the label, else the step texts with `›`).
9. **Proposed follow-ups** (`PROPOSED FOLLOW-UP:` notes on the phase bead):
   - catalog TTY metadata, so `gcal ack -n` refuses up front in the sheet and in chips;
   - the Bank row, if it was gated;
   - anything the census found but could not convert.
10. **Report.** Ask Bryan to reload Obsidian. Each other desktop host needs
    `chezmoi update` and `bob plugins sync` before command chips run there. Until then,
    the old folder keeps copy chips working and `run:` spans read as inline code.

## Non-goals

- A shell: no pipelines, redirects, variables, interpolation, `~` or glob expansion in
  the plugin, interactive TTY commands, or parallel steps.
- Chips that complete, freshen, or advance their task, or that auto-run on render, open,
  or schedule.
- Any bob CLI, catalog, or Bob Mac Capture change; the sheet accepting typed `a && b`; a
  "Copy as chip" action in the sheet; a new Vim mapping; a walker-footer run hint.
