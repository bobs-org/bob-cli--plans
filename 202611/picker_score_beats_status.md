---
tier: tale
title: Task pickers rank by match score; status only breaks ties
goal:
  Typing a filter in any task-linking picker shows the best match first, even when it is
  Blocked or done.
size: medium
decisions:
  empty_query_sections:
    ask:
      Keep the trailing BLOCKED / CLOSED section headers when the stage filter is empty?
    choices:
      keep:
        An empty filter keeps today's layout; only a typed filter becomes one ranked
        RESULTS list
      drop:
        Only CURRENT and RESULTS headers; Blocked and closed rows are marked per row,
        even unfiltered
    default: keep
    why:
      With no query all scores tie, so status runs are the real order and their headers
      stay truthful
    answer: keep
  decision_record:
    ask:
      Add a decisions-web strand recording that task pickers rank by match score and use
      status only to break ties?
    memory:
      - decisions
    default: false
    answer: false
proposed_by: bbugyi200.athena.1el
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-20 09:31:37
status: wip
---

# Plan: Task pickers rank by match score; status only breaks ties

## Problem

Bryan's screenshot shows the Obsidian **Depends on** stage (Ctrl+Shift+B / Task Card
`b`) with the filter `spike`. Eight Ready rows that match `spike` only as a scattered
subsequence (score 0) fill RESULTS. The Blocked task
`Spike S0: write the promise sentence…` matches as a field prefix (score 3), but it sits
in a trailing **BLOCKED** section, below the fold. The best match should be first.

**Rule (applies everywhere below):** match score always leads. Task status (open →
Blocked → closed/done) only orders candidates whose scores are **equal**, followed by
each picker's existing canonical order.

## Audit: where status currently beats score

| Surface                                                                      | Code                                                                                                                                                      | Today                                                                                                                                        | Change       |
| ---------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- | ------------ |
| nav **Depends on** stage                                                     | `bob-plugins` `plugins/bob-navigation-hotkeys/src/300-dependency-stage.js` `planDependencyStageView`                                                      | every `candidate.blocked` row goes into BLOCKED after all RESULTS, whatever its score; the ~60-row cap cuts Blocked rows first               | **fix**      |
| nav **Blocks** stage                                                         | `src/302-blocks-stage-view.js` (`planBlocksStageView` region, about lines 700–930)                                                                        | same split, using `blocked \|\| status === "?"`                                                                                              | **fix**      |
| nav **Links to** stage                                                       | `src/304-relation-stage-view.js` `planRelationStageView`                                                                                                  | closed and `done/` rows go into CLOSED after all RESULTS, whatever their score                                                               | **fix**      |
| nav **Linked from** stage                                                    | `src/304-relation-stage-view.js` `planLinkedFromStageView`                                                                                                | already one RESULTS list ranked by score; ties fall back to pool order (see the latent bug below)                                            | tie fix only |
| Obsidian `((` task wikilink picker (block-id-prompt; also its Pomodoro mode) | `plugins/block-id-prompt/src/032-task-wikilink-rank.js` `orderForWikilinkPicker`; `src/088-task-wikilink-picker-view.js` `buildTaskWikilinkFilteredItems` | open matches always precede completed history (`openOrder` is the primary sort key), plus a "Completed history" divider in the filtered list | **fix**      |
| bob-cli capture `&` dependency and `((` wikilink_task contexts               | `src/native/capture_dependency_tasks.rs` `order_for_picker` (used by `capture_complete/candidates.rs` and `capture_task_wikilinks.rs`)                    | `u8::from(!open)` is the primary key: completed history trails open matches at any score                                                     | **fix**      |
| bob-cli `:` task_link, `*` blocked_task, `~` relation contexts               | `capture_link_tasks::rank`, `candidates.rs`                                                                                                               | score first already                                                                                                                          | none         |
| Bob Mac Capture filtered pickers (dependency, relation, wikilink, `:` link)  | `Sources/CaptureCore/*PickerPresentation.swift`                                                                                                           | each filtered view re-ranks locally: score, then Bob's snapshot order (open before completed)                                                | none         |

**Latent tie bug in the nav stages.** Each nav planner sorts query results by the
candidate's index in `dependencyStageRank(...)` output, and that index is unique per
candidate. The canonical comparator after it never runs. `dependencyStageRank` keeps
ties in input order, and the input (`fresh`) is pool order, not canonical order. So
today's exact-score ties are effectively unordered. The new ordering below fixes this as
a side effect.

