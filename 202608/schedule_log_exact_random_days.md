---
tier: tale
title: Record exact random offsets in schedule-log entries
goal:
  Every machine-rolled schedule-log entry records the exact chosen relative day in bold
  alongside its configured priority window, with plugin and CLI behavior kept in parity.
proposed_by: bbugyi200.athena.vv
create_time: 2026-08-08 13:10:34
status: wip
---

# Plan: Record the exact random day offset in every automatic schedule-log entry

## Goal

Whenever Bob chooses a random relative day for a priority roll, the resulting
`🗓️ **SCHEDULE LOG**` entry must record both the exact chosen offset and the configured
window:

```markdown
- _2026-09-07_ — 🎲 priority P0 → P2 · random in **30** (8–30) days
```

Apply the same reason grammar to every machine-rolled path:

- `Ctrl+Shift+P` → a priority level on one task;
- `N<Ctrl+Shift+P>` → a priority level, with each task's independent offset;
- the pinned priority-roll suggestion in the `scheduled` stage, including a `Ctrl+R`
  re-roll;
- `bob capture <text> p:<N>`.

This is a `tale`: the work crosses two repositories, but it is one small parity contract
whose plugin, CLI, fixtures, docs, deployment, and verification should be completed by
one follow-up agent in sequence.

## Repositories and ownership

- Primary `bob-cli` checkout: update native `bob capture`, its Rust/unit/CLI tests,
  `README.md`, and `docs/projects.md`.
- Linked `bob-plugins` repository: first run
  `sase repo open bob-plugins -r "Implement exact rolled-day schedule-log reasons"` and
  use only the path it prints. Update Bob Navigation Hotkeys source, tests, manifest,
  and repository README. This is the source of truth for the `Ctrl+Shift+P`
  implementation and therefore should define the new reason contract before the CLI
  mirrors it.
- Do not edit the deployed plugin under `~/bob/.obsidian/plugins` directly. After the
  linked source changes pass tests, deploy through `bob plugins sync` as required by
  `bob-plugins/AGENTS.md`.
- No `chezmoi`, Hammerspoon, configuration-schema, or memory-file changes are needed.
  The keymap only invokes the plugin, and the configured `min_days` / `max_days` values
  remain unchanged.

## Existing behavior and data flow

Bob already chooses an inclusive integer offset and then loses it while constructing the
automatic reason:

- In `bob-plugins/plugins/bob-navigation-hotkeys/main.js`, `rollPriorityScheduledDate`
  computes `offset` but returns only a `Date`. `formatPriorityRollScheduleReason`
  receives only the priority level, so `formatPriorityRollWindowText` can render only
  `random in <min>–<max> days`. The single-task, counted-task, and pinned-suggestion
  writers all pass through that formatter.
- In `bob-cli/src/native/capture.rs`, `ResolvedPriority::rolled_offset` already retains
  the exact chosen `u64`, but the call to `capture_schedule_log::priority_roll_reason`
  passes only `min_days` and `max_days`.

The implementation must carry the value produced by the existing roll; it must never
call the random source a second time merely to format the reason. That is especially
important for counted sessions, where every target has its own roll, and for seeded CLI
runs, where note text and JSON must describe the scheduled date produced by that same
seed.

## Output contract and scope

1. Change only machine-generated `🎲` schedule-log reasons. Preserve priority transition
   heads exactly:

   | Gesture/result          | New reason shape                                                        |
   | ----------------------- | ----------------------------------------------------------------------- |
   | Priority changes        | `🎲 priority <from> → <to> · random in **<chosen>** (<min>–<max>) days` |
   | Missing prior priority  | `🎲 priority P0 → <to> · random in **<chosen>** (<min>–<max>) days`     |
   | Same priority re-picked | `🎲 priority <level> · random in **<chosen>** (<min>–<max>) days`       |
   | Pinned scheduled roll   | `🎲 <level> roll · random in **<chosen>** (<min>–<max>) days`           |

2. Keep the chosen day count bold with literal Markdown `**`; keep the window in
   parentheses, with the existing en dash (`U+2013`) and no spaces around it. Continue
   using `days` to match the existing configured-window grammar. Show both endpoints
   even if a future fixed window has `min_days == max_days`, so the exact choice and
   configured window remain distinct fields.
3. Do not change the pinned picker's detail row, priority selection rows, or notices.
   Those UI surfaces already show the exact ISO date/relative result and the configured
   range; this request is specifically about durable schedule-log sub-bullets.
4. Do not rewrite old log entries. Only newly generated reasons use the new grammar.
   Existing schedule-log parsing must continue accepting both old and new reason text.
5. Preserve all existing log guards: an automatic roll that lands on the current
   scheduled date writes no entry; `bob capture p:<N> s:<N>` writes no entry because the
   explicit schedule prevented a roll; manual reasons and `🤷 no reason given` are
   unchanged.
