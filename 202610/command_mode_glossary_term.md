---
tier: tale
title: Add a Command Mode glossary term
goal:
  Agents resolve "command mode" to Bob Command Mode's `:` command sheet, not its
  look-alikes.
size: small
decisions:
  memory_command_mode_term:
    ask:
      Should the glossary web gain a new `command-mode.md` strand defining Command Mode?
    memory:
      - glossary
      - glossary:command-mode
    default: true
    requested:
      Can you help me add an excellent new glossary term for Obsidian's "command-mode"
      (triggered via the `:` keymap)?
    answer: true
proposed_by: bbugyi200.athena.16g
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-30 15:26:14
status: wip
---

# Add A "Command Mode" Glossary Term

## Goal

Add one glossary strand that tells any agent what Bryan means by "command mode" in
Obsidian: the Bob Command Mode plugin's bottom command sheet, opened by `:` in Vim
normal mode. The term should identify the thing, explain the few properties that change
how an agent reasons about it, and rule out the look-alikes that share the name or the
`:` key.

## Why A Term Is Worth Its Cost

"Command mode" is ambiguous in four ways that matter here, and none of the 28 current
glossary terms covers it:

1. **Vim.** In vi usage, "command mode" can mean normal mode or the `:` command line (ex
   line). In this vault, `:` no longer opens the ex line in normal mode; `g:` does.
2. **Obsidian's command palette.** The palette is a separate surface, but it also lists
   **Bob: Command mode** and generated `Bob: <title>` run commands that use the same
   engine.
3. **Capture.** In a capture draft, a leading `:` opens the task-link picker. Free-text
   `bob capture TEXT` is excluded from the catalog, so command mode refuses it.
4. **SASE.** Bryan's task list also names a "sase TUI command-mode panel", which is a
   different product.

The roster line costs a few tokens on every turn. The body is read only on demand.

## Verified Facts The Strand Relies On

The implementer must re-check each fact against these sources before writing, and must
reword the strand rather than keep a claim that no longer holds:

- Plugin and keys: `docs/reference/plugins/bob-command-mode.md` and
  `docs/guides/run-bob-from-obsidian.md`. The plugin id is `bob-command-mode`. It is
  desktop-only. The palette command is **Bob: Command mode** (`open`).
- Vim binding: the vault's Vimrc Support plugin loads `obsidian_vimrc.md`. That file
  contains `exmap bob_command obcommand bob-command-mode:open`,
  `nnoremap : :bob_command<CR>`, and `nnoremap g: :`.
- Catalog: `docs/command-catalog.md`. The source is `src/native/catalog/`. Rows come
  from the hidden, versioned `bob __catalog --protocol 1` handshake, and the plugin
  never hard-codes a command list. `bob capture` is in the catalog's `excluded` list.
- Run engine: the bob-plugins source at `plugins/bob-command-mode/src/`
  (`060-runner.js`, `170-sheet.js`, `080-api.js`). Each run is `spawn(bobPath, argv)`
  with no shell. Records and output go under the host-local
  `${XDG_STATE_HOME:-~/.local/state}/bob-cli/command-mode/runs/`. Writers run one at a
  time after a save-every-open-editor guard, and reads run in parallel. Sibling plugins
  use `api.commandMode`, and bob-ledger-tools refreshes on
  `bob-command-mode:run-finished`.
- Capture's `:` picker: `docs/capture.md` (search for "A solo leading `:` opens the
  task-link picker").

## Changes

> [!decision] memory_command_mode_term = no Do not create the strand or run any
> generators. Instead, file one `memory` task bead through `/sase_new_task` whose
> description contains the strand below verbatim and names
> `sase/memory/glossary/command-mode.md` as its path.

### 1. Write the Command Mode strand

Use the `/sase_memory_write` skill first. The strand's path is
`sase/memory/glossary/command-mode.md`. Write this strand, adjusting only facts that the
check above disproves:

```markdown
---
keyword: Command Mode
aliases:
  - "Bob Command Mode"
  - "command sheet"
---

The `bob-command-mode` Obsidian desktop plugin's bottom command sheet: a fuzzy finder
for running `bob` commands without leaving the vault. With Obsidian's Vim mode on, `:`
in normal mode opens it, because the vault vimrc (`obsidian_vimrc.md`) maps
`nnoremap : :bob_command<CR>` onto `exmap bob_command obcommand bob-command-mode:open`;
**Bob: Command mode** in the command palette opens it in any view. Its rows come from
bob's hidden, versioned `bob __catalog` handshake, so the binary, never the plugin,
decides which commands, arguments, and write policies exist, and excluded commands are
refused. An empty query shows RECENT runs, FOR THIS NOTE rows, FROM CLIPBOARD URLs, and
one group per catalog section. `Enter` runs a row, `Tab` enters argument mode,
`Shift+Enter` previews, `Ctrl+Enter` runs now or applies a preview, and a trailing `?`
shows help; reports stream into an output pager. Each run spawns `bob` directly, with no
shell, and its record and output stay host-local under
`${XDG_STATE_HOME:-~/.local/state}/bob-cli/command-mode/runs/`, never in the vault.
Writers run one at a time and only after every open Markdown editor saves; a writer
without a preview asks first on a confirm card, while reads run in parallel. Catalog
rows that need no arguments, preview, or confirmation also appear in the command palette
as `Bob: <title>`, and sibling plugins run commands through the same engine
(`api.commandMode`). In Bob, "command mode" never means Vim's normal mode or its `:`
command line: `g:` in normal mode, or `::` in the sheet, still opens Vim's ex line, and
Vim-shaped input such as `:%s/a/b/` or `:42` is routed there. Not a [[capture]]:
free-text `bob capture TEXT` is refused here, and a capture draft's leading `:` opens
the task-link picker instead. Not the SASE TUI's command-mode panel. See
`docs/guides/run-bob-from-obsidian.md`.
```

Authoring rules this draft already follows. Keep them if you edit it:

- The keyword is `Command Mode`, so the roster sorts it right after `Capture`. The
  `command-mode` slug spelling, which Bryan uses, already resolves through the file
  stem, so it does not need to be an alias. Keep the aliases to the plugin's product
  name and its UI noun.
- Link only `[[capture]]`. The glossary uses `closure: mentions`, so each link pulls
  another definition into every read of this term. Capture adds only Task Relation. Keep
  other surfaces as plain prose or paths.
- Do not copy the output pager's keys, settings, toasts, or the status chip. The plugin
  reference owns those; the glossary only defines the term.
- Do not name bead IDs. `bob-docs gen` refuses a generated page that contains one.

### 2. Regenerate memory outputs

Run `sase memory init`. It should update `sase/memory/glossary.md` (the roster now
includes `Command Mode (Bob Command Mode, command sheet)`), `AGENTS.md`, `CLAUDE.md`,
and `sase/memory/README.md`. Never hand-edit those files.

### 3. Regenerate the docs glossary page

Run `just docs-gen`. It rewrites the generated `docs/reference/glossary.md`, which
should gain a `### Command Mode {#command-mode}` section after Capture with the aliases
line and `[Capture](#capture)`. No other docs page needs to change. The guide and the
plugin reference already describe the behavior, and `sase/memory/docs.md` routes
glossary changes through regeneration only.

## Verification

1. Each of these prints the new term, followed by Capture and Task Relation:
   - `sase memory read glossary:"command mode" -r "verify new term"`
   - `sase memory read glossary:"Bob Command Mode" -r "verify alias"`
   - `sase memory read glossary:command-mode -r "verify slug"`
2. `just docs-check` passes.
3. `git status` shows changes only to: the new strand, `sase/memory/glossary.md`,
   `AGENTS.md`, `CLAUDE.md`, `sase/memory/README.md`, and `docs/reference/glossary.md`.
   `just docs-gen` may also rewrite CLI goldens. If any golden changes, inspect it and
   revert it unless it reflects real drift on master.

## Out Of Scope

- No changes in bob-plugins or bob-mac-capture, and no behavior changes.
- No new decision record. This is vocabulary, not a policy choice.
- No edits to the guide or plugin reference.