The Links / Linked-from stages keep the accepted `blocked_ties = peer` rule
(`plan:202610/links_stage_blocked_ranking.md`, bob-cli `docs/task-relations.md` LG17).
In those stages, equal-score Blocked rows share Ready's lane. Closed and archived rows
still sink on ties through `compareRelationStageCanonical`.

## Changes

Open `bob-plugins` with `/sase_repo` and read its `AGENTS.md`. Edit only `src/`
fragments, regenerate `main.js` with `npm run build`, and keep every hand-edited
fragment at or below 1000 lines. `300-dependency-stage.js` is at 966, so the edit there
must be net-neutral or smaller. Put shared helpers in `290-local-task-items.js` (about
694 lines). Top-level helper names share one script scope per plugin, so grep for
collisions before adding one. Do not edit bob-cli `Cargo.toml` or `CHANGELOG.md`
(release-train decision).

### 1. Shared score helper (bob-navigation-hotkeys `290-local-task-items.js`)

- Add a helper that returns the matching candidates with their summed tier scores, for
  example `dependencyStageScoredMatches(candidates, query)` → `[{ candidate, score }]`.
  It uses the same term split, AND rule, and `dependencyStageTermTier`. An empty query
  returns every candidate with score 0.
- Rebuild `dependencyStageRank` on top of it with identical output: stable sort by score
  descending. The DK vectors and the ranker-and-pool suite must stay unchanged.
- Add one ordering helper, for example
  `orderDependencyStageMatches(candidates, query, tieCompare)`. It returns only the
  matches, sorted by score descending, then `tieCompare(left, right)`. Every nav planner
  below uses it instead of the `rankIndex` sort.

### 2. Depends on stage (`300-dependency-stage.js` `planDependencyStageView`)

- Tie comparator: open (`!candidate.blocked`) before Blocked, then
  `compareDependencyStageCanonical(left, right, dependentPath)`. At equal score
  (including the all-zero empty query), this reproduces today's RESULTS-then-BLOCKED
  order exactly. It also makes typed-query ties canonical instead of pool order.
- The empty-query pool filter (same note, In Progress `/`, Next `*`) is unchanged.
- Build one ordered list. Apply the ~60-row cap (`room`) to that merged score order, so
  a strong Blocked match is never folded into `type to search N more`.
- Each row keeps `stageSection` as its **status class**: `"results"` or `"blocked"`.
  That value still drives the `bob-cnp-dep-blocked` muted title, the `is-stage-*` class,
  the `🔒 waits on N / scheduled D / blocked` badge (`shape(…, "blocked")`), and the
  counted-mixed badge order. Do not change those consumers.
