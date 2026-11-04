---
tier: epic
title: Transfer previews — see every `>` move and merge before it lands
goal: 'When Bryan types a `>` transfer line (`@a+x @b+y > @c…`) in Bob Mac Capture,
  the preview reads like the draft: the tasks he named, an arrow, and where they go.
  Every source shows its real title and fate, the destination shows its real title
  and container, and every ripple (dependents, warnings, today''s links, offers, repairs)
  is one plain sentence that bob wrote. Below that, landing cards show the exact bytes
  bob will write to the target, the tombstones, and moved tasks. While the destination
  is unfinished the panel stays calm and teaches the four destination forms. A refusal
  reads as a refusal, not as red noise. The Mac app still parses nothing: every fact
  comes from bob''s JSON.'
decisions:
  tombstones:
    ask: How should merged sources appear below the target card?
    choices:
      cards: A small card per source with its struck line, new LINKS line, and a folded
        moved-notes row
      rows_only: Sources stay rows in the transfer card, and the landing area shows
        only the target
    default: cards
    why: You see every byte bob writes, and folding keeps each source card to about
      three rows
    answer: cards
phases:
- id: bob-landing-blocks
  title: bob emits landing blocks for transfers (target, tombstones, moved tasks,
    sections)
  depends_on: []
  size: medium
  description: 'bob-landing-blocks: make every capture `>` transfer report batch-level
    `task_blocks` with new roles `merge_target`, `merge_source`, and `moved`. Moved
    blocks carry `moved_from` and a before-image taken from the source note. Add a
    batch-level `section_blocks` array for section merges. All JSON is additive.'
- id: bob-transfer-records
  title: bob enriches merge and move records, serializes merge offers, and fixes their
    copy
  depends_on: []
  size: medium
  description: 'bob-transfer-records: add status names, target status, container-created
    flags, and Bob-authored `consequences[]` sentences (titles, not keys) to MergeRecord;
    add landing line, section, and status to move records; serialize merge hand-off
    offers as item `merge_offers`; give merge-only offers their own Pull-In copy;
    and print titles in the CLI''s dependent rows.'
- id: bob-transfer-pending
  title: bob describes an unfinished transfer on dry run
  depends_on: []
  size: small
  description: 'bob-transfer-pending: when a dry run fails because a transfer''s destination
    is unfinished, attach a read-only `transfer_pending` object to the JSON failure.
    It holds each source resolved to its title and status (or Bob''s resolution error)
    and the four destination forms, with the one being typed marked active.'
- id: mac-transfer-card
  title: Mac transfer card, status line, offers, notifications, and VoiceOver
  depends_on:
  - bob-transfer-records
  size: medium
  description: 'mac-transfer-card: replace the monospace transfer bands and the wire-word
    standard header with one designed transfer card that mirrors the draft. The card
    shows the verb, the source rows, an `↓ into` connector, the destination, consequence
    rows, the TODAY badge, and the offer rows. Also rewrite the status line, notification
    copy, and VoiceOver summary, regenerate rich recorded fixtures, and add design
    renders and docs.'
- id: mac-landing-cards
  title: Mac landing cards for target, tombstones, moved tasks, and sections
  depends_on:
  - bob-landing-blocks
  - mac-transfer-card
  size: medium
  description: 'mac-landing-cards: render bob''s transfer-role task blocks with role
    capsules and a `from inbox.md:5` caption. Fold each tombstone''s removed runs
    into one expandable row. Add a section block card, keep bob''s block order, and
    tint the editor''s `>` arrow by verb. Add fixtures, design renders, and docs.'
- id: mac-pending-and-refusals
  title: Mac calm pending card and transfer refusal card
  depends_on:
  - bob-transfer-pending
  - mac-landing-cards
  size: medium
  description: 'mac-pending-and-refusals: never flash red while a transfer destination
    is unfinished. Run the dry run and render bob''s `transfer_pending` as a calm
    card with the resolved sources (typos caught early) and the four destination forms.
    Render transfer refusals as a designed card with Bob''s message. Add fixtures,
    design renders, docs, and the manual checks list for Bryan.'
proposed_by: bbugyi200.athena.188
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-04 14:57:17
status: wip
bead_id: bob-cli-h4
---

- **PROMPT:** [prompts/202611/transfer_previews.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/transfer_previews.md)
- **BEAD:** [bob-cli-h4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-h4/README.md)

# Plan: Transfer previews — see every `>` move and merge before it lands

## Why

Bryan typed
`@bob+ch-jumper-keymaps @bob+capture-ctrl-h-l > @bob+always-capture-ref-time` in Bob Mac
Capture (screenshot `~/tmp/screenshots/20261104_143200.png`). The capture `>` transfer
grammar comes from epic `bob-cli-go` (plan `plan:202611/task_move_merge_split.md`, §2
and §8). The panel showed:

