---
tier: tale
title: Show Next and Blocked sources in the Task Card's Linked from CURRENT
goal:
  The Task Card's ⇧L Linked from stage lists every task that links to the current task
  under CURRENT, whatever the source's checkbox status (Ready, Next, In Progress,
  Blocked, Done, Canceled), and ↵ on such a row removes that link and its mirror entry.
size: small
proposed_by: bbugyi200.athena.12k.f0
proposed_with: claude/opus
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.12k.f0](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.12k.f0.md)
- **COMMITS:**
  - [d4927af](https://github.com/bobs-org/bob-plugins/commit/d4927af24de4a935fe3cba5a012bbd2053eb253b)
    — fix(ledger-tools): index relations from tasks of every checkbox status

# Show Next and Blocked sources in the Task Card's Linked from CURRENT

## Diagnosis (established; do not re-derive)

**Symptom.** On the Task Card (Ctrl+Shift+P), `⇧L` opens the Linked from stage. It
should list every task whose LINKS line points at the current task under CURRENT, where
↵ removes the link. Bryan sees no CURRENT rows, so he cannot remove existing links. His
scratch pair in `sase.md` (vault history `33e9bf8d`) was:

```
- [ ] #task foo bar baz [created::2026-10-20] ^foo-bar-baz
	- 🪢 **LINKED FROM:** [[#^bar]] (son of)
- [*] #task bar baz buz [created::2026-10-20] ^bar
	- 🪢 **LINKS:** son of [[#^foo-bar-baz]]
```

The source `bar baz buz` is `[*]`, which is Next in Bob's Tasks config. The other custom
statuses are `/` In Progress, `?` Blocked, and `-` Canceled.

**Root cause (bob-ledger-tools, not nav).** The ⇧L stage builds CURRENT only from
`api.relationIndex.incoming(owner.path, owner.blockId)` (`decorateLinkedFromVaultStage`,
`plugins/bob-navigation-hotkeys/src/427-relation-stage-plugin.js`). The index decides
which lines are tasks with `relationTaskInfo` in
`plugins/bob-ledger-tools/src/145-relation-index-model.js`:

```js
if (!relationListMarker(text) || !/\[[ xX/-]\]/u.test(text) || !/(?:^|\s)#task(?:\s|$)/u.test(text)) return null;
...
const statusMatch = text.match(/\[([ xX/-])\]/u);
```

Only `[ ]`, `[x]`, `[X]`, `[/]`, and `[-]` count as tasks. In `parseTaskRelationFile`, a
LINKS line under a `[*]` or `[?]` task, or under any other checkbox symbol, finds a
parent list item that is "not a task" and is dropped. So that edge never reaches
`outgoing`, `incoming`, or `labelSuggestions`.

Everything else treats every checkbox symbol as a task:

- nav's `OBSIDIAN_TASK_LINE_RE` (`[^\]\n]`);
- bob-cli reconcile, which wrote the mirror above;
- ledger-tools' own chip owner lookup, `relationOwnerTaskLine` in
  `278-plugin-relation-model.js`. It calls `dependencyChipIsTaskLine` under the comment
  "Sources are checkbox `#task` lines, as the live index requires".

**Reproduced** with a throwaway Node harness. It loads both real `main.js` builds, runs
the real ledger relation index over the fixture above, and builds nav's stage with
`buildVaultRelationStage({ …, direction: "from" })`.

- **Source `[ ]` or `[x]`:** CURRENT holds `bar baz buz` with label `son of → this`.
  `chooseRelationTask(row)` removes both the LINKS entry and the mirror line.
- **Source `[*]`, `[?]`, or `[>]`:** `incoming()` returns `[]`, CURRENT is empty, and
  `bar baz buz` appears as a RESULTS row ("link here") instead.
- **Fix applied in a scratch copy:** with `relationTaskInfo` accepting any checkbox
  symbol after the list marker, CURRENT lists the source for `[*]`, `[?]`, `[/]`, `[x]`,
  and `[-]`. ↵ removes the link and its mirror in every case, so the removal path needs
  no change.

**Other symptoms with the same cause, all fixed by the same change.** For sources whose
checkbox is not `[ ]`, `[x]`, `[/]`, or `[-]`:

- the Task Card `⇧L` row count and detail leave them out;
- their LINKED FROM chips on the target, and the LINKS chips under them, render as stale
  (dashed, tooltip "No longer links here"), because `stale: !edge`;
- their labels are missing from the relation field's suggestions.

The forward `l` stage is unaffected. It reads the owner's own LINKS line with nav's
parser.

**Secondary display defect in the index-only fallback (nav).** In
`relationStageIncomingRowsFromEdges` (`304-relation-stage-view.js`), nav may have no
record for a source. That happens when the Tasks cache is not `Warm` and the source note
is not open. The row is then built straight from the index edge, which differs from nav
records in three ways:

- `source.line` is **1-based**;
- `source.text` is the raw task line, or a Tasks description such as
  `#task bar baz buz`;
- `source.status` is a word (`TODO`, `DONE`, …) where nav expects a checkbox symbol.

As a result the row shows the raw line as its title, carries a line number one too high,
and is classed `closed: true` because `"TODO"` is not in `OPEN_OBSIDIAN_TASK_STATUSES`.
Removal still works, because the writer resolves by block id. The Task Card's `⇧L` row
detail (`taskCardLinkedFromSummary`, `312-task-card-relations.js`) prints `first.text`
uncleaned in every case.

**Why tests missed it.** No relation test uses a `[*]` or `[?]` source. The index tests
use `[ ]` and `[x]`, and the nav ⇧L tests stub `incoming()` with ready-made edges. No
test wires the real index into nav's Linked from stage.

## Work (bob-plugins linked repo)

Open the repo with your `/sase_repo` skill (`sase repo open bob-plugins -r "<reason>"`)
and read its `AGENTS.md`. All paths below are relative to that checkout. Edit only
`src/` fragments and test scripts. Never hand-edit a generated `main.js`, and keep each
hand-edited fragment at or below 1000 lines. `304-relation-stage-view.js` is at 848
lines now.

1. **Write the failing seam test first.** Create
   `scripts/test-relation-index-linked-from-stage.cjs` and add it to the `test` script
   in `package.json`, right after `scripts/test-navigation-relation-links-stage.cjs`.

   **Setup.**
   - Load both `plugins/bob-ledger-tools/main.js` and
     `plugins/bob-navigation-hotkeys/main.js` under one `Module._load` stub. It must
     cover:
     - `obsidian`: `MarkdownView`, `Modal`, a `Notice` that records messages,
       `Platform`, a `Plugin` class with no-op `registerEvent`, `normalizePath`, and
       `parseYaml`;
     - `@codemirror/state`: `Prec.highest`;
     - `@codemirror/view`: `EditorView` with a static `updateListener.of`, and
       `keymap.of`.

     Copy the shapes from `scripts/test-ledger-tools-relation-index.cjs` and
     `scripts/test-navigation-relation-links-stage.cjs`.

   - Build the real index: `new LedgerToolsPlugin(app)`, `initializeRelationIndex()`,
     `startRelationIndexScan()`, then wait until `api.ready()`. Give every fixture file
     a metadata `cache` with a `#^` link, so the startup prefilter keeps it.
   - Give the nav plugin
     `app.plugins.plugins["bob-ledger-tools"] = { api: { relationIndex } }`. It also
     needs `vault` (`getMarkdownFiles`, `getAbstractFileByPath`, `cachedRead`, `read`,
     `process`), the same `metadataCache`, and a `workspace` whose `getLeavesOfType`
     returns the owner's open `TestEditor`.

   **Fixture.**
   - Owner `- [ ] #task Owner ^owner` in `Tasks.md`.
   - Same-note sources with statuses `[*]`, `[?]`, `[/]`, `[x]`, and `[-]`. Each has a
     direct-child `🪢 **LINKS:**` line to `[[#^owner]]`. Give the `[*]` source the label
     `follow-up of`.
   - One cross-note `[*]` source in `Other.md` linking `[[Tasks#^owner]]`.
   - The owner's `🪢 **LINKED FROM:**` mirror, as reconcile would write it.

   **Assertions.**
   - Build the stage with
     `plugin.buildVaultRelationStage({ filePath: "Tasks.md", content, parentLines: [ownerLine], direction: "from" })`
     and plan it with `helpers.planRelationStageItemsFromVault(stage, "")`.
   - CURRENT lists all six sources exactly once. The `[*]` row's label is
     `followed up by`, its inverse phrase.
   - No source appears in RESULTS.
   - The cross-note row has no nav record, because `Other.md` is not open and no Tasks
     cache is provided. After step 3 it reads its clean title and is open. Add that
     assertion with step 3.
   - Build a picker the way `makeRelationPicker` does, with `vaultStage = stage`, and
     run `await picker.chooseRelationTask(row)` on the `[*]` same-note CURRENT row. Its
     LINKS line and the owner's mirror entry for it are removed, and every other
     relation line is byte-identical.

   Run `npm run build` and this file. Confirm the CURRENT assertion fails before any
   fix.

2. **Fix `relationTaskInfo`**
   (`plugins/bob-ledger-tools/src/145-relation-index-model.js`).
   - Recognise task lines with `dependencyChipIsTaskLine(text)` from
     `140-dependency-model.js`. Fragments share one scope, and it is the predicate the
     chip owner lookup already uses. It accepts any single checkbox character directly
     after the list marker. Drop the separate `relationListMarker` and `#task` tests.
   - Read the symbol from an anchored match,
     `/^[ \t]*(?:[-*+]|\d+[.)])[ \t]+\[([^\]\n])\]/u`. The old pattern searched for a
     bracket anywhere in the line.
   - Map the status word as follows: `x`/`X` → `DONE`, `/` → `IN_PROGRESS`, `-` →
     `CANCELLED`, `*` → `NEXT`, `?` → `BLOCKED`, anything else → `TODO`. The chip
     model's `relationStateFromTask` already reads `NEXT` and `BLOCKED`.
   - Keep the `RELATION_BLOCK_ID_RE` check and the returned shape.
   - Leave `relationListMarker` itself alone; the parent walk in `parseTaskRelationFile`
     still uses it.
3. **Harden nav's index-only rows.**
   - **Stage rows.** In `relationStageIncomingRowsFromEdges`
     (`plugins/bob-navigation-hotkeys/src/304-relation-stage-view.js`), keep every field
     from the nav `record` when one exists. Without a record, build the row from the
     edge:
     - `line` is `source.line - 1` when that is an integer ≥ 1, else `-1`;
     - `displayText` and `text` are `cleanTaskDisplayText(source.text)`, falling back to
       `^<blockId>`;
     - `status` is the checkbox symbol from a new small pure helper,
       `relationStageEdgeStatusSymbol(source)`. It returns the symbol that
       `OBSIDIAN_TASK_LINE_RE` captures when `source.text` is a task line. Otherwise it
       maps the word (`TODO` → `" "`, `NEXT` → `"*"`, `IN_PROGRESS` → `"/"`, `BLOCKED` →
       `"?"`, `DONE` → `"x"`, `CANCELLED` → `"-"`), and anything else gives `" "`;
     - `open` and `closed` come from that symbol and `OPEN_OBSIDIAN_TASK_STATUSES`, as
       today.

     Export the helper from `src/700-exports.js` next to
     `relationStageIncomingRowsFromEdges`.

   - **Card detail.** In `taskCardLinkedFromSummary`
     (`plugins/bob-navigation-hotkeys/src/312-task-card-relations.js`), print
     `cleanTaskDisplayText(first.text)`, falling back to `^<blockId>` and then
     `source task`.

4. **Unit regression tests.** Do not weaken any existing assertion.
   - **`scripts/test-ledger-tools-relation-index.cjs`.**
     - `parseTaskRelationFile` over sources `[ ]`, `[*]`, `[/]`, `[?]`, `[x]`, `[-]`,
       and `[>]` yields one LINKS edge each, with the right source `blockId`. The status
       words are `TODO`, `NEXT`, `IN_PROGRESS`, `BLOCKED`, `DONE`, `CANCELLED`, and
       `TODO`.
     - Each of those tasks is in `tasks`. `- #task text [x] ^stray` (bracket not after
       the marker) and `- [ ] untagged ^plain` are not.
     - In a `relationIndex` test, `api.incoming(target)` returns the `[*]` and `[?]`
       sources and `api.outgoing` returns their edges.
     - A `[?]` target with no cached block resolves from text.
     - `labelSuggestions()` counts a label authored under a `[*]` source.
   - **`scripts/test-navigation-relation-links-stage.cjs`.**
     - `helpers.relationStageIncomingRowsFromEdges` gets an empty index and a
       ledger-shaped edge: `source.line: 4`,
       `text: "- [*] #task bar baz buz [created::2026-10-20] ^bar"`, `status: "NEXT"`.
       The row has line `3`, displayText `bar baz buz`, status `*`, and is open and not
       closed.
     - A Tasks-description edge (`text: "#task Done source"`, `status: "DONE"`) gives
       status `x` and `closed: true`.
     - Add a card test in which the stub `incoming()` returns
       `text: "- [*] #task Source task ^source"`. Its row detail is `1 · Source task`.
       Leave the existing card ⇧L test as it is.
5. **Docs.** In `docs/task-relations.md`, add one sentence to the Linked from paragraph:
   sources of every task status count, including Next `[*]` and Blocked `[?]`. No other
   doc text changes.
6. **Version and build.** Patch bumps, as earlier fixes took:
   - `plugins/bob-ledger-tools/manifest.json`: `1.60.1` → `1.60.2`;
   - `plugins/bob-navigation-hotkeys/manifest.json`: `2.50.1` → `2.50.2`.

   Run `npm run build`, then `npm run docs:gen` to refresh the versions in
   `docs/index.md` and the two plugin pages.

7. **Validate.** `npm run build:check`, `npm test`, `npm run validate`, and
   `npm run docs:check`.
   - Six `npm test` failures already exist on the clean base `4ca3b50`:
     - four unblocked-glyph tests (bob-cli-6v);
     - "Keep counts once and saturates at 999" (bob-cli-6w);
     - "forward conversion carries a Depends-On line" (bob-cli-7i).
   - The stage-ranker 16 ms perf assertion can flake under load (bob-cli-3w).
   - Any other failure blocks the change.
8. **Deploy on athena** (bob-plugins `AGENTS.md` requires it). Run
   `bob plugins sync -n -r "<opened bob-plugins path>"`, then `bob plugins list`.
   Confirm `bob-ledger-tools 1.60.2` and `bob-navigation-hotkeys 2.50.2` read `synced`.

Commit subject:
`fix(ledger-tools): index relations from tasks of every checkbox status`. The body must
name the root cause: `relationTaskInfo` accepted only `[ xX/-]`, so Next and Blocked
sources vanished from `incoming()` and the ⇧L CURRENT list. It should also mention the
nav fallback hardening.

## MacBook rollout

Bryan hit the bug on the Mac. A bare sync there pulls origin, so only a pushed fix
reaches it.

1. Commit the bob-plugins change now with `/sase_git_commit`. Bryan's approval of this
   plan is the explicit request to commit it before the Mac deploy.
2. Confirm the commit is on `origin/master`.
3. Run `ssh mac 'bob plugins sync && bob plugins list'`. The bare sync pulls the Mac's
   bob-plugins checkout first.
4. Confirm ledger-tools shows `1.60.2` and nav shows `2.50.2`. Do not kill or relaunch
   Obsidian, because Bryan is using it.
5. In the final report, ask Bryan to reload Obsidian (command palette → "Reload app
   without saving").

**Reaching the Mac.** Read `tailnet.md` with `/sase_memory_read` first; SSH to `mac` is
best-effort. The last rollout reached it. Retry
`ssh -o ConnectTimeout=20 -o BatchMode=yes mac true` in one foreground loop for up to
about 10 minutes, with a matching Bash timeout, or hand that loop to `/sase_monitor`. If
the Mac never answers, give Bryan the exact steps instead: once the commit is on
`origin/master`, run `bob plugins sync && bob plugins list` on the Mac (expect
ledger-tools `1.60.2` and nav `2.50.2`), then reload Obsidian.

**Live acceptance check for the final report.**

1. After reloading, put a `[*]` task's LINKS line on any task.
2. Open Ctrl+Shift+P on the target → `⇧L`. The source should appear under CURRENT with
   its inverse label.
3. Press ↵. That should remove both the LINKS entry and the target's LINKED FROM entry.

Bryan's original `foo bar baz` / `bar baz buz` pair no longer exists. He deleted it at
vault commit `21509756`. Do not edit the vault.

## Not in scope

- Relation index performance and startup (bob-cli-a5, bob-cli-a6).
- The epic note #1 doc fixes in bob-cli (`docs/guides/link-related-tasks.md` quoting and
  the stale `bob capture-parse --help` text).
- The six clean-base `npm test` failures listed above. They are tracked by their own
  beads.
