---
tier: tale
title: Write a SCHEDULE LOG entry for `bob capture` `p:<N>` rolls
goal:
  A `bob capture` that rolls a scheduled date from a `p:<N>` token also writes the managed `🗓️
  **SCHEDULE LOG**` child bullet and one dated `🎲 priority P0 → <label> · random in <min>–<max>
  days` entry beneath the captured line, byte-for-byte identical to what the Obsidian `Ctrl+Shift+P`
  picker writes when a priority level is chosen with no reason prompt.
proposed_by: bbugyi200.athena.uq
create_time: 2026-08-07 09:57:29
status: wip
---

# Plan: Write a `SCHEDULE LOG` entry for `bob capture` `p:<N>` rolls

## Repositories

- Primary `bob-cli` checkout: all Rust, `README.md`, and `docs/projects.md` changes. This is the
  only repo that changes.
- `sase repo open bob-plugins -r "<reason>"` — read-only reference.
  `plugins/bob-navigation-hotkeys/main.js` and `scripts/test-navigation-hotkeys.cjs` are the source
  of truth for the exact bytes the CLI must reproduce. **Do not edit them.** The picker already
  writes these entries correctly; this plan only teaches the CLI to write the same thing.
- No `chezmoi` change. The Hammerspoon panel passes `p:<N>` through untouched and displays
  `decoded.text`, not the rendered block.

## Problem

`bob capture 'someday idea p:4'` writes the priority field and rolls a scheduled date:

```markdown
- [ ] #task someday idea [created::2026-08-07] [priority::lowest] [scheduled::2026-11-02]
```

The same gesture in Obsidian — `Ctrl+Shift+P` → `priority` → `P4` — writes the field, rolls the
date, **and** records why the date is what it is, without prompting:

```markdown
- [?] #task someday idea [priority:: lowest] [scheduled:: 2026-11-02] ^one
  - 🗓️ **SCHEDULE LOG**
    - _2026-11-02_ — 🎲 priority P0 → P4 · random in 91–365 days
```

So a CLI-rolled date arrives with no provenance. Three months later the note says "scheduled
2026-11-02" and nothing says a die picked it. Worse, the two entry points now produce structurally
different notes for the same decision, which is exactly the divergence `docs/projects.md` argues
against for the `[priority::]` value itself.

## What the picker writes today (the parity contract)

From `plugins/bob-navigation-hotkeys/main.js` (constants near `:244-275`, formatters near
`:1470-1800`), confirmed by the fixture at `scripts/test-navigation-hotkeys.cjs:4365-4367`:

| Piece                 | Exact text                              | Codepoints                             |
| --------------------- | --------------------------------------- | -------------------------------------- |
| Marker bullet text    | `🗓️ **SCHEDULE LOG**`                   | `U+1F5D3 U+FE0F` — keep the VS16       |
| Entry, no prior date  | `*<to>* — <reason>`                     | emphasis `*`, separator `U+2014`       |
| Entry, prior date     | `*<from> → <to>* — <reason>`            | transition `U+2192`                    |
| Auto reason head      | `🎲 priority <from-label> → <to-label>` | die is `U+1F3B2`                       |
| Auto reason separator | `·`                                     | `U+00B7`                               |
| Roll window           | `random in <min>–<max> days`            | en dash `U+2013`, **no** spaces around |
| Implicit prior level  | `P0`                                    | a task with no priority field          |

Two plugin rules matter for the shape the CLI emits:

- `formatScheduleLogEntryText` omits the `<from> → ` half entirely when there is no previous value.
  A brand-new capture has no previous `[scheduled::]`, so **every** capture entry is the short
  `*<to>* — <reason>` form.
- `getPriorityRollFromLevelLabel` falls back to `IMPLICIT_PRIORITY_LEVEL_LABEL` (`P0`) when the task
  has no priority field, and `formatPriorityRollScheduleReason` renders `priority <from> → <to>`
  whenever the labels differ. A brand-new capture always differs from `P0`, so the head is
  **always** `priority P0 → <label>` — capture never emits the unchanged-level (`priority P2 · …`)
  or pinned-suggestion (`P2 roll · …`) variants.

## Design decisions

**1. Log only when `p:<N>` actually rolled the date.** `p:2 s:1` writes `[priority::medium]` but
takes its date from `s:1`. The reason text would then claim a roll that never happened. It is also
exactly the plugin's "user typed a date" case, where the plugin prompts for a reason — and on a task
with no existing log, an empty prompt writes _nothing_ and creates no marker (`docs/projects.md`,
"Schedule-log reason prompt"). `bob capture` has no prompt and every capture is a brand-new line
with no log, so the faithful translation is: no roll, no entry. The trigger is therefore precisely
`resolve_priority` having returned a rolled offset, not merely "a `p:` token was present".