1. A status line that reads `Preview → bob.md:`. It ends in a dangling colon, because a
   transfer item has an empty `task_line`, and it never says "merge".
2. A `TODAY 2/13` badge as the first thing in the card, with no hint of why it is there.
3. A monospace band. Its header reads `↳ Merges 2 tasks into Always add capture time…`,
   and each source row is Bob's raw CLI `message`, with literal backticks and `[[org]]`.
4. The wire words `bob.md  updated  task_merge`, followed by a stray `bob.md` line.
   These come from the generic `standardPreviewItem` header and its trailing
   `relativeTarget`.
5. The only real diff is the BUILD Pomodoro card with two struck links. Nothing shows
   where the notes land in the target, its new `LINKED FROM` line, the tombstones, or
   the dependents that now wait on the target.

Investigating the code and a scratch-vault dry run found the causes:

- **No landing data.** `capture/transfer.rs` returns `task_block_refs: Vec::new()`, so
  transfer items never produce `task_blocks`. The Mac app has nothing to draw.
- **Consequences arrive as keys.** `merges[]` carries `dependents_repointed`,
  `prerequisites_left`, `successors_orphaned`, `links_pruned`, and `offers`, but only as
  keys and counts. The Mac decoder drops them all. Even the CLI prints
  `⛓ inbox.md#^cutover now waits on …`, a key instead of a title.
- **Invisible offers.** The merge hand-off offer rows are `#[serde(skip)]` in
  `RelationMergeOutput.offer_rows`. They count toward `pull_in.offer_count` but never
  reach the preview, so the Pull-In card appears at Return as a surprise. Its top copy
  also reads `Pull in successors of S`, which is wrong for a merge.
- **Red while typing.** `transfer_destination` and `drawer` are not calm states, so
  typing `>` and a space flashes red `transfer needs a destination…`. The `block_id` and
  `section` needs blank the preview to a bare "Preview".
- **No docs.** `docs/capture-previews.md` has no transfer section.

## The experience

The managed lines below are schematic. Their exact bytes always come from bob.

### Merge (Bryan's screenshot, redesigned)

```text
Would merge 2 tasks into “Always add capture time to ref marker note!” · bob.md   ← status line

▌ ⤵  Merge 2 tasks
▌    ◉ Add =<ch> jumper keymaps (see org)!                  Next → Cancelled   bob.md:80
▌        title only
▌    ◉ Add <ctrl+h/l> capture keymaps to jump to prev/n…    Next → Cancelled   bob.md:77
▌        title only
▌    ↓ into
▌    ◯ Always add capture time to ref marker note!                              bob.md:61
▌
▌    ⛓  “Cut over prod” now waits on “Always add capture time to ref marker note!”
▌    ⊖  “Add =<ch> jumper keymaps…” leaves BUILD                           TODAY 2/13
▌    Offer 1  Always add capture time to ref marker note!          → BUILD
▌
  [Merge target]  ◯ Ready · bob.md · line 61
  ┃   - [ ] #task Always add capture time to ref marker note! ^always-capture-ref-time
  ┃ + - 🪢 LINKED FROM: [[#^ch-jumper-keymaps]] (merged into), …
  ┃ + - Add `=<ch>` jumper keymaps (see [[org]])!
  ┃ + - Add `<ctrl+h/l>` capture keymaps to jump to prev/next word!
  [Merged]  ⊘ Cancelled · bob.md · line 80
  ┃ • - [-] #task Add `=<ch>` jumper keymaps (see [[org]])! [cancelled:: 2026-11-04] ^ch-jumper-keymaps
  ┃ + - 🪢 LINKS: merged into [[#^always-capture-ref-time]]
  [Merged]  ⊘ Cancelled · bob.md · line 77
  ┃ …
  ◌ Queued · line 61 — BUILD   (the existing Pomodoro card, struck links)

                                   Templates  Stash  Discard  Preview  [⤵ Merge]
```

### Move

```text
Would move 2 tasks into db.md › Tasks

▌ ↪  Move 2 tasks
▌    ◉ Benchmark logical replication            ^bench    inbox.md:5
▌    ◯ Add <ctrl+h/l> capture keymaps           ^ctrl-hl  inbox.md:11
▌    ↓ into
▌    🗎 db.md › Tasks
▌    ↻ 3 links repaired in 2 notes
  [Moved]  ◉ Next · db.md · line 9 · from inbox.md:5
  ┃   - [*] #task Benchmark logical replication … ^bench
  ┃     - Try pgbench at 10k TPS
  [Moved]  ◯ Ready · db.md · line 13 · from inbox.md:11
  ┃   …
```

