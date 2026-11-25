---
tier: epic
title: One tie per task pair — dependencies absorb plain links
goal: "A task pair can hold a dependency or a plain (unlabelled) link, never both, in
  either direction. Every Bob writer enforces it. Reconcile heals hand edits in the
  dependency's favour. The four Obsidian relationship stages show each pair's tie at a
  glance and convert it with Ctrl plus the Task Card key, and capture plus Bob Mac
  Capture explain the rule.

  "
decisions:
  ctrl_shift_l:
    ask:
      Should the stages also take ⌃⇧L ("it links to this") so all four Task Card
      relationship keys work as Ctrl chords?
    default: true
    why: Completes the ⌃+Task-Card-key grammar (b ⇧B l ⇧L) at almost no cost
    answer: true
  memory_decision_record:
    ask:
      Add a decisions memory record (dependency-absorbs-plain-link) for this policy once
      it ships?
    memory:
      - decisions:dependency-absorbs-plain-link
    default: false
    answer: false
  memory_glossary_relation:
    ask:
      Add a one-sentence plain-link exclusivity rule to the Task Relation glossary
      strand?
    memory:
      - glossary:task-relation
    default: false
    answer: true
phases:
  - id: tie-contract
    title: Contract, shared vectors, and the Rust tie helpers
    depends_on: []
    size: medium
    description:
      "tie-contract: write the plain-link/dependency contract and PT vectors in bob-cli
      docs, add the shared vector fixture, and add the pure Rust helpers that classify a
      pair's tie and drop plain entries byte-exactly."
  - id: reconcile-fold
    title: Reconcile fold and merge pruning
    depends_on:
      - tie-contract
    size: medium
    description:
      "reconcile-fold: bob task reconcile drops plain LINKS entries on dependency-joined
      pairs and reports absorbed_plain_links. bob task merge prunes the overlaps its
      repointing would create. Dry-run the fold against the live vault without applying
      it."
  - id: capture-writers
    title: Capture absorption, the plain-~ no-op, and the same-item error
    depends_on:
      - tie-contract
    size: medium
    description:
      "capture-writers: & and * absorb the pair's plain links in the same staged write.
      A plain ~ / ~~ over a dependency becomes a reported no-op, and & plus a plain ~ to
      one target in a single item is a teaching error. Both report additive JSON and
      human output."
  - id: capture-complete
    title: capture-complete tie fields
    depends_on:
      - tie-contract
    size: small
    description:
      "capture-complete: relation candidates carry dependency_tie in both directions
      (also_prerequisite stays as a deprecated alias), and & / * candidates carry
      plain_link."
  - id: nav-tie-planner
    title: Nav tie planner, mirrored vectors, and absorption in the dependency writers
    depends_on:
      - tie-contract
    size: medium
    description:
      "nav-tie-planner: add the pure JS twins and planTieChange in
      bob-navigation-hotkeys, pinned by mirrored PT vectors. Every existing nav
      dependency writer (Depends on, Blocks, counted, marked, api) then absorbs plain
      links."
  - id: nav-tie-visibility
    title:
      Tie pills, the ⇄ badge, tie bands, subtitles, and the Depends-on consequence strip
    depends_on:
      - nav-tie-planner
    size: medium
    description:
      "nav-tie-visibility: all four stages show each row's tie in both directions in a
      new pill cluster, mark converting rows with ⇄, list the other kind of tie in an
      empty-filter band, count it in the subtitle, give Depends on its consequence
      strip, and add the hand-edit notice."
  - id: nav-tie-writer
    title: applyTieChange — release, reverse, and convert writes
    depends_on:
      - nav-tie-visibility
      - reconcile-fold
    size: medium
    description:
      "nav-tie-writer: one guarded writer runs any planned tie change. It writes the
      dependency side first, keeps same-note changes in one undo step, reports honest
      notices and follow-up suffixes, and fixes reopenDependencyStageFresh."
  - id: nav-tie-keys
    title: The ⌃B / ⌃⇧B / ⌃L chords, ↵ conversions, and the armed release
    depends_on:
      - nav-tie-writer
    size: medium
    description:
      "nav-tie-keys: wire the Ctrl chords and ↵ conversions in all four stages with
      strict modifier helpers. A ↵ release takes two presses. Batches and counted
      sessions refuse conversions, and the label stage guards its plain row. Add the
      footers and docs, then deploy with bob plugins sync."
  - id: mac-tie-capsules
    title: Bob Mac Capture tie capsules and report rows
    depends_on:
      - capture-writers
      - capture-complete
    size: medium
    description:
      "mac-tie-capsules: decode the new capture and capture-complete fields, render the
      tie capsules, summaries, and report rows, refresh the fixtures and docs, and watch
      the exact-SHA macOS CI run to green."
proposed_by: bbugyi200.apollo.research.6f.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-25 18:16:39
status: wip
---

- **PROMPT:**
  [prompts/202611/plain_link_dependency_ties.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/plain_link_dependency_ties.md)

# Plan: One tie per task pair — dependencies absorb plain links

## Context

Bryan wants the state "A depends on B **and** A has a relationless link to B" made
impossible. He agreed to every requirement in the research report
`research:202611/plain_link_dependency_exclusivity/plain_link_dependency_exclusivity.md`
(read it with `sase artifact read` before starting any phase). The one exception is
memory, which changes only through this plan's memory decisions. This plan carries the
report's adjustments R1–R8. Where the report left room, the plan settles each design
choice.

Repositories:

- **bob-cli** is the primary repo. It holds the contract docs, reconcile, capture,
  capture-complete, and merge.
- **bob-plugins** is a linked repo; open it with `/sase_repo`. Its
  `bob-navigation-hotkeys` plugin owns the four relationship stages. Read its
  `AGENTS.md`. Edit the `src/` fragments only and never `main.js`. Each fragment has a
  1000-line cap, so put new code in new fragments. These fragments are already at or
  near the cap: `470` (1000), `426`, `350`, `700` (999), and `428` (996). Run
  `npm run build`, add new test files to `test:unit`, run `npm test` and
  `npm run docs:check`, then `bob plugins sync`.
- **bob-mac-capture** is a linked repo, a thin client of `bob`. Read its `AGENTS.md`.
  Only macOS CI gates the macOS targets.

Rules from memory that every phase follows:

- Never edit the bob-cli `Cargo.toml` version or `CHANGELOG.md` (the daily release train
  owns them). Every JSON change here is additive.
