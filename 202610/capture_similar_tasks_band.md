---
tier: epic
title: Similar tasks band in Bob Mac Capture
goal: 'When a capture draft will create new tasks, Bob Mac Capture quietly shows the
  few existing tasks each one may duplicate. The rows appear after typing settles,
  at the foot of the new task''s own preview card, and take zero pixels otherwise.
  One keyboard-driven draft edit can fold, relate, or link a match, and typing, the
  live preview, and Return stay exactly as fast as today.

  '
decisions:
  duplicate_default:
    ask: Which action should ↩ apply on a focused duplicate row?
    choices:
      split: ⌫ Discard mine for Same title, + Add as note for Possible duplicate
      note_both: + Add as note for both, so ↩ never removes your words
    default: split
    why: An exact recapture is almost always a forgotten task; a lexical near-match
      may not be
    answer: split
  calibration:
    ask: Should the calibrate phase wait for your ~100 labels before it fixes the
      thresholds?
    choices:
      labels: Calibrate pauses for your labels and sets the thresholds from them
      starting_bars: Ship the research's starting thresholds; labelling becomes a
        follow-up task
    default: labels
    why: R10 says the thresholds come from your labels, not researcher labels
    answer: labels
  decision_record:
    ask: Record the Similar tasks band as a new accepted decisions record?
    memory:
    - decisions
    - decisions:similar-tasks-are-a-settled-heads-up
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: true
phases:
- id: engine
  title: Matching engine and the --query harness
  depends_on: []
  size: medium
  description: 'engine: add bob capture-related with the Bob-aware tokenizer, the
    per-call TF-IDF scorer, candidate discovery (open tasks in routable notes plus
    a closed window), bands, and --query human/JSON output, with table tests and a
    synthetic-vault perf test.'
- id: contract
  title: Draft mode and the capture-related JSON contract
  depends_on:
  - engine
  size: medium
  description: 'contract: accept a capture draft, gate task-creating items, exclude
    and report draft-linked tasks, emit item anchors, actions, defaults, and replacements
    as schema_version 1, with goldens, a perf gate, and docs.'
- id: calibrate
  title: Calibrate thresholds and measure on the Mac
  depends_on:
  - contract
  size: medium
  description: 'calibrate: build the leave-one-out sampling script, collect Bryan''s
    labels, set the bars from them, time a release build on the Mac against the 50
    ms gate, and file a calibration report in the research sidecar.'
- id: mac_lane
  title: Mac related lane, settle policy, and setting
  depends_on:
  - contract
  size: medium
  description: 'mac_lane: decode the contract, add the cancellable related lane with
    old-bob fallback, a pure band-state reducer, the 350 ms settle wiring in CapturePanelModel,
    the Settings toggle, signposts, fake-bob support, and protocol docs.'
- id: mac_band
  title: Band view inside the preview card
  depends_on:
  - mac_lane
  size: medium
  description: 'mac_band: render the band at the foot of each task-creating card (header,
    rows, capsules, bulk rows, folded line), fold before scrolling, cross-fade, VoiceOver
    announcements, render fixtures, and capture-previews docs.'
- id: mac_actions
  title: Row focus, action strip, and draft-edit actions
  depends_on:
  - mac_band
  size: medium
  description: 'mac_actions: add ⌘1–⌘5 focus, the sigil action strip, key routing,
    click handling, undoable splices for relate/note/needs/blocks on rows with IDs,
    Open that keeps the draft, Discard mine, footer hints, a11y actions, and keyboard
    docs.'
- id: mac_idless
  title: ID-less rows and the +N reveal
  depends_on:
  - mac_actions
  size: small
  description: 'mac_idless: route link actions on ID-less rows through the existing
    Add block ID prompt, reveal hidden matches with ↓ while focused, and finish the
    docs, fixtures, and end-to-end checks.'
- id: record
  title: Decisions record for the band
  depends_on:
  - calibrate
  - mac_idless
  size: small
  description: 'record: write the accepted decisions record through /sase_memory_write,
    citing the research, this epic, the phase commits, and the Mac measurements, then
    republish memory.'
proposed_by: bbugyi200.apollo.research.27.linker.w0
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-20 17:44:11
status: done
bead_id: bob-cli-ab
---

- **PROMPT:** [prompts/202610/capture_similar_tasks_band.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/capture_similar_tasks_band.md)
- **BEAD:** [bob-cli-ab](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ab/README.md)

# Plan: Similar tasks band in Bob Mac Capture

## Context

Bryan wants the capture preview to warn when a new task probably already exists, and to
make relating or linking it easy. The research report
`research:202610/capture_similar_tasks_band/capture_similar_tasks_band.md` (read it with
`sase artifact read`) critiques that idea and recommends a design. **Bryan accepted
every requirement in that report**, so these are fixed for this epic:

- **R1 Similar, not related.** The band is named "Similar tasks". A row gets a **Same
  title** or **Possible duplicate** mark only when the evidence supports it. No
  percentages, and no score is ever shown as a probability.
- **R2 Zero pixels unless a match clears the bar.** No header, no placeholder, and never
  a "No duplicates" line.
- **R3 Settle-gated.** Search after about 350 ms without edits. Rows on screen hold
  while the user types and swap only if the settled result changed.
- **R4 Scope.** Every open task (Ready, Blocked, Next, In Progress) in routable notes,
  plus tasks closed in the last 30 days at the duplicate bar only (at most 1 row).
- **R5 Gate.** Only items that create a task: `task`, `pomodoro_task`, and the `^prj`
  title of `project_note`/`pomodoro_project_note`. Require ≥ 2 meaningful terms, with a
  stricter bar under 3 terms except for an exact title match. Skip `@@template:` drafts.
- **R6 Cap.** 2 rows for a single-item draft plus a "+N" capsule, 1 row per item in bulk
  drafts, and at most 5 per draft.
- **R7 Actions are explicit draft edits only.** Return is never intercepted or
  re-confirmed, nothing is auto-linked, and `@` and `!` are not v1 actions.
- **R8 Numbers.** Zero extra spawns per keystroke. Preview p95 is unchanged with the
  feature on versus off. The related lane's p95 on the Mac is ≤ 50 ms.