**2. Reproduce the plugin's bytes exactly, including `*` emphasis.** `SCHEDULE_LOG_ENTRY_RE` in
`main.js` matches `(?<emphasis>\*\*?)` — one or two asterisks and nothing else. `docs/projects.md`
renders its examples with `_…_`, but that is prose formatting; an underscore-emphasized entry would
be invisible to the plugin's own parser, so a later `Ctrl+Shift+P` on a captured task would append a
second marker instead of reusing the log. Use `*`.

**3. Indent with the note's dominant indent unit, not a hard tab.** The plugin uses
`SCHEDULE_LOG_INDENT_UNIT = "\t"` because it is editing an existing task in an existing note.
`bob capture` already resolves child indentation from the target note for clipboard children and for
sub-bullet captures (README: "the note's dominant tab-or-two-space indentation and falls back to a
tab"). Being internally consistent inside one captured block beats matching a constant the plugin
only reaches when a task has no existing entries. In a tab vault — the real one — the two agree
anyway.

**4. The log block is the last thing in the capture block, after any clipboard children.** The
plugin appends the marker "as the last direct child of the task … after any hand-written notes or
dependency links". Clip children are capture's equivalent of that pre-existing content, so
`%`-captured bullets come first and the marker follows.

**5. Every capture kind gets the log.** `p:<N>` already writes `[priority::]` and `[scheduled::]` on
tasks, Pomodoro tasks, section bullets, and sub-bullets alike (README: rendered "for consistency").
Suppressing the log for some of them would create a second rule to remember with no payoff, and the
rendering is one shared code path.

**6. New module `src/native/capture_schedule_log.rs`.** `capture.rs` is already 4 261 lines, and the
repo's convention is `capture_clip.rs` / `capture_sections.rs` / `capture_targets.rs` for
capture-supporting concerns. A dedicated module gives the parity constants and their codepoint
assertions one obvious home.

**7. Replace `resolve_priority`'s 4-tuple with a struct.** The reason text needs `min_days` and
`max_days` too; a 6-tuple of `String`s and `Option<u64>` is unreadable. Introduce `ResolvedPriority`
and update the three call sites.

## Implementation

### 1. `src/native/config.rs`

`PriorityLevel::min_days` and `max_days` are private module fields. Add two accessors beside
`label()` / `value()`:

```rust
pub(crate) fn min_days(&self) -> u64 { self.min_days }
pub(crate) fn max_days(&self) -> u64 { self.max_days }
```

No parsing, validation, or `roll_offset` changes.

### 2. New `src/native/capture_schedule_log.rs`

Register it in `src/native.rs` in alphabetical order (between `capture_sections` and
`capture_targets`; note `capture_schedule_log` sorts before `capture_sections`, so it goes directly
after `capture_clip`).

Contents — pure functions, no I/O:

- Constants mirroring `main.js`, each with a comment naming the `main.js` constant it tracks and its
  codepoints: `MARKER_TEXT` (`🗓️ **SCHEDULE LOG**`), `ENTRY_EMPHASIS` (`*`), `SEPARATOR` (`—`),
  `TRANSITION` (`→`), `AUTO_REASON_EMOJI` (`🎲`), `AUTO_REASON_SEPARATOR` (`·`),
  `IMPLICIT_LEVEL_LABEL` (`P0`).
- `pub(crate) fn priority_roll_reason(from_label: &str, to_label: &str, min_days: u64, max_days: u64) -> String`
  — `🎲 priority {from} → {to} · random in {min}–{max} days`, collapsing to `🎲 priority {to} · …`
  when the labels are equal so the helper stays faithful to `formatPriorityRollScheduleReason` even
  though capture only ever passes `P0`.
- `pub(crate) fn entry_text(from: Option<&str>, to: &str, reason: &str) -> String` —
  `*{from} → {to}* — {reason}`, or `*{to}* — {reason}` when `from` is `None`.
- `pub(crate) struct ScheduleLog { pub reason: String, pub lines: Vec<String> }`, `Serialize`, with
  `pub(crate) fn plan(indent_unit: &str, scheduled: &str, reason: String) -> ScheduleLog` building
  the two lines: `{indent}- {MARKER_TEXT}` and `{indent}{indent}- {entry_text}`.

Derive `Serialize` on `ScheduleLog` so it drops straight into `CaptureResult`; field order `reason`,
`lines`.

### 3. `src/native/capture.rs`

- Replace the `resolve_priority` return tuple with
  `struct ResolvedPriority { name: String, value: String, label: String, min_days: u64, max_days: u64, rolled_offset: Option<u64> }`
  (`:634`). Update the destructuring at `:417-428` and the `CaptureResult` construction at
  `:582-583`.
- Hoist the child-indent computation (`:461`). Today it is
  `parsed.clip.as_ref().map(|_| clip_indent_unit(&target))`; it must also run when a schedule log
  will be written, and must still be computed **once** (it reads the target file). Compute a
  `schedule_log_reason: Option<String>` first — `Some` only when `priority.rolled_offset.is_some()`
  — then resolve the indent when `parsed.clip.is_some() || schedule_log_reason.is_some()`. Rename
  `clip_indent_unit` to `child_indent_unit` since it is no longer clip-specific; leave its body
  alone.
- Build the log after `scheduled` is resolved:
  `let schedule_log = schedule_log_reason.map(|reason| capture_schedule_log::plan(indent, scheduled_as_str, reason));`
  The `scheduled` string is guaranteed `Some` whenever `rolled_offset` is `Some`, but express that
  with a combinator rather than an `unwrap` — a rolled offset that overflowed the date already
  returned `Err` earlier.
- Extend the `capture_block` assembly (`:506`) to append `schedule_log.lines` after the clip lines.
  Keep it a single `\n`-joined string so `plan_sub_bullet_capture`'s uniform re-indent (`:975`) and
  every placement path keep working unchanged.
- Add `#[serde(skip_serializing_if = "Option::is_none")] schedule_log: Option<ScheduleLog>` to
  `CaptureResult` (`:2433`), placed after `clip` so JSON key order follows note order.
- In `print_human_success`, print the log lines dim immediately after the clip-lines loop (`:2510`)
  and before the Pomodoro link block, matching the order they appear in the note.

### 4. Tests

**`src/native/capture_schedule_log.rs` unit tests.** Assert the rendered strings _and_ their
codepoints, quoting `scripts/test-navigation-hotkeys.cjs:4365-4367` in a comment as the source of
truth:

- `plan("\t", "2026-11-02", priority_roll_reason("P0", "P4", 91, 365))` produces exactly
  `["\t- 🗓️ **SCHEDULE LOG**", "\t\t- *2026-11-02* — 🎲 priority P0 → P4 · random in 91–365 days"]`.
- A codepoint test walking the marker line and asserting `U+1F5D3` is followed by `U+FE0F`, and the
  entry line contains `U+2014`, `U+1F3B2`, `U+2192`, `U+00B7`, and `U+2013` — a stray VS16 drop or
  an em-dash/en-dash swap must fail loudly rather than diff as identical-looking text.
- Two-space indent unit produces `"  - …"` / `"    - …"`.
- `entry_text(Some("2026-08-13"), "2026-09-02", r)` renders the `→` form; `None` renders the short
  form.
- `priority_roll_reason("P2", "P2", 8, 30)` collapses to `🎲 priority P2 · random in 8–30 days`.

**`src/native/capture.rs` unit tests.** One test that the assembled capture block for a clip +
priority capture orders lines as: capture line, clip children, marker, entry.

**`tests/cli.rs`.** Five existing tests assert exact note contents and must be updated, not deleted:

- `capture_priority_level_one_rolls_scheduled_date_in_window` (`:2708`) — inbox is now three lines;
  assert the full expected block including the entry text with the P1 window (`random in 2–7 days`).
- `capture_priority_level_four_rolls_scheduled_date_in_window` (`:2749`).
- `capture_priority_with_explicit_schedule_skips_roll` (`:2783`) — **must stay two-line-free**:
  assert the inbox is exactly the single task line and contains neither `SCHEDULE LOG` nor `🎲`.
  This is decision 1's regression guard.
- `capture_priority_json_includes_priority_fields_only_when_set` (`:2884`) — add
  `json["schedule_log"]["reason"] == "🎲 priority P0 → P3 · random in 31–90 days"`, a two-element
  `lines` array, and `unset_json.get("schedule_log").is_none()`.
- `capture_priority_renders_before_scheduled_and_before_pomodoro_block_id` (`:2955`) — the routed
  note gains the two indented lines after the `^foobar` task line; the block ID stays last on the
  task line itself.

`priority_scheduled_offset_days` (`:3014`) scans for the first `[scheduled::` in stdout and is
unaffected — the log lines contain no inline fields. Do not change it.

New `tests/cli.rs` cases:

- A two-space-indented target note plus `p:2` renders the marker at two spaces and the entry at
  four.
- `p:1` with a `%` clipboard capture: clip children come first, marker and entry last.
- A sub-bullet capture (`-r <route> -t <block-id>`) with `p:2`: the whole block, log included, sits
  one level under the parent task and the parent line is untouched.
- `--dry-run` with `p:2` prints the two log lines and writes nothing.

Also add `schedule_log` to the omitted-field list in `json_success_shape_is_stable` (`:3758`) and to
the struct literal it builds.

### 5. Docs

`README.md`:

- In the `p:<N>` paragraph (`:107-128`), after the sentence about `s:<N>` winning the scheduled
  date, add: a rolled `p:<N>` date also writes a `🗓️ **SCHEDULE LOG**` child bullet with one dated
  `🎲 …` entry, matching the Obsidian `Ctrl+Shift+P` picker; a `p:<N> s:<N>` capture writes no entry
  because the date was not rolled. Show the three-line rendered example.
- In the JSON prose (`:326-328`), document `schedule_log` (`reason` plus the exact rendered
  `lines`), omitted when no entry was written.

`docs/projects.md`:

- In "Priority property and scheduled rolls", extend the `bob capture <text> p:<N>` paragraph
  (`:275-282`) to say the CLI writes the same schedule-log entry, always as a `P0 → <label>`
  transition since a fresh capture has no priority field, and that `p:<N> s:<N>` writes no entry.
- In "Schedule-log reason prompt", add a row to the gesture table: `bob capture <text> p:<N>` →
  `🎲 priority P0 → <to> · random in <window>`, and note capture never prompts (it has no
  interactive stage) and never writes the `🤷 no reason given` fallback, since a brand-new line can
  never already keep a log.
- Keep the file's ~79-column wrapping and re-wrap every paragraph touched.

## Verification

From the `bob-cli` workspace:

```bash
just all     # cargo fmt --check, cargo clippy --all-targets --all-features, cargo test
```

Manual smoke with `--dry-run` against the real deployed config:

```bash
BOB_PRIORITY_ROLL_SEED=1 cargo run -- capture --dry-run -- 'someday idea p:4'
BOB_PRIORITY_ROLL_SEED=1 cargo run -- capture --dry-run -f json -- 'this week p:1 @dev'
cargo run -- capture --dry-run -- 'exact date p:2 s:1'    # no SCHEDULE LOG lines
```

Byte-parity check against the plugin — the whole point of the change:

```bash
# from the bob-cli workspace, after a real capture into a scratch vault
bob capture -b /tmp/scratch-vault 'parity check p:4'
diff <(sed -n '2,3p' /tmp/scratch-vault/mac_inbox.md) \
     <(printf '\t- 🗓️ **SCHEDULE LOG**\n')   # first line only; compare the entry by eye for the date
```

Then, in Obsidian, put the cursor on that captured task and run `Ctrl+Shift+P` → `scheduled` → any
date with a typed reason. The new entry must land **above** the captured entry under the _same_
marker bullet — no second `🗓️ **SCHEDULE LOG**` appears. That is the real proof the CLI's bytes
satisfy `SCHEDULE_LOG_PARENT_RE` and `SCHEDULE_LOG_ENTRY_RE`. If a second marker appears, the
emphasis character or the marker text is wrong.

## Risks and edge cases

- **The variation selector.** `🗓️` is two codepoints. Dropping `U+FE0F` still renders in most fonts
  but fails `SCHEDULE_LOG_PARENT_RE`, orphaning the log silently. The codepoint unit test exists
  specifically for this.
- **En dash vs. em dash vs. hyphen.** The window uses `–` (`U+2013`) with no surrounding spaces; the
  reason separator uses `—` (`U+2014`) with spaces. They are visually close and trivially wrong.
- **Every `p:<N>` capture is now three lines.** That is the request, and it matches the picker.
  There is deliberately no `--no-schedule-log` opt-out: the escape hatch is `p:<N> s:<N>`, or
  omitting `p:`. If Bryan later wants a flag, file it as a task bead rather than pre-building it.
- **Section and Tasks-block placement.** `insert_task_line` and the section inserters already
  receive a multi-line `capture_block` for clipboard captures, so a longer block is not a new code
  path — but a routed bullet capture into a section with `p:` is worth one integration test to
  confirm placement is unchanged.
- **The plugin writes `[priority:: lowest]`; capture writes `[priority::lowest]`.** Pre-existing,
  intentional divergence in capture's field style. Do not "fix" it here — it is out of scope and
  would churn many tests.
- **`min_days == 0` in a future config.** The rolled date could equal today. The plugin's
  `unchanged-date` guard cannot apply: a brand-new capture has no previous date, so the entry is
  still correct and is still written.
- **Sub-bullet captures put the log three levels deep**, under the captured child rather than under
  the parent task. That is correct — the child is what carries the priority — but it means
  `Ctrl+Shift+P` on the _parent_ will not find it. Expected; mention it in neither doc unless it
  proves confusing in practice.

## Out of scope (file as task beads if wanted)

- Any `bob-plugins` change. The picker is the reference implementation here.
- Teaching `bob capture` to _append_ to an existing task's log (that is the picker's job, and
  `@route^block-id` sub-bullet captures already have a hand-written escape hatch).
- A `--reason` flag for `bob capture` to write a human reason instead of the `🎲` auto reason.
- Surfacing the P-level in the Hammerspoon capture banner.
