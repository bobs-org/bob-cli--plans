---
tier: tale
title: Add a p:<N> priority token to bob capture
goal:
  "`bob capture` accepts a terminal `p:<N>` token (N = 1-4, the picker's P-level rows) that writes `[priority::<value>]`
  and rolls `[scheduled::YYYY-MM-DD]` inside that level's configured day window, reading the same
  `~/.config/bob/config.yml` levels the Obsidian picker reads, composing with `s:<N>`, `%` clipboard markers, and every
  route kind, with the Hammerspoon capture panel treating `p:<N>` as a terminal marker exactly like `s:<N>`."
proposed_by: bbugyi200.athena.t1.f0
create_time: 2026-08-05 14:34:06
status: wip
---

# Plan: Add a `p:<N>` priority token to `bob capture`

## Repositories

- Primary `bob-cli` checkout: all Rust, README, and `docs/` changes.
- `sase repo open chezmoi -r "<reason>"` — owns `home/dot_hammerspoon/task_capture.lua`,
  `tests/hammerspoon/task_capture_spec.lua`, and `home/dot_config/bob/config.yml` (read-only reference here; the config
  file itself does **not** change). Per that repo's `CLAUDE.md`, run `chezmoi update -a --force` after committing there.
- No `bob-plugins` change. The Obsidian picker already owns the interactive path; this plan only teaches the CLI to
  write the same fields.

## Problem

`Ctrl+Shift+P` in Obsidian can set a P-level (P1-P4) that writes `[priority:: <value>]` and rolls a random `scheduled`
date inside that level's window. `bob capture` cannot do this at all: its only date affordance is `s:<N>`, an exact
offset in days. Capturing "someday, low priority" from the CLI or the Hammerspoon panel today means capturing, then
opening the note in Obsidian and running the picker.

The deployed levels (`~/.config/bob/config.yml`, chezmoi-managed, read by `bob-navigation-hotkeys`) are:

| Picker row | `label` | `value`  | `min_days` | `max_days` |
| ---------- | ------- | -------- | ---------- | ---------- |
| 1          | `P1`    | `high`   | 2          | 7          |
| 2          | `P2`    | `medium` | 8          | 30         |
| 3          | `P3`    | `low`    | 31         | 90         |
| 4          | `P4`    | `lowest` | 91         | 365        |

A task with no `priority` field is implicit P0 (do it now, no roll), so there is no `p:0`.

## Design decisions

**1. Read the levels from `~/.config/bob/config.yml`; do not hard-code them.** The picker and the CLI must agree on
labels, written values, and day windows, and Bryan just renumbered those levels once. A hard-coded Rust table would
silently diverge the next time `config.yml` changes — `p:2` writing `medium` while the picker's P2 means something else
is a wrong-data bug with no error. `serde_yaml` is already a dependency, the file is small, and it is _bob's own_ config
file (`bob/config.yml`), so the CLI reading it is natural rather than a new coupling. The config is loaded **lazily**: a
capture without a `p:` token never touches it, so every existing capture path is byte-for-byte unchanged even on a
machine with no config file.

**2. `p:<N>` selects the Nth configured level (1-based), not the label `P<N>`.** N is "the Nth row of the picker", which
is what the user sees, and the accepted range is derived from `levels.len()` — adding a fifth level to `config.yml`
makes `p:5` work with no code change. Index selection also survives a label rename. The error message names the
configured labels so the mapping stays discoverable (`use p:1 through p:4 (P1, P2, P3, P4)`).

**3. A digit-shaped `p:<N>` in the terminal region that names no configured level is a usage error, not literal text.**
`s:<N>` keeps malformed tokens literal because every non-negative offset is valid, so there is nothing to reject. For
`p:`, `p:0` and `p:9` are unambiguously an attempt to set a priority; capturing a task whose body literally contains
"p:9" and no priority field is a silent wrong result. This matches the marker families that already validate and fail
(`@route:` Pomodoro markers, standalone `#...` markers). Non-digit `p:` tokens (`p:`, `p:soon`, `p:1.5`, `P:1`) and any
`p:` token outside the terminal marker region stay literal body text, exactly like `s:`. A digit run that overflows
`u64` also stays literal, matching `parse_schedule_token`'s existing overflow behavior.