- **R9 Settings toggle.** "Show similar tasks while capturing", on by default.
  Thresholds live in bob, not in Settings.
- **R10 Calibrate first.** A harness, a labelled set, and a new `decisions` record.

Governing records (read them with `sase memory read` before changing related behavior):

- `decisions:mac-capture-is-a-thin-client`: bob owns meaning; the app owns timing and
  pixels and never decides syntax.
- `decisions:idle-capture-shows-ledger-agenda`: the eye line, the below-eye-line budget,
  and stale-while-revalidate stores.
- `decisions:unblocked-tasks-are-offered`: offer choices, never apply them implicitly.
- `decisions:capture-link-drops-blocking-deps`: this is why `@route:id` is not a one-key
  action. Most open tasks are Blocked, and that gesture drops prerequisites.

Repositories:

- **bob-cli** is this repo.
- **bob-mac-capture** is the linked repo. Open it with `/sase_repo`
  (`sase repo open bob-mac-capture -r "<why>"`) and follow its `AGENTS.md`.
- **bob-cli--research** is the research sidecar. Open it with `/sase_repo` and register
  new files with `/sase_artifact`.

## Design

### Principles

1. **A heads-up, never a gate.** The band answers one question: "did I already write
   this down?" It cannot block, confirm, or change what Return does.
2. **bob owns meaning; the app owns timing and pixels.** Eligibility, tokens, weights,
   bands, hysteresis, exclusions, action availability, defaults, splice anchors, and
   replacement strings all come from `bob capture-related`. The app schedules the call,
   holds and fits rows, and applies the edits bob describes.
3. **Calm over clever.** The band appears only after a pause and only when something
   clears the bar. It never reorders while you type. It cross-fades once and never
   animates height.
4. **The keys are the grammar.** The action strip's keys are the capture sigils
   themselves (`~ + & * /`), so using the band teaches the language instead of adding
   vocabulary.
5. **Every action is a draft edit.** bob's live preview then explains the real
   consequence before anything is written, and ⌘Z undoes it. The one exception is the
   existing, explicit Add block ID prompt for ID-less targets.

### What the user sees

**Settled, single item.** The band sits below the card's destination line, separated by
a hairline:

```text
┌ preview card ──────────────────────────────────────────────────────────────┐
│ + dev · end of Tasks · task                                                  │
│ - [ ] #task Install UPS battery [created:: 2026-10-20]                       │
│                                                                      dev.md  │
│ ─────────────────────────────────────────────────────────────────────────── │
│ ⧉ SIMILAR TASKS                                                       ⌘1–2  │
│ ⌘1  ⏸  Install new **UPS battery**   ( Possible duplicate )   dev · Blocked │
│ ⌘2  ○  Order a replacement **UPS battery** for athena          dev · Ready  │
└──────────────────────────────────────────────────────────────────────────────┘
```

**⌘1 pressed.** The row is focused and its action strip opens on a second line. The
default action is preselected. The footer swaps to key hints, so the meaning of ↩ is
always visible:

```text
│ ⌘1  ⏸  Install new **UPS battery**   ( Possible duplicate )   dev · Blocked │  ← accent fill
│        [~ Relate] [+ Add as note] [& Needs it] [* Blocks it] [/ Open] [⌫ Discard mine]
footer:  ↩ Add as note   ← → choose   ↑ ↓ rows   esc back
```

**After `~`.** The draft gains ` ~dev:install-ups-battery`, and bob's preview shows the
🪢 LINKS row. On the next settle bob reports the task as linked, so the row stays with a
mint **Related** capsule instead of disappearing.

**Bulk draft.** Each task-creating card gets at most one row and no header line. A
leading `⧉` mark identifies the row. Numbering continues down the stack:

```text
│ 1  mac_inbox · end of Tasks · task
│ ⧉ ⌘1  ✓  Close all open task beads          ( Done 8d ago )            sase │
│ 2  mac_inbox · end of Tasks · task
│ ⧉ ⌘2  ○  Rename **xprompts** to hash-prompts      ( +1 )         sase_better │
```

**Tight space.** If the band would make the auxiliary region scroll, it folds to one
line before anything else moves:
`⧉ 2 similar tasks · Possible duplicate: Install new UPS battery   ⌘1`.

### Visual spec

- **Placement.** The band goes inside the item's card, after every capture consequence
  (destination, transitions, relationship rows, blocks, budget). It never displaces any
  of them.
- **Hairline.** Reuse `BlockDiffCard.hairline`: separator colour at 0.5 opacity, full
  strength under Increase Contrast, 0.5 pt, 8 pt above the header.
- **Header** (single-item only, about 16 pt):
  - `square.on.square` (caption2, tertiary);
  - "SIMILAR TASKS" (caption2, semibold, +0.6 tracking, secondary);
  - an optional "+N" capsule (caption2, tertiary on a quaternary capsule);
  - a spacer, then a trailing hint `⌘1–2` or `⌘1` (caption2 monospaced, tertiary). The
    hint hides while a row is focused.
- **Row** (one line, about 22 pt, callout):
  - a keycap chip `⌘N` (reuse `CaptureOpenHintChip` styling);
  - the status glyph from `CaptureEditorPalette.taskStatus`;
  - the title: primary colour, matched ranges **semibold**, tail truncation;
  - a kind glyph (`folder` for a `^prj` project task, `book` for a `#ref` reading task;
    tertiary; it has an accessibility label);
  - at most one capsule;
  - a spacer of at least 8 pt;
  - the locator `route · Status` (caption, secondary).

  The locator truncates first, then the title. Capsules never wrap or shrink.

- **Capsules.** Use caption2 semibold, padding 7/3:
  - **Same title** and **Possible duplicate**: orange text on `orange.opacity(0.15)`,
    raised to 0.25 under Increase Contrast. This is the app's existing warning tint.
  - **Done 8d ago** and **Canceled 8d ago**: secondary text on quaternary.
  - **Related**, **Prerequisite**, and **Blocked by this**: these appear only on a held
    row after you act. Use the palette's relation (mint), dependency, and blocked-task
    (teal) category colours.
  - **+N**: tertiary on quaternary.
- **Closed rows.** The title is secondary and never struck through, and the glyph is
  green `checkmark.circle` or `xmark.circle`.