### While the destination is unfinished (`@inbox+bench @inbox+bnch > @db+`)

```text
▌ ⇢  2 tasks ready — finish the destination
▌    ◉ Benchmark logical replication            inbox.md:5
▌    ✕ no task ^bnch in inbox.md                                 ← caught before Return
▌    @note               ↪ move into the note
▌  ▸ @note+task          ⤵ merge into a task                     ← active: you are typing it
▌    @note+task#drawer   ⤵ merge into a task's drawer
▌    @note#section       ⤵ merge into a note section
```

### When bob refuses

```text
▌ ⤵  Can't merge
▌    “Benchmark logical replication” has 2 open subtasks; move or close ^sub first
▌    merge_source_open_subtasks
```

## Design principles

Every phase holds to these:

1. **The card mirrors the draft.** Sources come first, then `↓ into`, then the
   destination: the same order as `@a+x @b+y > @c`. Landing cards follow, then Pomodoro
   cards.
2. **One verb, one tint, one glyph.**
   - Move uses the route accent and `arrow.right` (`↪` in copy).
   - Merge uses purple and `arrow.triangle.merge` (`⤵`).
   - The verb carries through the editor's `>` arrow span, the card glyph and rail, the
     `↓ into` connector, the target's role capsule, and the footer's primary button.
     Verb names always appear in words too, so colour is never the only signal (CL6).
3. **Bob writes, the Mac draws.** Following decision `mac-capture-is-a-thin-client`,
   every sentence, status name, title, and line comes from bob's JSON. Swift maps a
   `kind` to an SF Symbol and a tint, styles inline Markdown with `TaskDisplayText`
   (code spans, wikilink display text), and folds or orders rows. It never resolves a
   task, counts links, or derives a consequence.
4. **Exact bytes, honest diffs.** The landing cards are bob's `task_blocks` and
   `section_blocks`, drawn with the existing `BlockDiffCard`. Tinting never adds, drops,
   or reorders characters.
5. **Calm while typing.** Red appears only when the draft is complete and bob refuses.
6. **Additive and graceful.** Every new JSON field is additive, and `schema_version`
   stays 1. With an older bob, the Mac card falls back to Bob's `message` rows, with no
   landing cards and no pending card. It never shows a blank or broken card.
7. **Dry run equals commit.** The committed JSON carries the same blocks, records, and
   consequences as the dry run, so notifications and the post-commit status read the
   same facts the preview showed.

## Shared JSON contract (additive)

All keys are snake_case. "Item" means one capture item result. "Batch" means the
top-level success.

### A. Transfer roles on batch `task_blocks` (phase `bob-landing-blocks`)

- `merge_target` is T's final block. It includes the payload title bullets and their
  children, the `LINKED FROM` line, and any drawer title that the merge created.
- `merge_source` is each merged S's tombstone. Its headline is `changed` (with
  `before`), its `LINKS` line is `added`, and its cut children are `removed`.
- `moved` is each moved task at its destination. Its before-image comes from the source
  note, so carried lines read `unchanged`, a freshness or ID change reads `changed`, and
  `created` is false. It also carries `moved_from: {relative_target, line}`, where
  `line` is the 1-based source line.
- Order follows first touch:
  - for a merge, T first, then sources in typed order;
  - for a move, moved tasks in typed order.
- Blocks are emitted only for sources whose outcome is `merged` (or moves that
  happened). An all-`unchanged` re-run emits no transfer blocks.
- Blocks already tracked by earlier items keep their roles and gain the new one.

### B. Batch `section_blocks` (phase `bob-landing-blocks`)

The key is present only when a section merge happened, with this shape:

```text
{relative_target, route, line, heading, created, roles: ["merge_target"], lines: [BlockLine]}
```

- `lines` uses exactly the task-block line schema (`text`, `depth`, `change`, `before`).
- The headline is the `## Heading` line.
- The section ends at the next heading of the same or a higher level, or at the end of
  the note.
- `created` is true when this batch appended the H2.

### C. `merges[]` (MergeRecord) additions (phase `bob-transfer-records`)

- `previous_status_name` is the source's status name before the merge.
- `status_symbol` and `status_name` give the source's status after the merge (`-`,
  Cancelled, using the vault's names).
- `target` gains these fields:
  - `line` (1-based, post-merge);
  - `created` (true when the merge created the drawer or section);
  - `status_symbol` and `status_name`, for `task` and `drawer` kinds.