**4. An explicit `s:<N>` wins the date; `p:<N>` still writes the priority.** `bob capture pay taxes p:2 s:1` means
"medium priority, but do it tomorrow" — a coherent request that should not error, and the explicit offset must beat the
random roll. When `s:` is present, the roll is skipped entirely (no wasted randomness, and `--dry-run` matches the real
run). Both tokens are distinct marker kinds, so the existing extractor already collects both in any order.

**5. The rolled date always lands in `[scheduled::...]`, and the config's `schedules:` key is validated against that.**
Capture's scheduled property is hard-coded end to end (the `s:<N>` token, the `scheduled` JSON field, the Blocked
reconciliation in `bob task-status-hooks`). Rather than make capture generic over date property names — which would fork
the JSON contract — it requires the priority property's `schedules` to be `scheduled` and fails with a clear message
otherwise. Silent divergence from the picker becomes a loud, one-line error.

**6. Capture still writes `- [ ] ...`; `bob task-status-hooks` owns the Blocked marker.** The picker marks a
future-scheduled task `[?]` inside its own editor transaction, but `bob capture s:5` already writes `[ ]` with a future
date and lets `task-status-hooks` reconcile `[?]` from task-level `[scheduled:: ]`. `p:<N>` follows the existing CLI
policy; do not add status logic to capture.

**7. `p:<N>` composes with every capture kind, like `s:<N>`.** Task, bullet, sub-bullet, and Pomodoro lines all render
the priority field. Rendering `[priority:: ]` on a non-task bullet is meaningless to Obsidian Tasks, but that is already
true of `[scheduled:: ]` on a bullet and README documents it as deliberate consistency (`README.md:244-246`). The
alternative — rejecting `p:` for bullets — adds a kind-specific error for no benefit.

**8. The priority field is rendered immediately after `[created::...]` and before `[scheduled::...]`.** Both keys are
recognized by every parser in play, so ordering is free — but Tasks-format parsers read trailing fields right to left
and stop at the first unrecognized key, so putting priority to the _left_ of scheduled means a hypothetical
misconfigured property name can only hide `created`, never `scheduled`. Block IDs stay last on Pomodoro lines.

**9. Randomness: a seeded splitmix64 mix, with `BOB_PRIORITY_ROLL_SEED` for determinism.** The crate has no `rand`
dependency and does not need one for a single 275-value draw. The seed comes from `BOB_PRIORITY_ROLL_SEED` when it
parses as a `u64`, otherwise from `SystemTime` nanoseconds since the epoch mixed with `std::process::id()` — the same
process-id ingredient capture already uses for temp-file names (`capture.rs:1234-1237`). Modulo bias over a span of at
most 366 against a mixed 64-bit value is ~2^-56 and irrelevant. The env override exists so integration tests can pin
exact dates and so a `--dry-run` preview can be reproduced.

**10. No `--priority` flag.** The request is token syntax, the Hammerspoon panel passes text through verbatim, and
`p:<N>` is unambiguous (unlike `%`, which needed `--clip`/`--no-clip`). Adding an option would also pull in the
`sase/memory/cli_rules.md` option conventions for no user-visible gain. Out of scope, not blocked.

## Implementation

### 1. New module `src/native/config.rs`

Declare it in `src/native.rs` between `mod collect_done;` and `mod dataview;` (the list is alphabetical).

Path resolution mirrors the plugin (`main.js:333-350`), plus a `BOB_CONFIG_FILE` override that follows the crate's
env-override convention (`env.rs` for the `*_DIR` paths, `pomodoro.rs:208` for the `BOB_DAY_FILE` exact-file naming).
Keep the resolver pure so it is unit-testable without mutating process env:

```rust
const CONFIG_RELATIVE_PATH: &str = "bob/config.yml";

pub(crate) fn config_path() -> PathBuf {
    resolve_config_path(
        env::var_os("BOB_CONFIG_FILE"),
        env::var_os("XDG_CONFIG_HOME"),
        bob_env::home_dir(),
    )
}

fn resolve_config_path(
    config_file: Option<OsString>,
    xdg_config_home: Option<OsString>,
    home: PathBuf,
) -> PathBuf { /* first non-empty wins; expand_tilde both env values */ }
```

Public surface for capture:

```rust
pub(crate) struct PriorityProperty { name: String, levels: Vec<PriorityLevel> }
pub(crate) struct PriorityLevel { label: String, value: String, min_days: u64, max_days: u64 }

pub(crate) enum ConfigError { Read(String), Invalid(String) }

pub(crate) fn load_priority_property(path: &Path) -> Result<PriorityProperty, ConfigError>;
fn parse_priority_property(text: &str, path: &Path) -> Result<PriorityProperty, ConfigError>; // unit-tested directly

impl PriorityProperty {
    pub(crate) fn name(&self) -> &str;
    pub(crate) fn level(&self, number: u64) -> Option<&PriorityLevel>; // 1-based
    pub(crate) fn level_count(&self) -> usize;
    pub(crate) fn labels(&self) -> String;                             // "P1, P2, P3, P4"
}

impl PriorityLevel {
    pub(crate) fn label(&self) -> &str;
    pub(crate) fn value(&self) -> &str;
    pub(crate) fn roll_offset(&self, seed: u64) -> u64;                // inclusive [min_days, max_days]
}

pub(crate) fn roll_seed() -> u64;
```

Deserialize loosely so an unrelated property shape never breaks the priority lookup, then validate explicitly. Every
field is optional at the serde layer and every rejection produces its own message:

```rust
#[derive(Deserialize)]
struct RawConfig { #[serde(default)] properties: Vec<RawProperty> }

#[derive(Deserialize)]
struct RawProperty {
    #[serde(default)] name: Option<String>,
    #[serde(default)] values: Option<serde_yaml::Value>,   // "date" | "priority" | "local_task_id" | sequence
    #[serde(default)] schedules: Option<String>,
    #[serde(default)] levels: Option<Vec<RawLevel>>,
}

#[derive(Deserialize)]
struct RawLevel {
    #[serde(default)] label: Option<String>,
    #[serde(default)] value: Option<serde_yaml::Value>,    // scalar; may be a YAML number/bool
    #[serde(default)] min_days: Option<i64>,               // i64 so negatives get a real message
    #[serde(default)] max_days: Option<i64>,
}
```

Selection and validation, mirroring `normalizeBulletPriorityLevel()` (`main.js:405-470`) so the CLI accepts exactly what
the picker accepts:

- Pick the first property whose `values` is the string `priority`. None → `Invalid`:
  `no priority property is configured in {path}`.
- `schedules` must be exactly `scheduled` → otherwise `Invalid`:
  `priority property "{name}" in {path} must schedule "scheduled"` (append `; it schedules "{other}"` when present).
- `levels` must be non-empty → `priority property "{name}" in {path} configures no levels`.
- Per level: non-empty trimmed `label`; scalar `value` that trims non-empty and contains none of `[`, `]`, `::`, `\n`;
  `min_days`/`max_days` present, non-negative, `min_days <= max_days`. Messages carry the level's label when it parsed
  and `#<1-based index>` when it did not.
- `value` is **not** checked against Obsidian Tasks' priority names. The plugin does not check either (it documents the
  constraint in the config comment); duplicating a whitelist here would reject a config the picker accepts.

Read errors: `io::ErrorKind::NotFound` → `Read` with the actionable hint
`p:<N> needs {path}; run 'chezmoi apply ~/.config/bob/config.yml'`. Any other IO error → `Read`: `read {path}: {error}`.

Roll:

```rust
pub(crate) fn roll_offset(&self, seed: u64) -> u64 {
    let span = self.max_days - self.min_days + 1;      // validated min <= max
    self.min_days + mix64(seed) % span
}

fn mix64(value: u64) -> u64 { /* splitmix64 finalizer: add 0x9E3779B97F4A7C15, xor-shift-multiply twice */ }

pub(crate) fn roll_seed() -> u64 {
    // BOB_PRIORITY_ROLL_SEED when it parses as u64 (trimmed); otherwise
    // UNIX_EPOCH nanos rotated against process::id(). Unparseable values are ignored, like BOB_NOW.
}
```

### 2. `src/native/capture.rs`

- **Token parser**, next to `parse_schedule_token()` (`:1655-1663`), with a doc comment in the same voice:

  ```rust
  /// Parse one whitespace-free token as a priority level (`p:<N>`), returning the
  /// 1-based level number. Non-digit or overflowing tokens stay literal.
  fn parse_priority_token(token: &str) -> Option<u64> { /* strip_prefix("p:"), all ASCII digits, parse::<u64>() */ }
  ```

- **Marker extraction** (`:1682-1767`): add `priority_level: Option<u64>` to `TerminalMarkers` (`:1335-1339`) and to
  `ParsedCaptureText` (`:1326-1333`); thread it through `RouteToken::into_parsed()` (`:1352-1366`) and the four
  `ParsedCaptureText` construction sites in `parse_capture_text_with_clip_control()` (`:1409`, `:1424`, `:1459`, plus
  the `into_parsed` calls). Inside `extract_terminal_markers()`, add `parse_priority_token(token).is_some()` to the
  `marker_like` closure — this is what lets the route scan look _past_ a `p:` token — and add the priority arm to
  `extract_terminal_marker()` with the same duplicate rule as the other kinds (a second `p:` token stops the scan and
  stays literal, so `p:1 p:2` yields level 2 with `p:1` left in the body, mirroring `s:1 s:2` at `:2582-2584`).
  `parse_clip_markers` does not gate priority; `--no-clip` and `--clip` must not change `p:` behavior.

- **Line formatters** (`:616-658`): add a `priority: Option<&str>` parameter after `created` (after `body` for
  `format_sub_bullet_line`) and a helper beside `append_scheduled_property()`:

  ```rust
  fn append_priority_property(line: &mut String, property: &str, value: Option<&str>) {
      if let Some(value) = value {
          line.push_str(&format!(" [{property}::{value}]"));
      }
  }
  ```

  The key name comes from the config property's `name` (`priority` today), and the rendered spacing matches capture's
  existing style: `[priority::high]`, no space after `::`, like `[created::...]` and `[scheduled::...]`. Priority is
  appended before the scheduled property in all four formatters; `^block-id` stays last for Pomodoro lines. Update the
  eight existing formatter call sites in the unit tests (`:3137-3160`, `:3730-3745`).

- **`capture()`** (`:407-430`): after `created` and before the `capture_line` match, resolve the priority once:

  ```rust
  let priority = match parsed.priority_level {
      Some(number) => Some(resolve_priority(number)?),   // (field name, value, Option<rolled offset>)
      None => None,
  };
  ```

  `resolve_priority()` loads `config::load_priority_property(&config::config_path())`, maps `ConfigError::Read` to
  `CaptureError::io` and `ConfigError::Invalid` to `CaptureError::usage`, then looks up the level. An unconfigured
  number is a usage error: `p:9 is not a configured priority level; use p:1 through p:4 (P1, P2, P3, P4)` (both bounds
  and the label list come from the loaded property). The rolled offset is computed only when
  `parsed.scheduled_offset.is_none()`, and the existing `scheduled_date_string(today, offset)` turns it into the date so
  the overflow guard is reused.

- **`CaptureResult`** (`:2330-2363`): add, next to `scheduled`,

  ```rust
  #[serde(skip_serializing_if = "Option::is_none")]
  priority: Option<String>,        // written value, e.g. "high"
  #[serde(skip_serializing_if = "Option::is_none")]
  priority_label: Option<String>, // configured label, e.g. "P1"
  ```

  Non-priority captures serialize exactly as before. Human output is unchanged on purpose: the echoed task line already
  shows `[priority::high]`, and there is nothing a second line would add.