6. Keep the CLI JSON schema stable. `schedule_log.reason` and the rendered
   `schedule_log.lines` strings change contents, but no field is added or removed.

## Implementation

### 1. Preserve the plugin's roll result as date plus exact offset

In `bob-plugins/plugins/bob-navigation-hotkeys/main.js`, introduce one pure roll
primitive that returns the date and chosen day offset together. Keep
`rollPriorityScheduledDate` as a compatibility wrapper returning the `Date` if that
avoids unnecessary churn in exported helpers and existing callers/tests. The primitive
must retain the current inclusive/clamped random semantics, including `random() === 1`,
and must invoke the supplied random function exactly once per roll.

Thread the returned offset through every writer that creates an automatic roll reason:

- `createPriorityRollDateItem` stores the offset on the pinned item. A `Ctrl+R`
  replacement naturally gets the replacement roll's offset, and
  `buildPriorityRollScheduleLogForItem` passes that exact value onward.
- `setBulletPriorityValue` uses one roll result for the scheduled date and for both the
  project-frontmatter and ordinary-inline schedule-log payloads.
- `setCountedBulletPriorityValue` retains one roll result per original task line. Derive
  both `scheduledValueByLine` and `reasonByLine` from that same per-line result; do not
  create two maps by rolling twice. This ensures a three-task batch can correctly log
  three different choices such as 2, 5, and 7 days.

Update `formatPriorityRollScheduleReason` (and `buildPriorityRollScheduleLog`'s details
contract) to require the chosen offset for a valid generated reason and render:

```text
random in **<chosen>** (<min>–<max>) days
```

Leave `formatPriorityRollWindowText` as the range-only picker-detail formatter and
update its misleading comment: it is no longer shared with durable log reasons. Validate
that the chosen offset is an integer within the configured inclusive range before
formatting, so a future missed call site cannot emit `**undefined**` or a reason that
contradicts its level. All production call sites must supply a valid value; the normal
roll paths must not silently lose a log.

The entry parser's `reason` capture is intentionally free-form and should need no
grammar change. Add a round-trip assertion containing `**<chosen>**` so the new Markdown
is explicitly protected.

### 2. Mirror the contract in `bob capture`

In `bob-cli/src/native/capture_schedule_log.rs`, extend `priority_roll_reason` to accept
the selected `rolled_days: u64` as well as the level labels and min/max bounds. Render
the same bytes as the plugin:

```text
🎲 priority <head> · random in **<rolled_days>** (<min>–<max>) days
```

In `bob-cli/src/native/capture.rs`, obtain the reason from the existing
`ResolvedPriority::rolled_offset` value and pass that exact value to the formatter. Do
not derive it from the resulting ISO date, generate another seed, or change
`config::PriorityLevel::roll_offset`. The existing `Option<u64>` is also the correct
guard for suppressing a log when `s:<N>` wins.

Keep `ScheduleLog`, capture-block ordering/indentation, human output, and JSON structure
unchanged apart from their rendered reason strings. Update parity comments in
`capture_schedule_log.rs` so they describe the new plugin formatter and fixture rather
than the superseded range-only bytes.

### 3. Plugin tests and release metadata

In `bob-plugins/scripts/test-navigation-hotkeys.cjs`:

- Extend roll-helper boundary coverage to assert both date and offset for the minimum,
  midpoint, maximum, and clamped-`1` random inputs.
- Update `formatPriorityRollScheduleReason` coverage for all four heads to pass a chosen
  offset and expect bold exact days plus the parenthesized range. Add
  invalid/missing/out-of-window chosen-offset coverage matching the formatter's guard.
- Update `buildPriorityRollScheduleLog` expectations and verify the unchanged date guard
  still returns `null`.
- Update exact editor fixtures for ordinary priority changes, missing/current priority,
  project schedules, counted batches, and the pinned scheduled-roll row. The counted
  runtime test must prove each task logs its own offset (for the existing deterministic
  sequence: 2, 5, and 7), rather than reusing one batch-wide number.
- Assert a re-rolled pinned suggestion carries the replacement offset into the eventual
  log, not the offset from the initially displayed suggestion.
- Keep the picker detail assertion range-only and add/retain a schedule-log format/parse
  round trip whose reason contains bold Markdown.

Bump `plugins/bob-navigation-hotkeys/manifest.json` from `1.21.0` to `1.22.0`. Update
the matching version references and Bob Navigation Hotkeys description in
`bob-plugins/README.md` to state that automatic schedule-log reasons record the exact
chosen relative day as well as its configured window.

Run from the linked `bob-plugins` checkout:

```bash
npm test
npm run validate
```

### 4. CLI tests and documentation