- Read `sase/memory/docs.md` (via `/sase_memory_read`) before touching docs.
- Run `just docs-gen` when clap help changes.
- Run `just docs-check`.

## The rule

**Vocabulary.**

- **Plain link:** a LINKS entry with no label. An alias label counts as a label (LK5).
- **Labelled relation:** any LINKS entry with a label.
- **Tie:** the exclusive relationship between two tasks, either a dependency or a plain
  link, in either direction.
- **Absorb:** adding a dependency removes the pair's plain links.
- **Release:** turning a dependency back into a plain link.
- **Reverse:** swapping which task waits on which.

**Invariant (pair-level, both directions).** `D(A,B)` means A's Depends-On names B.
`P(A,B)` means A's LINKS line holds a plain entry for B.

```text
Forbidden on LINKS lines of open tasks in writable notes:
    (D(A,B) or D(B,A)) and (P(A,B) or P(B,A))
Always legal: labelled relations beside a dependency; mentions (M7 unchanged).
```

The rule is structural, not keyed on status. A dependency on a completed prerequisite,
or one whose Depends-On line sits in an archived note, still ties the pair. A few things
are never auto-edited:

- `done/` and the previous daily note;
- a closed task's own lines;
- anything Bob is unsure about: malformed Depends-On or LINKS lines, duplicate block
  ids, uncertain dependents, quiet-deferred notes.

**Asymmetry (R3).** Adding a dependency absorbs a plain link in every writer. That loses
nothing. Releasing a dependency can unblock a task, possibly not the one on screen, or a
whole `^prj` project. So only an explicit Obsidian stage gesture releases, and only
after its consequence is on screen. Capture, agents, hand edits, and reconcile never
release.

**Writer table** (the contract phase publishes it):

| Writer                                                 | Dependency added over a plain link          | Plain link added over a dependency                         |
| ------------------------------------------------------ | ------------------------------------------- | ---------------------------------------------------------- |
| Nav Depends on / Blocks ↵, counted, marked, plugin api | absorb, same gesture                        | —                                                          |
| Nav ⌃B / ⌃⇧B (any stage)                               | absorb (and reverse an opposite dependency) | —                                                          |
| Nav ⌃L (and ⌃⇧L) / Links or Linked-from ↵ on a ⇄ row   | —                                           | release: chord commits on one press, ↵ arms a second press |
| Nav ⌃↵ label stage, empty label                        | —                                           | refused while a dependency ties the pair                   |
| Capture `&` / `*`                                      | absorb in the same all-or-nothing write     | —                                                          |
| Capture plain `~` / `~~`                               | —                                           | reported no-op (`subsumed_by: "dependency"`)               |
| Capture `&T ~T` in one item                            | teaching error                              | teaching error                                             |
| `bob task merge` repoint                               | prune in the merge transaction              | —                                                          |
| `bob task reconcile`                                   | fold: dependency wins, reported             | —                                                          |
| Hand edits                                             | reconcile folds; nav shows one notice       | reconcile folds; nav shows one notice                      |

## The Obsidian experience

The Task Card opens four stages: Depends on (`b`), Blocks (`⇧B`), Links (`l`), and
Linked from (`⇧L`). In this section, "this task" is the stage's owner O and "it" is the
highlighted row X.

### Key grammar: Ctrl plus the Task Card key