- **Focus.** The row gets an accent fill at 0.16 opacity (0.30 under Increase Contrast)
  with a 5 pt radius, matching picker rows.
  - The strip is a second line (about 20 pt) aligned to the title's leading edge.
  - Each chip shows a monospaced sigil plus a label on a hairline capsule. The chosen
    chip is accent-filled with white text.
  - Only the actions bob lists are shown, always in the fixed order `~ + & * / ⌫`.
- **Hover.** A quaternary fill, plus `.help` showing the full title and note path.
  Hovering never changes the layout.
- **Motion.** The content cross-fades over 120 ms when a settled swap changes rows. With
  Reduce Motion it changes instantly. Height is never animated.
- **Never show** percentages, sparkles, spinners, skeletons, per-row coloured
  backgrounds, or empty states.
- **Light and dark mode** use semantic colours only.

### Interaction spec

While no row is focused, the editor keeps every current binding. ⌘1–⌘5 are free today:
`CaptureKeyCommandRouter` binds no ⌘-digit, and the app menu uses none.

| Key                                 | When                                         | Effect                                                                                                                                          |
| ----------------------------------- | -------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| ⌘1 … ⌘5                             | Band rows visible; the editor owns the panel | Focus that row and open its strip. Pressing the same key again unfocuses it. In a folded band, ⌘1 shows that row and its strip until unfocused. |
| ← →                                 | Row focused                                  | Choose an action in the strip                                                                                                                   |
| ↑ ↓                                 | Row focused                                  | Move between rows. ↓ from the last visible row of an item reveals its next hidden match, up to bob's limit (`mac_idless`).                      |
| ↩                                  | Row focused                                  | Apply the chosen action. The default is preselected.                                                                                            |
| `~` `+` `&` `*` `/`                 | Row focused                                  | Apply that action directly                                                                                                                      |
| ⌫                                   | Row focused, and bob offers `discard`        | Discard mine                                                                                                                                    |
| esc                                 | Row focused                                  | Unfocus and return to typing. With nothing focused, Esc keeps today's panel behaviour.                                                          |
| any other key, including ⌘Z and ⌘↩ | Row focused                                  | Unfocus, then pass the key to the editor unchanged                                                                                              |
| click row / click chip              | —                                            | Focus the row / apply that action                                                                                                               |

Actions:

| Action       | Sigil | Draft edit (anchors from bob)                                                                                                                                                                       | Default for                      |
| ------------ | ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- |
| Relate       | `~`   | Insert `" " + relation_to` at the item's `marker_at`                                                                                                                                                | `similar`                        |
| Add as note  | `+`   | Replace the item's `route_range` with `parent`, or insert `" " + parent` at `marker_at` when the range is null. The item becomes a sub-bullet (with any authored children) under the existing task. | `possible_duplicate` (see below) |
| Needs it     | `&`   | Insert `" " + dependency` at `marker_at`                                                                                                                                                            | —                                |
| Blocks it    | `*`   | Insert `" " + blocked_task` at `marker_at`                                                                                                                                                          | —                                |
| Open         | `/`   | No edit. Open the task through the existing Go To URL path, then hide the panel **keeping the draft**.                                                                                              | closed rows                      |
| Discard mine | `⌫`   | Delete the item's `discard_range`                                                                                                                                                                   | `same_title` (see below)         |

> [!decision] duplicate_default = split bob emits `default_action: "discard"` for
> `same_title` rows whose item has a `discard_range`. It emits `"note"` for `same_title`
> rows whose item has authored children, and `"note"` for `possible_duplicate` rows.

> [!decision] duplicate_default = note_both bob emits `default_action: "note"` for both
> `same_title` and `possible_duplicate` rows. Discard stays available on its key and its
> chip, but is never preselected.

In both branches, `similar` rows default to `relate` and closed rows default to `open`.
The defaults live in bob, so they can be retuned without an app release.

**After each action** the app sets a status line and posts one VoiceOver announcement:
"Related to Install new UPS battery", "Added as a note under …", or "Removed your item —
⌘Z restores it".

### Band lifecycle

```text
None ──settle & match clears bar──▶ Shown ──keystroke──▶ Held ──settle──▶ Shown (swap only if changed)
  ▲                                   │                    │
  └──────── settle finds nothing ─────┴── item's mode changes or item count changes ──▶ None
Shown ──⌘N / click──▶ Focused ──esc / action / other key──▶ Shown
```

- **Held** means the last settled rows keep rendering whenever the card renders. The
  card itself still shows today's per-keystroke spinner, and its height is held by
  `CapturePreviewPaneHeightPolicy`.
- **A band renders on item `i` only while the latest parse reports item `i` with the
  same `mode` bob matched it as.** This drops the band immediately when, for example,
  `@dev+id` turns the item into a sub-bullet. The app needs no list of modes in Swift.
- **If the parse's item count changes,** every held band clears until the next settle. A
  band can never attach to the wrong card.
- **While a row is focused** the band is frozen. A settled result that arrives then (on
  panel re-show) is kept pending and applied on unfocus.

## bob contract: `bob capture-related`

### CLI

The command is a frozen-name `capture-*` sibling: read-only, in the "Capture protocol"
help section, placed alphabetically between `capture-pomodoros` and `capture-rewrite`.
It is never nested under `bob capture`, and every option has a short alias
(`sase memory read cli_rules.md`).

```text
Usage: bob capture-related [OPTIONS] [-- <DRAFT>]

List existing tasks that each new task in a capture draft may duplicate or relate to.
Read-only; never writes the vault.

Options:
  -b, --bob-dir <DIR>    Bob vault root; defaults to BOB_DIR or ~/bob
  -d, --days <N>         Also match tasks closed in the last N days [default: 30]
  -f, --format <FORMAT>  human or json [default: human]
  -h, --help             Show help
  -l, --limit <N>        Matches per item [default: 5]
  -q, --query <TEXT>     Rank TEXT as one new task title (evaluation harness)
```

`--query` and `<DRAFT>` are mutually exclusive, and one of them is required.

### Scope and eligibility