- `consequences[]` holds `{kind, tone, text, keys}`, where `tone` is `info` or
  `warning`. The text is Bob-authored with titles (never keys), in this order:
  - `dependent` (info), one per re-pointed dependent:
    `“Cut over prod” now waits on “Upgrade Postgres to 17”`;
  - `prerequisite_left` (warning), one per source that has any:
    `“S” waited on “P”; “T” does not · add &inbox:p` (names every prerequisite; the hint
    uses the real locator);
  - `successor_orphaned` (warning):
    `N task(s) start after “S”; they won't be offered when “T” finishes · relink with ~`;
  - `link_pruned` (info), one per open Pomodoro entry that lost S's link:
    `“S” leaves BUILD`. If the engine cannot name the entries, emit one row
    `“S” leaves N of today's Pomodoros`.
- `message` and every existing key stay byte-identical.
- The §10 copy table in `docs/task-merge.md` gains these rows.

### D. `moves[]` additions (phase `bob-transfer-records`)

`bob task move -f json` and capture transfers both add:

- `destination_line`, the 1-based landing line (from `MoveRecord.dest_line`);
- `destination_section`, the H2 the landing line sits under (`"Tasks"`), or null;
- `status_symbol` and `status_name`, the moved task's final status.

The two shapes stay identical, as the `TransferMoveJson` doc comment promises.

### E. Item `merge_offers[]` (phase `bob-transfer-records`)

- Serialize `RelationMergeOutput.offer_rows` as `merge_offers`, omitted when empty. Each
  row has the same `TaskCompleteUnblocked` shape as `relation_successors`, with
  `trigger: "merge"` and `offer.merged_from`, and carries the draft-global numbers that
  `pull_in.rs` already assigns.
- Label merges (`~…|"merged into"`) gain the same key.
- When every offer row in the draft is a merge hand-off, `pull_in.copy.title` reads:
  - `Link “T” in place of “S”` for one merged source;
  - `Link “T” in place of N merged tasks` for more than one.

  The capsule stays `Pull in`, and the glyph stays `hand_off`.

### F. `transfer_pending` on dry-run failures (phase `bob-transfer-pending`)

`bob capture --dry-run -f json` failures with any `incomplete_transfer*` code
(`incomplete_transfer`, `incomplete_transfer_block_id`, `incomplete_transfer_drawer`,
`incomplete_transfer_section`, or `incomplete_transfer_task`) add this object:

```text
transfer_pending: {
  item,                                   // 1-based capture item number
  need,                                   // transfer_destination | block_id | drawer | section (as capture-parse reports)
  sources: [ {marker, block_id, path, line, title, status_symbol, status_name}
           | {marker, block_id, error: {code, message}} ],
  forms:   [ {form: "@note",             verb: "move",  text: "move into the note",          active},
             {form: "@note+task",        verb: "merge", text: "merge into a task",           active},
             {form: "@note+task#drawer", verb: "merge", text: "merge into a task's drawer",  active},
             {form: "@note#section",     verb: "merge", text: "merge into a note section",   active} ]
}
```

- `active` is true for exactly the form the current need is completing:
  - `block_id` makes `@note+task` active;
  - `drawer` makes `@note+task#drawer` active;
  - `section` makes `@note#section` active;
  - `transfer_destination` makes none active.
- Source errors are only resolution errors: a missing note, an ambiguous note, a missing
  task, or not a task. The verb is unknown, so move and merge eligibility is not judged
  yet.
- This is read-only: no lock, no sync, no writes. Human output and non-dry-run failures
  are unchanged.

## Phase `bob-landing-blocks`: bob emits landing blocks for transfers

Work in bob-cli. Before touching docs, read the `docs.md` reference memory with
`/sase_memory_read`.

- **Roles.** Add `MergeTarget`, `MergeSource`, and `Moved` to `TaskBlockRole` in
  `src/native/capture/task_blocks.rs` (snake_case on the wire).
- **Refs from transfers.** In `capture/transfer.rs`, after a merge stages, rescan the
  staged texts by block ID and return `task_block_refs` per contract A, using 0-based
  post-state lines.
  - For a section merge, return a section ref (target path plus resolved heading)
    instead of a target task ref.
  - For moves, return one `Moved` ref per moved task at its staged destination line,
    with an origin (source absolute path plus old block ID).
- **Cross-note before-image.**
  - Give `TrackedTaskBlock` an optional origin.
  - In `TaskBlockTracker::finish`, when a tracked block with an origin has no
    before-image in its destination's original text, take the before-block from the
    origin note's original text by the old block ID, with `created = false`.
  - Add an optional `moved_from` to `TaskBlockJson`, omitted when `None`, so existing
    output stays byte-identical.
  - `remap_transfer_moves` sets the origin on blocks that earlier items already tracked.