- Section headers: add a row field `headerSection`
  (`"current" | "results" | "blocked" | "closed" | null`) and keep `firstInSection` true
  exactly when `headerSection` is set.
  - CURRENT rows: unchanged (`"current"` on the first one).
  - Typed query: only the first ranked row gets `headerSection: "results"`, whatever its
    status class.
  - Empty query:

    > [!decision] empty_query_sections = keep The first open row gets `"results"`, and
    > the first Blocked row gets `"blocked"` (today's look, since every score ties).

    > [!decision] empty_query_sections = drop Only the first ranked row gets
    > `"results"`, as in a typed query.

- The `stage-more` row takes the status class of the last picked row (or `"results"`).
  It never carries a header.

### 3. Blocks stage (`302-blocks-stage-view.js`)

Same change as §2:

- Status class `blocked` is `candidate.blocked || candidate.status === "?"`, the
  predicate the split uses today.
- Tie comparator: open before Blocked, then
  `compareBlocksStageResults(left, right, sourcePath, todayMap)`.
- Use the same merged cap and the same `headerSection` rules.
- Blocked rows keep `stageSection: "blocked"`, so the consequence strip still reads
  `Also waits on this (N open)` (`303-blocks-stage-copy.js`) and keeps the 🔒 badge.

### 4. Links to / Linked from stages (`304-relation-stage-view.js`)

- `planRelationStageView` (Links to):
  - Query order: score descending, then the existing `compareRelationStageCanonical`.
    That comparator already sinks archived, then closed, rows on ties and keeps Blocked
    as Ready's peer.
  - Build one merged list and apply one cap. Rows keep `stageSection` `"results"` or
    `"closed"` (closed or archived) as their status class.
  - Headers follow the §2 rules, with `"closed"` in place of `"blocked"` for the
    empty-query branch.
  - The empty-query pool (same note, `/`, `*`, closed, archived) is unchanged.
- `planLinkedFromStageView`:
  - Replace the `rank` index sort with the score helper and tie comparator
    `compareRelationStageCanonical`. The empty query keeps its open-first sort.
  - Rows stay one RESULTS section (add `headerSection: "results"` on the first row).

### 5. Renderer (`410-picker-filter-and-render.js` `renderTaskValueItem`)

- Print the section header when `item.firstInSection`, with the text
  `sectionNames[item.headerSection || item.stageSection]`.
- Leave every other `stageSection` use (blocked tint, badges, `is-stage-*`) as is.
- Update the comment above it: one RESULTS list ranked by score; BLOCKED / CLOSED
  headers appear only per `empty_query_sections`.

### 6. `((` task wikilink picker (block-id-prompt)

- `032-task-wikilink-rank.js` `orderForWikilinkPicker`: drop `openOrder` as the primary
  key. Sort by score descending, then the canonical parts already stored (`section` —
  completed history is 4 — then `hidden`, `note`, `line`). Rewrite both header comments:
  completed history ranks by score, and open rows lead only at equal score. Keep the
  per-row allocation-free shape; the performance suite
  (`scripts/test-block-id-prompt-task-wikilink-index-performance.cjs`) must stay green.
- `088-task-wikilink-picker-view.js` `buildTaskWikilinkFilteredItems`: stop inserting
  the "Completed history" divider. The filtered list becomes purely flat, because
  completed rows now interleave by score. Completed rows keep their existing completed
  styling.
- The empty-filter grouped view (`buildTaskWikilinkGroupedItems`) and the Pomodoro
  mode's open-only pool are unchanged.

### 7. bob-cli `order_for_picker` (`src/native/capture_dependency_tasks.rs`)

- Change the comparator to score descending, then `canonical_key`. That key's first
  component already sends completed history (section 4) last on ties.
- Rewrite the doc comment and the inline comment ("Completed history always trails open
  matches" / "at any score") to say completed history trails only at equal score.
- This fixes both the capture `&` dependency context and the `((` `wikilink_task`
  context. `docs/capture.md` already promises "open matches ahead of completed ones at
  equal score" for `((`.

### 8. Tests

bob-cli:

- `capture_dependency_tasks.rs` unit test: a closed task matching as a field prefix (for
  example `- [x] #task Widget launch ^widget-launch`) precedes an open task matching
  only as a later word (`- [ ] #task Polish the widget ^polish`). An equal-score
  open/closed pair still lists the open one first.
- `tests/cli/capture/complete_dependency.rs` and
  `tests/cli/capture/complete_wikilink_task.rs`: add one query each where a completed
  row outranks an open one and is listed first. Keep
  `query_ranks_open_matches_ahead_of_history` (its `kept` pair ties) and rename or
  comment it as the equal-score case.
- `tests/fixtures/task_wikilinks/vectors.json`: add one vector (for example
  `rank-history-best-match-first`) with the widget pair above. Generate its expected
  `rows` from bob's actual output and check them against the rule by hand. The Rust
  vector test must pass.

bob-plugins:

- Copy the updated `vectors.json` verbatim to
  `scripts/fixtures/task_wikilinks/vectors.json`. Update `EXPECTED_FIXTURE_SHA` and the
  sha in the header comment of `scripts/test-block-id-prompt-task-wikilink-model.cjs`.
- `scripts/test-block-id-prompt-task-wikilink-picker-view.cjs`: replace "filtered list
  is flat with a Completed history divider" with a test that the filtered list has no
  `section` items and that a better-matching completed row comes first.
- `scripts/test-navigation-dependencies-stage-ranker-and-pool.cjs`: update "stage view
  groups CURRENT, RESULTS, and BLOCKED with guards" to the new header rules. Add:
  1. **Screenshot repro:** Ready rows that match `spike` only as a subsequence, plus the
     Blocked `Spike S0: …` in another note. The first ranked row is the Blocked one,
     with `stageSection: "blocked"`, a 🔒 badge, and `headerSection: "results"`. No row
     has `headerSection: "blocked"`.
  2. **Equal-score ties:** open rows (In Progress, Next, Ready in canonical order)
     precede Blocked rows.
  3. **Cap:** 70 weaker Ready matches plus one strong Blocked match. The Blocked row is
     rendered, not folded into stage-more.
  4. **Empty query** per `empty_query_sections`.
- `scripts/test-navigation-blocks-stage-view.cjs`: the same four cases through
  `planBlocksStageItemsFromVault`.
- `scripts/test-navigation-relation-stage-render.cjs`: keep the existing render =
  planner deep-equality. Add a Links-to case where a closed row matching best is first
  and carries no CLOSED header. Adjust the empty-query CLOSED assertion per
  `empty_query_sections`. The peer tie test must still pass.
- A render test (pattern: the `render({...})` helper in
  `scripts/test-navigation-dependencies-stage-entry-and-view.cjs`): a first-ranked
  Blocked row prints `RESULTS` as its header and still shows its 🔒 badge.
- If any other existing assertion pinned the score-blind split, fix it to the new
  contract and say so in the commit message.

### 9. Docs

bob-cli (`just docs-check`):

- `docs/task-dependencies.md`:
  - §6.1 layout sentence: CURRENT and RESULTS sections, and a BLOCKED section only per
    `empty_query_sections`.
  - §6.3: replace "Blocked candidates render in their own BLOCKED section beneath
    RESULTS" with the rule. Match score leads, so a Blocked candidate that matches
    better ranks above Ready ones inside RESULTS. Ties order open candidates, then
    Blocked, each in canonical order. The cap counts the merged order.
  - §15.3: under "For a typed query…", say rows render in one RESULTS list with Blocked
    rows marked by 🔒.
  - §15.4: under `drop`, remove BLOCKED from the mock.
- `docs/capture.md` (~line 2328): replace "keep completed history findable behind open
  matches" with "rank completed history by score too, with open matches ahead only at
  equal score".
- `docs/task-relations.md` LG table: append the next free LG number:
  `Links stage query that a closed task matches best | the closed row ranks by score inside RESULTS; CLOSED appears only on an empty filter`
  (under `drop`: `…; the stage never shows a CLOSED section`).

bob-plugins (`npm run docs:check`):

- `docs/task-relations.md`, Links stage paragraph: RESULTS ranks every match (open,
  Blocked, closed, `done/` archives) by score. Exact-score ties put In Progress and Next
  first, then Ready and Blocked together, then closed, then archives. Describe the
  empty-filter CLOSED section per `empty_query_sections`.

Bob Mac Capture needs no code or doc change. Its filtered pickers already rank by score,
and its docs already describe them that way.

### 10. Memory (decision `decision_record`)

> [!decision] decision_record Use `/sase_memory_write`, then add
> `sase/memory/decisions/task-pickers-rank-by-match.md` in the existing strand format
> (`keyword`, `aliases`, `summary`, `metadata.status: accepted`, `decided`).
>
> - **Applies to:** nav Depends on / Blocks / Links / Linked-from stages, the `((`
>   picker, and bob-cli `order_for_picker`.
> - **Claim:** score leads, and status only breaks ties.
> - **Why:** best matches must be visible without scrolling. Rejected alternatives:
>   status sections, and an open-first partition.
> - **Cost:** status runs only exist on an empty filter.
> - **Reopens when:** a picker needs actionable rows pinned above better matches.
>
> Cite this plan and the touched docs. Then run `sase memory init`.

### 11. Build, verify, deploy

- bob-cli: `just fmt`, `just lint`, `just test` (or `just all`), `just docs-check`.
- bob-plugins: `npm run build`, `npm test`, `npm run docs:check`, then
  `bob plugins sync` (required by its `AGENTS.md`; follow its refusal rules, never `-F`
  on your own).
- **Live read-only check.** Write a scratch Node script outside both repos that never
  writes to `~/bob`. Load the built `plugins/bob-navigation-hotkeys/main.js` with the
  `obsidian` / `@codemirror/view` stubs the `scripts/test-*.cjs` files use. Build the
  Depends-on pool from `~/bob` with `helpers.createDependencySnapshotMap` and
  `helpers.collectVaultDependencyCandidatesFromSnapshots`, then run
  `planDependencyStageView` for the query `spike`. Confirm the Blocked
  `Spike S0: write the promise sentence…` task is the first ranked row. Report the
  before/after index in the final summary.

## Out of scope

- The ranker tiers (`dependencyStageRank` scoring, `capture_link_tasks::rank`, the DK
  vectors) and the Links-stage `blocked_ties = peer` tie rule.
- Bob Mac Capture's `/` Go To picker (`TaskLinkPickerPresentation` `.taskGoto`), which
  still lists closed rows after open ones at any score. It navigates rather than links.
  Mention it in the final summary as a possible follow-up.
- Successor offer pickers (grouped offers, not a vault search).