- **Open candidates.** Every open `#task` (statuses ` `, `/`, `*`, `?`) in routable
  notes: `mac_inbox`, top-level area notes, and non-terminal project notes. This is the
  same catalog as `capture_link_tasks` (`capture_targets::scan_capture_targets` plus
  `note_tasks::scan`). It includes `^prj` project tasks and `#ref` reading tasks, which
  carry `kind`.
- **Closed candidates.** Tasks done or canceled within `--days` (by `[completion:: D]`,
  `[cancelled:: D]`, or the emoji forms; see `ref_tasks::close_date`). They come from
  routable notes and from `done/**/*_done.md` archives whose mtime falls inside the
  window. Map archives to their source note for the locator
  (`collect_done::source_relative_path_from_archive`).
- **Excluded:** `#gtd` checklist rows, closed instances of recurring tasks
  (`task_complete::is_recurring_task_line`), and daily notes. Tasks the draft already
  references are excluded too and reported under `linked`.
- **Draft items.** Parse with `capture_language::parse_for_editor_with`, using URL
  routing loaded the same way `capture-parse` does, so a bare URL stays `ref`.
  - Only items whose mode is `task`, `pomodoro_task`, `project_note`, or
    `pomodoro_project_note` qualify. For project notes, only the title is matched.
  - `@@template:` drafts return `items: []`.
- **No vault I/O unless something qualifies.** Gating happens before discovery. A draft
  with no qualifying item returns `{"ok": true, "schema_version": 1, "items": []}` at
  spawn cost.

### Matching

- **Tokenizer.** Bob-aware and golden-tested.
  - Lowercase. Split camelCase, snake_case, and kebab-case. Apply light stemming (‑s,
    ‑es, ‑ies, ‑ed, ‑ing) and a stopword list. Reduce wikilinks to their alias, or to
    their basename when there is none.
  - Strip inline fields, `#tags`, `^id`, URLs, Markdown links' targets, emoji, and the
    💡 Keep link.
  - **Code spans and `<key+chords>` are atomic terms with double weight.**
  - Every token records its UTF-8 span in the display title, for `matched_ranges`.
- **Display title.** Use the agenda's clean title (`note_tasks::clean_description`, then
  `capture_pomodoro_close::close_task_text`) with wikilinks reduced to their alias.
  `matched_ranges` index into this string.
- **Score.** TF-IDF cosine with sublinear tf (1 + ln tf) and idf = ln((N + 1)/(df +
  0.5)). It is computed per call over the candidate corpus, which is sub-millisecond at
  about 1,000 documents. Thresholds compare raw scores.
- **Starting bars** (the `calibrate` phase replaces them):
  - `same_title`: normalized titles are equal (casefolded, with punctuation, tags,
    fields, and emoji removed). Applies to open or closed tasks, and to any term count.
  - `possible_duplicate`: ≥ 0.70, open tasks.
  - `similar`: ≥ 0.55, open tasks.
  - `closed_duplicate`: ≥ 0.70 or `same_title`, closed in the window. At most 1 per
    item, and only when the item has no open `same_title` or `possible_duplicate`.
  - Queries with fewer than 2 meaningful terms match only `same_title`. Queries with
    fewer than 3 terms raise every bar by 0.10.
  - `hold`: an open candidate scoring within 0.10 below the `similar` bar in effect.
- **Ordering.** First by band rank (`same_title` > `possible_duplicate` > `similar` >
  `closed_duplicate`). Then by score rounded to 0.01, descending. Ties break as follows:
  open before closed; then In Progress, Next, Ready, Blocked; then the item's own route
  (a tie-break only, never a reason to show a row); then recency (`created`, or the
  close date).
- **Bulk dedupe.** A candidate appears under only one item, the one where it scores
  highest.
- **Limits.** `--limit` caps the non-hold matches per item. Hold entries are capped at
  the same number.
- **Bars are bob-internal constants.** Retuning them is not a schema change.

### Item anchors, actions, and replacements

bob owns the splice positions, so the app never parses grammar. All offsets are UTF-8
byte offsets into the draft.

- **`marker_at`**: where `" " + marker` can be inserted on the item's parent line
  without changing any other token's meaning. It is the end of the parent line's
  content, before any trailing whole-item suffix that must stay last (for example a
  Pomodoro `=`/`=x` suffix).
- **`route_range`**: the item's own destination token (`@route`, `@route^id`, or a new
  task's `@route:id`), or null.
- **`discard_range`**: the item plus one adjacent blank-line separator (the following
  one, or the preceding one for the last item), so the remaining items stay separated.
  For a single-item draft it is the whole draft. It is null when the item has authored
  children.

**Matches and actions.**

- Each match carries `actions`: the subset of `relate`, `note`, `needs`, `blocks`,
  `open`, `discard` that is valid for it, plus a `default_action`.
  - Closed rows offer only `open` and `discard`.
  - `discard` requires a non-null `discard_range`.
- **bob verifies every offered link action** by simulating its edit on the draft and
  re-parsing:
  - `relate`, `needs`, and `blocks` must keep the item's mode and add exactly one
    marker;
  - `note` must yield `sub_bullet` with no error diagnostics.

  For ID-less rows, the simulation uses the first block-ID suggestion.

- **`replacements`** are bob's strings (`relation_to`, `parent`, `dependency`,
  `blocked_task`), built with the existing replacement helpers. They are present only
  when the task has a block ID.
- ID-less rows instead carry `requires_block_id: true`, `block_id_suggestions`, `route`,
  `note_path`, and `ref`. The app then runs the existing Add block ID flow
  (`capture-task-id --route … --task-ref … --block-id …`), which returns the same
  replacement strings.

### JSON (`schema_version: 1`, additive thereafter)