- **Section blocks.**
  - Add a small batch-level section tracker beside `task_blocks.rs` that the batch loop
    feeds and finishes like the task tracker. At finish it finds the heading in the
    final text, diffs the section against the same heading in the original text (or
    marks it `created`), and emits `section_blocks` per contract B through the shared
    `block_diff` helpers.
  - Wire the top-level key so it is omitted when empty.
- **Performance.** Do at most one extra scan per touched note per item. Run the existing
  capture performance test; a live preview runs on every keystroke.
- **Tests** (CLI tests in `tests/cli/capture/transfer.rs`, plus unit tests):
  - a task merge; a drawer merge into an existing drawer; a drawer merge that creates
    the drawer (its title row is `added`);
  - a section merge into an existing section, and one that creates the section;
  - the screenshot shape, where S and T are in the same note;
  - a two-source move across notes, where a freshness stamp makes the headline `changed`
    and `moved_from` is set;
  - a multi-item draft whose second item adds a sub-bullet under T, so T's card is
    cumulative and has both roles;
  - an all-`unchanged` re-run that emits no transfer blocks;
  - dry-run parity with the committed JSON.
- **Docs:**
  - `docs/capture.md`: the task-block roles and `section_blocks`;
  - `docs/reference/json-outputs.md`: the roles, `moved_from`, and `section_blocks`;
  - `docs/reference/compatibility.md`, if it lists additive capture keys.

  Then run `just docs-gen`.

**Done when:** `just fmt`, `just lint`, `just test`, and `just docs-check` pass. In an
isolated scratch vault (`HOME`/`XDG_*` isolated, `BOB_REF_JOBS_KICK=off`, never Bryan's
vault), each of the four destination forms prints the expected blocks from
`bob capture --dry-run -f json`.

## Phase `bob-transfer-records`: bob enriches merge and move records, serializes merge offers, and fixes their copy

Work in bob-cli. Read the `docs.md` reference memory first.

- **MergeRecord** (`src/native/task_merge/types.rs`, `plan.rs::build_records`):
  - Add the contract-C fields.
  - Resolve titles for dependents and prerequisites from the staged snapshot with
    `note_tasks::clean_description`, the same extractor `source.title` uses. Never fall
    back to printing a key when a title resolves; if one does not resolve, the row says
    `a task in inbox.md` rather than a raw key.
  - Get pruned entry names from the cancel prune results the retirement stage already
    computes.
  - Status names come from the vault's Tasks settings, the same source as `status_name`.
- **Moves:** add the contract-D fields to `TransferMoveJson` and the `bob task move`
  JSON record, keeping both identical.
- **Offers:**
  - Serialize `offer_rows` as `merge_offers` (contract E).
  - Verify that `pull_in.rs` numbering reaches them in previews, and that a dry run's
    numbers equal those of the `pull_in_choice_required` failure's `offers[]` for the
    same draft.
  - Add the merge-only top copy in `pull_in_copy.rs::top_copy`.
- **CLI human output:** `bob capture` and `bob task merge` print dependent rows from
  `consequences` (`⛓ “Cut over prod” now waits on “Upgrade Postgres to 17”`), and print
  the prerequisite and successor warnings with titles. Update the human-output tests.
- **Tests:**
  - JSON for every consequence kind, using one fixture vault that has a dependent, an
    open prerequisite, a `starts after` successor, and two of today's open Pomodoros
    that link S;
  - `target.created` for drawer and section merges;
  - the move fields for a landing under `## Tasks`;
  - `merge_offers` present on a capture dry run and on a `~…|"merged into"` label merge,
    with equal numbers on the dry run and the required-choice failure;
  - the merge-only top copy for one and for two sources;
  - existing `message` strings unchanged.
- **Docs:** update the JSON section and §10 copy table of `docs/task-merge.md`, the move
  JSON in `docs/task-move.md`, `docs/reference/json-outputs.md`, and the
  `docs/capture.md` transfer JSON paragraph. Then run `just docs-gen`.

**Done when:** all just gates pass. A scratch-vault
`bob capture --dry-run -f json -- '@inbox+bench @inbox+other > @db+upgrade#future-work'`
shows titled consequences, `merge_offers`, the new top copy, and the status names.

## Phase `bob-transfer-pending`: bob describes an unfinished transfer on dry run

Work in bob-cli. Read the `docs.md` reference memory first.

- When a dry run fails with one of the contract-F codes and `-f json`, attach
  `transfer_pending` to the failure envelope, beside the existing `code` and `error`.
  The failure envelope already carries extra payloads, such as the `offers[]` of
  `pull_in_choice_required`; follow that precedent.
- Recover the sources from the transfer lexer (`capture_language::transfer`), which
  already spans them on incomplete lines. Resolve each source read-only against a fresh
  vault scan, using the same note resolution transfers use (`resolve_transfer_note`
  semantics) and the block-ID lookup.
