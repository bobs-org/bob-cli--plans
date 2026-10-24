---
tier: epic
title: Relation label vocabulary — a chooser that works, hidden and one-off labels,
  and one global rename
goal: 'The Bob Mac Capture relation card is a reliable, beautiful chooser: Ctrl+N/P
  move a visible highlight through existing relation labels, Tab accepts, and nothing
  collapses. Relation labels can be hidden from suggestions (Ctrl+D, with undo), shown
  again (Ctrl+H section) and unhidden, created hidden with `~note:id||label` (with
  a clear warning when that hides an existing label), and renamed everywhere — from
  the card, `bob relations rename`, and Obsidian — in one previewed, revertable commit.

  '
decisions:
  hidden_key:
    ask: Which chord should toggle the relation card's Hidden section (Mac card and
      Obsidian stage)?
    choices:
      ctrl_h: Ctrl+H, mirroring `bob relations -H`; Ctrl+H stops deleting backward
        inside the card
      ctrl_shift_h: Ctrl+Shift+H; Ctrl+H keeps its Emacs delete-backward meaning in
        the card field
    default: ctrl_h
    why: The research's pick; Backspace still deletes and the ◌ N hidden pill is clickable
    answer: ctrl_h
  obsidian_manager:
    ask: 'Should Obsidian get a standalone ''Bob: Relation labels'' command to hide,
      unhide, and rename labels?'
    default: true
    why: Renaming from Obsidian should not require starting a link gesture first
    answer: true
  memory_decisions:
    ask: Should the records phase add four decision records for relation label hide,
      journal, rename, and card keys?
    memory:
    - decisions
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: true
  memory_glossary:
    ask: Should glossary:task-relation gain one sentence on relation labels (hide,
      one-off ||, global rename)?
    memory:
    - glossary:task-relation
    default: false
    answer: false
phases:
- id: mac-chooser
  title: Mac relation card as a navigate-then-accept chooser
  depends_on: []
  size: medium
  description: 'mac-chooser: fix the Tab collapse by separating query from selection
    (selection by key), add Ctrl+N/P and arrow navigation with wrap, Tab accepts or
    completes the top match, AppKit field with focus repair, local filtering of a
    once-fetched catalog, the redesigned vertical card, a collapsing fake-client regression
    test, and keyboard docs. No bob change.'
- id: label-catalog
  title: Shared preference journal, relation label catalog, and bob relations list/hide/unhide
  depends_on: []
  size: medium
  description: 'label-catalog: extract the theme journal into a namespaced shared
    module, add the relation_labels module (one key function, fast LINKS scan, display
    spelling, fold), the bob relations group with list/hide/unhide, the vault allowlist
    rollout, the label-vocabulary contract section, and shared vectors.'
- id: label-complete
  title: capture-complete relation_label rows from the catalog
  depends_on:
  - label-catalog
  size: small
  description: 'label-complete: switch relation_label suggestions to the fast catalog,
    add key/inverse/existing/count/hidden fields and hidden_count, omit hidden rows,
    adopt the catalog spelling for case variants, fix the dedupe mismatch, and measure
    the latency budget.'
- id: rewrite-txn
  title: Shared vault-rewrite transaction extracted from bob themes rename
  depends_on:
  - label-catalog
  size: medium
  description: 'rewrite-txn: extract lock, pre-sync, snapshot and stale check, dirty-path
    rules, atomic write with rollback, verify, single commit, and post-sync from the
    theme rename into one shared transaction, moving bob themes rename onto it byte-for-byte.'
- id: double-pipe
  title: The `||` one-off relation label capture syntax
  depends_on:
  - label-complete
  size: medium
  description: 'double-pipe: parse, rewrite glue, completion, dry-run hide fields
    and the hides_existing_relation_label warning, the after-commit journal append,
    shell completion, and the capture grammar docs for `~note:id||label`.'
- id: label-rename
  title: bob relations rename across every note, done/ included
  depends_on:
  - rewrite-txn
  size: medium
  description: 'label-rename: the strict, previewed relation label rename on the shared
    transaction — LINKS label spans everywhere including done/ and the previous daily
    note, LINKED FROM mirrors, case-only and merge rules, journal rename event, renamed_reappeared,
    the recorded LR7 exception, and the runbook.'
- id: mac-vocab
  title: Mac card hide, one-off, Hidden section, and warnings
  depends_on:
  - mac-chooser
  - double-pipe
  size: medium
  description: 'mac-vocab: decode the new fields, Ctrl+D ghost-row hide/unhide on
    a serial lane, the one-off toggle (Ctrl+D or a typed pipe), the Hidden section
    and pill, the amber warning in the card, preview, and success feedback, capsule
    states, version skew, fixtures, and docs.'
- id: obsidian-vocab
  title: Obsidian label stage parity, JS mirrors, and api.relationLabels
  depends_on:
  - label-catalog
  size: medium
  description: 'obsidian-vocab: copy the shared vectors, align the JS key, rank, and
    display spelling, add a read-only journal fold and api.relationLabels v1 with
    hide/unhide writes through bob, and give the nav relation-label stage the card''s
    Tab, Ctrl+D, pipe, Hidden, ghost, warning, and footer behavior.'
- id: mac-rename
  title: Mac card rename mode
  depends_on:
  - mac-vocab
  - label-rename
  size: medium
  description: 'mac-rename: Ctrl+R rename mode with a live dry-run preview, merge
    confirmation, apply on a serial long-timeout lane, the post-rename draft fix-up
    from bob-supplied ranges and replacements, version skew, tests, and docs.'
- id: obsidian-rename
  title: Obsidian rename mode and the relation labels command
  depends_on:
  - obsidian-vocab
  - label-rename
  size: medium
  description: 'obsidian-rename: Ctrl+R rename mode in the relation-label stage with
    the bob dry-run preview, dirty-editor refusal, and merge confirmation, plus the
    optional standalone Bob: Relation labels command, tests, docs, and deploy.'
- id: records
  title: Decision records, docs sweep, and end-to-end acceptance
  depends_on:
  - mac-rename
  - obsidian-rename
  size: small
  description: 'records: write the accepted memory changes, drop planned markers and
    check cross-links across bob-cli, Mac, and plugin docs, and run and record the
    end-to-end acceptance.'
proposed_by: bbugyi200.apollo.research.2e.linker.w0
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-24 13:34:42
status: wip
bead_id: bob-cli-bj
---

- **PROMPT:** [prompts/202610/relation_label_vocabulary.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/relation_label_vocabulary.md)
- **BEAD:** [bob-cli-bj](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-bj/README.md)

# Plan: Relation label vocabulary

## Outcome

Bryan types `Buy new SSD for athena! @dev:harddrive ~sase:` in Bob Mac Capture, picks
"What was athena doing at 07:57…", and presses Shift-Return. The relation card opens:

```text
╭─ 🪢 Relation ───────────────────────────────────────────── ◌ 1 hidden ⌃H ─╮
│  Buy new SSD for athena!  ───────────────▶  What was athena doing at 07:5… │
│ ┌────────────────────────────────────────────────────────────────────────┐ │
│ │ Search or name a relation▏                                             │ │
│ └────────────────────────────────────────────────────────────────────────┘ │
│    inspired by                                           3 links      ⇥    │
│    implements                                            2 links           │
│    duplicate of                       ⇄ duplicated by    1 link            │
│                                                                            │
│  ⇥ inspired by    ↩ plain link    ⌃N ⌃P choose    ⌃H hidden    esc back    │
╰────────────────────────────────────────────────────────────────────────────╯
```