- **Help text** (`:66-124`): add a paragraph after the `s:<N>` paragraph (`:72-76`):

  > Append a trailing lowercase `p:<N>` token, where N selects the Nth priority level configured in
  > `~/.config/bob/config.yml` (1-4 today: P1-P4), to write `[priority::<value>]` and roll a random
  > `[scheduled::YYYY-MM-DD]` inside that level's day window. A task with no priority field is implicitly P0, so there
  > is no `p:0`. An explicit `s:<N>` wins the scheduled date and `p:<N>` still writes the priority. Like `s:<N>` it is
  > recognized only in the terminal token region and may appear on either side of a trailing @route token.

  Also extend the `s:<N>`/`%` composition sentence (`:82`) to name `p:<N>`, add two `after_help` examples
  (`bob capture buy milk p:2` and `bob capture research rust p:4 @dev`) in the existing example ordering, and add the
  new environment entries to the alphabetically sorted `Environment:` block: `BOB_CONFIG_FILE` (exact bullet-property
  config file; defaults to `$XDG_CONFIG_HOME/bob/config.yml` or `~/.config/bob/config.yml`), `BOB_PRIORITY_ROLL_SEED`
  (fixed seed for `p:<N>` rolls; unset means random), and `XDG_CONFIG_HOME` last. `sase/memory/cli_rules.md` requires
  the help to stay scannable and the lists sorted; no new options means no other `cli_rules` obligations.

### 3. Tests

**`src/native/config.rs` unit tests** (YAML strings, no filesystem):

- The deployed config text parses to four levels with the exact labels, values, and windows from the table above.
- Each validation failure produces its `Invalid` message: no priority property; `schedules: due`; missing `schedules`;
  empty `levels`; blank `label`; missing/blank `value`; a `value` containing `::`; negative `min_days`;
  `min_days > max_days`; a non-integer `min_days`.
- A config whose _other_ properties are unusual (a `values:` sequence, an extra unknown key, a property with no `name`)
  still resolves the priority property — this is what the loose serde layer buys.
- `resolve_config_path()`: `BOB_CONFIG_FILE` wins; `XDG_CONFIG_HOME` is next; `~` expands; the fallback is
  `<home>/.config/bob/config.yml`; empty env values are ignored.
- `roll_offset()`: for every level, ~10k seeds all land in `[min_days, max_days]`; a span-6 level produces all six
  offsets across those seeds (uniformity smoke); a `min_days == max_days` level always returns that value; the P4 window
  hits both 91 and 365 for some seed.
- `roll_seed()` returns the parsed `BOB_PRIORITY_ROLL_SEED` — assert this only if it can be done without mutating
  process env from a test thread; otherwise cover it through the integration tests below and keep the unit tests pure.

**`src/native/capture.rs` unit tests** (extend the existing tables rather than adding parallel ones):

- `parses_priority_tokens()` beside `parses_schedule_tokens()` (`:2545`): `p:1`/`p:4`/`p:12` parse; `p:`, `p:abc`,
  `p:-1`, `p:1.5`, `P:1`, `px:1`, and an overflowing digit run stay `None`.
- Extraction: `p:` alone, on either side of a trailing route token, mid-body (stays literal), and duplicated (`p:1 p:2`
  → level 2, body keeps `p:1`).
- Composition: add cases to `extracts_clip_and_schedule_markers_from_terminal_region()` (`:2592`) covering
  `body p:2 s:1 % @groceries` and `body @groceries %log p:3 s:4` and assert all four of body/route/scheduled/priority.
- Formatters: task, bullet, sub-bullet, and Pomodoro lines with priority only, and with priority plus scheduled, e.g.
  `- [ ] #task buy milk [created::2026-06-15] [priority::high] [scheduled::2026-06-20]` and
  `- [*] #task ship it [created::2026-06-15] [priority::lowest] [scheduled::2026-09-20] ^ship`.