```json
{
  "ok": true,
  "schema_version": 1,
  "warnings": [],
  "items": [
    {
      "index": 0,
      "mode": "task",
      "range": { "start": 0, "end": 24 },
      "marker_at": 24,
      "route_range": { "start": 20, "end": 24 },
      "discard_range": { "start": 0, "end": 24 },
      "matches": [
        {
          "key": "dev.md#install-ups-battery",
          "band": "possible_duplicate",
          "title": "Install new UPS battery",
          "matched_ranges": [
            [12, 15],
            [16, 23]
          ],
          "kind": "task",
          "status_symbol": "?",
          "status_name": "Blocked",
          "status_type": "ON_HOLD",
          "closed": null,
          "route": "dev",
          "locator": "dev",
          "note_path": "dev.md",
          "target": "/Users/test/bob/dev.md",
          "line": 42,
          "ref": "42:ab12cd34",
          "block_id": "install-ups-battery",
          "requires_block_id": false,
          "block_id_suggestions": [],
          "replacements": {
            "relation_to": "~dev:install-ups-battery",
            "parent": "@dev+install-ups-battery",
            "dependency": "&dev:install-ups-battery",
            "blocked_task": "*dev:install-ups-battery"
          },
          "actions": ["relate", "note", "needs", "blocks", "open", "discard"],
          "default_action": "note"
        }
      ],
      "linked": [{ "key": "dev.md#ups-order", "link": "relation" }]
    }
  ]
}
```

Field notes:

- **`index`** is the item's 0-based position among _all_ draft items. That is the
  dry-run `captures` order the app already renders.
- **`key`** is stable across line moves (`note_path#block_id`, or `note_path#` plus a
  text digest). The app uses it for hold, linked, and dedupe identity.
- **`band`** is one of `same_title`, `possible_duplicate`, `similar`,
  `closed_duplicate`, or `hold`.
- **`closed`** is `{"status": "done" | "canceled", "date": "YYYY-MM-DD", "days_ago": N}`
  or null.
- **`kind`** is `task`, `project`, or `reading`.
- **`link`** is `relation`, `dependency`, `blocked_task`, or `parent`.
- **`target`** and **`line`** are exactly what the Go To URL needs.
- **Draft-mode JSON never carries a score.** `--query --format json` adds `score` (3
  decimals) for the harness only.
- **Errors** follow the sibling commands: `{"ok": false, "error": …}`, with a non-zero
  exit. Unreadable notes become bounded `warnings`.

**Human output** (default) is coloured and grouped per item `1/N`:

- each row shows the status glyph, the title with matched words bold, a band tag, and
  the locator;
- `--query` adds a score bar.

## Mac architecture

- **Lane.** `BobProcessClient.captureRelated(draft:)` runs
  `capture-related --format json -- <draft>` on lane `"related"`.
  - It cancels the previous request in the lane and has a 3 s advisory timeout.
  - It decodes off the main actor into `CaptureRelatedResponse`, which conforms to
    `SchemaVersioned` and uses `decodeIfPresent` everywhere. Unknown bands and actions
    decode to `.other(String)` and are ignored.
  - Exit 2 with `unrecognized subcommand 'capture-related'` maps to
    `.unsupportedCommand("capture-related")`. That is remembered per bob identity token
    (the `isUnsupportedCaptureTemplatesFailure` precedent), and the feature then hides
    silently until the identity changes.
- **Settle wiring** in `CapturePanelModel`:
  - Every `editorTextDidChange` cancels the settle task and any in-flight related
    request, bumps `relatedGeneration`, and starts a new 350 ms `Task.sleep` (the agenda
    dim-hold timer is the precedent).
  - When the timer fires, the model calls the lane only if all of these hold: the
    setting is on, the panel is visible, the draft is non-empty, bob is not known to be
    unsupported, and no picker, ID prompt, Pomodoro-name prompt, relation-label prompt,
    pull-in choice, stash picker, or template shelf owns the panel.
  - `prepareForPresentation` with a retained draft fires immediately, with no settle.
  - `prepareForDismissal` cancels everything.
  - Results whose generation is stale are discarded. Byte-identical stdout publishes
    nothing.
- **Band state reducer.** `SimilarTasksBandPolicy` lives in CaptureCore, is pure, and is
  Linux-testable. Inputs: the previous visible state, a new response, the latest parse
  items (index and mode), and the caps (single 2, bulk 1, draft 5). Output: per-item
  visible rows. The rules:
  - non-hold matches come in bob's order;
  - `hold` and `linked` entries appear only when their `key` is already visible for that
    item, and a linked entry renders with its link capsule;
  - "+N" counts the hidden non-hold matches;
  - ⌘ numbering runs down the stack;
  - the render gate covers mode equality and item-count changes;
  - focus freezes the state, and a pending result applies on unfocus.
- **Splices.** Apply bob's anchors through a native, undoable `NSTextView` bridge (the
  `applyNativeRewriteEdit` and `insertSavedTemplateInEditableTextView` precedent:
  `breakUndoCoalescing` plus `insertText(_:replacementRange:)`).
  - The app checks that the draft is unchanged since the response, and converts UTF-8
    offsets to UTF-16 with the existing helpers.
  - When the bridge cannot apply (marked text), fall back to `setPlainDraft` and
    `scheduleAnalysis`.
  - Anchors are only valid for the draft they came from. Focusing a held row after
    further typing triggers one immediate lane call, and the action applies against that
    fresh response (`mac_actions`).
- **Open** reuses `ObsidianOpenURL.taskGotoURL` and `taskGotoOpener`. On success it
  calls `panelDismisser()` **without** `discardDraft()`.
- **Settings.** `AppSettings.similarTasksEnabled` defaults to on (the `agendaEnabled`
  default-on pattern) and lives in a new "Similar Tasks" `SettingsView` section placed
  after Agenda. The toggle reads "Show similar tasks while capturing", with the caption
  "bob looks after you pause typing; nothing is linked unless you choose an action".
  AppDelegate pushes changes live into the model, and turning it off clears the bands
  and stops all related spawns.
- **Signposts** (metadata only, never draft text or titles):
  - `related.request` (an interval);
  - events `related.publish`, `related.empty`, and `related.failed`.

  List them in the Mac `diagnostics` doc page.

## Performance and reliability gates

Each gate below is enforced by a test or a measurement in its phase:

- **Keystrokes add no work.** A burst of edits inside the settle window spawns zero
  `capture-related` processes, and one after the pause (Mac model test with fake-bob
  argv recording).
- **The preview lane is untouched.** The same scripted edit sequence produces identical
  `capture --dry-run` and `capture-parse` invocations with the feature on and off.
- **Return never waits.** With fake-bob delaying `capture-related` by 5 s, submit
  completes as it does today.
- **No spawns while hidden or disabled.**
- **Stale results never paint.** A late result after an edit is discarded. A row whose
  task closed before the action fails safely in bob's preview, not on write.
