---
tier: tale
title: Shorten machine-rolled schedule-log reasons
goal:
  Every machine-rolled schedule-log sub-bullet reads `🎲 <levels> · in **<chosen>**
  (<min>–<max>) days`, dropping the redundant `priority` and `random` words, with plugin
  and CLI output kept byte-for-byte in parity.
proposed_by: bbugyi200.athena.wm
create_time: 2026-08-09 10:07:58
status: wip
---

# Plan: Make randomly-rolled schedule-log sub-bullets more concise

## Goal

A machine-rolled `🗓️ **SCHEDULE LOG**` entry currently spends two words saying what its
own die emoji and `P<N>` labels already say:

```markdown
- _2026-08-09 → 2026-08-28_ — 🎲 priority P0 → P2 · random in **19** (8–30) days
```

It should read:

```markdown
- _2026-08-09 → 2026-08-28_ — 🎲 P0 → P2 · in **19** (8–30) days
```

`🎲` already marks the entry as machine-rolled, so `random` is redundant; `P0 → P2` is
self-evidently a priority transition, so `priority` is redundant. Everything else about
the entry — the emoji, the `·` separator, the bold chosen offset, the parenthesized
configured window with its en dash, the trailing `days` — is unchanged.

This is a `tale`: the work crosses two repositories, but it is a single string-format
contract with no new data flow, and its plugin change, CLI mirror, fixtures, docs,
deployment, and verification should be completed by one follow-up agent in sequence.

## Repositories and ownership

- Linked `bob-plugins` repository — the source of truth for the reason grammar. Run
  `sase repo open bob-plugins -r "Shorten machine-rolled schedule-log reason text"`
  first and use only the path it prints. Update
  `plugins/bob-navigation-hotkeys/main.js`, `scripts/test-navigation-hotkeys.cjs`,
  `plugins/bob-navigation-hotkeys/manifest.json`, and `README.md`. Change and test this
  side first, then mirror it in the CLI.
- Primary `bob-cli` checkout — update `src/native/capture_schedule_log.rs`,
  `src/native/capture.rs`, `tests/cli.rs`, `README.md`, and `docs/projects.md`.
- Do not edit the deployed plugin under `~/bob/.obsidian/plugins` directly. Deploy the
  linked source with `bob plugins sync` as `bob-plugins/AGENTS.md` requires.
- Do not edit anything under `bob-cli/.sase/artifacts/`. Those files are the frozen
  record of the earlier `schedule_log_exact_random_days` plan, not live documentation.
- No `chezmoi`, Hammerspoon, config-schema, glossary, or sase-memory changes are needed.
  `sase/memory/glossary.md` defines "Schedule Log" without quoting the reason grammar.

## Existing behavior

Two independent implementations render these bytes and must not drift:

- `bob-plugins/plugins/bob-navigation-hotkeys/main.js`:
  `formatPriorityRollChosenWindowText(level, rolledDays)` returns
  `random in **<days>** (<min>–<max>) days` after validating that `rolledDays` is an
  integer inside the configured inclusive window;
  `formatPriorityRollScheduleReason(details)` prefixes the head with `priority ` when
  `details.source === "priority"` and uses `<label> roll` when it is `"scheduled"`, then
  joins head and window with `SCHEDULE_LOG_AUTO_REASON_SEPARATOR`.
- `bob-cli/src/native/capture_schedule_log.rs`: `priority_roll_reason` builds the same
  string for `bob capture <text> p:<N>`, which only ever takes the `source: "priority"`
  branch.

Nothing consumes the rendered reason as structured data. `SCHEDULE_LOG_ENTRY_RE`
captures `reason` as a free-form trailing group, the automatic reason never reaches a
notice or picker row, and `bob-cli` writes schedule logs but never parses them. The
change is therefore confined to the two formatters plus their fixtures and docs.

A separate formatter, `formatPriorityRollWindowText(level)`, renders the range-only
`random in <min>–<max> days` used in the pinned roll row's picker detail
(`2026-08-05 · Wed · random in 2–7 days`). It is a different surface with a different
job and is explicitly not part of this change; see "Out of scope".