**`tests/cli.rs` integration tests.** These must be hermetic: `bob_command()` inherits the real environment, so every
priority test sets `BOB_CONFIG_FILE` to a config written into its own `TempDir`. Write that fixture with a small helper
that emits the four deployed levels.

- `p:1` with `BOB_NOW=2026-06-15` and a fixed `BOB_PRIORITY_ROLL_SEED`: the inbox file and stdout contain the exact
  expected line. Derive the expected date by running the test once; also assert the date is inside
  2026-06-17..2026-06-22 so a future PRNG change fails with a meaningful diff instead of only an opaque string mismatch.
- `p:4` with the same seed: `[priority::lowest]` and a date inside 91-365 days.
- `p:2 s:1`: `[priority::medium] [scheduled::2026-06-16]`; the roll is not used.
- `p:9` and `p:0`: exit code 2, stderr names the configured range and labels, and no file is created.
- `BOB_CONFIG_FILE` pointing at a missing path with `p:2`: exit code 1, the message names the path and the
  `chezmoi apply` hint, and the vault is untouched.
- `-f json` with `p:3`: `priority` is `"low"`, `priority_label` is `"P3"`, `scheduled` is the rolled date; a capture
  without `p:` omits both keys.
- One routed/Pomodoro or sub-bullet case proving the field renders for another kind with the block ID still last.
- A no-`p:` capture with `BOB_CONFIG_FILE` pointing at a **nonexistent** path succeeds — the lazy-load guarantee.

### 4. chezmoi — Hammerspoon capture panel

`home/dot_hammerspoon/task_capture.lua` scans the terminal token region so interactive `@` markers survive alongside `%`
and `s:<N>` tokens. Without a `p:` arm, `Task @ p:2` stops the scan at `p:2`, the panel never sees the interactive `@`,
and the capture silently lands with a literal `@` in its body — `s:2` in the same position works today. Two small edits
restore parity:

- `terminal_marker_kind()` (`:35-58`): before the clip branch, return `"priority"` for `token:match("^p:(%d+)$")` when
  the digits fit `u64` (reuse `unsigned_integer_fits`, as the schedule arm does). Deliberately permissive about the
  numeric range: range checking belongs to `bob capture`, which errors clearly, whereas rejecting `p:9` here would
  silently drop the interactive marker.
- `split_interactive_terminal_token()` (`:81-111`): add a `seen_priority` flag with the same duplicate-stops-the-scan
  rule as `seen_clip`/`seen_schedule`.

`tests/hammerspoon/task_capture_spec.lua`: extend the crossed-marker test (`:271-305`) with `p:` cases in both orders
around `@Notes#`, `@:focus-123`, and `@Dev^`; add `p:2` to a canonical-synthesis retention case (`:352`, `:374`) so the
token survives `finalize`; and add `"Idea @Notes# p:1 p:2"` and `"Idea @Notes# p:18446744073709551616"` to the
leave-it-to-bob list (`:379-392`). Update the marker comment block in `init.lua:1172-1176` to name `p:<N>`.

The panel's capture notification (`init.lua:612-645`) shows `decoded.text` and the kind, not the rendered line, so it
will not display the level. Surfacing `P4` in the banner is a separate, optional polish item — out of scope here.

### 5. Docs

`README.md`:

- After the `s:<N>` paragraph (`:102-105`), a `p:<N>` paragraph: the token, the level table, where the levels come from
  (`BOB_CONFIG_FILE`, else `$XDG_CONFIG_HOME/bob/config.yml`, else `~/.config/bob/config.yml` — the same file the
  Obsidian picker reads), implicit P0, `s:<N>` precedence, that each capture rolls independently (so a `--dry-run`
  preview differs from the real capture unless `BOB_PRIORITY_ROLL_SEED` is set), that an out-of-range `p:<N>` fails
  instead of staying literal, and that `bob task-status-hooks` is what later marks a future-scheduled task Blocked.
- Add `p:<N>` to the composition sentences at `:120` and `:244`.
- Add `priority` and `priority_label` to the JSON field prose (`:295-301`).
- Add `p:<N>` to the Hammerspoon terminal-marker sentence (`:334`).