Update the focused unit tests in `bob-cli/src/native/capture_schedule_log.rs` to cover:

- the exact new parity fixture, including literal `**<chosen>**`;
- changed and unchanged priority-level heads;
- exact punctuation/codepoints (`U+2013` window dash remains distinct from the entry's
  `U+2014` separator);
- a fixed min/max range without collapsing away either endpoint.

Update all exact `p:<N>` fixtures in `bob-cli/tests/cli.rs`, including normal,
wide-window, JSON, routed/Pomodoro, two-space indentation, clipboard ordering,
sub-bullet nesting, and dry-run cases. Use the scheduled date minus `BOB_NOW` to assert
the known bold selected value in each seeded test; retain the existing range-bound
assertions. In particular, the current seed-1 fixtures must describe their actual
offsets (P1 `7`, P2 `11`, P3 `36`, and P4 `331` for the dates already asserted), which
catches any mismatch between the date and reason. Preserve the negative test that
`p:<N> s:<N>` emits neither a marker nor a die reason.

Update `bob-cli/README.md` and `bob-cli/docs/projects.md`:

- replace examples and the deterministic-reason table with the new
  `random in **<chosen>** (<min>–<max>) days` grammar;
- explain that the bold number is the actual relative offset selected for that scheduled
  date, while the parenthesized range is the configured priority window;
- state that counted tasks record their independent choices;
- update JSON prose/examples without claiming a schema change;
- retain the manual-reason, unchanged-date, and explicit-`s:<N>` exceptions.

Run from the primary `bob-cli` checkout:

```bash
just all
```

### 5. Deploy and parity smoke test

After the linked plugin source and manifest validate, sync only the modified plugin from
the linked checkout, using `--no-pull` so the implementation workspace itself is the
deployment source:

```bash
bob plugins sync -p bob-navigation-hotkeys -r "$PWD" --no-pull --dry-run
bob plugins sync -p bob-navigation-hotkeys -r "$PWD" --no-pull
```

Then perform deterministic smoke checks:

```bash
# From bob-cli; both the task date and reason should reflect the same seeded roll.
BOB_PRIORITY_ROLL_SEED=1 cargo run -- capture --dry-run -- 'exact roll p:2'
BOB_PRIORITY_ROLL_SEED=1 cargo run -- capture --dry-run -f json -- 'exact roll p:2'

# Explicit scheduling still suppresses the automatic log.
BOB_PRIORITY_ROLL_SEED=1 cargo run -- capture --dry-run -- 'explicit p:2 s:1'
```

In Obsidian after reloading Bob Navigation Hotkeys, smoke-test a normal priority pick, a
counted priority pick, and a pinned roll followed by `Ctrl+R`. Confirm the bold numbers
equal the displayed dates' relative offsets and that a later manual scheduled change
appends under the same `🗓️ **SCHEDULE LOG**` marker.

## Acceptance criteria

- Every newly written `🎲` schedule-log reason contains exactly one bold chosen day
  count followed by the configured range in parentheses.
- The recorded count is the same roll used to produce the entry's scheduled date;
  counted targets and re-rolls cannot leak or reuse another roll's count.
- Plugin and CLI reasons are byte-for-byte equivalent for the corresponding
  priority-transition form.
- Picker rows/notices, manual reasons, old log parsing, unchanged-date suppression,
  explicit CLI schedules, indentation, block ordering, and JSON shape remain
  behaviorally unchanged.
- `npm test`, `npm run validate`, and `just all` pass.
- `bob plugins sync` deploys the source-of-truth plugin successfully without directly
  editing the vault copy.

## Risks and safeguards

- **Accidental re-roll while formatting:** retain a compound roll result and consume it
  everywhere; counted tests assert distinct per-task values.
- **Stale pinned-roll metadata after `Ctrl+R`:** replace the whole item, including its
  exact offset, and select it before the write test.
- **Markdown/parser interaction:** reason text now contains `**`; round-trip it through
  `parseScheduleLogEntryBullet` and preserve its free-form reason capture.
- **DST/calendar arithmetic:** record the integer selected by the roll primitive instead
  of recomputing elapsed milliseconds from two dates.
- **Visual dash ambiguity:** exact-string and codepoint assertions distinguish the en
  dash inside `(min–max)` from the em dash before the reason.
- **Cross-repository drift:** change and test the plugin contract first, then update the
  CLI parity formatter, fixtures, and docs in the same tale.

## Out of scope

- Migrating historical `SCHEDULE LOG` entries to add inferred offsets.
- Adding a new JSON field for the offset or priority window.
- Changing random-number generation, priority configuration, date-picker options,
  notices, status transitions, Pomodoro pruning, or schedule-log placement/parsing rules
  beyond accepting the new free-form reason text.