- **Old bob degrades quietly.** The band stays absent, the preview and capture are
  unaffected, and the failing spawn happens once per bob identity.
- **bob speed.**
  - CI: a synthetic vault of about 1,500 tasks (about 40 notes plus `done/` archives)
    runs one draft call in-process under 250 ms in a debug build.
  - Measured: a release build on the Mac runs at p95 ≤ 50 ms (`calibrate`). Above that,
    file the in-app index escalation from the research (bob emits term vectors and the
    app ranks them) as a task bead. Do not block.
- **Main-thread publish is cheap.** Decoding happens off main, and only the reducer runs
  on main.

## Phases

### engine — Matching engine and the --query harness

All in bob-cli.

- Add `src/native/capture_related.rs` (the CLI: `build_cli`, `run`, request parsing,
  output) and a `src/native/capture_related/` module directory with:
  - `tokenize.rs`: Bob-aware terms with display spans;
  - `corpus.rs`: candidate discovery, open plus the closed window, with kind, key, and
    display title;
  - `score.rs`: per-call TF-IDF and cosine;
  - `bands.rs`: bars, short-query rules, hold, ordering, and limits;
  - tests.

  Follow the `capture_complete.rs` + `capture_complete/` layout and the
  `capture_pomodoros.rs` style: a `SCHEMA_VERSION` const, a serde success struct
  starting with `ok` and `schema_version`, `bounded_warning`, `print_error`, and a
  `Styler` human renderer.

- Register the command:
  - `src/native.rs`: the module, the `NativeCommand::CaptureRelated` variant, and the
    dispatch arm;
  - `src/runner.rs` `SUBCOMMANDS`: section CaptureProtocol, alphabetical. Update the
    `expected` array in `every_native_command_is_on_exactly_one_canonical_path`;
  - `src/native/completion/tree.rs`;
  - `src/native/completion/kinds.rs`: decisions for `days` and `query`;
  - the `justfile` `install-smoke` list.

  Ship only the `--query` mode here. Without `--query`, the command prints a usage
  error.

- **Discovery reuse.** Reuse `capture_targets` and `note_tasks::scan` directly. Do not
  call `capture_link_tasks::discover` if its day-file and Pomodoro work costs time; the
  related lane needs no Pomodoro data. Export the small helpers you need
  (`ref_tasks::close_date`, `collect_done::source_relative_path_from_archive`, a
  wikilink-alias reducer) as `pub(crate)` instead of copying them.
- **Tests.**
  - A tokenizer table: code spans, `<ctrl+shift+m>` chords, wikilink aliases, tags,
    inline fields, `^id`, URLs, emoji, camelCase and snake_case, stemming, stopwords.
  - Scorer pairs from the research:
    - "Add support for templated capture!" vs "Add support for bob capture templates!"
      must reach `possible_duplicate`;
    - "Make appt w/ Rahway Hospital for CT scan!" vs "Make appointment for CT scan!"
      must reach at least `hold`;
    - "Add `%d` short-hand for `%dispatch`!" vs "Add `!<cmd>` short-hand for
      `%proc("<cmd>")`!" must stay below `similar`.
  - Band edges, ordering, closed-window boundaries at 29, 30, and 31 days with `BOB_NOW`
    pinned, recurring and `#gtd` exclusions, and an in-process perf test on a synthetic
    1,500-task vault (< 250 ms debug).
- **Docs.** Clap help with examples, then `just docs-gen` (CLI reference page, index,
  and mkdocs nav). Update the root-help snapshots under `tests/fixtures/help/`.
- **Measure.** Time a release build's `--query` on the apollo vault (read-only; median
  and p95 of 30 runs) and report it in the phase's final notes.
- **Gate:** `just check`, `just docs-check`.

### contract — Draft mode and the JSON contract

All in bob-cli.

- Add `-- <DRAFT>`: parse, gate, discover once, score each qualifying item, apply bulk
  dedupe, and resolve the draft's own `&`, `*`, `~`, `@route:id`, and `@route+id`
  targets into `linked`.
- Emit the item anchors, `actions` (verified by simulated re-parse), `default_action`
  (per the duplicate_default decision), `replacements`, `matched_ranges`, `closed`,
  `kind`, `key`, `target`, and `line`, exactly as in the contract above.
- Return early with an empty result, before any vault I/O, for non-qualifying and
  template drafts.
- Add `capture-related` to `is_capture_text_command`
  (`src/native/completion/present.rs`) so the DRAFT argument gets live marker
  completion.
- **Goldens** in `tests/cli/capture/related.rs` with fixtures under
  `tests/fixtures/capture_related/`, following `pomodoros_agenda.rs` (`TempDir` vault,
  pinned `BOB_NOW`, normalized root, `BOB_UPDATE_GOLDENS`). Cover:
  - a short query (empty);
  - a recapture (`same_title`, with the `default_action` per the decision);
  - a cross-note `possible_duplicate`;
  - a code-span non-match;
  - a Blocked, ID-less row (`requires_block_id` and suggestions, no replacements);
  - a non-creating draft, which returns empty even with `--bob-dir` pointing at a
    missing directory (proof that no vault walk happens);
  - a template draft (empty);
  - a bulk draft (indexes, dedupe, `discard_range` for the first, middle, and last
    items);
  - the closed window and `closed_duplicate` suppression when an open duplicate exists;
  - `linked` reporting;
  - a project-note title;
  - a `pomodoro_task` item;
  - an anchor property test: inserting each offered marker at `marker_at` re-parses as
    expected.
- **Perf gate.** A CLI-level test on the synthetic vault, with the draft-mode call under
  250 ms (debug).
- **Docs.**
  - `docs/capture.md`: add a new `## bob capture-related` section placed with the other
    protocol commands. It is the behavior contract and covers scope, gate, bands,
    exclusions, anchors, actions and defaults, every JSON field, stability (bars and
    defaults may change without a schema bump; draft JSON never carries scores), and
    examples.
  - `docs/reference/json-outputs.md`: a new row in the capture/protocol table.
  - Run `just docs-gen` and `just docs-check`.
- Copy one representative golden into the phase notes for the Mac phases to mirror as a
  fake-bob fixture.
- **Gate:** `just check`, `just docs-check`.