## Output contract

Change only the machine-generated `🎲` reason text:

| Gesture/result                                    | New reason shape                                        |
| ------------------------------------------------- | ------------------------------------------------------- |
| Priority level picked, previous level differs     | `🎲 <from> → <to> · in **<chosen>** (<min>–<max>) days` |
| Priority level picked, task had no priority field | `🎲 P0 → <to> · in **<chosen>** (<min>–<max>) days`     |
| Priority level re-picked unchanged                | `🎲 <level> · in **<chosen>** (<min>–<max>) days`       |
| Pinned roll suggestion in the `scheduled` stage   | `🎲 <level> roll · in **<chosen>** (<min>–<max>) days`  |
| Reason prompt skipped on a task that has a log    | `🤷 no reason given` (unchanged)                        |

Worked examples, before → after:

```text
🎲 priority P0 → P2 · random in **19** (8–30) days   →  🎲 P0 → P2 · in **19** (8–30) days
🎲 priority P2 · random in **17** (8–30) days        →  🎲 P2 · in **17** (8–30) days
🎲 P1 roll · random in **7** (2–7) days              →  🎲 P1 roll · in **7** (2–7) days
```

Constraints:

1. Keep the ` roll` suffix on the pinned `scheduled`-stage head. Once `priority ` is
   gone it is the only thing distinguishing a re-picked-level entry (`🎲 P2 · …`) from a
   pinned roll (`🎲 P2 roll · …`), so dropping it would erase information rather than
   redundancy.
2. Keep every surviving codepoint exactly as it is: `🎲` (`U+1F3B2`), the `·` separator
   (`U+00B7`), the `→` transition (`U+2192`), the `—` entry separator (`U+2014`), and
   the window's `–` (`U+2013`). Keep the chosen count bold with literal `**`, keep both
   window endpoints even when `min == max`, and keep the trailing `days`.
3. Keep the existing validation guard in `formatPriorityRollChosenWindowText`: a
   missing, non-integer, or out-of-window `rolledDays` still yields `""`, which still
   suppresses the whole log entry. Shortening the prose must not weaken that check.
4. Do not touch entry structure, indentation, marker placement, newest-first ordering,
   manual typed reasons, or the `🤷 no reason given` fallback.
5. Preserve every existing log guard: an automatic roll landing on the task's current
   scheduled date writes nothing, and `bob capture p:<N> s:<N>` writes nothing because
   the explicit schedule prevented a roll.
6. Keep the CLI JSON schema stable. `schedule_log.reason` and `schedule_log.lines`
   change contents only; no field is added, removed, or renamed.
7. Do not rewrite historical entries, and do not add any parsing that depends on the new
   wording. Old and new entries coexist as free text.

## Implementation

### 1. Shorten the plugin's reason formatter

In `bob-plugins/plugins/bob-navigation-hotkeys/main.js`:

- In `formatPriorityRollChosenWindowText`, change the returned template from
  `random in **${days}** (${bounds.minDays}–${bounds.maxDays}) days` to
  `in **${days}** (${bounds.minDays}–${bounds.maxDays}) days`. Leave the
  `getPriorityRollBounds` guard and every early return untouched.
- In `formatPriorityRollScheduleReason`, drop the `priority ` prefix from the
  `details.source === "priority"` head so it becomes either
  `${details.fromLevelLabel}${SCHEDULE_LOG_TRANSITION}${level.label}` or bare
  `level.label`. Leave the `"scheduled"` branch's `${level.label} roll` alone and leave
  the final `${emoji} ${head}${separator}${windowText}` join alone.
- Update the two comments that now misstate the output: the block above
  `formatPriorityRollWindowText`, which currently presents the picker text and the
  durable reason as the same phrasing plus an offset, and the block above
  `formatPriorityRollScheduleReason`, which describes the `priority`/`scheduled` heads.
  State plainly that the two formatters have diverged: the picker detail still says
  `random in <min>–<max> days`, while the durable reason leans on `🎲` and says
  `in **<chosen>** (<min>–<max>) days`.

