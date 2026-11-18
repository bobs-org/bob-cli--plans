---
tier: epic
title: Walk attention marks in Bob Mac Capture's idle agenda
goal: 'Whenever the Obsidian walker selector shows attention for a family (an off-family
  dot or selected-family ink) — `s` review, `w` lanes, `b` bank — Bob Mac Capture''s
  idle agenda shows that family''s mark (its letter with a hue dot) at the title row''s
  trailing edge, computed by bob from the same contract, and shows nothing otherwise.

  '
decisions:
  mark_frame:
    ask: How should the walk marks be framed in the agenda title row?
    choices:
      bare: Letters with hue dots only; most concise, reads as status, never looks
        like link syntax
      brackets: Faint [[ … ]] around the marks; strongest echo of the Obsidian selector,
        ~4 glyphs wider
    default: bare
    why: The letter plus dot is already the selector's own glyph; brackets add width,
      not meaning
    answer: bare
  mark_counts:
    ask: Show a small count beside the s and b marks (due commitments, waiting rows)?
    default: false
    why: Obsidian's selector shows dots only; counts stay in the tooltip and VoiceOver
      label
    answer: true
  memory_walk_marks_record:
    ask: Record this design as a decision that partly supersedes the idle agenda's
      only-fact-source claim?
    memory:
    - decisions
    - decisions:idle-capture-shows-ledger-agenda
    default: true
    corrects: (`bob capture-pomodoros --tasks` as the agenda's only fact source)
    answer: true
phases:
- id: scan-once
  title: Freshness scan parses the vault once
  depends_on: []
  size: small
  description: 'scan-once: share one Tasks settings read and one TaskIndex across
    the freshness scan''s READY/PENDING/NEXT/OPEN queries and its all-task pass, with
    byte-identical outputs and recorded before/after timings.

    '
- id: bob-walks
  title: bob walks attention report and contract
  depends_on: []
  size: medium
  description: 'bob-walks: add the read-only `bob walks` command (human + JSON schema
    1) that reports s/w/b attention with per-family failure isolation, plus the docs/walks.md
    contract, conformance vectors, goldens, and doc updates.

    '
- id: plugin-parity
  title: Pin the walker footer to the walks conformance vectors
  depends_on:
  - bob-walks
  size: small
  description: 'plugin-parity: run the docs/walks.md vectors against bob-ledger-tools''
    real attention pushes in a JS test, fix any drift in favor of the contract, link
    the contract from the plugin docs, and sync.

    '
- id: mac-walks-core
  title: CaptureCore walks models, client, tones, and presentation
  depends_on:
  - bob-walks
  size: medium
  description: 'mac-walks-core: decode `bob walks --format json` on its own process
    lane, add the tier hue table, title-row walk marks in the agenda presentation,
    the pure refresh policy, fake-bob support, protocol docs, and green macOS CI.

    '
- id: mac-walk-marks
  title: Walk marks store, view, docs, and decision record
  depends_on:
  - mac-walks-core
  size: medium
  description: 'mac-walk-marks: wire the walks store and refresh triggers, paint the
    marks at the title row''s trailing edge with motion, contrast, and no-wrap guarantees,
    update docs and renders, watch CI green, and write the decision record.'
proposed_by: bbugyi200.apollo.es
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-17 17:59:50
status: done
bead_id: bob-cli-mx
---

- **PROMPT:** [prompts/202611/mac_walk_attention_marks.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/mac_walk_attention_marks.md)
- **BEAD:** [bob-cli-mx](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-mx/README.md)

# Plan: Walk attention marks in Bob Mac Capture's idle agenda

## Background (verified while planning)