### calibrate — Thresholds from Bryan's labels, timed on the Mac

- **Sampling script.** `scripts/dev/capture_related_calibrate`, in the style of
  `note_snapshot_corpus_gate`: read-only, with output outside the vault, and added to
  `just check-scripts`.
  - It runs leave-one-out over the live catalog: each open task's title goes through
    `bob capture-related --query … --format json`, and self-matches are dropped by note
    path and line.
  - It stratifies about 100 top pairs across score bins and bands, including
    `same_title` and closed rows, and reports firing rates per bar.
- **Labels.**

  > [!decision] calibration = labels Collect Bryan's labels through `/sase_questions`.
  > Read that skill first and batch pairs to fit its limits; if 100 pairs cannot fit,
  > write a worksheet artifact and ask one question that points to it. Labels are
  > **duplicate**, **useful heads-up**, or **noise**. Then choose bars that keep
  > `possible_duplicate` precision ≥ 80% and `similar` usefulness ≥ 60% while firing on
  > about 1 task capture in 8 to 10. Update the constants and goldens, and note any
  > tokenizer fixes the labels expose.

  > [!decision] calibration = starting_bars Keep the research's starting bars. Generate
  > the worksheet artifact anyway, and file one `/sase_new_task` task bead for Bryan to
  > label it and retune.

- **Mac timing.**
  1. rsync the bob-cli source to a scratch directory on `mac` (excluding `target`,
     `.git`, and `sase`) and run `cargo build --release` there.
  2. Time 30 runs each of a qualifying draft and a non-qualifying draft against `~/bob`,
     which is read-only for this purpose. Report the median and p95.
  3. Delete the scratch directory.

  Never install over or touch Bryan's `~/.cargo/bin/bob`. If the Mac is offline, say so,
  and use the apollo numbers plus the research's Mac/apollo ratio as a provisional
  estimate. If p95 is above 50 ms, file the index-escalation bead.

- **Report.** Add `202610/capture_similar_tasks_band/calibration.md` to the research
  sidecar and register it as an artifact. It records the method, the label counts,
  precision and firing rate per bar, the chosen bars, and the Mac and apollo timings.
- **Gate:** `just check`, `just check-scripts`, `just docs-check` (if `docs/capture.md`
  bar values changed).

### mac_lane — Lane, settle policy, and setting

Done in bob-mac-capture.

- **CaptureCore.**
  - `CaptureRelatedModels.swift`: response, item, match, band, action, closed, and
    linked types, with snake_case `CodingKeys` and `decodeIfPresent`.
  - The `SchemaVersioned` conformance.
  - `BobProcessClient.captureRelated` plus the unsupported-subcommand mapping and the
    identity cache.
  - `SimilarTasksBandPolicy.swift`: the pure reducer.
  - Tests in `CaptureCoreTests`:
    - decoding, including unknown bands and fields, and the contract golden copied from
      bob-cli;
    - the reducer's caps, hold, linked, numbering, "+N", the mode and count render gate,
      and the frozen/pending focus behaviour;
    - client lane cancellation and the unsupported mapping.
- **BobMacCapture.**
  - `CapturePanelModel`: the settle task, `relatedGeneration`, the trigger and
    suppression rules, the show/dismiss hooks, the published `similarTasks` state
    consumed by later phases, and the signposts.
  - `AppSettings`, `SettingsView`, and the AppDelegate wiring for the toggle.
- **fake-bob.** A `capture-related` branch with `FAKE_BOB_RELATED_FIXTURE`, a
  `FAKE_BOB_RELATED_UNSUPPORTED` switch (clap's `unrecognized subcommand` text, exit 2),
  and delay support.
- **Model tests** for every gate in "Performance and reliability gates".
- **Docs** (Mac `docs/`):
  - `protocol`: Required/Optional commands and a Compatibility row (feature, absent
    behaviour, reader action);
  - `settings`: the new section's table;
  - `diagnostics`: the signposts;
  - `develop/architecture`: the related lane.
- **Verify** per the Mac `AGENTS.md`:
  - `just portable-check` (on athena when local `swift` is missing);
  - a scratch-copy `Scripts/bundle.sh` plus `just typecheck-tests` on `mac`;
  - `just docs-check`.

  If the Mac is offline, state that the macOS targets are unverified.

### mac_band — Band view

Done in bob-mac-capture.

- **The band view.** Add `SimilarTasksBandView.swift`, built from the visual spec:
  header, row, capsules, kind glyph, closed styling, the bulk row variant with its `⧉`
  lead-in, and the folded line.
- **Placement.** Insert it at the foot of `standardPreviewItem` (after the
  `relativeTarget` line) and in the project-note card for its title. Insert it only when
  the reducer's render gate passes for that capture index.
- **Fold before scroll.**
  1. The controller already measures the auxiliary ideal height against the screen limit
     (`applyContentMetrics`).
  2. Compute the ideal height with every band at full height: the measured height, minus
     the current band heights, plus the full band heights. Band heights come from fixed
     row metrics, so this does not oscillate.
  3. Publish `similarTasksFolded` when that ideal height exceeds the available height,
     and fold every band.
- **Motion.** A 120 ms cross-fade keyed on the visible row keys, with no height
  animation, honouring Reduce Motion.
- **Accessibility.**
  - Each row is one element, labelled "<title>, <capsule>, <route>, <status or closed
    text>".
  - One `AccessibilityNotification.Announcement` per settled visible change that is
    non-empty and differs from the last. Never announce per keystroke, and never take
    focus.
- **Render fixtures** (`RenderFixtureWriter`, 760 and 620 pt, light and dark): no band,
  1 row, 2 rows plus "+N", a `same_title` row, a closed row, project and reading kinds,
  bulk, folded, and linked capsules.
- **Docs.** Add a "Similar tasks" section to the Mac `capture-previews` page, with
  curated images through `docs/media.yml` where that is the established pattern.
- **Verify** as in `mac_lane`.

### mac_actions — Focus, strip, and draft-edit actions

Done in bob-mac-capture.

- **Routing.**
  - Add `similarRowsVisible` and `similarRowFocused` to `CaptureKeyRoutingContext`, and
    new `CaptureKeyCommand` cases.
  - In `CaptureKeyCommandRouter`, match ⌘1–⌘5 by key code or by
    `charactersIgnoringModifiers` in the main-editor branch next to ⌘S. Modal branches
    keep precedence, so a picker or prompt naturally shadows these keys.
  - While a row is focused, route ← → ↑ ↓ ↩ esc, the sigils, and ⌫. Any other key
    unfocuses and passes through.
- **The strip view.** Build `SimilarTasksActionStrip` from the visual spec, wire its
  chips and click handling, and render the focused row state.
- **Fresh anchors.** Held rows can outlive the draft they were computed for.
  - When ⌘N or a click focuses a row whose response predates the current draft, focus it
    immediately and fire the related lane at once, skipping the settle. This is a user
    gesture, not a keystroke, so it does not break the zero-spawns-per-keystroke gate.
  - When the fresh response arrives, re-key the focus by `key`. If the task is gone,
    unfocus and announce "Similar tasks changed".
  - An action chosen before the fresh response arrives waits for it, within the lane
    timeout, and then applies against the fresh anchors.
- **Footer.** Swap the footer to key hints while a row is focused (the
  `CapturePickerKeyHints` precedent). The ↩ hint names the default action.
- **Actions for rows with a block ID.**
  - `relate`, `note`, `needs`, and `blocks` go through the undoable bridge using bob's
    anchors and replacements, then run `scheduleAnalysis(trigger: .edit)`.
  - `discard` is undoable the same way.
  - `open` keeps the draft.
  - Every splice re-checks that the draft still equals the response's draft. On a
    mismatch, which should be impossible after the fresh-anchor step, announce "Similar
    tasks changed" and edit nothing.
  - After an action, unfocus. Set the status text and post the announcement.
- **Accessibility custom actions** on each row, one per available action.
- **Tests.**
  - Router tables for every key and context.
  - Model tests: each action's exact draft result on fake-bob fixtures (single item,
    bulk middle item, a `route_range` replacement, an append), undo restoring the draft,
    focusing a held row after more typing (one immediate lane call, then the action
    applies with fresh anchors), the stale guard, Open retaining the draft, and Return
    unchanged when nothing is focused.
  - A render fixture of the focused strip.