`docs/projects.md`, in "Priority property and scheduled rolls" (`:225-268`): after the `Choosing P1, P2, P3, or P4`
paragraph (`:251`), add a short paragraph that `bob capture <text> p:<N>` writes the same `[priority:: ...]` field and
rolls the same window from the CLI, that N is the picker row, and that capture leaves the `[ ]` marker for
`bob task-status-hooks` instead of marking Blocked itself. Keep the file's ~79-column wrapping and re-wrap any paragraph
touched.

## Verification

From the `bob-cli` workspace:

```bash
just all     # cargo fmt --check, cargo clippy --all-targets --all-features, cargo test
```

Manual smoke from the workspace, against the real deployed config, using `--dry-run` so nothing is written:

```bash
cargo run -- capture --dry-run -f json -- 'someday idea p:4'
cargo run -- capture --dry-run -f json -- 'this week p:1 @dev'
cargo run -- capture --dry-run -- 'bad level p:9'   # exit 2, names P1, P2, P3, P4
BOB_PRIORITY_ROLL_SEED=1 cargo run -- capture --dry-run -- 'seeded p:2'   # repeat: identical date
```

From the `chezmoi` checkout:

```bash
stylua --check ./home/dot_hammerspoon ./tests/hammerspoon
just test-hammerspoon    # needs busted
```

`busted` is **not** installed on athena (`lua` and `luarocks` are). Try `luarocks install --local busted` and run
`~/.luarocks/bin/busted ./tests/hammerspoon`. If the install fails, say so explicitly in the final report and mark the
Lua suite unrun rather than implying it passed — do not claim a green Lua suite you did not observe.

Deploy after committing, per the chezmoi repo's `CLAUDE.md`:

```bash
chezmoi update -a --force     # from the chezmoi checkout
```

`bob` itself must be rebuilt/reinstalled from the committed `bob-cli` master before the token works outside the
workspace; that install is Bryan's call, not part of this plan.

Then hand-check in a scratch note (not a real one):

1. `bob capture -b /tmp/scratch-vault 'try p:4'` writes `[priority::lowest]` plus a date 91-365 days out.
2. `Ctrl+Shift+P` on that task in Obsidian reports `current: P4` — proof the CLI and picker agree on the value.
3. In the Hammerspoon panel, `try p:2 @` opens the note picker and the captured line carries `[priority::medium]`.

## Risks and edge cases

- **A stale config on the mac.** `p:4` fails with a clear range error if `~/.config/bob/config.yml` there predates the
  P4 level. Fix is `chezmoi update`, not a code change; the error message already points at `chezmoi apply`.
- **New failure mode for `p:` only.** Every non-`p:` capture path must keep working with a missing or malformed config.
  The lazy-load integration test above is the guard; do not hoist the load out of the `Some(number)` branch.
- **Body text that looks like a marker.** `p:` plus digits at the very end of the input is now consumed. "call p:mom"
  and "p:2 is the fix" (mid-body) stay literal, so realistic collisions need a trailing bare `p:<digits>`. Accepted, and
  called out in the README paragraph.
- **`p:0` is an error, not "clear priority".** Implicit P0 is expressed by omitting the token; a token that writes
  nothing would be indistinguishable from omitting it and would invite "why did nothing happen?".
- **Random dates can land on a weekend.** Same as the picker's roll. Intentional; no weekday weighting.
- **`--dry-run` previews a different date than the real capture.** Inherent to rolling per capture; documented, and
  `BOB_PRIORITY_ROLL_SEED` exists for anyone who needs reproducibility.
- **Sorting is unchanged.** Obsidian Tasks sorts a no-priority (P0) task as `Normal`, between P2 and P3 in
  `sort by priority`. Pre-existing behavior from the picker's numbering; out of scope here.
- **Do not touch `home/dot_config/bob/config.yml`.** This plan consumes the existing shape; no config change is needed,
  and editing it would desync the picker's tests in `bob-plugins`.