- The Obsidian `[[ s w n b ]]` selector lives in bob-navigation-hotkeys
  (`src/547-bracket-mode-indicator.js`, `src/547a-bracket-mode-view.js`, `styles.css`
  around the `.bob-bracket-mode*` rules). bob-ledger-tools pushes per-family attention
  through nav `api.walkMode.setAttention(type, {tier, label})`:
  - `s` — `src/122-walker-footer-host.js` `walkerFooterPushReviewAttention`: on when the
    review queue's **commitments** (sum of tiers PRE, NEW, PROJECTS, PENDING, NEXT,
    RECURRING, TICKLER; ROTTEN and POST are upkeep and excluded) are > 0. `tier` is the
    first non-zero commitment tier in that order. Label:
    `Review: {walk} due · {commitments} commitments — {TIER} first` where `walk` is the
    whole queue (all nine tiers) and TIER uses the footer labels PRE, NEW, PROJECTS, WIP
    (pending), NEXT, RECUR (recurring), TICKS (tickler).
  - `w` — `src/123-lane-walker-footer.js` `walkerFooterPushLaneAttention`: on when any
    lane is strictly over its cap (`workBadgeTone(count, cap) === "red"`, i.e.
    `count > cap`; exactly at cap is magenta, not attention). Lanes are the `]w` lane
    walk's off-Today counts (`src/071-lane-walk-model.js` `laneWalkQueueFromTasks`):
    PENDING `[/]`, NEXT `[*]`, READY (TODO type, minus NEW/ROTTEN review buckets), each
    excluding Today-linked tasks, `dash.md`, `_templates`/`_conflicts`, `#hide`,
    future-scheduled, done, and dependency-blocked rows. Caps: `plan.max_pending` (10),
    `plan.max_next` (15), `plan.max_ready` (100). Label:
    `Lanes over limit: WIP 12/10 · NEXT 16/15`; tier `over`.
  - `b` — `src/125-bank-walker-footer.js` `walkerFooterPushBankAttention`: on when any
    bank row waits (unacknowledged) across every `bank/YYYY-MM.md` month note. Label:
    `Bank: 3 waiting · oldest Mon Nov 3`; tier `waiting`.
  - `n` never carries attention.
  - Off-family attention paints a 5 px dot; the selected family paints the same hue as
    letter ink. Same fact, two treatments. Hues (nav CSS, `data-attention-tier`): pre
    cyan, new red, projects purple, pending yellow, next green, recurring pink, tickler
    blue, over red (the work badge red), waiting `--text-warning` (amber/orange).
- All three are computed in plugin JavaScript over Obsidian's in-memory Tasks cache and
  bank index. Bob Mac Capture never reads the vault
  ([[decisions:mac-capture-is-a-thin-client]]), so bob must own these facts.
- bob-cli already has the pieces: `freshness::scan::scan` (the native freshness engine
  that `bob freshness list -f json` uses for `counts.by_tier`, kept in parity with the
  plugin by the conformance vectors in `docs/freshness.md`); its `Snapshot` carries the
  READY/PENDING/NEXT lane rows with `is_today`, and `row_bucket` gives NEW/ROTTEN;
  `config::plan` has the caps; `bank::scan::scan_bank` + `bank::list::build` give
  `counts.waiting` across every month. No command combines them, and bob has no native
  off-Today lane counts.
- `freshness::scan::scan` rebuilds the Tasks settings and the whole-vault `TaskIndex`
  five times (four `query_rich_tasks` calls plus `scan_all_rich_tasks`);
  `bob freshness list -f json` took 2.3–2.9 s on apollo's vault.
- The Mac idle agenda's title row (`Sources/BobMacCapture/CaptureAgendaView.swift`
  `titleView` / `titleAccessoryView`) is a single `HStack`: leading
  `Today · Tue 17 Nov[ · Nothing running]`, a spacer, then the completed summary, the
  dashed off-clock chip, and the "Couldn't refresh" marker. Header rows never fold
  (`CaptureCore/CaptureAgendaFitPlanner.swift`), so a same-row accessory costs the fold
  ladder nothing as long as the row stays one line.

## Design

### What Bryan sees

```
Today · Tue 17 Nov                          5 done · 2h 40m    s• w• b•
TODAY 4/10
NOW  FIX · 18m left …
```

At the title row's trailing edge — the top-right corner of the agenda, a fixed place the
eye learns — one small monospaced lowercase letter per family that needs attention, each
with a tiny hue dot at its top-trailing corner, in the selector's canonical order `s`,
`w`, `b`. They are the same glyphs as the Obsidian selector's off-family letters, so
they read as "the walker dots" without any learning. Calm families render nothing; when
all are calm the corner is empty, with no placeholder and no "all clear" text.

### The rule (the "iff")

A family's mark is visible exactly when both hold:

1. The capture panel shows its default view: the idle agenda pane is mounted (empty
   draft, agenda setting on, no picker, card, or preview owning the region), in any of
   its states (agenda, no open Pomodoros, no daily note, "Loading today…").
2. bob's `walks` report for **today** says that family is `available` and has
   `attention`. By contract (`docs/walks.md`, next section) that is the same predicate
   the walker footer uses to push its dot or ink.

The Mac does not mirror Obsidian's _selected_ family. It lives in Obsidian's local
storage, only governs `]]` repeats, and changes no fact: the dot and the ink announce
the same attention. So the panel uses one treatment, the off-family dot. `n` never
appears.

### Why bob computes it

- **bob owns facts; the app owns pixels.** This follows
  [[decisions:mac-capture-is-a-thin-client]]. A Swift computation would be a second
  engine.
- **Rejected: the plugin publishes its attention to a file.** That would give parity by
  construction. But it works only while Obsidian runs, goes stale when Obsidian is
  closed or another host rolls the day, and writes on every footer repaint.
- **Rejected: an optional field on `capture-pomodoros --tasks`.** That would put a
  whole-vault scan into every agenda refresh. The agenda's revalidation would then wait
  on the slowest fact.
- **Parity is held by contract.** `docs/walks.md` states the three predicates and their
  conformance vectors. bob-cli's Rust tests and bob-ledger-tools' JS tests both run them
  verbatim, which is how `docs/freshness.md` already keeps the two freshness engines
  aligned.

### Data flow

```
bob walks --format json ──► WalkAttentionStore (own lane, own cadence)
                              │  last good snapshot in memory, date-guarded
                              ▼
CaptureAgendaPresentation title row `walkMarks` ──► CaptureAgendaView marks
```

## Phase scan-once: Freshness scan parses the vault once

bob-cli only.

- In `src/native/dataview/tasks/mod.rs`, add a shared-index reader, for example
  `RichTaskIndex::read(vault, now)`:
  - It reads `TasksSettings` and `TaskIndex` once.
  - It offers `query(&self, query: &str) -> Result<Vec<RichTask>, DataviewError>` and
    `all(&self) -> Vec<RichTask>`.
  - Keep the per-query `maybe_sandbox` decision, so a query without `by function` still
    never builds the JS sandbox.
  - `query_rich_tasks` and `scan_all_rich_tasks` become thin wrappers with unchanged
    signatures and behavior.
- Switch `src/native/freshness/scan.rs` `scan` to one `RichTaskIndex` for READY,
  PENDING, NEXT, OPEN, and all. Use the same `now` everywhere, as today.
- Outputs must be byte-identical:
  - Every existing freshness, ready (`note_ready`), period-brief, and plan test and
    golden passes unchanged.
  - Add a unit test proving the shared-index results equal the per-call results on a
    fixture vault.
- Measure `bob freshness list -f json` before and after, read-only, against the local
  vault (`BOB_DIR` or `~/bob`), with the release build. Put both timings in the commit
  body.
- Out of scope: `plan_budget::count_lanes` (`query_matching_descriptions`). If sharing
  pays off there too, file a task bead through `/sase_new_task`.
- Gate: `just` test/lint targets per `AGENTS.md`. Commit as `perf(freshness): …`.

## Phase bob-walks: bob walks attention report and contract

bob-cli only. Read `sase/memory/cli_rules.md` and `docs.md` with `/sase_memory_read`
first.

### Command

- Usage: `bob walks [-b|--bob-dir DIR] [-f|--format human|json] [-h|--help]`.
  - A read-only leaf command with no subcommands.
  - Reuse the exact `bob_dir_arg`/`format_arg` shapes from
    `src/native/freshness/cli.rs`.
- Code lives in a new module `src/native/walks/` (`mod.rs` for the computation, `cli.rs`
  for clap, human, and JSON output).
- Register it:
  - in `src/runner.rs` `SUBCOMMANDS`, in the Daily workflow section, alphabetically;
  - as `NativeCommand` in `src/native.rs`;
  - in `src/native/completion/tree.rs`.
- About line: `Show which bracket walks need attention: s review, w lanes, b bank`.
- Long help names the three families, says the report is read-only and never stamps or
  walks, and points to `docs/walks.md`.