Inside any of the four stages, Ctrl plus a Task Card key **sets the pair's tie** to that
relationship. The chords are desired-state commands, never toggles. Pressing one when
the pair already holds that tie writes nothing and shows a quiet notice ("Already waits
on it"), so a double press never flips the state back.

| Chord   | Result for O and X                                                                                                                            |
| ------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| **⌃B**  | ⛓ This task waits on it. Absorbs plain links both ways, reverses an opposite dependency, and keeps labelled relations.                       |
| **⌃⇧B** | 🧱 It waits on this task. Same absorption and reversal.                                                                                       |
| **⌃L**  | 🪢 Releases any dependency between them. If no relation already joins them, adds a plain link O→X. A labelled O→X entry already satisfies it. |
| **⌃⇧L** | 🪢 Same as ⌃L, but adds the plain link X→O.                                                                                                   |

> [!decision] ctrl_shift_l = no Drop the ⌃⇧L row. ⌃L is the only link chord, adding O→X
> from every stage, and the footers, docs, and vectors list three chords instead of
> four.

`↵` keeps each stage's native meaning. On a row whose pair is tied the other way, the
row is marked `⇄` and ↵ converts:

| Stage (native edge)       | untied    | plain O→X          | plain X→O          | ⛓ O waits on X                       | 🧱 X waits on O                      |
| ------------------------- | --------- | ------------------ | ------------------ | ------------------------------------- | ------------------------------------ |
| Depends on (O waits on X) | add       | ⇄ add + absorb     | ⇄ add + absorb     | CURRENT: remove                       | 🔒 cycle (strip hints `⌃B reverses`) |
| Blocks (X waits on O)     | add       | ⇄ add + absorb     | ⇄ add + absorb     | 🔒 cycle (strip hints `⌃⇧B reverses`) | CURRENT: remove                      |
| Links (O→X)               | add plain | CURRENT: unlink    | add O→X (as today) | ⇄ release to O→X, **armed**           | ⇄ release to O→X, **armed**          |
| Linked from (X→O)         | add X→O   | add X→O (as today) | CURRENT: remove    | ⇄ release to X→O, **armed**           | ⇄ release to X→O, **armed**          |

**When a press commits.** A dependency is removed only after its consequence has been on
screen:

- Absorbing (adding a dependency) commits on one press with any key.
- A chord release or reversal commits on one press. The strip already shows its result
  (below).
- An ↵ release arms first. The strip turns amber and reads
  `⚠ Removes the dependency · "X" becomes Ready — ↵ again · esc keeps it`. The second ↵
  commits.

The arming reuses the cancel and merge confirm-nonce machinery in
`426-picker-relation-stage.js`. Moving the selection, editing the query, pressing Esc,
or entering a stage disarms it.

**Guards** reuse what exists:

- dependency eligibility (closed, archived, and daily-note Event Tasks cannot become
  prerequisites), with the refusal reason shown as a notice;
- trial-graph cycle checks, which exclude a reversed edge;
- the `^prj` project guards;
- the `＋ id` block-ID prompt;
- stale-row refusal;
- refusing a malformed Depends-On or LINKS line.

Conversions act on one pair only. With Tab marks or in a counted session (R8), each
chord refuses with `⌃B acts on one task · clear marks first` (or `needs a single task`).
A marked Links or Linked-from batch that contains a ⇄ row refuses before writing, naming
the row: `↵ can't replace a dependency in a batch · unmark "X" or use ⌃L on it`. Marked
Depends-on and Blocks batches absorb normally, because absorbing loses nothing.

### Seeing the tie

These changes use the existing glyphs, add no new palette, and keep line heights:

- **Tie pills on every row, both directions, in all four stages.** Each stage hides the
  pill for its own native edge:

  | Pill              | Meaning               |
  | ----------------- | --------------------- |
  | `⛓ prerequisite` | this task waits on it |
  | `🧱 dependent`    | it waits on this task |
  | `🪢 linked`       | plain O→X             |
  | `🪢 links here`   | plain X→O             |
  - In the dependency stages, labelled relations also show as muted pills: the outgoing
    label, or `← label` for an incoming one.
  - Each tooltip states the fact plus the key that converts it.
  - The pills move out of the 190px `.bob-cnp-task-badge` into a trailing
    `.bob-cnp-row-ties` cluster, where each pill truncates with an ellipsis. This fixes
    today's hard clip. The badge keeps only the action glyph.
  - The tooltip `Also a prerequisite · linking stays inert` is retired, and so is the 🍅
    tooltip's `linking stays inert`.

- **The `⇄` action badge** replaces `＋` whenever ↵ would convert. The stage's verb
  stays: `⇄ add`, `⇄ block`, `⇄ link`, `⇄ link here`. Its tooltip names what gets
  replaced.
- **A tie band** sits between CURRENT and RESULTS. It appears only when the filter is
  empty and only when it has rows:
  - `TIED BY A DEPENDENCY` in Links and Linked from;
  - `PLAIN LINKS` in Depends on and Blocks.

  It lists the owner's ties of the other kind, in both directions. Its rows move out of
  RESULTS rather than repeating there, and they are fully actionable.

- **Subtitles count the other kind:** `Links to 2 · 1 tied by a dependency · …` and
  `Blocks 2 · frees 1 · 1 plain link · …`. The Links subtitle drops `does not block`,
  because ⌃B now can.
- **The Depends-on stage gets a consequence strip**, matching the Blocks copy style. It
  is the only one of the four stages without one today. Its copy:
  - CURRENT: `Stops waiting on it · Ready again`, `· still Blocked (waits on N)`, or
    `· status unchanged`. It ends with `· ⌃L keeps a plain link`.
  - Open result: `Waits on it · this task becomes Blocked`, then the lane phrase
    (`it takes Next`), then `⚠ removes this task's link from JOB 10:10–10:35`. When the
    task is already Blocked: `Also waits on it (N open)`.
  - Closed prerequisite: `Waits on it · already done · status unchanged`.
  - `⇄` rows add the prefix `Replaces the 🪢 link · `. A labelled pill adds the suffix
    `· keeps "follow-up of"`.
  - Cycle-locked 🧱 rows: `⟲ It waits on this task · ⌃B reverses`.
  - A `^prj` owner uses the Blocks project copy.
  - Copy always comes from the planned result, never from the gesture.
- **Strips on tied rows in the other stages.**
  - In Links and Linked from, a ⛓ or 🧱 row shows the release outcome, e.g.
    `⇄ Makes it a plain link · stops waiting on it · Ready again`.
  - A chord whose outcome differs from ↵ appears as one muted trailing segment, e.g.
    `· ⌃⇧B reverses · it becomes Blocked`.
  - A `^prj` release names the fan-out: `unblocks 12 residents of ‹sase›`.
- **Footers** gain one contextual hint group listing the chords that would change the
  highlighted row: `⌃B wait on it`, `⌃⇧B make it wait`, `⌃L plain link`,
  `⌃⇧L link here`. A chord is listed only when it differs from ↵, is not a no-op, and is
  not guarded. The footer stays on one line at the default modal width. When space runs
  short, drop `↑↓ navigate` first.
- **Notices name what was replaced:**
  - `⛓ Now waits on "X" · replaced the 🪢 link · Blocked`
  - `🧱 "X" now waits on this · replaced the 🪢 link`
  - `🪢 Now links to "X" · no longer waits on it · Ready again`
  - `⇄ Reversed · "X" now waits on this · this task is Ready again`
  - `⛓ Now waits on "X" · kept the "follow-up of" relation`
  - The failure suffixes are `· 🪢 link not removed (reconcile will fold it)` and
    `· 🪢 link not added`.
- **Hand edits.** When nav's existing hand-edit path sees a hand-typed plain link or
  Depends-On entry create an overlap, it shows one notice:
  `"X" is already a dependency · reconcile will drop this plain link`. Nav writes
  nothing mid-keystroke.
- **Chips** (bob-ledger-tools) need no change. Under the invariant, a pair never renders
  two tie chips.

## Reliability rules (all engines)

- **One predicate, one drop.** Use one pair predicate, reusing `dependency_joins` (Rust)
  and the existing identity keys (JS `path\x00id` / `path#^id`). Do not mint a third
  equality. Use one byte-exact plain-entry drop, with the separator rules from the PT
  vectors: a non-first entry takes its preceding separator, and the first entry takes
  its following one. When a line ends up with no entries, delete it.
- **Undo.** A same-note conversion is one editor transaction and one Ctrl+Z. Never split
  the drop and the add.
- **Cross-note order: dependency side first.** The dependency write goes first, through
  the existing guarded writers with their preflight, guards, stamps, lane transfer, and
  Pomodoro consumption. If it fails, nothing else is written. The link side follows
  best-effort, as mirrors do today. A leftover plain link is folded by reconcile. A
  plain link that was never added leaves a valid, merely untied pair, and the notice
  says so.
- **No-ops and stamps.** A no-op is byte-identical, with no history entry and no
  freshness stamp. A tie change that adds or removes a dependency is a dependency write,
  so it follows dependency freshness and lane rules. A pure relation edit stays inert
  and unstamped.
- **Ids and fields.** Removing the last dependency clears `[dependsOn::]` with the line,
  or R2 re-adopts the edge. Never strip a target's `[id::]`. Mirrors (LINKED FROM,
  BLOCKS) get courtesy updates and are otherwise repaired by reconcile.

## Phase tie-contract — contract, shared vectors, and the Rust tie helpers

This phase writes the contract that every later phase implements, plus the Rust
primitives.