- For multi-item drafts, report the failing transfer item.
- Skip the object (and keep today's failure) when the failing item is not a transfer.
- **Tests:**
  - each need (`> `, `> @`, `> @db+`, `> @db+upgrade#`, `> @db#`) with the right
    `active` form;
  - a source typo (missing task) and a missing note;
  - two sources;
  - a multi-item draft;
  - non-dry-run and human output unchanged.
- **Docs:** add the object to `docs/capture.md` (dry-run failures) and
  `docs/reference/json-outputs.md`, then run `just docs-gen`.

**Done when:** all just gates pass, and the scratch-vault runs above print the
contract-F object.

## Phase `mac-transfer-card`: Mac transfer card, status line, offers, notifications, and VoiceOver

Open `bob-mac-capture` with `/sase_repo` and read its `AGENTS.md`. Never land Swift you
have not compiled. Run `just portable-check` on athena. Compile a scratch copy on the
mac over the tailnet (rsync without `.build`/`.git`, then `bundle.sh` and
`just typecheck-tests`, then delete the copy), as `AGENTS.md` describes. If the mac is
offline, say so in the final report.

- **Fixtures first.**
  - Build bob from this bob-cli checkout and pass it to the generator as `BOB_BIN`.
  - Extend `Scripts/generate-transfer-preview-fixtures.sh` with a realistic vault. Give
    it a Tasks plugin `data.json` with real status names, so rows never read
    `Unknown → Cancelled`.
  - Record these drafts:
    - the screenshot shape (S and T in one note, titles with code spans and a wikilink);
    - a two-source merge with a dependent, an open prerequisite, a successor, and links
      in two of today's open Pomodoros (producing an offer);
    - a merge that creates a drawer;
    - a section merge;
    - a two-source move;
    - an all-`unchanged` re-run against the post-merge vault.
  - Keep the script's assertion block, and assert the new keys.
- **Decode:** add the contract C, D, and E fields to `CaptureModels.swift` /
  `CaptureTransferPresentation.swift`, tolerantly (`decodeIfPresent`, malformed rows
  dropped). Add `merge_offers` to the offer collections the preview feeds to the Pull-In
  summary (`allUnblockedRows + allKeepMovingRows + allRelationSuccessors`).
- **Presentation** (`CaptureCore`, pure and unit-tested): replace
  `CaptureTransferMovePresentation` and `CaptureTransferMergePresentation` with one
  `CaptureTransferPresentation`:
  - `verb` is `.move` or `.merge`, and the state is `.planned` or `.alreadyMerged`
    (every record `unchanged`).
  - Source rows carry the title as `TaskDisplayText`, the current status, the target
    status (Cancelled for a merge; for a move, the status Bob reports), the locator
    `bob.md:80`, and a detail line:
    - for a merge: `title only` or `N notes move`;
    - for a move: `^bench`, or `^bench → ^bench-2` in orange when renamed.
  - The destination row carries the kind (note, task, drawer, or section), title,
    status, locator, container label (`› FUTURE WORK`, `§ Future Work`, or `› Tasks`),
    and a `new` flag from `target.created`.
  - Consequence rows take Bob's `text` and map `kind` to a symbol and tint: `dependent`
    gets `link` (secondary), `prerequisite_left` and `successor_orphaned` get
    `exclamationmark.triangle` (orange), and `link_pruned` gets `minus.circle`
    (secondary).
  - It also produces the repair text (`3 links repaired in 2 notes`), the status-line
    text, the notification title and body, and a VoiceOver summary.
  - **Older-bob fallback:** without the new fields, source rows render Bob's `message`
    through `TaskDisplayText`, and consequences are omitted.
- **View** (`Sources/BobMacCapture/CaptureTransferCardView.swift`, a new file, so
  `CapturePanelView.swift` does not grow):
  - The layout follows the mockups. A 3 pt leading rail and a header glyph in a soft
    circle carry the verb tint. The title (`Merge 2 tasks`) is semibold.
  - Source rows show the status glyph from `CaptureEditorPalette.taskStatus`, the styled
    title (code spans monospaced on a quiet fill, links in the wikilink tint), and a
    transition chip (`Next → Cancelled`) as two status-tinted capsules. A trailing
    monospaced caption locator truncates first, then the title; chips never wrap or
    shrink.
  - Next comes the `↓ into` connector, then the destination row. Drawers get an outlined
    container capsule, and a created drawer or section gets a dashed `new` capsule.
  - Then the consequence rows, the repair row, and a TODAY row that hosts the shared
    `TodayBadge` when the item reports `plan_budget`.
  - Last, the shared Pull-In offer rows (`pullInPreviewSummary`) inside the card, so
    Return's Pull-In card is forecast.
  - An `.alreadyMerged` card renders muted, with `Already merged into “T”` and
    `Nothing to write`.
  - There is no animation; the pane updates on every keystroke.
- **Preview integration** (`CapturePanelView.swift` `previewItem`):
  - A transfer item renders only the transfer card. It replaces the two monospace bands,
    `planDestinationRow`, `standardPreviewItem`'s wire-word header and trailing path,
    and the item-level Pull-In summary.
  - When the batch is a single transfer item, skip the standalone plan-budget meter
    above the items; the card hosts the badge.
  - Keep `inboxRoutePreviewSummary`, so a merge from an inbox still forecasts its route
    card.
  - Multi-item drafts render the card per transfer item, between dividers.
- **Status line** (`CapturePanelModel.captureSummary`/`captureStatus`): a single
  transfer reads like these examples:
  - `Would merge 2 tasks into “T” · bob.md` (with ` › FUTURE WORK` for a container);
  - `Would move 2 tasks into db.md › Tasks`;
  - `Merged …` / `Moved …` once committed.
- **Footer:** keep **Merge** and **Move**. Show the verb glyph on the primary button if
  the footer already supports a leading symbol; otherwise text only.
- **Notifications** (`NotificationService`):
  - The title stays `Merged 2 tasks into “T”` or `Moved 2 tasks into db.md`.
  - The body has one line per source, with styled text flattened (for example
    `Add =<ch> jumper keymaps · Next → Cancelled · title only`), then the consequence
    sentences, then the repair line.
- **VoiceOver:**
  - The card is one element whose summary reads every row in words.
  - When the verb flips for the same draft (typing `> @db`, then `+upgrade`), post one
    announcement: `Now merging into “Upgrade Postgres to 17”`.
- **Tests:**
  - presentation tests from every recorded fixture: rows, chips, the fallback, and
    `.alreadyMerged`;
  - `CapturePanelModel` tests for the status line and footer verb;
  - `NotificationService` tests;
  - a `CaptureTransferCardDesignTests` render (light and dark, at 760 and 620 pt), in
    the existing design-test pattern.
- **Docs:** `docs/capture-previews.md` gains a
  `## Task transfers (\`>\`)`section covering the card, status line, offers, notifications, and VoiceOver. Run`just
  docs-check`.

**Done when:** the repo's checks pass (`just portable-check`, and the mac scratch build
plus `just typecheck-tests`; CI runs the tests). Every fixture renders the expected card
in the design render.