### Computation

Each family is computed independently. A failure in one makes only that family
`available: false` with a `reason`; the others still report.

- **s**
  - One `freshness::scan::scan(bob_dir)`.
  - Counts must come from the **same counting code** `bob freshness list` uses. Factor a
    shared helper out of `freshness/cli.rs` `collect_list` if needed, so `walks` and
    `freshness list` can never disagree. A test asserts `walk` and `by_tier` equality on
    a fixture vault.
  - `commitments` = pre + new + projects + pending + next + recurring + tickler.
  - `attention` = `commitments > 0`.
  - `tier` = the first non-zero commitment tier in that order.
- **w**
  - Uses the same snapshot. Lane counts:
    - **pending**: `snapshot.pending` rows;
    - **next**: `snapshot.next` rows;
    - **ready**: `snapshot.ready` rows whose `row_bucket` is neither `new` nor `rotten`.
  - All three exclude Today-linked rows (`is_today`) and the vault-root `dash.md`
    (case-insensitive), matching the dashboard sections and the `]w` walk.
  - Caps come from `config::plan` (`max_pending`, `max_next`, `max_ready`).
  - `over` = `count > cap`, and `attention` = any lane over.
  - An invalid plan config makes `w` unavailable, with the loader's message as the
    reason.
- **b**
  - `bank::scan::scan_bank` plus `bank::list::build` with the waiting filter, giving
    `counts.waiting` across every month and the oldest waiting row's date.
  - A teaching-empty bank is available with waiting 0.
  - An invalid bank config makes `b` unavailable.
  - Confirm bob's acknowledged predicate against the plugin's (`x`/`X` acknowledged,
    anything else waiting; bob-ledger-tools `src/241i-plugin-bank-source.js`). The
    contract states the rule `docs/bank.md` already defines. If the two differ, the
    plugin-parity phase aligns the plugin.
- **Labels.** bob owns them, byte-for-byte equal to the walker footer's push labels
  minus the plugin-only ` — click to show` / `; click to show` hints:
  - `s`: `Review: 14 due · 4 commitments — NEXT first`
  - `w`: `Lanes over limit: WIP 12/10 · NEXT 16/15` (over lanes only, in lane order)
  - `b`: `Bank: 3 waiting · oldest Mon Nov 3`. Use the bank walk's long day label
    format, and drop ` · oldest …` when no date is known.
  - Verify each wording against the plugin sources listed in Background. Open
    bob-plugins read-only with `/sase_repo`.
- **Exit codes.**
  - 0 whenever the report prints, including with unavailable families.
  - Nonzero only for clap usage errors or an unusable vault root, with the existing
    unversioned `{"ok": false, "error": …}` JSON envelope.

### JSON (schema_version 1)

New optional fields keep version 1; only a breaking change bumps it.

```json
{
  "ok": true,
  "schema_version": 1,
  "date": "2026-11-17",
  "walks": [
    {
      "family": "s",
      "name": "review",
      "available": true,
      "attention": true,
      "tier": "next",
      "label": "Review: 14 due · 4 commitments — NEXT first",
      "counts": { "walk": 14, "commitments": 4 }
    },
    {
      "family": "w",
      "name": "lanes",
      "available": true,
      "attention": true,
      "tier": "over",
      "label": "Lanes over limit: WIP 12/10",
      "lanes": [
        { "lane": "pending", "label": "WIP", "count": 12, "cap": 10, "over": true },
        { "lane": "next", "label": "NEXT", "count": 9, "cap": 15, "over": false },
        { "lane": "ready", "label": "READY", "count": 61, "cap": 100, "over": false }
      ]
    },
    {
      "family": "b",
      "name": "bank",
      "available": false,
      "attention": false,
      "tier": null,
      "label": null,
      "reason": "invalid bank config: …"
    }
  ]
}
```

- `walks` is always `s`, `w`, `b`, in that order.
- `tier` and `label` are non-null exactly when `attention` is true.
- An available `b` carries `"counts": {"waiting": N}` and
  `"oldest_waiting": "YYYY-MM-DD" | null`.
- `date` is the local today that the evaluation used.

### Human output

```
s ● Review: 14 due · 4 commitments — NEXT first
w ● Lanes over limit: WIP 12/10
b · Bank clear
```