1. **`docs/task-relations.md`.** Add a new top-level section
   `## 22. Plain links and dependencies`, appended so that no existing section is
   renumbered. It covers:
   - the vocabulary and invariant above;
   - writable scope and protected history;
   - the asymmetry and the writer table;
   - a short key-grammar summary that points to `docs/task-dependencies.md` §6 and §15
     and to the plugin docs for stage detail;
   - the JSON field names, their row shapes, and the PT vectors (below).

   Also make these edits elsewhere in the file:
   - §1: add "plain link".
   - §2: add "A tie change that adds or removes a dependency is a dependency write, not
     a relation write" beside "A link that needs to block is a dependency".
   - §7 item 5: becomes "Reconcile never edits a LINKS line except the §22 fold, which
     drops plain entries on dependency-joined pairs". The fold is a narrow exception
     that must not grow into general relation rewriting.
   - §12: list the new output fields.
   - §13 guarantee 2: relation writes stay inert, and tie changes are carved out as
     dependency writes.
   - §13 guarantee 3: add the PT vectors.
   - M7 (line ~1865): becomes "Dependencies suppress unlabelled relation entries;
     labelled relations stay."

2. **`docs/task-dependencies.md`.** In §5, add one paragraph: a dependency absorbs plain
   links, and a pointer to §22. Do not touch the stage sections (§6 and §15); the nav
   phases own them.
3. **Shared vector fixture.** Create `tests/fixtures/plain_link_ties/vectors.json`.
   Mirror its schema on `tests/fixtures/task_mentions/vectors.json`. It holds two
   groups:
   - `drop` (PT1–PT12, line text in and line text out);
   - `pair` (PT13–PT21, resolved edge sets in and a tie classification plus the entries
     to drop out).

   The docs §22 table lists every PT id, including the JS-only planner vectors PT22 and
   up, which `nav-tie-planner` writes.

   Starting set (extend it as needed, but keep these ids):

   | #         | Input                                                      | Outcome                                                    |
   | --------- | ---------------------------------------------------------- | ---------------------------------------------------------- |
   | PT1       | `🪢 **LINKS:** [[#^a]]`, drop ^a                           | line deleted                                               |
   | PT2       | `[[#^a]] • see [[#^b]]`                                    | `see [[#^b]]`                                              |
   | PT3       | `see [[#^b]] • [[#^a]]`                                    | `see [[#^b]]`                                              |
   | PT4       | `see [[#^b]] • [[#^a]] • also [[#^c]]`                     | `see [[#^b]] • also [[#^c]]`                               |
   | PT5       | `follow-up of [[#^a]]`                                     | unchanged (labelled)                                       |
   | PT6       | `[[#^a\|merged into]]`                                     | unchanged (alias label, LK5)                               |
   | PT7       | `see [[#^a]] • [[#^a]]`                                    | `see [[#^a]]` (only the plain LK8 duplicate goes)          |
   | PT8       | `~~[[#^a]]~~ • see [[#^b]]`, `![[#^a]] • see [[#^b]]`      | struck or embedded plain entry dropped                     |
   | PT9       | `,` / `·` / whitespace separators                          | entry plus its own separator dropped; all other bytes kept |
   | PT10      | `[[other#^a]]` from another note vs a same-note `[[#^a]]`  | matched by resolved path + id                              |
   | PT11      | malformed `[[#^a]] until Friday`                           | untouched                                                  |
   | PT12      | CRLF, 4-space child indent, no final newline               | preserved                                                  |
   | PT13–PT17 | none / plain O→X / plain X→O / O waits on X / X waits on O | classified, nothing to drop                                |
   | PT18      | O waits on X and O plain X                                 | drop O's entry                                             |
   | PT19      | O waits on X and X plain O (the live `bob.md` shape)       | drop X's entry                                             |
   | PT20      | O waits on X and O labelled X                              | legal                                                      |
   | PT21      | O waits on X, O labelled X, and O plain X                  | drop the plain entry only                                  |

4. **Rust helpers.** Add a new module `src/native/task_relations/ties.rs`, re-exported
   from `task_relations`. It holds:
   - `drop_plain_link_entries`: pure and byte-exact over one LINKS line, given a
     predicate on the resolved target identity. It returns `Unchanged`, `Rewritten`
     (plus the removed identities), `DeleteLine` (plus the removed identities), or
     `Malformed`. Build it on `block_link_spans` and the existing entry and label spans
     in `task_relations/parse.rs`.
   - A `PairTies` classification computed from already-resolved edge sets.

   Unit tests run every `drop` and `pair` vector from the JSON fixture.

5. **Done when:**
   - `just test` passes (or the project's standard Rust test and clippy recipes).
   - `just docs-check` passes.
   - No behaviour changes yet.

## Phase reconcile-fold — reconcile fold and merge pruning

1. **The fold.** It lives in `src/native/task_status_hooks/relations.rs`, or in a
   sibling `relations_fold.rs` (the file is already 1342 lines). Order inside
   `RelationsPass::run()`:
   - Compute `DependencyState` unconditionally (today it exists only when mentions are
     on).
   - Fold **before** `collect_edges`, so that mints and the LINKED FROM projection see
     the post-fold text.
   - Re-parse the touched files.

   For each accepted LINKS child of an **open** task S, in an active, writable,
   non-quiet-deferred note, drop every plain entry whose resolved target T satisfies
   `dependency_joins` in either direction. Generalise its signature so LINKS edges can
   use it. Dependency evidence may come from active or archived notes, at any status.

   Skip the fold when:
   - S or T carries a duplicate block id;
   - either task's Depends-On line is malformed (add a loc-keyed malformed set for
     id-less dependents);
   - the target is uncertain (thread `reconcile.uncertain_dependents` through both call
     sites: `sync.rs` and `task_status_hooks/mod.rs`);
   - S's LINKS line is malformed;
   - the target is unresolved.

   Follow the relations kill switch: no relations reconcile means no fold. Edits go
   through `touch()` / `apply_edits()`, so `original_contents`, the guarded output, and
   the quiet interval all apply. The pass is idempotent: a second run is clean.

2. **Reporting.**
   - JSON: add `absorbed_plain_links[]` to `SyncResult` in `model.rs`, next to the
     relation fields, using the §22 row shape
     `{path, line, source, target, dependency}`. `dependency` is
     `"source_waits_on_target"` or `"target_waits_on_source"`. Endpoints are
     `{path, line, block_id?, text}`. Update the test literal in `retry.rs`.
   - Human: add a count on the `Relations:` line in `output.rs` (`🪢 −N plain (⛓)`),
     plus a row group in `print_relation_line_sections`.
   - Classify the new array in `scripts/dev/reconcile-drift`.