## Phase `mac-landing-cards`: Mac landing cards for target, tombstones, moved tasks, and sections

Same repo, build, and compile rules as `mac-transfer-card`.

- **Fixtures:** regenerate with a bob built from bob-cli master that includes
  `bob-landing-blocks`. The recorded JSON now carries `task_blocks` roles, `moved_from`,
  and `section_blocks`.
- **Role capsules** (`CaptureTaskBlockPresentation`, `TaskBlockView`): in the existing
  New/Created capsule style:
  - `merge_target` gets a purple `Merge target` capsule;
  - `merge_source` gets a secondary `Merged` capsule;
  - `moved` gets an accent `Moved` capsule, and its caption appends `· from inbox.md:5`
    from `moved_from`.

  Decode `moved_from` tolerantly.

> [!decision] tombstones = cards Render `merge_source` cards. In those cards only, fold
> every run of `removed` rows into one expandable row, `⤵ N lines moved to the target`,
> in the merge tint. It expands on click, like the quiet-run folds, and joins **Show all
> lines**. The headline and `added` rows always show.

> [!decision] tombstones = rows_only Skip `merge_source` blocks in the landing area.
> Each source row in the transfer card gains a muted caption `→ tombstone keeps LINKS`
> instead.

- **Section cards:**
  - Decode the batch `section_blocks`.
  - Add a `SectionBlockView` that reuses `BlockDiffCard`, with the caption
    `§ Future Work · dev.md · line 40`, a pink `New`/`Created` capsule when `created`,
    and headline emphasis on the heading row.
  - Folding matches task blocks.
- **Order:**
  - Landing cards render after the item stack and before the Pomodoro blocks, in bob's
    order, with section blocks placed beside their role peers.
  - `CaptureRelationshipPresentation.ownerFirstTaskBlocks` must not reorder blocks that
    carry transfer roles.
  - Dimming rules (`closePendingText`) apply unchanged.
- **Editor arrow tint:** the `transfer_arrow` span takes the verb tint from the parse's
  `mode`: `task_move` gets the route accent, `task_merge` gets purple, and anything else
  keeps the neutral placeholder tint. Swift never scans the draft for `>`.