No other production call site changes. Confirm this by grepping the plugin for
`random in` and for a `priority ` literal after the edit; the only surviving `random in`
should be inside `formatPriorityRollWindowText`.

### 2. Update plugin fixtures

In `bob-plugins/scripts/test-navigation-hotkeys.cjs`, update every exact-bytes
expectation that contains a `🎲` reason — roughly two dozen sites covering ordinary
priority changes, missing/current priority, project-frontmatter schedules, counted
batches, the pinned scheduled-roll row and its `Ctrl+R` re-roll, and the
`buildPriorityRollScheduleLog` / parse round-trip cases. In particular:

- The `formatPriorityRollScheduleReason covers all four shapes` test must assert the
  four new strings verbatim, and its invalid/missing/out-of-window `rolledDays` cases
  must still expect `""`.
- Leave the picker-detail expectation `"2026-08-05 · Wed · random in 2–7 days"` and the
  computed `formatPriorityRollWindowText`-suffix assertion unchanged. They are the
  regression guard proving the picker surface did not move with the log surface.
- Add two small assertions that lock the divergence in place: a generated schedule-log
  reason does not contain `random` or the word `priority`, and the pinned roll row's
  detail still does contain `random in`.

### 3. Mirror the contract in `bob capture`

In `bob-cli/src/native/capture_schedule_log.rs`, change `priority_roll_reason`'s format
string to render the same bytes as the plugin:

```text
🎲 <head> · in **<rolled_days>** (<min>–<max>) days
```

That is, remove the literal `priority ` before `{head}` and the literal `random ` before
`in`. The `head` collapse (`from == to` renders one label) stays as written, as do the
`AUTO_REASON_EMOJI` / `AUTO_REASON_SEPARATOR` / `IMPLICIT_LEVEL_LABEL` constants and the
function signature. Update the doc comment above the function, which describes the
mirrored `source: "priority"` branch, and re-check the module header's `main.js:250-274`
constant-block citation still points at the right lines after step 1.

Update the module's own unit tests: the picker-parity fixture, the unchanged-level
collapse case, and the fixed-window (`min == max`) case. Update the two exact strings in
the capture-block assembly tests in `bob-cli/src/native/capture.rs`.

Nothing else in `bob-cli` changes. `ResolvedPriority::rolled_offset`, the `ScheduleLog`
struct, capture-block ordering and indentation, human output, and JSON structure are all
untouched apart from the rendered reason bytes.

### 4. Update CLI integration fixtures

In `bob-cli/tests/cli.rs`, update the nine exact `p:<N>` schedule-log expectations,
which cover the normal case, the wide P4 window, JSON output, routed/Pomodoro capture,
two-space note indentation, clipboard child ordering, sub-bullet nesting, and dry-run
output. Keep every asserted date and bold offset pairing exactly as it is — only the
prose between them shrinks — so the tests still prove the reason describes the same
seeded roll that produced the date. Preserve the negative test asserting that
`p:<N> s:<N>` emits neither a `SCHEDULE LOG` marker nor a `🎲` reason.

### 5. Documentation and release metadata

In `bob-plugins`: bump `plugins/bob-navigation-hotkeys/manifest.json` from `1.22.0` to
`1.23.0` and update the matching version cell in the repository `README.md` table. The
Bob Navigation Hotkeys description in that table paraphrases the deterministic reason
without quoting its grammar, so it needs no wording change.

In `bob-cli/README.md`: update the `p:<N>` capture example's schedule-log line and the
surrounding prose about the bold number and parenthesized range. The JSON-output section
describes `schedule_log` structurally and stays accurate.

In `bob-cli/docs/projects.md`: update the `Ctrl+Shift+P` priority example, the
deterministic-reason table, and the `bob capture` paragraph that currently names a
`priority P0 → <to>` transition (it becomes a `P0 → <to>` transition). Keep the
surrounding rules — manual reasons, the `🤷` fallback, unchanged-date suppression, and
the explicit-`s:<N>` exception — as they are.

### 6. Verify, deploy, smoke-test

From the linked `bob-plugins` checkout:

```bash
npm test
npm run validate
```

From the primary `bob-cli` checkout:

```bash
just all
```

Then deploy only the modified plugin from the linked checkout, so the implementation
workspace itself is the deployment source:

```bash
bob plugins sync -p bob-navigation-hotkeys -r "$PWD" --no-pull --dry-run
bob plugins sync -p bob-navigation-hotkeys -r "$PWD" --no-pull
```

Deterministic CLI smoke checks (the same seed must produce a date and a reason that
agree, and the explicit-schedule case must still stay silent):

```bash
BOB_PRIORITY_ROLL_SEED=1 cargo run -- capture --dry-run -- 'concise roll p:2'
BOB_PRIORITY_ROLL_SEED=1 cargo run -- capture --dry-run -f json -- 'concise roll p:2'
BOB_PRIORITY_ROLL_SEED=1 cargo run -- capture --dry-run -- 'explicit p:2 s:1'
```

After reloading Bob Navigation Hotkeys in Obsidian, smoke-test a plain priority pick, a
counted priority pick, and a pinned roll row followed by `Ctrl+R`. Confirm each written
sub-bullet uses the short grammar, that the pinned row's own detail line still reads
`… · random in <min>–<max> days`, and that a subsequent manual scheduled change still
appends its typed reason above the rolled one under the same marker.

## Acceptance criteria

- Every newly written `🎲` schedule-log reason matches
  `🎲 <levels> · in **<chosen>** (<min>–<max>) days`, with no `priority` or `random`
  word, and the pinned form keeps its ` roll` suffix.
- Plugin and CLI reasons remain byte-for-byte identical for the corresponding
  priority-transition form.
- The pinned picker row's detail still reads `random in <min>–<max> days`; picker rows,
  notices, typed reasons, the `🤷` fallback, log placement and ordering, unchanged-date
  suppression, `p:<N> s:<N>` silence, indentation, and JSON shape are all behaviorally
  unchanged.
- A missing or out-of-window rolled offset still suppresses the entry rather than
  emitting `**undefined**`.
- `npm test`, `npm run validate`, and `just all` pass.
- `bob plugins sync` deploys the source-of-truth plugin without editing the vault copy
  directly.

## Risks and safeguards

- **Silent parity drift between the two implementations.** The bytes are duplicated in
  JS and Rust with no shared fixture. Change the plugin first, then copy its exact
  output into the Rust parity test, and keep both repositories' test suites green before
  deploying.
- **Shortening the wrong formatter.** `formatPriorityRollWindowText` and
  `formatPriorityRollChosenWindowText` differ by one word plus the bold offset. Editing
  the wrong one silently changes the picker instead of the log. The retained
  picker-detail fixture catches this.
- **Weakening the offset guard while editing the template.** The bounds check lives in
  the same short function as the string being changed; keep the early returns
  byte-identical and keep the invalid-input tests.
- **Mixed grammar during rollout.** Between the plugin deploy and a `bob-cli` reinstall,
  the two writers can emit different wordings. This is harmless because reasons are free
  text that nothing parses, but land both repositories together to keep it brief.
- **Stale vault snapshot when judging migration need.** The finding that no historical
  entries exist is from the 2026-08-06 vault snapshot; a handful of entries may have
  been written since. Any such entry keeps the old wording, which is acceptable — see
  "Out of scope".

## Out of scope

- The pinned roll row's picker detail (`… · random in 2–7 days`) and every other picker
  row, notice, or toast. Those describe what a roll _will_ do, where "random" is doing
  real work; this request is specifically about the durable log sub-bullet, where the
  die already carries that meaning.
- Migrating or rewriting existing `🗓️ **SCHEDULE LOG**` entries. The 2026-08-06 vault
  snapshot contains none at all, so a migration script would have nothing to do; any
  entry written in the last few days can be left in the old wording or hand-edited.
- Adding a JSON field for the offset or window, changing the roll source or priority
  configuration, or altering schedule-log placement, ordering, or parsing rules.
- The `🤷 no reason given` fallback, typed reasons, and the reason-prompt UI.