- The `●` takes the family's tier hue through the existing styler, honoring `NO_COLOR`
  and non-TTY output.
- Calm lines are dim: `Review clear`, `Lanes within limits`, `Bank clear`.
- An unavailable family prints `? unavailable — <reason>`, dim.

### Contract, vectors, docs

- **New root contract page `docs/walks.md`.** It holds:
  - the purpose: one attention fact per walk family, shared by the walker footer and Bob
    Mac Capture;
  - the three predicates, the label formats, the JSON schema, and failure isolation;
  - the conformance vector table below, with stable ids.

  It says that bob-ledger-tools' selector pushes and Bob Mac Capture's idle-agenda marks
  both follow it. Add it to `mkdocs.yml` nav beside the freshness and plan pages.

- **Vectors.** These are encoded verbatim in Rust tests here, and later in JS:

  | Id  | Setup                                                               | Expected                                    |
  | --- | ------------------------------------------------------------------- | ------------------------------------------- |
  | S1  | only ROTTEN due                                                     | `s` off                                     |
  | S2  | only POST due                                                       | `s` off                                     |
  | S3  | NEW 1 and ROTTEN 1                                                  | on, tier `new`, walk 2, commitments 1       |
  | S4  | PRE 1 and NEXT 2                                                    | tier `pre`                                  |
  | S5  | TICKLER only                                                        | tier `tickler`, label ends `TICKS first`    |
  | W1  | PENDING 11, cap 10                                                  | on, `WIP 11/10`                             |
  | W2  | PENDING 10, cap 10                                                  | off                                         |
  | W3  | a Today-linked `[/]` task                                           | not counted                                 |
  | W4  | an unstamped (NEW) Ready task                                       | not in READY                                |
  | W5  | a ROTTEN Ready task                                                 | not in READY                                |
  | W6  | rows in `dash.md`                                                   | not counted                                 |
  | W7  | future-scheduled, `#hide`, dependency-blocked, or `_templates` rows | not counted                                 |
  | W8  | READY 101, cap 100                                                  | on, `READY 101/100`                         |
  | B1  | one `[ ]` row under a day heading                                   | waiting 1                                   |
  | B2  | `[x]` and `[X]` rows                                                | waiting 0                                   |
  | B3  | waiting rows in two months                                          | oldest comes from the earlier month         |
  | B4  | invalid bank config                                                 | `b` unavailable; `s` and `w` still reported |

- **Tests.** `tests/cli/walks.rs` uses fixture vault(s), with goldens in
  `tests/fixtures/walks/` regenerated through the repo's `BOB_UPDATE_GOLDENS=1` pattern.
  At minimum: `walks-clear.json`, `walks-all.json` (s next, w over, b waiting), and
  `walks-bank-unavailable.json`. These goldens become the Mac fixtures.
- **Docs.**
  - Add a `docs/reference/json-outputs.md` row. Stay inside that page's word budget; it
    was just trimmed.
  - In `docs/plan.md`'s READY section, replace "this feature adds no native READY count"
    with: `bob walks` reports the READY lane count for attention, but there is still no
    `bob plan` lint, capture enforcement, or tmux meter.
  - Add one linking line in each of `docs/freshness.md` (the `s` rule) and
    `docs/bank.md` (the `b` rule).
  - Run `just docs-gen` (CLI reference and catalog) and `just docs-check`.
- Commit as `feat(walks): …`. The change is additive, not a contract break.

## Phase plugin-parity: Pin the walker footer to the walks conformance vectors

bob-plugins only. Open it with `/sase_repo` and read its `AGENTS.md`.

- Add `scripts/test-ledger-tools-walk-attention-vectors.cjs` and wire it into
  `npm test`. It runs the `docs/walks.md` S/W/B vectors verbatim through the real code
  paths, using the existing ledger-tools harness:
  - `walkerFooterPushReviewAttention`;
  - `laneWalkQueueFromTasks` with `walkerFooterPushLaneAttention`;
  - the bank snapshot with `walkerFooterPushBankAttention`.

  Each runs against a stub `walkMode` (version 6) that records `setAttention` calls.
  Assert presence, tier, and the label with the click hint stripped.