3. **Merge pruning.** In `task_merge/retirement.rs::apply_retirement`, after
   `repoint_dependents` and the provenance step, drop the plain entries between each
   repointed dependent D and the target T, in both directions, inside the same `staged`
   map. Use the tie helper. Report them additively as `absorbed_plain_links` in the
   merge JSON. A merge postimage must still be a reconcile fixed point
   (`reconcile_merge_snapshot_for_tests`). Add a test where repointing creates an
   overlap.
4. **Tests.** Add CLI vectors in `tests/cli/task_status_hooks/relations.rs`, following
   the LR pattern (dry-run JSON, then apply, then bytes, then a clean second dry run):
   - LR27: same-note O→X plain.
   - LR28: reverse X→O, cross-note.
   - LR29: a labelled entry kept.
   - LR30: the only entry; the line is deleted and the target's LINKED FROM is
     re-projected in the same pass.
   - LR31: each uncertainty skip.
   - LR32: closed owner, previous daily note, or `done/`; all untouched.
   - LR33: archived dependency evidence; still folded.
   - LR34: kill switch off.
   - LR35: dry-run JSON equals the apply JSON.

   Also add a merge test in `task_merge/tests.rs`.

5. **Docs.** Update:
   - `docs/task-status-hooks.md`: the relations section, the JSON example, and the field
     glossary.
   - `docs/reference/configuration.md`: the line that says the kill switch "leaves LINKS
     alone".
   - The `relations.rs` module doc.
   - `docs/task-merge.md` §11.
   - The LR table in `docs/task-relations.md` §15.
6. **Live-vault check (read-only).** Run `bob task reconcile --dry-run` against the real
   vault with JSON output, and paste the `absorbed_plain_links` rows into the commit
   message or phase notes. The expected effect is the single `bob.md` pair from the
   research (`^ref-plugin-sync-unblocking` → `^no-bob-plugins-sync-qs`), with nothing in
   `done/`. **Do not apply.** The hooks apply the fold once the release train ships it.

## Phase capture-writers — capture absorption, the plain-~ no-op, and the same-item error

1. **Absorb on `&` and `*`.**
   - `&` goes through `capture/dependencies.rs::plan_existing_task_dependencies`; `*`
     goes through `capture/blocked_tasks.rs`.
   - Whenever the item leaves the pair (owner O, target T) joined by a dependency, drop
     the pair's plain entries in both directions on the batch overlay, through
     `planner.stage`. Use the tie helper and resolved identities
     (`(display_relative(path), block_id)`).
   - This includes a pair that was already dependency-joined with a hand-made overlap:
     `&T` heals it.
   - Remove the matching LINKED FROM mirror entries as a courtesy, through
     `relations_mirror.rs::stage_linked_from_remove`.
   - A new-task owner has nothing to absorb.
2. **Plain `~T` / `~~T` over a dependency.** This is a reported no-op: write nothing.
   The `relation_updates[]` row gets `action: "unchanged"` and
   `subsumed_by: "dependency"`. Add a batch-level structured warning
   `plain_link_subsumed_by_dependency`:
   `"~T" adds nothing: a dependency already ties these tasks · give it a |label to relate them`.
   A labelled `~T|label` writes normally.
3. **Same-item contradiction.** One item that names the same resolved existing target
   with a dependency sigil (`&`/`*`) and a plain relation sigil (`~`/`~~`) is refused
   with code `dependency_and_plain_link`:
   ``"`~T` names the same task as `&T`: a dependency already ties them · drop the `~` or give it a `|label`"``.
   - Model the code on `duplicate_capture_target` in `plan.rs`.
   - The whole capture aborts, as with every all-or-nothing refusal.
   - No positional "last sigil wins".
4. **Order across items** is documented and tested. If item 1 adds `~T` and item 2 adds
   `&T`, item 2 absorbs item 1's staged plain link. If item 1 adds `&T` and item 2 adds
   `~T`, item 2 is a subsumed no-op. Either way the final state holds only the
   dependency.
5. **Output.**
   - Per item, add `absorbed_plain_links[]` with the same row shape as reconcile, using
     `RelationEndpointJson`, omitted when empty. Carry it on a flattened sub-struct with
     `Default`, following `transfer_move_output`, so the 31 `CaptureItemResult` literals
     stay untouched.
   - Add `subsumed_by` to `RelationUpdateJson`, with `skip_serializing_if`.
   - Human output: after the dependency rows, add
     `🪢 {would replace|replaced} the link to "T"`. A subsumed relation renders
     `🪢 already tied by a dependency "T"`.
   - `--dry-run` and apply must agree (`capture_json_dry_run_matches_real`).
6. **Tests.** Add CLI tests for LX17–LX25:
   - LX17: absorb O→T on `&`.
   - LX18: absorb T→O on `*`.
   - LX19: subsumed `~`.
   - LX20: labelled `~` written.
   - LX21: same-item error.
   - LX22 / LX23: cross-item order.
   - LX24: `&` heals an overlap.
   - LX25: dry-run parity.

   Add unit tests next to the existing `#[cfg(test)]` modules.

7. **Docs.**
   - `docs/capture.md`: the relationship sections, "Dependency updates and consumed
     links" (new additive fields), and the warning list.
   - `docs/task-relations.md`: the LX table and the §12 `relation_updates` action notes.
   - `docs/task-dependencies.md`: the §8 capture `&` row (today it says capture never
     removes links) and the ST vectors for `*`.
   - `docs/reference/json-outputs.md`.

## Phase capture-complete — capture-complete tie fields

- **Relation candidates (`~`/`~~`).** In
  `capture_complete/candidates.rs::relation_candidates`, add
  `dependency_tie: "prerequisite" | "dependent"` (omitted when none):
  - `prerequisite`: the owner waits on the candidate.
  - `dependent`: the candidate waits on the owner, read from `graph.get(candidate)`.

  Both directions count the same-item typed `&` and typed `*`. Typed `*` is not passed
  today, so pass it in `engine.rs` and `shell.rs`. Keep `also_prerequisite` as a
  deprecated alias meaning `dependency_tie == "prerequisite"`. Document it in
  `docs/reference/json-outputs.md` with the "retained for one release train" wording
  used for the earlier `untracked` precedent. Do not remove it in this epic.