Two Ctrl+N presses move a highlight to **implements**. The field does not change, bob is
not called, and the sentence reads `… ── implements ──▶ …`. Tab inserts `|implements`.
Ctrl+D on a row hides that label from suggestions everywhere, and the row stays as a
dimmed ghost with `⌃D undo`. Typing `prompted the purchase of` and pressing `|` (or
Ctrl+D) makes it a one-off:
`↩ inserts ||"prompted the purchase of" · won't be suggested again`. Ctrl+R on
**inspired by** opens a rename mode. It previews "3 links in 3 notes · mirrors updated ·
1 commit", and Return renames the label in every note, `done/` included. The same keys
work in the Obsidian relation-label stage, and `bob relations` lists, hides, unhides,
and renames from the terminal.

## Context

- **Research (accepted in full by Bryan):**
  `research:202610/relation_card_label_vocabulary/relation_card_label_vocabulary.md`.
  Read it with `sase artifact read` before starting any phase. Its adjustments A1–A13
  are requirements of this epic. Its design (card states A–G, the contracts, the rename
  transaction) is the baseline. This plan settles its open points and splits the work.
- **The reported bug (verified in code):** `cycleRelationLabelSuggestion(by:)` in
  `Sources/BobMacCapture/CapturePanelModel.swift` writes the chosen row into
  `authoredLabel` and re-queries bob. For a full label, bob returns a single typed row,
  so the list collapses to one entry and every later Tab computes `(0+1) mod 1 = 0`.
  Fast Tabs reach "the nth" label only by walking the stale list before bob replies. The
  index also outlives its list, starts at 0 (so the first Tab skips the most-used
  label), and is never drawn. Ctrl+N/P and ↑/↓ are swallowed by the shared
  `isolatedPromptCommand` in `CaptureKeyCommandRouter.swift`. The SwiftUI `TextField`
  sits outside `repairPromptFieldFocusIfOrphaned`.
  `testRelationCardTabCyclesSuggestions` uses a fixed array, so it cannot see the
  collapse.
- **Freshness protocol to keep:** bob-cli-al's `suggestionAuthoredLabel`,
  `pendingSubmit`, and generation guards. They are correct. Only navigation stops going
  through them.
- **bob today:** `relation_label_suggestions` in
  `src/native/capture_complete/candidates.rs` calls `capture_dependency_tasks::discover`
  over the whole vault on every keystroke (0.28–0.85 s warm). Its catalog folds with
  `to_lowercase`, but the typed-row dedupe uses `eq_ignore_ascii_case`. Ranking is
  `capture_complete/support.rs::rank`: prefix matches, then substring matches, keeping
  catalog order (count desc, lowercase name, name).
- **Vault facts (read-only, 2026-10-24):** 8 LINKS lines, 6 labelled entries, 3 labels
  (`inspired by` ×3, `implements` ×2, `duplicate of` ×1), and 1 LINKS line in `done/`.
  `bob capture-parse 'Buy SSD ~sase:x||oneoff'` reports `invalid_relation` today, so
  `||` is free to claim.
- **Contract:** `docs/task-relations.md` (bob-cli) is the relation contract. Grammar §3,
  mirror projection §6, capture JSON §12, guarantees §13 (LR7: no relation write touches
  `done/`), vectors §15.
- **Precedent, being built right now:** epic bob-cli-bc (Pomodoro themes). On bob-cli it
  has shipped `src/native/pomodoro_themes/` (`journal.rs` PT4 fold, `write.rs`
  hide/unhide, `rename.rs` PT10 transaction) and `bob themes`. Still in flight are bc.7
  (Mac Ctrl+D ghost rows and a `theme-pref` process lane), bc.10 (Obsidian picker
  Ctrl+D), bc.11 (Obsidian themes manager), and bc.13 (records). Reuse what has landed
  and never change theme behavior. Note that master's `just check` currently fails
  `every_value_arg_has_a_decision` for `themes rename:old/new`. That epic owns the fix,
  so compare against a clean-base run.
- **Obsidian today:** the nav relation-label stage
  (`plugins/bob-navigation-hotkeys/src/426-picker-relation-stage.js`) already has a
  filter field, a vertical list with `used N×`, Ctrl+N/P, and a row-0 typed/plain row.
  Labels come from `api.relationIndex.labelSuggestions` in bob-ledger-tools
  (`145-relation-index-model.js`, `171-relation-index-lifecycle.js`). The theme mirror
  precedent is `279-plugin-pomodoro-theme-mirrors.js` and
  `279-plugin-pomodoro-themes.js`.
- **Usage is low** (bead bob-cli-am was cancelled for "such low usage"). That is why the
  bug fix ships first and alone, and why rename comes last.

## Design

`docs/task-relations.md` gains a new **§16 Label vocabulary**. It holds the normative
version of rules RV1–RV9 below, written by `label-catalog` and extended by later phases.
A rule that has not shipped yet is marked `planned (phase <id>)` there until its phase
lands.

### RV1 Words

- **Relation label** (contracts, CLI, JSON, code) is the optional text on a LINKS entry.
  The UI mostly says **relation**. Bryan's "relation description" is the same thing.
  Never use "description" in code or contracts, because bob already uses it for task
  text.
- **Hide / Unhide / Show hidden.** Never "disable": `disabled_reason` already means
  "cannot be inserted", and a hidden label can still be inserted.
- **One-off** means a label created hidden (`||`). It is wording that is never suggested
  again, not an expiring link. Relations are history.
- **Naming in code:** `relation_labels` (Rust module), `RelationLabel*` (Swift),
  `api.relationLabels` (JS).

### RV2 Key, catalog, and display spelling

- **Key.** `relation_label_key(label)` applies the writer normalization (trim, collapse
  whitespace runs, strip one trailing `:`) and then Unicode `to_lowercase`. This one
  function feeds the catalog, the typed-row match, the journal, rename, and, through
  vectors, the JS mirror. Do not add NFC or any other new equivalence.
- **Catalog.** One row per key, counting every accepted LINKS entry with that label:
  open, closed, `done/`, daily notes, nested projects, and references.
  - Walk every `.md` in sorted path order. Skip `_conflicts/`, `_templates/`,
    dot-directories, and fenced blocks.
  - Byte-prefilter for `**LINKS`, then parse direct children of `#task` lines with
    `task_relations::parse::links_children_of`.
  - Malformed lines are skipped with a bounded warning. Completion may tolerate them;
    rename may not (RV8).
  - Never call `capture_dependency_tasks::discover`.
- **Display spelling.** The most-used spelling of the key, with ties going to the first
  seen. `spellings[]` lists every spelling with its count, so case drift is visible.
- **Inverse.** The §10 display-only table (`merged into` → `merged from`, and so on).
  Rust gets a `task_relations::label::inverse_phrase` if it does not already have one.
- **Journal-only keys.** A hidden key with zero uses stays listable (count 0) from the
  journal's stored name, so it can always be found and unhidden.

### RV3 Preferences journal

- **Location:** `.bob/relations/<host>.jsonl`, one append-only file per host. The host
  label is the vault-sync host label.
- **Events (`v: 1`):**

  ```json
  {"v":1,"ts":"2026-10-24T09:41:07-04:00","host":"mac","kind":"relation_label.hide","key":"implements","name":"implements","via":"bob-mac-capture"}
  {"v":1,"ts":"2026-10-24T09:42:00-04:00","host":"mac","kind":"relation_label.unhide","key":"implements","name":"implements","via":"bob-mac-capture"}
  {"v":1,"ts":"2026-10-24T10:02:11-04:00","host":"apollo","kind":"relation_label.rename","from":"inspired by","to":"prompted by","from_name":"inspired by","to_name":"prompted by","via":"bob-cli"}
  ```