- If a vector fails, the contract wins. Fix the plugin's `src/` fragment, run
  `npm run build`, and keep hand-edited fragments ≤ 1000 lines. If the contract itself
  looks wrong, do not edit bob-cli. Record a `PROPOSED FOLLOW-UP:` on this phase's bead.
- Docs:
  - `docs/bob-ledger-tools.md`'s attention paragraph links bob's `docs/walks.md`
    contract and notes that Bob Mac Capture mirrors `s`/`w`/`b` attention.
  - Fix the stale `[[ s w n ]]` mention near the top of
    `docs/bob-navigation-hotkeys.md`.
- Gate: `npm test`, `npm run build:check`, `npm run docs:check`. Then run
  `bob plugins sync`, following `AGENTS.md`'s refusal rules (never `-F` on your own).

## Phase mac-walks-core: CaptureCore walks models, client, tones, and presentation

bob-mac-capture, `CaptureCore` target and fixtures. Open the repo with `/sase_repo` and
read its `AGENTS.md`. This code must compile on Linux via `just portable-check`.

### Models

`Sources/CaptureCore/WalkAttentionModels.swift`:

- `WalkAttentionSnapshot { schemaVersion, date, walks: [WalkAttention] }`.
- `WalkAttention { family, name, available, attention, tier?, label?, reason? }`.
- Every field after `family` decodes with `decodeIfPresent` and a default.
- Unknown families decode and are then ignored, never thrown.

### Client

`BobProcessClient.walkAttention(previous:)` mirrors `captureAgenda(previous:)`:

- It runs `["walks", "--format", "json"]` on its **own lane** (`"walks"`), so it never
  cancels or delays the agenda lane.
- Byte-identical stdout returns `.unchanged`.
- clap's unrecognized-subcommand error maps to `.unsupported`.
- `schema_version != 1` maps to `schemaMismatch`; anything else that fails to parse maps
  to `malformedJSON`.

### Tones

`Sources/CaptureCore/WalkAttentionTone.swift` maps a tier to light and dark hex. It
mirrors nav's `data-attention-tier` CSS on Obsidian's default palette:

| Tier        | Hue            | Light               | Dark                |
| ----------- | -------------- | ------------------- | ------------------- |
| `pre`       | cyan           | `#00bfbc`           | `#53dfdd`           |
| `new`       | red            | `#e93147`           | `#fb464c`           |
| `projects`  | purple         | `#7852ee`           | `#a882ff`           |
| `pending`   | yellow         | `#e0ac00`           | `#e0de71`           |
| `next`      | green          | `#08b94e`           | `#44cf6e`           |
| `recurring` | pink           | `#d53984`           | `#fa99cd`           |
| `tickler`   | blue           | `#086ddd`           | `#027aff`           |
| `over`      | work badge red | `WorkBadgeTone.red` | `WorkBadgeTone.red` |
| `waiting`   | amber          | `#ec7500`           | `#e9973f`           |

- Unknown tiers return nil, and the view falls back to the accent color.
- Reuse the `WorkBadgeTone` constants wherever the hexes coincide, and test every entry.
- Vault CSS-snippet overrides are not mirrored. Say so in a comment.

### Presentation

- Add `CaptureAgendaWalkMark { family, tier, label }` and a title-row field
  `walkMarks: [CaptureAgendaWalkMark]`.
- `CaptureAgendaPresentation` takes an optional `WalkAttentionSnapshot` beside the
  agenda snapshot. Marks contain only entries that are `available && attention`, in `s`,
  `w`, `b` order.
- Marks are empty when the walks snapshot is nil, from another day, or unsupported.
- Marks appear in every idle-agenda state that has a title row, including loading.
- The title row's accessibility label appends
  `Walks needing attention: <labels joined by "; ">`.
- Add `walkMarks` to `CaptureAgendaRowKey` and `hasAccessory`, so the measured height
  cache invalidates when marks change.
- Forward the field at every `CaptureAgendaRow` copy site in the fit planner.

### Refresh policy

A pure `WalkAttentionRefreshPolicy` in CaptureCore, with named constants:

- **Panel show:** refresh when dirty, never refreshed, or the last success is older than
  60 s. Otherwise paint the cached marks and spawn nothing.