- **Docs.**
  - The Mac `keyboard` page: band keys in the editor table plus a "Similar tasks keys"
    table.
  - `capture-previews`: the actions and their defaults.
- **Verify** as in `mac_lane`.

### mac_idless — ID-less rows and the +N reveal

Done in bob-mac-capture.

- **ID-less rows.** For a row with `requires_block_id` and a link action, present the
  existing Add block ID prompt with a new `CaptureTaskIDPromptPurpose` case that carries
  the action, item index, anchors, and draft snapshot. The prompt is prefilled from
  `block_id_suggestions`.
  - On submit, call `assignCaptureTaskID(route:…)` and splice the matching
    `CaptureTaskIDSuccess` replacement with the same bridge.
  - Esc cancels with no write.
  - The prompt's copy says the ID is written now, before Return, as the pickers already
    do.
- **+N reveal.** ↓ from the last visible row of an item reveals that item's next hidden
  non-hold match, up to bob's limit. Unfocusing collapses the reveal.
- **Finish.**
  - Complete the `capture-previews` and `keyboard` docs, and add render fixtures for an
    ID-less focused row and a revealed row.
  - Run an end-to-end check on the Mac scratch build against a scratch vault built from
    bob-cli's fixture, never Bryan's vault, if a real `bob` with `capture-related` can
    be pointed at it. Otherwise state what remains unverified.
- **Verify** as in `mac_lane`.

### record — Decisions record

> [!decision] decision_record Use `/sase_memory_write`.
>
> 1. Add `sase/memory/decisions/similar-tasks-are-a-settled-heads-up.md`, titled
>    "Capture Shows Similar Tasks As A Settled Heads-Up", in the house format: keyword,
>    aliases, summary, `metadata.status: accepted`, `decided`, and **Applies to / Claim
>    / Why / Cost / Reopens when** sections.
> 2. Run `sase memory init`.
>
> The claim covers: bob owns eligibility, tokens, weights, bands, hysteresis, actions,
> defaults, and anchors; the app owns the 350 ms settle and the pixels; transport is one
> call per pause, with no in-app index; zero pixels unless a match clears the bar;
> actions are draft edits; Return is never gated; nothing is auto-linked; and the bars
> come from Bryan's labels.
>
> - **Rejected alternatives:** a live per-word list, an in-app index, pull-only,
>   triage-time-only detection, a "related" label, and percentages.
> - **Reopens when:** the Mac p95 exceeds 50 ms; Bryan wants rows without a pause; a
>   second client wants the ranking; or labelled precision falls.
> - **Evidence:** the research ref, this plan, the phase commits, the `calibrate` report
>   with its Mac timings, and the `[[mac-capture-is-a-thin-client]]` and
>   `[[idle-capture-shows-ledger-agenda]]` links.

## Out of scope for v1

These stay out of v1; file follow-ups only if usage asks for them:

- `@` "do it now" and `!` "mark it done" actions;
- seeding the `~`, `&`, `*`, and `+` pickers by similarity;
- a NEW-tier "Possible duplicate" chip in the `]s` walk;
- route hints;
- the in-app index;
- `@@template:` drafts and project sub-bullet tasks;
- Obsidian plugin surfaces;
- stamping `[fresh:: today]` when a recapture is added as a note (sub-bullet captures do
  not stamp today, and changing that is a separate freshness decision).

## Risks

- **Lexical blind spots.** "Enable backups" vs "Disable backups", or "… for Alice" vs "…
  for Bob", look alike to any lexical scorer. The strong label is therefore "Possible
  duplicate", and "Same title" is reserved for exact equality.
- **Large Swift files.** `CapturePanelModel` and `CapturePanelView` are very large. Keep
  new logic in new files (the reducer, the band view, the action helpers) and add only
  thin hooks.
- **Mac availability.** The Mac is best-effort. Every Mac phase must report its
  verification state honestly, and CI on `macos-26` is the test authority for
  `BobMacCaptureTests`.
- **Installation order.** bob and the app ship independently. The app must hide the band
  silently on an older bob, and the contract may only grow additively after
  `schema_version` 1.