- **Fold:** exactly PT4's rule, implemented once in a shared namespaced module (not
  copied):
  - read every `*.jsonl` file and skip malformed lines with a bounded warning;
  - sort by `(ts instant, host, path, line)`; the last writer wins per key;
  - a rename moves the source key's explicit state unless the target already has state;
  - events after a rename on the old key apply to the old key.
- **Writers:** bob only. Appends happen under `bob_sync.lock` with a 3 s wait (busy
  gives `lock_busy`), one `O_APPEND` write per event, and the directory is created on
  first use.
- **Vault allowlist:** `!/.bob/relations/*.jsonl` in `.gitignore`, and
  `.bob/relations/*.jsonl merge=union` in `.gitattributes`.

### RV4 What hiding means

- **Hiding removes a label from suggestions only.** It affects card rows, the Obsidian
  stage, `relations list` defaults, and zsh completion. LINKS text, LINKED FROM mirrors,
  chips, counts, and typed use are unchanged. This is the twin of
  `decisions:theme-hide-is-recent-only`.
- **Using a hidden label never unhides it.** Typing its exact name still works and is
  flagged `hidden`.
- **No silent state.** Nothing auto-hides. A hide is always an explicit Ctrl+D,
  `bob relations hide`, or `||`.

### RV5 The relation card (Mac; the Obsidian stage mirrors it in RV9)

**State.** The card's state is:

```text
query        field text; only typing edits it
selectedKey  a label key or nil, never an array index
showHidden   whether the Hidden section is open
oneOff       a pending "hide on capture" for what Return inserts
mode         choose | rename
catalog      visible rows; fetched once on open and after every write
hiddenRows   hidden rows; prefetched when hidden_count > 0
ghosts       key → hidden / visible-again; cleared when the query changes
```

Navigation changes only `selectedKey` and the sentence pill. It never edits `query`,
spawns a process, or clears freshness. Typing clears `selectedKey` and refreshes only
the typed-row validation probe. The list is always ranked locally from `catalog` (and
`hiddenRows`) with bob's documented rule. A refresh keeps `selectedKey` when the key
survives and clears it when the key disappears; a different row never inherits it.

**Keys (choose mode).** One rule ties them together: Ctrl+D hides what Return would
insert.