- **Tests:**
  - presentation tests for each role (capsule, caption, folding) and for section blocks;
  - a check that ownerFirst ordering leaves transfer blocks in bob's order;
  - design renders for the target, tombstone, moved, and section cards.
- **Docs:** add the landing cards to the transfer section, and name the roles in the
  "Pomodoro and task blocks" paragraph. Run `just docs-check`.

**Done when:** the same gates as `mac-transfer-card` pass, and the recorded fixtures
render target, tombstone, moved, and section cards in the design renders.

## Phase `mac-pending-and-refusals`: Mac calm pending card and transfer refusal card

Same repo, build, and compile rules as `mac-transfer-card`.

- **Calm states** (`CapturePanelModel`, around `pickerNeed` and the live-preview
  scheduler):
  - When the parse has a `transfer_source` span and needs `transfer_destination`,
    `block_id`, `drawer`, or `section`, skip the quiet-picker shortcut for those needs.
    Keep the completion menu, run the live dry run, and never set red preview text for
    `incomplete_transfer*` failures.
  - Non-transfer drafts keep today's behavior byte-for-byte.
  - With an older bob that sends no `transfer_pending`, put Bob's diagnostic message in
    the status line, muted, and keep the preview idle.
- **Pending card** (decode `transfer_pending` on the failure; render the card in place
  of the failure text, as `CaptureNotePullEmptyCard` does for empty pulls):
  - The header is `⇢ N tasks ready — finish the destination`, in a neutral tint.
  - Resolved sources render as the transfer card's source rows, without a transition.
    Unresolved sources show `xmark.octagon` in red with Bob's message.
  - The forms legend shows each `form` monospaced, its parts tinted like the editor
    spans (`@note` route accent, `+task` block-ID indigo, `#drawer`/`#section` purple),
    a verb glyph, and the `text`. The `active` row gets a quiet highlight fill.
  - VoiceOver reads the card as a list.
- **Refusal card:** a transfer refusal is a dry-run failure whose code starts with
  `transfer_` or `merge_`, or any dry-run failure while the parse mode is `task_move` or
  `task_merge`.
  - Render the verb glyph, `Can't merge` or `Can't move` (`Can't transfer` when the verb
    is unknown), Bob's message verbatim through `TaskDisplayText`, and the code in muted
    monospaced caption.
  - Submit-time failures keep today's error callout.
- **Tests:**
  - record pending fixtures for `> `, `> @db+`, `> @db#`, and a typo source, plus
    refusal fixtures for `merge_source_open_subtasks` and `transfer_extra_text`, all
    with the generator;
  - model tests proving that transfer needs never flash red and that non-transfer needs
    are unchanged;
  - presentation tests;
  - design renders.
- **Docs:** add pending and refusal paragraphs to the transfer section. Run
  `just docs-check`.
- **Final report:** list the manual checks for Bryan on the Mac:
  - the screenshot draft;
  - a move;
  - a drawer merge that creates the drawer;
  - a section merge;
  - typing `>` slowly through every need;
  - a typo source;
  - a refusal;
  - Return on a merge with an offer.

**Done when:** the same gates as `mac-transfer-card` pass, and the pending and refusal
fixtures render in the design renders.

## Coordination

- **Parallel tracks:** the three bob phases are independent and can run in parallel.
  They all touch `src/native/capture/transfer.rs`, so rebase before committing. The Mac
  phases are serialized because they share `CapturePanelView.swift`,
  `CapturePanelModel.swift`, and the fixture generator.
- **In-flight epics:** `bob-cli-h0` (h0.8) and `bob-cli-h2` (h2.6) edit the same Mac
  files and capture output. Rebase onto them, and keep their cards untouched.
- **Fixtures:** every Mac phase regenerates fixtures with a bob built from the current
  bob-cli master. Never hand-edit recorded JSON beyond the generator's scratch-path
  normalization.
- **Commits:** use Conventional Commits, `feat(capture):` with no `!`, because every
  change is additive. Never edit the `Cargo.toml` version or `CHANGELOG.md` (decision
  `bob-cli-releases-ride-a-daily-train`).
- **Hygiene:** never run live commands against Bryan's vault. Use isolated scratch
  vaults with `HOME`/`XDG_*` isolated and `BOB_REF_JOBS_KICK=off`.

## Non-goals

- Changing transfer semantics, grammar, or completion rows.
- Inbox routing of merged tombstones. Its existing Inbox row stays in the preview.
- Previews for the Obsidian Links-stage merge (`nav-merge`).
- Animations or transitions.
- Importing CI render images into the docs. `just docs-media` can do that after CI, as a
  follow-up.