- **Filtered vault event:** mark dirty and schedule a trailing debounce: 2 s while the
  panel is visible, 10 s while hidden. Use the same filter as the agenda,
  `CaptureAgendaRefreshFilter`.
- **Submit, wake, unlock, day change, clock change:** refresh now.
- **Recheck Bob or executable change:** reset, then refresh.
- **Concurrency:** one refresh in flight with at most one follow-up. Reuse
  `CaptureAgendaRefreshState` if it is generic enough; otherwise add a sibling.

Rationale: the walks scan is whole-vault. Debouncing keeps active Obsidian typing from
spawning back-to-back scans, while show-time revalidation and the 60 s age bound catch
config edits outside the vault.

### Fixtures and tests

- **Fixtures.**
  - Copy the bob-walks goldens to `Tests/Fixtures/walks-*.json`.
  - Teach `Tests/Fixtures/fake-bob` the `walks` subcommand, with
    `FAKE_BOB_WALKS_FIXTURE`, `FAKE_BOB_WALKS_UNSUPPORTED`, and `FAKE_BOB_WALKS_FAIL`.
- **CaptureCoreTests** cover:
  - decoding: fixtures, missing optional fields, unknown family and tier;
  - the tone table;
  - presentation: order, filtering, the date guard, all states, the accessibility label,
    and the row key change;
  - policy decisions under a fake clock.

### Docs and gate

- `docs/protocol.md`:
  - list `bob walks --format json`;
  - add the optional fields;
  - add a compatibility row: `bob walks` | s/w/b marks on the idle agenda title row |
    older Bob: no marks, nothing else changes | upgrade `bob`.
- Run `just docs-check` with `BOB_CLI_DIR`.
- Gate:
  1. `just portable-check`.
  2. `Scripts/macos-ci-check`.
  3. Push, then watch the exact-SHA macOS CI run to green with `/sase_monitor`.

## Phase mac-walk-marks: Walk marks store, view, docs, and decision record

bob-mac-capture app target. The phase also edits bob-cli memory.

### Store and wiring

- `Sources/BobMacCapture/WalkAttentionStore.swift` is `@MainActor` and modeled on
  `CaptureAgendaStore`:
  - it keeps the last good snapshot in memory;
  - statuses are idle, loading, ready, stale, failed, and unsupported;
  - a snapshot from another day is never published;
  - a failed refresh keeps the last good same-day snapshot;
  - observers handle wake, unlock, day change, and clock change;
  - every decision is delegated to `WalkAttentionRefreshPolicy`.
- Wiring:
  - **`AppDelegate`:**
    - prefetch at launch, after the agenda prefetch, so the agenda's first refresh is
      never delayed;
    - send the same filtered vault events;
    - refresh after submit;
    - reset on Recheck Bob or a change of `bob` executable or directory.
  - **`CapturePanelController`:** report show/hide, which drives the policy's visible
    flag.
  - **`CapturePanelModel`:** subscribe to the store, pass the snapshot into the
    presentation build, and re-plan when marks change.

### View

`CaptureAgendaView` `titleView` gets a trailing-most `WalkMarksView`, after the summary,
off-clock chip, and stale marker. It is non-interactive: no click and no focus.