- **Dependency (`&`) and blocked-task (`*`) candidates.** Add
  `plain_link: "to" | "from" | "both"` (omitted when none), from vault LINKS only,
  through `task_links_catalog`:
  - `to`: the owner's LINKS line holds a plain entry for the candidate.
  - `from`: the candidate's LINKS line holds a plain entry for the owner.

  Build the catalog at most once per request. Check that the candidate latency does not
  regress against the existing tests and perf notes.

- **Human render.** `render.rs` gets ` · prerequisite`, ` · dependent`, and ` · linked`.
- **Models and docs.** Update `model.rs`, the `cli.rs` help text if it names the fields
  (then run `just docs-gen`), `docs/capture.md`, and `docs/task-relations.md` §12. The
  schema version stays 1, because the fields are additive.
- **Tests.** In `capture_complete/tests/relations.rs`, cover both tie directions and
  typed `*`. Add new `&` / `*` candidate tests for `plain_link` in each direction.

## Phase nav-tie-planner — pure twins, planTieChange, and absorption in the dependency writers

All work is in bob-plugins `plugins/bob-navigation-hotkeys`. Open the repo with
`/sase_repo`.

1. **Vectors mirror.** Add `scripts/plain-link-tie-vectors.cjs`, copied verbatim from
   bob-cli `tests/fixtures/plain_link_ties/vectors.json`, with a header naming the
   source. Follow the precedent of `scripts/relation-line-vectors.cjs`.
2. **Pure twins.** Add a new fragment, e.g. `087-plain-link-ties.js`. List it in
   `fragments.json` after the `085`/`086` relation planners. It holds:
   - `dropPlainLinkEntries(line, isTarget)`, the byte-exact twin of the Rust helper.
   - `classifyPairTie(...)`, computed from resolved Depends-On and LINKS identities.
   - `planTieChange({owner, other, to, notes, resolve})`. `to` is `owner-waits`,
     `other-waits`, `plain-out`, or `plain-in`. It returns one of:
     - `noop` with its notice;
     - `refuse` with a reason;
     - `change` with: the dependency delta (dependent, add, remove), the LINKS deltas
       (drops and an optional add per note), the ids needed, and whether every edit is
       same-note.

   `planTieChange` composes `planDependencyEdit`, `planLinksEdit`, and the drop helper.
   Its desired-state rules match the key-grammar table above. Use the existing identity
   keys (`dependencyStageRowKey`, `blocksIdentity`).

3. **Planner vectors.** Write PT22 and up and add them to the bob-cli
   `docs/task-relations.md` §22 table in the same change. Cover:
   - each chord against each tie state;
   - ⌃L with a labelled O→X entry (release only, keeping the label, adding no plain
     entry);
   - ⌃L when a plain or labelled X→O already exists (a no-op);
   - reversal with a third-task cycle (refused);
   - closed, archived, and Event Task eligibility refusals;
   - a done prerequisite (status unchanged).

   > [!decision] ctrl_shift_l The vectors include `plain-in` (⌃⇧L). If the decision is
   > no, `planTieChange` keeps `plain-in` for the Linked-from ↵ conversion, but no chord
   > reaches it.