| Key                 | Action                                                                                                                                                                                                                                                                  |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- | --- | ------------------------------- |
| ⌃N / ↓              | Select the next row. From no selection, select the first. Past the last, return to the field (wrap).                                                                                                                                                                    |
| ⌃P / ↑              | Select the previous row. From no selection, select the last. Before the first, return to the field.                                                                                                                                                                     |
| ⇥                   | Accept the selected row. With no selection, accept the top match, marked `⇥` on its row. With no match, insert nothing and show `No match — ↩ uses “…”`.                                                                                                               |
| ↩                  | Accept the selected row. Otherwise insert the typed text through bob's typed row (the existing `pendingSubmit` path). An empty field inserts a plain link and drops the pipe.                                                                                           |
| ⌃D                  | **Selected row:** hide it now, or unhide a hidden row. The row stays as a ghost; ⌃D on the ghost undoes. **No selection:** toggle one-off for the typed label (hint on an empty field: `Type a label to make it one-off`). Always consumed; key auto-repeat is ignored. |
| `                   | `                                                                                                                                                                                                                                                                       | Turn one-off on. It never enters the field, so `~note:id |     | ` typed straight through works. |
| ⌫ on an empty field | If one-off is on, turn it off. Otherwise unchanged behavior.                                                                                                                                                                                                            |
| ⌃H                  | Show or hide the Hidden section (`hidden_key`). The query and selection are kept.                                                                                                                                                                                       |
| ⌃R                  | Rename the selected row, or the typed label when it equals an existing key. Otherwise hint `Select a label to rename`.                                                                                                                                                  |
| Esc / ⌃[            | Leave rename mode first. Otherwise go back to the picker or editor, as today.                                                                                                                                                                                           |
| ⇧⇥                  | Consumed, no-op.                                                                                                                                                                                                                                                        |
| Click               | A row accepts it. The eye icon on a hovered or selected row hides or unhides it. The `◌ N hidden` pill toggles the Hidden section.                                                                                                                                      |

> [!decision] hidden_key = ctrl_shift_h Everywhere this plan says ⌃H for the Hidden
> section, use ⌃⇧H instead (Mac card, Obsidian stage, footers, and docs), and leave
> Ctrl+H's text-editing meaning in the card field alone.

- **One-off applies to whatever Return inserts.** With a row selected and one-off on,
  Return inserts `||row`.
- **Return accepts ghost and hidden rows** like any row. Hidden labels stay usable and
  stay hidden.
- **Claimed text keys.** ⌃D (forward delete) and ⌃H (backspace) are claimed only inside
  this one-line card. fn-Delete and Backspace still work, and the app already overrides
  Ctrl+A/E/T/U. Never claim Cmd+A/C/V/X/Z, IME composition keys, or Control+Option.

**States.** The mocks below are the visual target. A and B ship in `mac-chooser`; C–F
ship in `mac-vocab`; G ships in `mac-rename`.

```text
B · ⌃N twice: the highlight moves, the field stays, the pill follows
│  Buy new SSD for athena!  ── implements ──▶  What was athena doing…      │
│    inspired by                                           3 links         │
│ ▌  implements                                            2 links    👁   │
│  ↩ inserts |implements    ⌃D hide    ⌃R rename everywhere    esc back    │

C · A new label made one-off (⌃D, or a typed "|")
│ │ prompted the purchase of▏                          ◌ one-off · new   │ │
│  ↩ inserts ||"prompted the purchase of" · won't be suggested again     │

D · One-off on an existing visible label: the warning (amber)
│ │ inspired by▏                                    ◌ one-off · 3 links  │ │
│  ⚠ Capturing will also hide “inspired by” (3 links) from suggestions   │
│    everywhere. Existing links keep their label.     ⌃D keep visible     │
│  ↩ inserts ||"inspired by"                                              │

E · ⌃D on a selected row: a ghost row with undo
│ ▌◌ implements                             hidden everywhere · ⌃D undo   │

F · ⌃H: the Hidden section
│    inspired by                                           3 links        │
│    ── Hidden ─────────────────────────────────────────────────────────  │
│ ▌◌ duplicate of                                          1 link         │
│  ↩ inserts |"duplicate of" (stays hidden)    ⌃D unhide    ⌃H hide list  │

G · ⌃R: rename mode
│  Rename “inspired by” everywhere to                                     │
│ │ prompted by▏                                                        │ │
│    3 links in 3 notes · 2 mirrors · 1 in done/ · 1 commit               │
│  ↩ rename everywhere    esc cancel                                      │
│  (an existing NEW: "Merges into “implements” (2) → 5 links · ↩ ↩ merge")│
```

**Visual rules:**

- **One icon.** 🪢 in the mint `relationSigil` accent. Drop the extra
  `arrow.left.arrow.right` symbol.
- **The sentence is the hero.**
  - Owner and target appear in primary text, truncated in the middle.
  - The label rides the arrow as a small pill and always shows what Return would insert
    (the selected row, else the typed text).
  - A one-off label wears `eye.slash`.
  - `~~` flips the sentence so the direction is unmistakable.
- **Rows look like picker rows.**
  - Same height and accent selection as the pickers. Selection is a leading bar plus a
    fill, never color alone.
  - Matched text is bold.
  - Counts are right-aligned tertiary `N links`, because they count LINKS entries, not
    tasks.
  - Inverse phrases are quaternary.
  - The `⇥` keycap marks Tab's target when nothing is selected.
- **The field's trailing capsule** shows the typed label's status: `new`, `N links`,
  `hidden`, `one-off · …`, or a red invalid reason. The list holds only existing labels;
  "use what I typed" is always Return on the field.
- **Hidden rows** use `eye.slash` and secondary text. They are never red and never
  struck through.
- **Colors.** Amber means a warning; red means only an invalid label.
- **The footer** shows at most five context-sensitive keycaps and replaces "Tab cycles
  suggestions". Its `↩ inserts |…` text keeps teaching the grammar, including `||`.
  Keep Cancel and Insert for pointer users.
- **Stable layout.** The list has a bounded height of about six rows and scrolls, so
  replies never move the field or the capture controls. The highlight moves instantly;
  only a ghost fades.
- **Accessibility.** VoiceOver announces the selected label, its count, its position,
  and its hidden state. Verify light, dark, increased contrast, long titles, and long
  labels.

### RV6 Capture grammar: `||`

```text
~note:id|label            relation with a label
~note:id||label           same; hide the label from suggestions once the capture succeeds
~note:id||"multi word"    quoted form
~~note:id||label          reverse direction, same meaning
~note:id||                incomplete: needs relation_label (hidden: true)
~note:id|||label          error: "use || to hide a label, | to name one"
```

The stored Markdown is identical for `|` and `||`. ` ||` is glued the way ` |` is glued.
How each label state is treated:

| Existing label state | `                  | label`                                                           | `   |     | label` |
| -------------------- | ------------------ | ---------------------------------------------------------------- | --- | --- | ------ |
| Unknown              | create; visible    | create; hidden after capture (`◌ one-off`)                       |
| Visible              | use; stays visible | use, then hide everywhere; **amber warning with the link count** |
| Hidden               | use; stays hidden  | use; stays hidden; neutral "already hidden", no event            |

- **Warn, never block.** The hide happens only after the notes commit succeeds. A dry
  run or a failed capture leaves no trace. A failed append never fails the capture.
- **One event per key.** In a multi-item draft, repeated hides of one key collapse to
  one event. Mixing `|x` and `||x` leaves `x` hidden whatever the order. Counts come
  from the pre-capture catalog.
- **Version skew.** An older app with a newer bob still parses `||` typed in the editor,
  because bob owns the grammar. The app never downgrades `||` to `|` on its own.

### RV7 Contracts (additive; every `schema_version` stays 1)

- **`capture-complete` `relation_label`** (built by `label-complete`; `double-pipe` adds
  `hide_replacement` and `one_off`):
  - Suggestion rows are `{label, key, count, replacement, inverse?, hide_replacement}`.
    Hidden labels are omitted, as theme Recent rows are, so older clients never suggest
    them.
  - The typed row is
    `{label, key, replacement, typed: true, existing?, count?, hidden?, disabled_reason?, hide_replacement}`.
    When its key matches the catalog, `label` and `replacement` use the catalog display
    spelling (A12).
  - The response gains `hidden_count` (present, possibly 0, in this context) and
    `one_off: true` when the probe sits in a `||` label.
  - `replacement` keeps the probe's pipe count. `hide_replacement` is always the `||`
    form. The app never builds either token itself.
- **`bob relations`**: a group in **Tasks and projects**, between `projects` and `task`.
  Its membership rule is "the relation-label vocabulary" (the `bob themes` precedent: a
  catalog noun whose `rename` also rewrites notes). A bare group runs the read-only
  `list`. Hide and unhide do not rewrite task lines, so they stay out of `bob task`, and
  nothing goes under `bob capture`.

  ```text
  bob relations [list] [-a/--all] [-f/--format human|json] [-H/--hidden] [-n/--limit N] [-q/--query Q]
  bob relations hide LABEL…   [-f/--format human|json]
  bob relations rename OLD NEW [-d/--dry-run] [-f/--format human|json] [-m/--merge] [-o/--offline]
  bob relations unhide LABEL… [-f/--format human|json]
  ```

  - Keep option parity with `bob themes` wherever an option applies, for example a
    vault-directory override.
  - Hidden internal arguments: `--via CLIENT` (default `bob-cli`) on every writer, and
    `--allow-unused` on `hide`. Clients that hide a label they just wrote, before bob
    can see it on disk, pass `--allow-unused`.
  - **Human list** (mint relation accent on a TTY, dim counts; still readable with
    `NO_COLOR`):

    ```text
    RELATIONS  3 labels · 6 links · 1 hidden
      inspired by      3 links
      implements       2 links
      duplicate of     1 link     ⇄ duplicated by     ◌ hidden   (with -a)
    ```

  - **JSON list:**
    `{schema_version, ok, counts:{labels, links, hidden}, labels:[{key, name, count, hidden, inverse, spellings, replacement, hide_replacement}], renamed_reappeared, warnings}`.
  - **Writers** emit `{schema_version, ok, …}`. Human hide output reads:
    `🪢 Hid “implements” from relation suggestions · 2 links keep their label · undo: bob relations unhide implements`.
  - **Refusal codes:** `unknown_label`, `invalid_label`, `unchanged`, `label_exists`,
    `malformed_links_line`, `lock_busy`, `dirty_paths`, `not_a_worktree`, `sync_failed`,
    `verify_failed`, `stale_note`.

- **`capture` / `--dry-run`, per item** (built by `double-pipe`):
  - `relation_label_hides[] {label, key, existing_count, was_hidden, action: hide|already_hidden}`.
  - A visible existing label also adds `warnings[]`
    `{code: "hides_existing_relation_label", label, count, message}` with a bob-authored
    message. Clients render that message and never compute the count.
  - A failed append adds `{code: "relation_label_hide_failed", message}`. The message
    includes the exact retry command.
  - Human output adds the line `🪢 hides "inspired by" from suggestions (3 links)`.
  - `capture-parse` `relations[]` gains `hidden: true`, and the separator span covers
    both pipes.

### RV8 The rename transaction

This follows `bob themes rename` (PT10), and runs on the shared transaction that
`rewrite-txn` extracts from it.

1. **Lock and sync.** Take `bob_sync.lock`, pre-sync unless `--offline`, and plan from
   the synced files.
2. **Plan strictly.**
   - Find every LINKS entry whose key equals OLD across every scanned note (RV2 scope).
   - Relabel each entry in place with the existing per-edge planner (LW4), preserving
     entry order.
   - Rewrite the matching LINKED FROM label spans of writable targets in the same batch.
   - **Refuse rather than under-cover:** a malformed LINKS line that mentions OLD, or an
     unreadable note, gives `malformed_links_line`.
3. **Rules.**
   - A case-only rename is allowed (same key, new spelling) and unifies every spelling.
   - A normalized-identical rename with no spelling change is `unchanged`.
   - A different existing key is `label_exists` unless `--merge`.
   - Hidden state moves to NEW unless NEW already has explicit state.
4. **Write and verify.**
   - Snapshot every target and compare right before writing (`stale_note`).
   - Write atomically, then append `relation_label.rename`.
   - Rescan: OLD's key is gone, entry counts are unchanged, and only label spans differ.
     On failure, restore the before-images (`verify_failed`).
   - Make one commit, `bob relations rename: OLD → NEW`, then post-sync. Report "renamed
     locally" and "sync pending" separately.
5. **Preview (`-d`).** It lists:
   - files and entries, with before and after lines;
   - link, note, and mirror counts, including how many are in `done/`;
   - whether the hidden state moves;
   - inverse-phrase changes (renaming `follow-up of` loses the built-in "followed up
     by");
   - `replacement` and `hide_replacement` for NEW, so clients can fix up drafts.
6. **Afterwards.**
   - `bob relations list` reports OLD in `renamed_reappeared` when an offline host
     writes it later. It only reports, never rewrites.
   - Undo is `git revert` of the one commit, or a reverse rename when no merge happened.
7. **The LR7 exception.** `done/` and the previous daily note stay read-only for every
   relation write except rename. Rename may change **label spans only** there: it never
   reopens tasks, mints IDs, prunes edges, or reorders entries. Amend §13 guarantee 6,
   §6, and the decision records.

### RV9 Obsidian

- **One ranker and one key.** The JS key, rank, display spelling, and fold match Rust,
  gated by vectors copied byte-for-byte from bob-cli.
- **Read side (desktop and mobile).**
  - bob-ledger-tools folds `.bob/relations/*.jsonl` read-only through the vault adapter
    (the theme-mirror precedent).
  - `api.relationIndex.labelSuggestions` omits hidden labels.
  - A new `api.relationLabels` v1 adds:
    - `version`;
    - `rows(query, {hidden: "exclude"|"only"|"include"}) → [{key, name, count, hidden, inverse}]`;
    - `hiddenCount()`;
    - `onChange`;
    - desktop-only writes: `hide(label, {allowUnused, via})`, `unhide(label, {via})`,
      and (from `obsidian-rename`) `rename(old, new, {dryRun, merge, via})`.
  - Writes spawn `bob relations … -f json --via <plugin id>`, then refresh. On mobile,
    writes resolve `{ok: false, code: "desktop_only"}`.
- **The nav relation-label stage uses the card's keys and copy.**
  - Its row-0 typed/plain row plays the Mac's "no selection": Ctrl+D on row 0 toggles
    one-off, and Tab on row 0 accepts the top existing match.
  - Ctrl+D here acts on vocabulary. The Task Card's _Links to_ Ctrl+D ("remove every
    outgoing relation") acts on edges. Both footers and the help docs say so.
- **One-off in Obsidian.** Commit the relation through the existing
  `commitRelationEdit`. Only if that succeeds, call `hide(label, {allowUnused: true})`.
  A failed hide keeps the relation and shows a Notice with the retry command.

### Performance

- **Mac.** Navigation is local and instant. The card fetches the catalog once on open
  and after each write. Per keystroke, the only call is the typed-row validation probe,
  and the list never waits on it.
- **bob.** Measure the warm `capture-complete` relation_label probe on apollo against
  the live vault, read-only, with `hyperfine` (30 runs), before and after
  `label-complete`. The budget is p95 ≤ 150 ms. If the fast scan misses it, add a
  PT3-style per-file fingerprint cache under `bob_cli_cache_dir()/relations/`.

## Cross-cutting rules for every phase

- **Contracts first.** New behavior lands in bob-cli and its docs before clients consume
  it. JSON grows additively at `schema_version` 1. Clients decode new fields with
  `decodeIfPresent` or feature detection.
- **Shared vectors.** `tests/fixtures/relation_labels/vectors.json` in bob-cli is the
  single source.
  - Its sections are `key`, `catalog`, `fold`, `rank`, `grammar` (`rust_only`), and
    `rename` (`rust_only`).
  - bob-plugins copies it byte-for-byte to
    `plugins/bob-ledger-tools/scripts/fixtures/relation_labels/`, with a README that
    records the bob-cli base commit.
- **CLI and docs rules.** Read `cli_rules.md` and `docs.md` with `/sase_memory_read`
  before touching commands, help, or docs.
  - Every public long option gets a short alias.
  - Register every value-taking argument in `src/native/completion/kinds.rs`, so
    `every_value_arg_has_a_decision` passes for `relations`.
  - Run `just docs-gen`, then `just docs-check`.
- **Release train.** Never edit `Cargo.toml` versions or `CHANGELOG.md`. Use
  conventional commits (`feat(relations): …`, `feat(capture): …`,
  `fix(relation-card): …`). Mark only a true contract break with `!`; none is planned.
  bob-plugins and bob-mac-capture follow their own versioning conventions.
- **Cross-repo work.**
  - Open bob-mac-capture, bob-plugins, and the research sidecar only through
    `/sase_repo`, and read each repo's `AGENTS.md`.
  - Plugin phases run `npm test` and then `bob plugins sync`. Deploying needs an
    installed bob that already has `bob relations`, so install bob-cli master first if
    needed.
- **Mac builds.** Follow bob-mac-capture's `AGENTS.md`:
  - Never land Swift you have not compiled.
  - Run `just portable-check`, using athena over Tailscale SSH if the local host lacks
    Swift (`tailnet.md`).
  - Best effort: build a scratch copy on the mac (`bundle.sh` + `just typecheck-tests`).
  - A Mac phase closes only on a green macOS CI run. If the mac is unreachable, say the
    macOS targets are unverified.
- **Coordination with bob-cli-bc.**
  - If bc.7's Mac ghost-row presentation or `theme-pref` lane has landed, reuse and
    generalize it. Otherwise follow the same template-delete ghost pattern in a form
    bc.7 can share.
  - Never change theme behavior. Rebase carefully over concurrent bc commits to the same
    files.
- **Vault safety.**
  - Tests use scratch vaults with `BOB_DIR`, `BOB_DAY_FILE`, `BOB_NOW`, and
    `BOB_CONFIG_FILE`.
  - Against `~/bob`, only read and measure. The single exception is the allowlist
    rollout in `label-catalog`.
  - No live-vault hide, unhide, or rename is run by any phase.
  - Never write into `~/bob/.obsidian/plugins` by hand.
- **Baselines and follow-ups.**
  - Compare test failures against a clean-base run.
  - Phase workers create no beads. They record discoveries as `PROPOSED FOLLOW-UP:`
    notes on their phase bead.

## Phases

### mac-chooser

**Repo:** bob-mac-capture. This phase fixes the reported bug and needs no bob change.

1. **Separate query from selection** in `CaptureRelationLabelPromptState` and
   `CapturePanelModel.swift`:
   - Replace `selectedSuggestionIndex` with `selectedKey: String?`.
   - Add a `catalog` filled once at open from the existing empty-label probe (`|"`).
   - Delete `cycleRelationLabelSuggestion`'s write into `authoredLabel` and its refetch.
   - Typing keeps today's freshness protocol (bob-cli-al) for the typed row only.
   - Rows come from the catalog, ranked locally by bob's rule: prefix, then substring on
     the lowercased, whitespace-collapsed label, in catalog order. Drop rows flagged
     `hidden` (a no-op today, and forward-compatible).
   - The key is the lowercased, whitespace-collapsed label; `mac-vocab` switches to
     bob's `key` field.
2. **Keys** in `CaptureKeyCommandRouter.swift` and `CapturePanelController.swift`:
   - Add card-scoped cases **before** the shared `isolatedPromptCommand` fallthrough. Do
     not edit that helper, which also serves the task-ID and Pomodoro-name prompts.
   - Cover ⌃N/↓ and ⌃P/↑ (wrapping through the field), ⇥ (accept the selection or the
     top match), ⇧⇥ (consumed), and ↩ (accept the selection, else today's typed path).
   - Accepting a catalog row inserts its bob `replacement` with no extra probe.
3. **Field.** Replace the SwiftUI `TextField` with an AppKit `RelationLabelField` built
   like `BlockIDField` and `PomodoroNameField`. Register it in
   `repairPromptFieldFocusIfOrphaned`, and correct the model comment that claims the
   field is AppKit-owned.
4. **View.** Rebuild `RelationLabelPromptCard` in `CapturePanelView.swift` to RV5 states
   A and B and the visual rules:
   - one 🪢 icon;
   - the sentence hero with its label pill;
   - a bounded vertical list with a drawn selection, bold matches, `N links`, and the
     `⇥` keycap;
   - the trailing capsule (`new` / `N links` / invalid);
   - the keycap footer;
   - the Cancel and Insert buttons;
   - VoiceOver labels.
5. **Tests** in `Tests/BobMacCaptureTests/CaptureRelationPanelTests.swift`. Replace
   `testRelationCardTabCyclesSuggestions` with tests built on a fake process client that
   filters by query the way bob does, so a collapsing list fails. Cover:
   - k presses of ⌃N then Tab insert the k-th label;
   - navigation sends zero fetches and leaves the field unchanged;
   - slow and fast sequences select the same rows;
   - ⌃N from an empty field selects the most-used label;
   - a reorder keeps the key, and a missing key clears the selection;
   - Tab with no selection inserts the top match, and Tab with no match is a no-op with
     its hint;
   - every existing bob-cli-al freshness test still passes.
6. **Docs:** add a **Relation card keys** table to `docs/keyboard.md`, and update the
   relation card text in `docs/relationship-pickers.md` and `docs/capture-panel.md`
   ("Tab cycles" goes away).

**Acceptance:**

- `just portable-check` is green, the Swift is compiled per `AGENTS.md`, and macOS CI is
  green.
- The keyboard docs match the router.
- If the mac is reachable, attach light and dark screenshots of states A and B to the
  phase note.

### label-catalog

**Repo:** bob-cli, plus the vault allowlist rollout. Covers RV1–RV4 and the read and
hide side of RV7.

1. **Shared preference journal.**
   - Move the generic parts of `src/native/pomodoro_themes/journal.rs` into
     `src/native/pref_journal/`, keyed by a namespace:
     - `themes` → `.bob/themes`, `theme.*` kinds;
     - `relations` → `.bob/relations`, `relation_label.*` kinds.
   - The generic parts are the paths, host label, timestamps, serialize and append under
     the lock, read-all with bounded malformed warnings, sort, last-writer-wins fold,
     and rename state moves.
   - `theme.color` stays theme-specific.
   - `bob themes` output and the theme `fold` vectors must stay byte-identical and
     green.
2. **`src/native/relation_labels/`** provides:
   - the RV2 `key` (also used by `task_relations` where labels are compared);
   - the fast catalog scan, display spelling, `spellings`, the inverse phrase, the RV3
     fold, and journal-only hidden keys;
   - a `Catalog` API that later phases call: load for a vault (with warnings), look up a
     key, rank rows for a query with a hidden filter, and report the hidden count.
3. **CLI.** The `bob relations` group with `list` (also the bare group), `hide`, and
   `unhide`, per RV7.
   - Register the group in `src/runner.rs` (`SUBCOMMANDS`, Tasks and projects) and in
     the completion kinds.
   - Help is excellent and options are alphabetical.
   - `unknown_label` refuses a hide unless `--allow-unused` is passed. `unchanged` is a
     no-op that writes nothing.
4. **Vault rollout.**
   - Add `!/.bob/relations/*.jsonl` to `~/bob/.gitignore` and
     `.bob/relations/*.jsonl merge=union` to `~/bob/.gitattributes`.
   - Verify with `git check-ignore -v` and `git check-attr merge`.
   - Commit in the vault (`chore(vault): allow relation label journal`) and run
     `bob vault-sync`.
   - Document it in `docs/vault-git-sync.md` beside the theme journal.
5. **Docs.**
   - Add `docs/task-relations.md` §16 Label vocabulary: RV1–RV4, the RV7 list/hide
     contract, and RV5–RV6/RV8–RV9 marked `planned (phase …)`.
   - Add a **Relation label** entry to `docs/reference/glossary.md`.
   - Generate the CLI reference.
6. **Vectors.** `tests/fixtures/relation_labels/vectors.json` with the `key`, `catalog`,
   `fold`, and `rank` sections, all consumed by Rust tests.

**Acceptance:**

- On the live vault, read-only, `bob relations` lists inspired by 3, implements 2, and
  duplicate of 1 (6 links). `-f json` matches the JSON shape.
- A scratch-vault hide/unhide round trip folds correctly across two host files, and
  concurrent appends from two processes both survive.
- On the live vault, `git check-attr merge` reports `union`.
- Theme tests are unchanged and green, and the full gate is green against the clean
  base.

### label-complete

**Repo:** bob-cli. Covers the `capture-complete` part of RV7.

1. **Fast catalog.** `relation_label_suggestions` reads the `Catalog` (no `discover`).
   The relation task pickers keep their current path. Record before and after
   `hyperfine` numbers in the phase note. If the budget is missed, add the cache.
2. **Fields.**
   - Rows gain `key` and `inverse`; hidden labels are omitted.
   - The typed row gains `key`, `existing`, `count`, and `hidden`, and uses the catalog
     spelling on a key match.
   - The response gains `hidden_count`.
   - Replace the `eq_ignore_ascii_case` dedupe with key equality.
3. **zsh** (`capture_complete/shell.rs`): describe labels as `N links`. Hidden labels
   are absent.
4. **Docs and tests.**
   - Update `docs/task-relations.md` §12 and §16 and `docs/reference/json-outputs.md`.
   - Add JSON snapshot tests for a case variant (`Inspired By` → `inspired by`), a
     hidden exact match, a hidden prefix match (absent), and `hidden_count`.

**Acceptance:** the snapshots pass, the budget is met with numbers recorded, and the
full gate is green.

### rewrite-txn

**Repo:** bob-cli. A pure refactor with no behavior change.

1. **Extract** the transaction steps of `src/native/pomodoro_themes/rename.rs` into
   `src/native/vault_rewrite/`:
   - bounded lock wait (`lock_busy`);
   - the worktree check (`not_a_worktree`);
   - pre-sync unless offline (`sync_failed`);
   - planning through a caller closure over the synced files;
   - before-image snapshots and the stale check (`stale_note`);
   - dirty-path rules (offline refuses with `dirty_paths`; online notes pre-sync
     inclusion);
   - atomic writes with rollback;
   - a post-write hook for the journal append;
   - a verify hook that restores on failure (`verify_failed`);
   - one commit with the caller's subject, unstaging on failure, and
     committed-despite-error handling;
   - post-sync, with "sync pending" reported separately.
2. **Move `bob themes rename` onto it** with identical JSON, human output, and exit
   codes. The existing theme rename tests stay unchanged and green.
3. **Unit-test the transaction** with a fake plan: stale abort, verify-failure restore,
   offline dirty refusal, one commit, and post-sync failure.
4. **Scope.** Do not migrate `bob ref rename`. Leave a `PROPOSED FOLLOW-UP:` note
   instead.

**Acceptance:** the theme rename suites pass byte-for-byte, the new transaction tests
pass, and the full gate is green against the clean base.

### double-pipe

**Repo:** bob-cli. Covers RV6 and the capture part of RV7.

1. **Parse** (`src/native/capture_language/relations.rs` and the parse JSON):
   - `||label`, `||"multi word"`, and `~~` forms;
   - `relations[].hidden`;
   - a separator span covering both pipes;
   - the empty `||` incomplete need with `hidden: true`;
   - the `|||` teaching error;
   - unrelated pipes and fenced text left untouched.
2. **Rewrite.** Glue ` ||` the way `attach_relation_label` glues ` |`, and update the
   `Fix ~tasks:target ||` exclusion in `src/native/capture_language/tests/rewrite.rs`.
3. **Completion.**
   - A `||` probe is the `relation_label` context with `one_off: true`, and its
     `replacement` keeps `||`.
   - Every row and the typed row gain `hide_replacement`.
   - `relations list` JSON gains `hide_replacement`.
   - zsh completes after `||` too.
4. **Execution.**
   - Dry run: `relation_label_hides[]`, the `hides_existing_relation_label` warning, and
     the human `🪢 hides …` line.
   - Commit: after the notes commit succeeds, append one `relation_label.hide` per
     distinct key that is not already hidden (`via: bob-cli`).
   - Use the same lock discipline as `bob relations hide`. If capture already holds the
     maintenance lock, append within it.
   - A failed append reports `relation_label_hide_failed` with the retry command, and
     the capture still succeeds.
5. **Docs.**
   - Update `docs/capture.md` (relation grammar and capture-parse table),
     `docs/reference/capture-syntax.md`, and `docs/task-relations.md` §12 and §16.
   - Add `LD` vectors.
   - Shell examples quote the draft, so the shell never sees `||`.

**Acceptance:**

- Vectors pass for `~`/`~~` × `|`/`||`, quoted labels, ` ||` glue, empty `||`, `|||`,
  Unicode byte ranges, and untouched pipes.
- In a scratch vault:
  - `bob capture '… ~x:y||oneoff'` writes the plain `oneoff` LINKS label and one hide
    event;
  - an existing visible label warns in JSON and human output;
  - an already-hidden label writes no event;
  - a read-only journal directory still captures and reports the warning;
  - `-d` writes nothing.
- The full gate is green.

### label-rename

**Repo:** bob-cli. Covers RV8.

1. **CLI.** `bob relations rename OLD NEW [-d] [-f] [-m] [-o]` plus the hidden `--via`.
   Register it in the completion kinds; OLD and NEW complete labels from the catalog.
2. **Planner.** Run the strict RV8 plan over the RV2 scope with the LW4 per-edge
   relabel, and update LINKED FROM label spans on writable targets. Apply the rules for
   case-only, `unchanged`, `label_exists`, `--merge`, and the hidden-state move.
3. **Transaction.** Run on `vault_rewrite`, append the `relation_label.rename` event,
   and use the commit subject `bob relations rename: OLD → NEW`.
4. **Preview and JSON.** Produce the human dry run (one line per entry, before → after,
   plus the counts, state, and inverse lines) and the JSON that clients consume,
   including `replacement` and `hide_replacement`. Add `renamed_reappeared` to `list`.
5. **Docs.**
   - Add the §16 rename contract with the refusal table and the `git revert` runbook.
   - Amend §13 guarantee 6 and §6 for the label-span-only LR7 exception.
   - Mention the command in `docs/vault-git-sync.md` operations.
   - Add `LN` vectors.

**Acceptance:** scratch-vault tests cover:

- a label used in open, closed, `done/`, previous-daily, and nested-project notes;
- mirrors rewritten;
- case-only; `unchanged`; `label_exists`; merge;
- malformed-line refusal; stale-note abort and restore; verify-failure restore;
- the hidden-state move;
- exactly one commit, with `git revert` restoring every byte;
- every status, `^id`, field, and freshness stamp byte-identical (inertness).

No live-vault rename is run, and the full gate is green.

### mac-vocab

**Repo:** bob-mac-capture. Covers RV5 states C–F, RV6 in the app, and version skew.

1. **Decode** (`Sources/CaptureCore/CaptureModels.swift`, `decodeIfPresent`):
   - row `key`, `inverse`, and `hide_replacement`;
   - typed `existing`, `count`, and `hidden`;
   - `hidden_count` and `one_off`;
   - `relations[].hidden`;
   - dry-run `relation_label_hides` and the warning codes;
   - a `bob relations list` model.
2. **Process lanes** (`BobProcessClient.swift`):
   - `relationsList(hiddenOnly:)` on a cancellable `relation-labels` lane, prefetched on
     open when `hidden_count > 0`;
   - `relationsHide` / `relationsUnhide` with `--via bob-mac-capture -f json` on a
     serial, non-cancelling `relation-label-pref` lane;
   - an older bob maps to "Update bob to hide relation labels".
3. **Card behavior per RV5:**
   - Ctrl+D ghost rows: optimistic, restored with bob's message on failure, auto-repeat
     ignored, and a pending write survives dismissal.
   - The one-off toggle (Ctrl+D with no selection, a typed `|`, Backspace on an empty
     field).
   - The card opens in one-off mode when it opens on a `||` token.
   - The Hidden section with the `◌ N hidden` pill and Ctrl+H (`hidden_key`).
   - Capsule states, the inverse phrase, and state-D's amber warning with
     `⌃D keep visible`.
   - Footers per state.
   - Insertion uses bob's `replacement` or `hide_replacement`.
4. **Outside the card:**
   - the live preview shows bob's `hides_existing_relation_label` message as an amber
     row (`docs/capture-previews.md`);
   - the success notification appends `· hid “x” from relation suggestions`;
   - an older bob shows no pill, Ctrl+D and Ctrl+H give the update hint, and `||` is
     never downgraded.
5. **Fixtures, tests, and docs.**
   - Regenerate fixtures from real bob output on a scratch vault.
   - Test decoding, presentation, ghost and undo, the one-off rules, the warning state,
     and skew.
   - Update `docs/keyboard.md`, `docs/relationship-pickers.md`,
     `docs/capture-previews.md`, and the `docs/protocol.md` compatibility rows.

**Acceptance:** `just portable-check` is green, the Swift is compiled per `AGENTS.md`,
macOS CI is green, the docs match the router, and screenshots of states C–F are attached
if the mac is reachable.

### obsidian-vocab

**Repo:** bob-plugins (bob-ledger-tools and bob-navigation-hotkeys). Covers RV9 without
rename.

1. **Vectors.** Copy `relation_labels/vectors.json` with its README. The JS `key`,
   `catalog`, `fold`, and `rank` sections must pass.
2. **bob-ledger-tools.**
   - Relation-index label counts use the shared key and display spelling.
   - Add the read-only journal fold, refreshed on load, on adapter changes to
     `.bob/relations/`, and after its own writes.
   - Add `api.relationLabels` v1 (rows, hidden count, `onChange`, hide/unhide).
   - `labelSuggestions` omits hidden labels.
   - Document the API in `docs/develop/plugin-apis.md`.
3. **The nav relation-label stage** (`426-picker-relation-stage.js`,
   `305-relation-stage-copy.js`):
   - Rows show `N links` instead of `used N×`, inverse phrases, `eye.slash` for hidden
     rows, and the `⇥` marker.
   - Keys: Tab, Ctrl+D (ghost and undo, or one-off on row 0), a typed `|`, and Ctrl+H
     (`hidden_key`).
   - Add the amber warning when one-off would hide an existing visible label.
   - The one-off commit then hides with `allowUnused`.
   - Footers follow the card copy. Mobile shows a "needs the desktop app" Notice for
     writes.
4. **Docs, tests, deploy.** Update `docs/bob-navigation-hotkeys.md` (including both
   meanings of Ctrl+D), `docs/task-relations.md`, and `docs/bob-ledger-tools.md`. Add
   `scripts/test-*.cjs` coverage listed in `package.json`, run `npm test`, then
   `bob plugins sync`.

**Acceptance:** `npm test` passes, apart from known baseline failures. On desktop, with
a stubbed bob, Ctrl+D hides and undoes, the Hidden section toggles, and a one-off link
commits and then hides.

### mac-rename

**Repo:** bob-mac-capture. Covers RV5 state G.

1. **Rename mode.**
   - Ctrl+R opens it for the selected row, or for the typed label when that label is an
     existing key. The field is prefilled with OLD and selected.
   - A live dry run (`bob relations rename OLD NEW -d -f json --via bob-mac-capture`)
     runs on a cancellable `relation-rename-preview` lane, debounced at 150 ms. It
     renders bob's counts, state, and inverse lines.
   - Refusal messages render inline.
   - A merge preview arms on the first ↩
     (`↩ again to merge into “implements” · 5 links`) and applies on the second.
   - Esc returns to choose mode and writes nothing.
2. **Apply.**
   - Run on the serial `relation-label-pref` lane with a 60 s timeout, behind a progress
     state (`Renaming… syncing vault`).
   - On success: refetch the catalog, select NEW, and set the status to
     `Renamed “A” → “B” · N links in M notes · 1 commit`, plus `· sync pending` when
     that applies.
3. **Draft fix-up.** After success, rewrite every relation in the current draft whose
   label key was OLD. Use the `label_range` byte ranges from `capture-parse`
   `relations[]` and bob's `replacement` / `hide_replacement` from the rename JSON, so
   the app makes no grammar decisions.
4. **Skew.** An older bob without `relations rename` shows
   `Update bob to rename relation labels` on Ctrl+R.
5. **Tests and docs.** Test preview staleness, merge arming, failure, the draft fix-up,
   and skew. Update `docs/keyboard.md`, `docs/relationship-pickers.md`, and
   `docs/protocol.md`.

**Acceptance:** `just portable-check` is green, the Swift is compiled per `AGENTS.md`,
macOS CI is green, and a state-G screenshot is attached if the mac is reachable.

### obsidian-rename

**Repo:** bob-plugins.

1. **`api.relationLabels.rename`** spawns `bob relations rename … -f json`, with
   `dryRun` and `merge`.
2. **Stage rename mode.**
   - Ctrl+R in the relation-label stage gives the Mac's rename mode: prefilled input,
     the dry-run preview list (files and before/after entries), merge arming, and inline
     refusals.
   - Before preview and before apply, save the active editor. Refuse when an open editor
     with unsaved changes holds an affected note: the live relation index overlays
     unsaved text, and a disk hash cannot protect it.
   - Finish with the Notice `Renamed “A” → “B” in N notes · one commit`.

> [!decision] obsidian_manager 3. **Command `Bob: Relation labels`** (no default
> hotkey), a `FilteredPickerModal` that reuses the stage's rows and keys without a link
> target:
>
> - Visible rows first, then a Hidden section (Ctrl+H, `hidden_key`).
> - Ctrl+D hides or unhides in place, with a ghost and undo.
> - Ctrl+R or Enter opens rename mode for the selected label.
> - Mobile shows the desktop-only Notice.

> [!decision] obsidian_manager = no 3. **No standalone command.** Rename and hide from
> Obsidian happen only inside the relation-label stage.

4. **Docs, tests, deploy.** Update `docs/bob-navigation-hotkeys.md` (keymaps and the
   command) and `docs/task-relations.md`. Test against a stubbed bob, run `npm test`,
   then `bob plugins sync`.

**Acceptance:** `npm test` passes, apart from known baseline failures. A scratch-vault
flow test drives preview → apply through a real bob in a temp vault and gets exactly one
commit. The dirty-editor refusal is covered.

### records

**Repos:** bob-cli memory and docs.

> [!decision] memory_decisions Through `/sase_memory_write`, add four `decisions`
> strands, written from what shipped. Each gives where it applies, the claim, why, the
> rejected alternatives (from the research's "Alternatives considered and rejected"),
> the cost, what reopens it, and evidence: this plan, the research ref, and the landed
> commits.
>
> 1. `relation-label-hide-is-suggestions-only`: hiding removes a label from suggestions
>    only, and use never unhides it. `||` hides after a successful capture and warns,
>    never blocks, on an existing label. This is the twin of
>    `theme-hide-is-recent-only`.
> 2. `relation-label-prefs-live-in-per-host-journal`: `.bob/relations/<host>.jsonl`,
>    union-merged, folded by the shared PT4 module, with bob as the only writer.
> 3. `relation-label-rename-rewrites-history`: one previewed transaction and one
>    revertable commit, rewriting label spans in `done/` and the previous daily note
>    (the recorded LR7 exception).
> 4. `relation-card-tab-accepts-ctrl-d-hides`: Ctrl+N/P navigate without touching the
>    query, Tab accepts or completes the top match, selection is by key, and Ctrl+D
>    hides what Return would insert.
>
> Run `sase memory init`.

> [!decision] memory_glossary Through `/sase_memory_write`, add one sentence to
> `glossary:task-relation`: a label can be hidden from suggestions (Ctrl+D,
> `bob relations hide`, or a one-off `||`) and renamed everywhere with
> `bob relations rename`. Run `sase memory init`.

**Also:**

- Drop every `planned (phase …)` marker from `docs/task-relations.md` §16.
- Check the cross-links between `docs/capture.md`, the reference pages, the Mac docs,
  and the plugin docs, then run `just docs-check`.
- Run the end-to-end acceptance below and record the results in the phase note.

## End-to-end acceptance

1. **Chooser.** In the Mac card with three labels, ⌃N ⌃N Tab inserts `|implements`.
   Repeated ⌃N cycles every visible label and wraps to the field, at any typing speed.
2. **Hide.** Ctrl+D on `implements` in the Mac card:
   - `bob relations` on apollo shows it hidden after vault-sync;
   - the Obsidian stage omits it;
   - Ctrl+H shows it;
   - Ctrl+D unhides it.
3. **One-off.** In a scratch vault, `~x:y||prompted by` captures the plain label and
   hides it. `~x:y||inspired by` warns in the card, the preview, the CLI, and the
   notification.
4. **Rename.** A scratch rename of `inspired by` → `prompted by` previews identically
   from the CLI, the Mac card, and the Obsidian stage, then makes one commit that
   `git revert` fully undoes.
5. **Agreement.** Keys, counts, display spellings, and rank order match across
   `bob relations -f json`, `capture-complete`, and `api.relationLabels` on the vectors
   and on the live vault (read-only).
6. **Performance and docs.** The latency budget holds with numbers. `just docs-check` is
   green, and the CLI reference lists `bob relations` under Tasks and projects.

## Risks

- **Card regressions in a 10k-line model.** Keep the change scoped to the relation card
  state. The filtering fake client makes the collapse class of bug impossible to
  reintroduce silently.
- **Concurrent theme work (bob-cli-bc)** touches the same Swift files, plugin pickers,
  and `pomodoro_themes`. Phases rebase over it, reuse its landed pieces, and keep theme
  behavior byte-identical.
- **The LR7 exception.** Rewriting `done/` is a deliberate break of a stated guarantee.
  It is limited to label spans, verified by rescan, revertable as one commit, and
  recorded.
- **Ctrl+D and Ctrl+H muscle memory.** Both are claimed only inside the card or stage,
  and only for a one-line field. `hidden_key` offers Ctrl+Shift+H.
- **Journal allowlist lag.** A host that has not synced `.gitignore` keeps its hides
  local until it does. This is documented.
- **Clock skew.** Last writer wins across hosts and trusts the clocks. That is an
  accepted risk for one user, as with themes.
- **Low usage** (3 labels today). That is why the fix ships first and alone. If scope
  must shrink, cut the rename UIs (`mac-rename`, `obsidian-rename`) before the
  `bob relations rename` CLI.