- **Mark:**
  - the letter is `Text("s")` in `.system(.caption, design: .monospaced)`,
    `.weight(.medium)`, `.secondary`, inside a fixed-width cell (about 1.3 em, like
    nav's segment);
  - a 5 pt `Circle` in the tier hue sits at the cell's top-trailing corner, touching but
    never covering the glyph;
  - marks are 4 pt apart.
- **Contrast:** under Increase Contrast, the dot is 6 pt with a 0.5 pt
  `.primary.opacity(0.35)` stroke, and the letter is `.primary`. The letter already
  carries the meaning without color.
- **Motion:**
  - A mark that appears or clears **while the panel is visible** fades, with the dot
    scaling from 0.6 to 1, over 0.16 s ease-out.
  - The first paint on show is never animated, so cached marks are simply there.
  - Reduce Motion turns all of this off.
- **Help:** `.help(labels joined by "\n")`. VoiceOver reads the title row's combined
  label, so the marks themselves are `accessibilityHidden`.
- **No wrap, ever:** the leading title gets `.lineLimit(1)` and `.truncationMode(.tail)`
  at lower layout priority; the marks are `.fixedSize()` with `.layoutPriority(1)`. The
  marks never truncate, and the title row stays one line.

> [!decision] mark_frame = bare The marks render as listed above, with no frame.

> [!decision] mark_frame = brackets Wrap the marks in faint `[[` and `]]` (monospaced
> caption, `.quaternary`, letter spacing −0.08 em like nav's keys). With one mark it
> still reads `[[ b• ]]`. Include the brackets in the height-consistency width case.

> [!decision] mark_counts Show a `.caption2.monospacedDigit()` `.tertiary` count right
> after the `s` mark (commitments) and the `b` mark (waiting). `w` stays a bare mark,
> because its label already lists the over lanes. Counts come from the JSON `counts`,
> never from the app.

### Diagnostics, tests, docs, gate

- **Diagnostics.** Settings' agenda diagnostics area gains one walks line: status, last
  refresh time, and the error or "Upgrade bob for walk marks".
- **Tests.**
  - `CaptureAgendaHeightConsistencyTests` at widths 584 and 724, with the busiest title
    row (summary + off-clock chip + stale marker + all three marks): one line, within 2
    pt of the plan.
  - `CaptureAgendaDesignTests` renders light and dark with marks.
  - Store tests through fake-bob cover unchanged output, unsupported, failure keeping
    the last good same-day snapshot, the date guard, debounce, and the show-refresh
    rule. Use timed waits only, never untimed continuations.
- **Docs.**
  - `docs/idle-agenda.md` gains a "Walk attention" paragraph: the rule, placement, hues,
    cadence, and older-bob behavior.
  - Update `docs/settings.md` if it describes diagnostics.
  - Regenerate `docs/images/capture-agenda{,-dark}.png` with marks visible, through the
    render-fixtures pipeline (`docs/develop/render-fixtures.md`).
  - Run `just docs-check`.
- **Gate.**
  1. `just portable-check`.
  2. Best-effort scratch-copy typecheck on the Mac (`bundle.sh`,
     `just typecheck-tests`), as `AGENTS.md` describes. If the Mac is offline, say the
     macOS targets are unverified.
  3. `Scripts/macos-ci-check`.
  4. Push, then watch the exact-SHA run to green with `/sase_monitor`.

### Decision record

> [!decision] memory_walk_marks_record Use `/sase_memory_write` in bob-cli:
>
> - **New strand.** Add `sase/memory/decisions/idle-capture-mirrors-walk-attention.md`,
>   "Idle Capture Mirrors Walk Attention". The claim: the idle agenda's title row shows
>   `s`/`w`/`b` marks exactly when `bob walks` reports attention for today; bob owns the
>   predicates in `docs/walks.md`, held in parity with the walker footer by shared
>   conformance vectors; the app fetches `bob walks` on its own lane and cadence;
>   Obsidian's selected family is never mirrored. Rejected alternatives: a
>   plugin-published state file, folding into `capture-pomodoros --tasks`, Swift-side
>   evaluation, and showing all four letters. Cost and reopen conditions as in this
>   plan. Cite the phase commits and the green CI SHA as evidence.
> - **Supersession mark.** Mark `decisions:idle-capture-shows-ledger-agenda` with
>   `metadata.status: superseded-in-part` and `superseded_by` pointing at the new
>   record. Add a one-line back-link stating that its only-fact-source claim no longer
>   covers the title row's walk marks. Do not otherwise edit its body.
> - **Regenerate.** Run `sase memory init`, then `just docs-gen` (which regenerates
>   `docs/develop/decisions.md`) and `just docs-check`.

## Live verification for Bryan (after the epic lands and bob's release train ships `walks`)

- With review commitments due, a lane over its cap, and a waiting bank row, the Obsidian
  selector shows `s`/`w`/`b` attention, and the empty capture panel shows `s• w• b•` at
  the top right, with the `s` hue matching the selector's.
- Clearing each one in Obsidian clears the same mark on the next panel show, within
  about a debounce interval when the panel is already open.
- Typing any character hides the marks along with the agenda; clearing the draft brings
  them back.
- Light and dark modes, Increase Contrast, and Reduce Motion all read well. VoiceOver
  reads the labels from the title row.