4. **Absorption in the existing writers.** `applyDependencyEdit` (`580`) and
   `applyBlocksEdit` (`592`) absorb on every dependency add:
   - Plain entries on the same note fold into the same editor transaction.
   - Cross-note entries (the other task's LINKS line, plus LINKED FROM mirror courtesy)
     become follow-ups after the dependency commit, using the existing follow-up
     machinery.

   This covers every caller: Depends-on ↵, `setLocalTaskDependency`, Blocks ↵, counted
   and marked batches (`420`, `430`, `270`), and the plugin api v3. Audit the
   project-subtree, conversion, and promotion writers (`218`, `665`, `667`). Route any
   path that creates a new dependency pair through the same absorption, or record in the
   phase notes why it cannot create one. Notices gain `· replaced the 🪢 link`.

5. **Tests.**
   - Add a new `scripts/test-navigation-plain-link-ties.cjs` that runs every mirrored PT
     vector and checks Rust/JS byte agreement.
   - Extend the DW vectors (inline in `test-navigation-dependencies.cjs`) with DW20 and
     up: an add over a plain link absorbs in one transaction; a cross-note drop is a
     follow-up.
   - Add the same for the Blocks writers (`test-navigation-blocks-writers.cjs`).
   - Add the new file to `test:unit` and keep `test:perf` green.
6. **Docs and deploy.**
   - bob-cli `docs/task-dependencies.md`: §6.6 and §15.6 ("One gesture writes") gain
     absorption, plus the DW and RB vector rows.
   - bob-plugins `docs/develop/validation.md`: list the new suite.
   - Build, run `npm test` and `npm run docs:check`, then `bob plugins sync`.

## Phase nav-tie-visibility — pills, ⇄, tie bands, subtitles, and the Depends-on strip

Implement **Seeing the tie** above, display only. This phase adds no new write paths.

1. **Tie data per row.** Build an owner tie index once per stage open. It is a
   `Map<rowKey, {dependency, plainOut, plainIn, labelledOut, labelledIn}>`. Sources:
   - the owner's Depends-On and LINKS children;
   - reverse dependencies, from the `edges` map in `buildVaultDependencyStage` or from
     the Blocks current collector;
   - incoming relations, from the ledger-tools `api.relationIndex`.

   Then:
   - The Links and Linked-from builders (`427`) gain reverse dependencies. Today they
     have only `prerequisiteIds`.
   - The dependency builders (`590`) gain the owner's LINKS entries and incoming
     relations.
   - When the relation index is unavailable, degrade gracefully: show no incoming pills
     and never refuse.
   - Keep `test:perf` within budget.

2. **Rendering.**
   - Add the `.bob-cnp-row-ties` cluster and its CSS in `styles.css`, including mobile.
     Move the label, `⛓`, 🍅, and Archived pills into the cluster.
   - Add the `⇄` action badge.
   - Add the band section with the `TIED BY A DEPENDENCY` / `PLAIN LINKS` headers,
     reusing the CURRENT / RESULTS header style.
   - Add the subtitle counts.
   - Retire the inert tooltips.
3. **Consequence copy.** Add a new copy fragment (e.g. `306-tie-stage-copy.js`) holding:
   - `dependencyStageConsequenceStrip`, the new Depends-on strip, which also refreshes
     on an in-place stage switch so no stale strip text survives;
   - the tied-row strip variants for Links, Linked from, and Blocks;
   - pill and tooltip text;
   - `planTieConsequence(row, tie, to)`, which is pure and built from stage row data
     (open-prerequisite counts, `lastReason`, `todayLink`, lane, `^prj` resident
     counts).

   Copy comes from the planned result, never from the gesture.

4. **Hand-edit notice.** Hook into the existing Depends-On and relation hand-edit paths
   (`planDependencyHandEditMirror` in `070`, `594a`). Show the one-time overlap notice
   when the cursor leaves the edited line.
5. **Tests.** Extend:
   - `test-navigation-relation-links-stage.cjs`, where the existing `⛓ prerequisite`
     test sits near line 1296;
   - `test-navigation-dependencies-stage-mirror-and-badge.cjs`;
   - the Blocks stage-view tests;
   - `test-relation-index-linked-from-stage.cjs`.

   Cover all four stages, both directions, and every tie state, plus the band visibility
   rules and strip copy.

6. **Docs and deploy.**
   - bob-cli `docs/task-dependencies.md`: §6 (pills, band, strip) and §15.
   - bob-cli `docs/task-relations.md`: the Links-stage badge text.
   - bob-plugins `docs/task-relations.md` and `docs/task-card.md`.
   - Build, test, and `bob plugins sync`.

## Phase nav-tie-writer — applyTieChange

1. **`applyTieChange(owner, other, to, {expected})`.** Put it in a new fragment, e.g.
   `596-plugin-tie-change.js`. It re-reads the touched notes, checks `expected`
   snapshots (stale-row refusal), runs `planTieChange`, and then:
   - For a dependency delta, it calls `applyDependencyEdit` (when O is the dependent) or
     `applyBlocksEdit` (when X is the dependent). Each gets a new optional `tieLinks`
     parameter, which folds same-note LINKS adds and drops into the dependency writer's
     single editor transaction. Same-note means one Ctrl+Z.
   - Cross-note link work runs only after the dependency commit succeeds, through
     `applyRelationEdit` and the existing follow-ups.
   - A pure relation add (`⌃L` on an untied pair) goes straight to `applyRelationEdit`.
     It stays inert and unstamped.
   - Missing ids go through the existing block-ID prompt flow.
   - It returns an outcome object. `buildTieChangeNotice` renders the notices listed
     above from that outcome, including the follow-up failure suffixes.
2. **Fix `reopenDependencyStageFresh`** (`440`). A stale refusal must reopen the stage
   the user was actually in: `reopenLinksStage` / `reopenLinkedFromStage` for the
   relation stages, while keeping the counted `taskSession`. The callers are at
   `426:532`, `426:587`, and `426:673`.
3. **Tests.** Add a new `scripts/test-navigation-tie-change-writer.cjs` covering:
   - same-note single transaction (one undo), and cross-note dependency-first;
   - a failure after each authored step (dependency fails: nothing written; link side
     fails: the right suffix);
   - removing the last versus one of several dependencies, including the `[dependsOn::]`
     clearing;
   - a `^prj` release;
   - id prompts;
   - stale snapshots;
   - byte-identical no-ops with no stamp;
   - a regression test for the stale reopen.
4. **Docs and deploy.** bob-cli `docs/task-dependencies.md` §6.6 and §15.6 (the
   conversion writes and order), then build, test, and `bob plugins sync`.

## Phase nav-tie-keys — chords, ↵ conversions, and the armed release

1. **Strict modifiers.**
   - Fix `isCtrlKey` to require `shiftKey !== true`, and add `isCtrlShiftKey`, mirroring
     `isCtrlLeftBracketKeydown`.
   - Audit the callers: Ctrl+Shift+N/P/R currently match by accident and should stop
     matching.
   - `470` is at exactly 1000 lines, so move code to stay at or under the cap.
   - Chords match Ctrl only, never Meta, so macOS `⌘B` stays Bold. Ignore `repeat` and
     `isComposing`. Call `preventDefault` and `stopPropagation`.
   - Chords are active only in the four stages' list mode, never in the relation-label
     field or the block-ID prompt.
   - Verify in the harness that the modal's scope keeps the global Ctrl+Shift+B and
     Ctrl+Shift+L commands from firing.
2. **Chords.** Add the handlers in a new keydown fragment (e.g.
   `451-picker-tie-chords.js`), dispatched from `450` before the generic handlers. ⌃B,
   ⌃⇧B, and ⌃L route to `applyTieChange`.

   > [!decision] ctrl_shift_l ⌃⇧L routes `plain-in` as well, and appears in footers,
   > docs, and vectors.

3. **↵ conversions.** On `⇄` rows, ↵ converts. The Depends-on and Blocks absorbs commit
   on one press. The Links and Linked-from releases arm, using the confirm-nonce pattern
   from `426:956-973` and `429a`, with the amber banner and footer `↵ again to confirm`.
   Moving the selection, editing the query, pressing Esc, or entering a stage disarms
   them.
4. **Refusals.** Implement the batch and counted-session refusals (R8) and the
   ⇄-in-batch refusal.
5. **Label stage.** In the ⌃↵ label stage, the "Plain link (no relation)" row and any
   relabel-to-empty are disabled while a dependency ties the pair. The reason reads
   `A dependency ties them · ⌃L makes it a plain link`. Relabelling an empty entry to a
   labelled one never brings back a released dependency.
6. **Stage-flag leak.** `showLocalTaskValueStage` / `showBlocksStage` must reset
   `blocksStage` / `relationStage` on an in-place switch, so that ↵ and the chords never
   route to the wrong stage.
7. **Footers.** Add the contextual chord hint group to all four stage footers.
8. **Tests.** Add a new `scripts/test-navigation-tie-chords.cjs` covering:
   - 4 stages × each chord × each tie state (none, plain same / reverse / both,
     dependency each way, labelled, labelled plus dependency, invalid pair);
   - no-op notices;
   - the arm, disarm, and commit cycle;
   - refusals in batches, in counted sessions, and from guards (cycle and eligibility),
     each leaving the original link intact;
   - repeat and IME events ignored, no leak of ⌃B into editor Bold, `⌘B` untouched;
   - the label-stage guard;
   - the flag-leak regression.
9. **Docs.**
   - bob-plugins `docs/task-card.md`: a gestures row for the chords, and the Links stage
     copy.
   - bob-plugins `docs/task-relations.md` and `docs/bob-navigation-hotkeys.md`.
   - bob-cli `docs/task-dependencies.md` §6.4 and §15 keys, and `docs/task-relations.md`
     LG rows (LG-next: ⌃B in Links, the armed ↵ release, the guarded empty label).
10. **Deploy.** Build, run `npm test` and `npm run docs:check`, then `bob plugins sync`.
11. **Memory**, written last, after everything above is true and deployed.

    > [!decision] memory_decision_record Use `/sase_memory_write` to add the decisions
    > strand `dependency-absorbs-plain-link`.
    >
    > - **Applies to:** bob-cli, bob-plugins, Bob Mac Capture.
    > - **Claim:** one tie per pair; a dependency absorbs plain links; only explicit,
    >   previewed Obsidian gestures release; reconcile heals.
    > - **Why:** cite the research report, the vault evidence, and the rejected
    >   alternatives: symmetric removal; labelled relations in scope; per-panel ⌃B; a
    >   toggle; refusing a release from Links; last-sigil-wins; a hard refusal of plain
    >   `~`; a status-keyed invariant.
    > - Also record the **cost**, what **reopens** it, and the evidence commits.
    >
    > The new record links out to `[[decisions:task-deps-are-depends-on-links]]` and
    > `[[decisions:capture-link-drops-blocking-deps]]`. Accepted records are immutable,
    > so leave those records untouched. Then run `sase memory init`.

    > [!decision] memory_glossary_relation Add one sentence to `glossary:task-relation`:
    > an unlabelled relation is a plain link, and a plain link and a
    > [[task-dependency-link]] never join the same pair in either direction. A
    > dependency absorbs it, and only an explicit Task Card stage gesture turns a
    > dependency back into one. Run `sase memory init` and `just docs-gen` (the docs
    > glossary is generated from this web).

## Phase mac-tie-capsules — Bob Mac Capture tie capsules and report rows

The work is in bob-mac-capture. Open it with `/sase_repo` and follow its `AGENTS.md`.

1. **Decoding.** Use the tolerant `try? decodeIfPresent ?? default` pattern.
   - `CaptureCompletionCandidate` (`CaptureModels.swift`): add `dependencyTie` and
     `plainLink`. Keep `alsoPrerequisite` and fall back to it when `dependency_tie` is
     absent.
   - `CaptureRelationUpdate`: add `subsumedBy`.
   - `CaptureCommandSuccess`: add a new `CaptureAbsorbedPlainLink` array.
   - Update the hand-copied init in
     `Tests/BobMacCaptureTests/CaptureRelationPanelTests.swift` (`withCancelSource` /
     `withCompleteSource`), or the copies there will silently drop the new fields.
2. **Capsules and copy.** Make the changes in `DependencyPickerPresentation.swift`
   (`chipText`, `summary`) and `CompletionRowContent.swift`:

   | Row                                     | Chip                              | Summary                                                                               |
   | --------------------------------------- | --------------------------------- | ------------------------------------------------------------------------------------- |
   | Relation candidate tied as prerequisite | `Prerequisite · ~ needs a label`  | `Already tied by a dependency · a plain ~ adds nothing; add a \|label to relate them` |
   | Relation candidate tied as dependent    | `Waits on this · ~ needs a label` | same                                                                                  |
   | `&` candidate with `plain_link`         | `Linked · & replaces the link`    | `Replaces the plain link · …` (existing copy follows)                                 |
   | `*` candidate with `plain_link`         | `Linked · * replaces the link`    | same pattern                                                                          |
   - An existing `Already linked · label` chip keeps precedence for labelled relations.
     The tie then shows in the summary.
   - The chip-only-when-filtering rule stays.

3. **Report rows.** In `CaptureRelationshipPresentation.swift`:
   - A subsumed relation renders `🪢 Already tied by a dependency` instead of "already
     linked", and does not count as a link in notifications.
   - `absorbed_plain_links` renders `🪢 Replaced the link to "T"` under its ⛓ / 🧱 row.
   - `suppressRelationshipLines` also considers the absorbed rows.
4. **Fixtures.** Regenerate the relation and dependency complete and preview fixtures
   with the generator scripts. Point `BOB_BIN` at a bob built from current bob-cli
   master, which already contains the capture phases. Add new fixtures for:
   - `dependency_tie` in both directions;
   - `plain_link` on `&` and `*`;
   - a subsumed `~`;
   - an absorbing `&`;
   - the `dependency_and_plain_link` error.
5. **Tests.** Extend `DependencyDecodingTests`, `DependencyPickerPresentationTests`,
   `CompletionRowContentTests`, and `CaptureRelationshipPresentationTests`.
6. **Docs.** Update `docs/relationship-pickers.md`, `docs/capture-previews.md`, and
   `docs/protocol.md` (the optional fields and the compatibility table). Run
   `just docs-check`.
7. **Verification.**
   - Run `just portable-check` on athena for CaptureCore.
   - Compile the macOS targets from a scratch copy on the mac, as `AGENTS.md` describes.
   - Run `Scripts/macos-ci-check` before landing.
   - After the push, watch the exact-SHA macOS run to green with `/sase_monitor`.
   - Mac-side convert keys in its pickers are out of scope. Record a
     `PROPOSED FOLLOW-UP:` note.

## Cross-phase conventions

- **Engine agreement.** Rust and JS must agree byte for byte on every shared PT vector.
  When a vector changes, change the bob-cli JSON and the bob-plugins mirror in lockstep.
  The later phase updates both.
- **Docs ownership.**
  - bob-cli owns the contract: `task-relations.md` §22, the stage contracts in
    `task-dependencies.md` §6 and §15, and the capture docs.
  - bob-plugins owns the plugin how-to pages.
  - bob-mac-capture owns its mounted reference pages.

  Each phase updates only the sections whose behaviour it changes, and leaves docs true
  at its own commit.

- **Serialised bob-plugins phases.** The nav phases run in series, so the generated
  `main.js` and `bob plugins sync` deploys never collide. If sync refuses an overwrite,
  follow the bob-plugins `AGENTS.md` procedure and never pass `-F` on your own.
- **Out of scope, recorded as follow-ups:**
  - desired-state batch conversions (v2);
  - mutual plain links A↔B (deferred);
  - Mac picker convert keys;
  - removing the deprecated `also_prerequisite` after Mac migrates.
