---
tier: epic
title: "Close the day, tag the week: plan budget, #now, and ledger guardrails"
goal: "Today's Pomodoro plan is a visible, capped, closed list (GTD + 3 themes, about 10
  Task Links) and this week's bets live in a `#now` tag. One shared budget definition
  shows the same numbers in `bob plan`, task-status-hooks, tmux, `bob capture`, Bob Mac
  Capture, and Obsidian. New gestures let Bryan drop, defer, and tag work from the
  ledger itself, and nothing rewrites the plan behind Bryan's back.

  "
phases:
  - id: plan-core
    title: "bob-cli: shared plan-budget core, config block, and `bob plan`"
    depends_on: []
    size: medium
    description: "plan-core: add the `plan:` config block, a pure ledger budget and lint
      engine, a NOW counter built on the native Tasks engine, the read-only `bob plan`
      command (human and JSON), and `docs/plan.md` as the authoritative definition with
      conformance examples.

      "
  - id: vault-now
    title: "Vault: NOW chip, NOW section, and the gtd_daily chore swap"
    depends_on: []
    size: small
    description: "vault-now: add a NOW chip and a `### NOW Tasks` section to `dash.md`,
      and replace the three migrate/review chores in `gtd_daily.md` with one daily pick
      and one weekly review, so the trial can start without waiting for code.

      "
  - id: hooks-tmux
    title: "bob-cli: plan budget in task-status-hooks and the tmux segment"
    depends_on:
      - plan-core
    size: small
    description: "hooks-tmux: add a read-only `plan_budget` to task-status-hooks JSON
      and human output, make the multiple-open-timed error name the entries and suggest
      `=x`, and append the budget meter (reversed when over the cap) to `bob
      tmux-pomodoro`.

      "
  - id: capture-budget
    title:
      "bob-cli: capture plan-budget warnings, strict mode, and implicit destination"
    depends_on:
      - plan-core
    size: medium
    description: "capture-budget: `bob capture` reports before/after `plan_budget` when
      a batch changes today's ledger, warns only when it grows past a cap, refuses new
      non-start themes past the cap in strict mode, reports where a Task Link lands
      (`role`: current/next_up/named/created), and marks capture-complete create rows
      with the resulting theme count.

      "
  - id: close-drop
    title: "bob-cli: `~<K>` drop outcome for `=x` closes"
    depends_on:
      - plan-core
      - capture-budget
    size: medium
    description: "close-drop: extend the close grammar to `=x[<N>][!<M>][~<K>]`, where
      dropped links are removed from the closed session, not carried, and not started;
      add editor spans, JSON (`drop`, outcome and role `dropped`, and a `now` flag on
      close task rows), human output, help, and docs.

      "
  - id: now-token
    title: "bob-cli: first-class `#now` in capture"
    depends_on:
      - close-drop
    size: medium
    description: "now-token: accept `#now` after the route marker, color it with a
      `now_tag` span, complete a partial `#n`/`#no` to `#now`, and list Ready `#now`
      tasks (flagged `now`) in the `^` active-task picker.

      "
  - id: ledger-plan-view
    title: "bob-plugins: Bob Ledger Tools plan view, `bob-plan` block, and public API"
    depends_on:
      - plan-core
    size: medium
    description: "ledger-plan-view: mirror the plan-budget definition in
      bob-ledger-tools; render a live ```` ```bob-plan ```` block (PLAN and NOW chips,
      today's themes with the highlight starred, and lint lines); and expose a versioned
      `api` (caps, planBudget, nowBudget) for the dash and the other plugins.

      "
  - id: link-picker
    title: "bob-plugins: Ctrl+Shift+P edits the task behind a Task Link"
    depends_on: []
    size: medium
    description: "link-picker: on a dedicated Task Link bullet, the bullet-property
      picker targets the linked task in its own note. Counted `N<Ctrl+Shift+P>` covers
      the next N sibling links. Future-date picks still prune those links from today's
      open Pomodoros.

      "
  - id: now-toggle
    title: "bob-plugins: toggle #now from task lines and Task Links"
    depends_on:
      - link-picker
      - ledger-plan-view
    size: medium
    description: 'now-toggle: add a counted "Toggle #now" command (default Alt+N) and a
      `#now` row in the Ctrl+Shift+P picker. Both work on task lines and Task Link
      lines, place the tag before the fields, and report the NOW count in the Notice.

      '
  - id: link-notice-budget
    title: "bob-plugins: plan budget in the Ctrl+Shift+Enter Notice"
    depends_on:
      - ledger-plan-view
    size: small
    description: "link-notice-budget: append `plan T/3 · L/10`, marked 🔴 when over, to
      block-id-prompt's link, unlink, and Task Link Notices, using the ledger-tools API
      on the post-write daily content. It warns and never refuses.

      "
  - id: mac-budget
    title:
      "Bob Mac Capture: plan budget meter, destination row, and create-row cap badge"
    depends_on:
      - capture-budget
    size: medium
    description: 'mac-budget: decode `plan_budget`, the destination `role`, the
      create-row theme counts, and the failure `code` tolerantly. Render a destination
      row, a themes/links meter with delta chips and warnings, and a cap badge on
      "create Pomodoro" completion rows. Use real-bob fixtures and green macOS CI.

      '
  - id: mac-close-now
    title: "Bob Mac Capture: drop outcome, `#now` token, and NOW badges"
    depends_on:
      - mac-budget
      - close-drop
      - now-token
    size: medium
    description: "mac-close-now: render the `dropped` close outcome and the `~` span,
      the `now_tag` span color and completion row, and NOW badges on active-task and
      close rows, all decoded tolerantly and verified on macOS CI.

      "
  - id: rollout
    title:
      "Rollout: PLAN chip, daily template block, config knobs, install, and end-to-end
      check"
    depends_on:
      - plan-core
      - vault-now
      - hooks-tmux
      - now-token
      - ledger-plan-view
      - now-toggle
      - link-notice-budget
      - mac-close-now
    size: small
    description:
      "rollout: switch the dash NOW chip to the API and add the PLAN chip; add the
      `bob-plan` block to the daily template and to today's note; add the commented
      `plan:` block to the chezmoi-managed config; reinstall `bob` on apollo and sync
      the plugins; run an end-to-end check; hand Bryan the manual checklist."
proposed_by: bbugyi200.apollo.38
create_time: 2026-09-29 18:09:55
status: wip
---

- **PROMPT:**
  [prompts/202609/pomodoro_plan_budget_now_tag.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/pomodoro_plan_budget_now_tag.md)

# Plan: Close the day, tag the week — plan budget, `#now`, and ledger guardrails

## Context

This epic implements the recommendations of the research report
`research:202609/pomodoro_closed_day_now_tag_automation/pomodoro_closed_day_now_tag_automation.md`
("the report"), plus its follow-up `research:202609/now_tag_vs_in_progress_status.md`.
Read both with `sase artifact read` before starting any phase.

- **Phase 0 of the report.** This epic implements the tool-side parts: views, dash,
  template, and chores.
- **Phase 1 of the report.** This epic implements all of it, with one deliberate
  deferral: `stale_link` (see "Deliberately not doing").
- **Phase 2 of the report.** Out of scope. The report makes it conditional on the
  two-week trial.

Three repos plus the vault are involved:

| Surface          | Where                                            | How to open                                                                                                                                                                           |
| ---------------- | ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bob` CLI        | this repo (`bob-cli`)                            | your workspace                                                                                                                                                                        |
| Obsidian plugins | `bob-plugins`                                    | `sase repo open bob-plugins -r "<why>"`. If that fails because the linked primary checkout is missing, use `sase repo open gh:bobs-org/bob-plugins -r "<why>"`. Read its `AGENTS.md`. |
| Mac capture app  | `bob-mac-capture`                                | `sase repo open bob-mac-capture -r "<why>"`, falling back to `gh:bobs-org/bob-mac-capture`                                                                                            |
| Vault            | `~/bob` (git-synced by `bob-vault-sync.service`) | edit in place; never commit the vault by hand                                                                                                                                         |
| Config           | chezmoi source for `~/.config/bob/config.yml`    | `sase repo open chezmoi -r "<why>"`                                                                                                                                                   |

## Design

### The rule (one index card)

> **Today is closed:** GTD plus at most 3 themes. The first theme is the highlight.
> Nothing is copied forward by default. **This week is `#now`:** at most 15 tasks.
> Dropping a link from today never loses it, because `#now` keeps it in view.
> **Everything else is READY (next) or deferred with a P-level (later).**

The tools make this rule **visible** at every place Bryan already looks, and **cheap to
follow** with a few new gestures. They never rewrite the plan: nothing is pruned,
deferred, or migrated by cron, and yesterday's note is never written.

### One definition, many surfaces

| Surface                          | What Bryan sees                                                                                                  | Phase                          |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------- | ------------------------------ |
| `bob plan`                       | The full plan report: meters, today's themes (★ highlight, ▶ running), and lints with hints                     | plan-core                      |
| Daily note, above `## Pomodoros` | A live ` ```bob-plan ` block: `PLAN 3/3 · 7/10`, `NOW 12/15`, then `★ GOALS · DECKS · BOB`                       | ledger-plan-view, rollout      |
| `dash.md` chip bar               | `NOW 12/15` and `PLAN 3/3 · 7/10` chips, red when over the cap; a `### NOW Tasks` section                        | vault-now, rollout             |
| tmux status line                 | `[<23m] 0945-1015 — GOALS · plan 3/3 · 7/10 \| `, reverse video when over                                        | hooks-tmux                     |
| `bob task-status-hooks`          | A `plan_budget` JSON block and one human line                                                                    | hooks-tmux                     |
| `bob capture` / Bob Mac Capture  | Where the Task Link lands ("→ GOALS · next up"), a before→after meter, cap warnings, and optional strict refusal | capture-budget, mac-budget     |
| Obsidian Notices                 | `· plan 3/3 · 11/10 🔴` on Ctrl+Shift+Enter; `#now added · NOW 13/15` on the toggle                              | link-notice-budget, now-toggle |

**One definition.** `docs/plan.md` in bob-cli, written in plan-core, is the
authoritative definition. The Rust engine implements it. bob-ledger-tools mirrors it in
JavaScript, and every other Obsidian surface calls ledger-tools' public API instead of
re-implementing it. Both implementations test against the same conformance examples.

### The plan budget (authoritative rules)

1. **Ledger.** The lines of today's daily note (`YYYY/YYYYMMDD.md`) from a
   `## Pomodoros…` heading up to the next `## ` heading. Fenced code blocks are skipped.
2. **Entries.** Column-0 checkbox list items (`- [c] …`). An entry is **open** when `c`
   is not `x`, `X` or `-`. Only open entries count; completed and cancelled entries are
   history.
3. **Names.**
   - An entry's name is the text after `—` (em dash) that follows the leading `()`
     placeholder or time range.
   - A merged name (`BOB + DECKS`) splits on `+` into **components**.
   - Components compare case-insensitively after collapsing whitespace.
4. **Exempt entries.** Components listed in `plan.exempt` (default `[GTD]`) are never
   themes. An entry whose components are all exempt is ignored for links too.
5. **Themes.** The number of distinct non-exempt components across open entries. An
   **unnamed** open entry counts as one theme, shown as `(unnamed)`, but only when it
   holds at least one counted link. An empty `()` placeholder or `\t- ` stub never
   counts.
6. **Links.** The number of distinct `(target, block_id)` pairs among block links in any
   descendant bullet of an open, non-exempt entry. Accepted forms:
   - `[[target#^id]]`, `![[target#^id]]`, and `[[target#^id|alias]]`;
   - with or without 🍅 markers;
   - with or without a trailing `#` move-only marker.

   Excluded:
   - links inside `~~…~~` (struck);
   - links in fenced code.

   The target is compared with any `.md` suffix removed. An empty target means the daily
   note itself. A link planned under two entries counts once.

7. **Highlight** is the first open non-exempt entry in ledger order. **Running** is the
   open entry whose body starts with a time range.
8. **Status** is `over` when themes > `max_themes` or links > `max_links`; otherwise it
   is `ok`. Being exactly at the cap is fine.
9. **Lints.** Each lint has a stable code, a message, and a 1-based `line` when one
   applies:
   - `plan_theme_cap_exceeded`, `plan_link_cap_exceeded`
   - `duplicate_open_pomodoro_name`: the same non-exempt component on two open entries
   - `inventory_label_open`: an open component that is in `plan.inventory_labels`. It
     still counts as a theme; the lint explains why it shouldn't be one.
   - `subheading_in_pomodoros`: a `###`–`######` heading inside the section. It splits
     the heading's time total.
   - `now_cap_exceeded`: only on surfaces that compute NOW.

### NOW (this week's bets)

A NOW task is a `#task` line that the native Tasks engine (the one behind
`bob query --tasks`) matches with this query:

```text
not done
tags include #now
is not blocked
tags do not include #hide
folder does not include _templates
path does not include _conflicts
(no scheduled date) OR (scheduled on or before today)
```

- This matches the dash's own defaults exactly, so the chip, the `### NOW Tasks` section
  and `bob plan` always agree.
- `has_now_tag(text)` means the case-sensitive whole token `#now`: it must be preceded
  by the line start or whitespace, and followed by the end or whitespace.
- `#now` is **never** a Next source. It never changes task status.

### Config (`~/.config/bob/config.yml`)

The block is optional. A missing file or a missing block means these defaults, shared by
Rust and JavaScript:

```yaml
# Today's plan budget (bob plan, capture, tmux, Obsidian) and the weekly #now cap.
plan:
  max_themes: 3 # distinct open Pomodoro names besides the exempt ones
  max_links: 10 # distinct open Task Links outside exempt entries
  max_now: 15 # open #now tasks visible today (see docs/plan.md)
  strict: false # refuse #NAME captures that would create a theme past max_themes
  exempt: [GTD] # open entries that never count as themes
  inventory_labels: [LATER, MISC, NEW FEATURES, SASE] # open names that are storage, not themes
```

- **Validation.** Caps are integers ≥ 1, and the lists hold non-empty strings.
- **Invalid values:**
  - `bob plan` exits 2 with a clear message.
  - Every other surface falls back to the defaults and says so once: as a warning
    string, a stderr line, or a Notice. An invalid plan config must never break capture,
    hooks, tmux, or a keymap.

### Visual language (shared across every surface)

- **Meters** read `count/cap`, themes first and then links, labelled `plan`/`PLAN`. NOW
  is its own meter: `NOW n/15`.
- **Colors:** green or neutral within the cap, red when over. In tmux, reverse video
  when over. In Notices, a trailing 🔴 when over.
- **Glyphs:** ★ marks the highlight theme and ▶ marks the running entry. Lint codes
  appear verbatim, dimmed, after the human message.
- **Contracts are additive only.**
  - Existing JSON keys, `schema_version: 1` values and exit codes do not change.
  - New fields are omitted when empty or false, unless a section below says otherwise.
  - Bob Mac Capture must decode every new field tolerantly and keep working against an
    older `bob`.

## Cross-cutting rules for every phase

- **Read first.** Read the report and the follow-up with `sase artifact read`.
- **New CLI surface.** Before adding CLI surface, read `sase/memory/cli_rules.md` with
  `/sase_memory_read`:
  - alphabetical subcommands and options;
  - a short alias for every long option;
  - excellent `--help`;
  - color on a TTY and none when piped or under `NO_COLOR`.
- **bob-cli checks.** `just all` (fmt, clippy, tests) must pass.
  - Keep new or touched Rust files under about 1500 lines where practical. The repo
    splits oversized modules into directory modules.
  - Update `README.md` and `docs/*.md` in the same phase that changes behavior.
- **bob-plugins checks.** `npm test` and `npm run validate` must pass.
  - Bump the touched plugin's `manifest.json` minor version and update its README row
    and description.
  - Deploy to apollo's vault with
    `bob plugins sync -n -r "<opened bob-plugins path>" -p <id>`. The default repo path
    does not exist on this host.
- **bob-mac-capture checks.** This host has no Swift toolchain, so macOS CI is the gate.
  - Follow the repo's established commit and push flow for opened repos.
  - The phase is not done until the CI run for your commit is green.
  - Regenerate every new JSON fixture from a real `bob` built from bob-cli master
    (`cargo build`), following the existing fixture conventions and the command lists in
    its tests.
- **Epic workers** record `PROPOSED FOLLOW-UP:` notes on their own phase bead instead of
  creating beads.

## Phase: plan-core — shared plan-budget core, config block, and `bob plan`

**Config.**

- Add a `plan` field to `RawConfig` in `src/native/config.rs`, plus `PlanConfig` and
  `load_plan_config(path) -> Result<PlanConfig, ConfigError>`.
- A missing file or block returns the defaults above. This is unlike
  `load_priority_property`, which errors on a missing file.
- Unknown keys stay ignored.
- Unit-test the defaults, overrides, and each invalid case.

**Engine.** Create a directory module, `src/native/plan_budget/`, with these parts:

- **`compute(contents, &PlanConfig) -> LedgerBudget`.** A pure function implementing
  rules 1–9.
  - Build on `capture_pomodoros::scan` for entries, names, state and time ranges. Add a
    link collector with struck and fence handling; reuse or align with
    `task_status_hooks::pomodoro::block_link_occurrences` and
    `capture_pomodoro_close/links.rs`.
  - `[-]` entries must be neither open nor counted.
  - The result holds:
    - per-entry rows: `line`, `name`, `components`, `exempt`, `running`, `highlight`,
      `time_range`, and the entry's distinct link count;
    - the `themes` and `links` meters (`{count, cap, over}`);
    - `status`;
    - the ordered theme names;
    - the lints.
- **`has_now_tag(text)`** and **`count_now(bob_dir, today, &PlanConfig) -> NowBudget`.**
  - `count_now` runs the NOW query through the native Tasks engine used by
    `bob query --tasks`, so it honors the vault's Tasks settings. Find that engine's
    entry point in `src/native/dataview/tasks/`.
  - It emits `now_cap_exceeded` when over the cap.
- **Serializable report structs, shared by `bob plan`, the hooks, and capture:**

  ```json
  {
    "date": "2026-09-30",
    "daily_file": "2026/20260930.md",
    "caps": { "max_themes": 3, "max_links": 10, "max_now": 15, "strict": false },
    "status": "ok",
    "themes": { "count": 3, "cap": 3, "over": false },
    "links": { "count": 7, "cap": 10, "over": false },
    "now": { "count": 12, "cap": 15, "over": false },
    "entries": [
      {
        "line": 24,
        "name": "GOALS",
        "components": ["GOALS"],
        "exempt": false,
        "running": true,
        "highlight": true,
        "time_range": "0945-1015",
        "links": 3
      }
    ],
    "warnings": [
      {
        "code": "inventory_label_open",
        "message": "MISC is an inventory label, not a theme",
        "line": 60
      }
    ]
  }
  ```

**`bob plan` command.**

- Add it to `SUBCOMMANDS` in `src/runner.rs`, in alphabetical order. The about text is
  "Show today's plan budget and this week's NOW count".
- Options: `-b/--bob-dir DIR`, `-f/--format human|json`, `-h/--help`.
- Environment: `BOB_DIR`, `BOB_DAY_FILE`, `BOB_NOW`, `BOB_CONFIG_FILE`, `NO_COLOR`.
  Document each in `--help` with examples.
- JSON is `{"ok": true, "schema_version": 1, …report…}`.
- The human output should look like this, with the NOW meter on the same line when it
  fits:

  ```text
  bob plan · Wed 2026-09-30 · 2026/20260930.md

    PLAN  3/3 themes · 7/10 links        NOW  12/15

    ★ GOALS    ▶ 0945-1015   3 links
      DECKS                  2 links
      BOB                    2 links
      GTD      exempt        1 link

    ⚠ MISC is an inventory label, not a theme (line 60)  inventory_label_open
  ```

  - Meters are green when within the cap and red when over.
  - ★ is yellow and ▶ is cyan; exempt rows are dimmed.
  - Lints are yellow, with the code dimmed.

- **No daily note or no Pomodoros section:**
  - The header says `no daily note yet` or `no Pomodoros section`.
  - NOW is still shown.
  - The command exits 0.
- **Exit codes:** 0 for a report, 1 for an I/O failure, 2 for usage or an invalid plan
  config.

**Docs.**

- Add `docs/plan.md`, covering:
  - the rules and the NOW query;
  - config and validation;
  - JSON;
  - the lint codes;
  - "Surfaces", with a forward-looking table that later phases fill in;
  - **Conformance examples**, used by ledger-plan-view as its JS test vectors. At
    minimum:
    1. merged `BOB + DECKS` next to a separate `DECKS`: counts DECKS once and raises
       `duplicate_open_pomodoro_name`;
    2. a struck link, GTD's `[[#^gtd]]`, a link planned twice, an embedded link, and a
       `#`-marked link;
    3. an unnamed placeholder with and without links;
    4. `### Notes` inside the section;
    5. an open `LATER`;
    6. four themes (over the cap);
    7. a `[-]` entry and a completed entry.
- Index it in `docs/README.md`.
- Update `README.md`: the command table, a new section, and the `BOB_CONFIG_FILE`
  environment paragraph.
- Add `bob plan --help` to the justfile's `install-smoke`.

**Tests.**

- Unit tests for every rule and every conformance example.
- A NOW fixture vault covering:
  - open `#now`;
  - done or cancelled `#now`;
  - `#hide`;
  - `_templates/`;
  - a future `scheduled` date, and a `scheduled` date of today;
  - dependency-blocked;
  - `#nowadays` and `#now/x`, which must not match.
- CLI tests for human output (no ANSI when piped) and for JSON.
- The help tests must keep passing, including the short-flag guard.

## Phase: vault-now — NOW chip, NOW section, and the gtd_daily chore swap

This phase has no code dependency, so Bryan can start the trial (2026-09-30 →
2026-10-13) right away. Edit `~/bob` in place, keeping each file's existing formatting.

1. **The `dash.md` chip bar** (the `dataviewjs` block).
   - Add `counts.now`: the active tasks that are not dependency-blocked and satisfy
     `t.tags.includes("#now")`. Use the same `active` and `dependencyBlocked` sets the
     WIP, NEXT and READY chips use, so the chip matches the NOW query.
   - Add the chip `{ key: "now", target: "dash#NOW Tasks", label: "NOW" }` first in the
     bar.
   - The value renders as `n/15` with the cap hard-coded for now; rollout switches it to
     the API.
   - Give it a purple accent (`var(--color-purple)`). Add a `task-count-over` modifier
     class that switches any chip's accent to the red used by BLOCKED when it is over
     the cap, and include the cap in the `aria-label`.
2. **The `dash.md` section.** Add `### NOW Tasks` as the first subsection under
   `## Tasks`, above `### WIP Tasks`:

   ````markdown
   ```tasks
   not done
   tags include #now
   sort by priority
   ```
   ````

   The page's `TQ_extra_instructions` supply the rest of the NOW query. Check the
   query's semantics with `bob query --tasks` against the vault.

3. **`gtd_daily.md`.**
   - Cancel the open instances of these three chores:
     - "Migrate unfinished Pomodoro tasks…";
     - "Review [[dash#READY Tasks|READY]] tasks";
     - "Review [[dash#WIP Tasks|WIP]] + [[dash#NEXT Tasks|NEXT]] … plan daily
       "Pomodoros"".

     To cancel one, set it to `[-]`, append `  [cancelled:: <today>]`, and keep its
     other fields. Cancel rather than delete so the nightly archive keeps the history.
     Leave the completed history lines alone.

   - Add two tasks in the file's style (two spaces between fields):
     - `- [ ] #task Pick today: ≤3 themes from yesterday + [[dash#NOW Tasks|NOW]] (highlight first)  [repeat:: every day when done]  [created:: <today>]  [scheduled:: <tomorrow>]`
     - `- [ ] #task Weekly review: re-tag [[dash#NOW Tasks|NOW]] to ≤15, promote from [[dash#READY Tasks|READY]], defer with P-levels, check hours and created vs closed  [repeat:: every week on Monday when done]  [created:: <today>]  [scheduled:: <next Monday>]`

4. **Sync and verify.** Run `bob vault-sync`, then confirm with
   `bob vault-sync status --json`. Do not touch yesterday's or today's daily notes.

## Phase: hooks-tmux — plan budget in task-status-hooks and the tmux segment

**task-status-hooks** (`src/native/task_status_hooks/`):

- Add a top-level `plan_budget` key to `SyncResult` JSON. It holds the plan-core report,
  including NOW. It is `null` when there is no daily note or no Pomodoros section.
- Count NOW with plan-core's function, sharing the predicate. If the hooks' own vault
  scan can supply parsed tasks cheaply, use it instead of a second walk; the semantics
  must stay identical.
- Human output adds one stats line, for example:
  `plan 3/3 themes · 7/10 links · NOW 12/15 · 2 plan warnings (run bob plan)`.
  - Over-cap meters are red.
  - Do not print individual plan lints every 15 minutes.
- The budget never changes the exit code and never writes anything.
- An invalid plan config yields `plan_budget: null` and a single stderr warning.

**Multiple open timed entries.** Keep the stable prefix "Bob daily note has multiple
open timed Pomodoros" in both the JSON `error` and the human output. Append each entry's
name (or `(unnamed)`), line and range, plus the hint: close all but one with
`bob capture -- =x`, or mark it `[x]`. Update
`tests/cli/task_status_hooks/structure.rs`.

**tmux** (`pomodoro::run_tmux`):

- Append the budget meter to the segment:
  - `<status> · plan T/Tc · L/Lc | ` when a Pomodoro status exists;
  - `plan T/Tc · L/Lc | ` when none does;
  - nothing when there is no daily note or no Pomodoros section.
- When over the cap, wrap just the meter in `#[reverse]…#[noreverse]`.
- Errors stay swallowed.
- `bob pomodoro` output is unchanged, because `notify.rs` depends on it.
- The `BOB_CLI_USE_SCRIPT` script fallback stays budget-less. Say so in
  `tmux-pomodoro --help` and in the README's "Pomodoro status" section.
- Update `tests/cli/pomodoro.rs`. Its fixture has no named entries, so it expects
  `· plan 0/3 · 0/10`.
- Add tests for a named ledger and for the over-cap reverse-video output.

**Docs.** Update `docs/task-status-hooks.md` (the JSON contract and the human output)
and `docs/plan.md` ("Surfaces").

## Phase: capture-budget — capture plan-budget warnings, strict mode, and implicit destination

**Budget in the result.**

- After planning a batch (dry-run and real use the same planner), compare today's daily
  note before and after.
- If its Pomodoros section changed, add a top-level `plan_budget` to `CaptureResult`. It
  is not per item, and it is omitted otherwise:

  ```json
  "plan_budget": {"status": "over",
    "themes": {"count": 4, "cap": 3, "over": true, "before": 3},
    "links": {"count": 8, "cap": 10, "over": false, "before": 7},
    "added_themes": ["BOB"], "warnings": [{"code": "plan_theme_cap_exceeded",
    "message": "today's plan now has 4/3 themes (adds BOB); queue it with ^, keep it this week with #now, or defer with p:<N>"}]}
  ```

- **When warnings fire.** Only `plan_theme_cap_exceeded` and `plan_link_cap_exceeded`,
  and only when `after > cap` and `after > before`. A capture that doesn't grow an
  over-cap plan stays quiet.
- **Human output.**
  - Print one meter line after the result, e.g.
    `plan 4/3 themes · 8/10 links  (+1 theme: BOB)`.
  - Print each budget warning on stderr as `bob capture: warning: …`.
  - Do not duplicate them into the existing `warnings: [String]`.
- **Invalid plan config.** Skip the budget and push one plain string into `warnings`.

**Strict mode** (`plan.strict: true`).

- Refuse the whole batch, atomically, when all of these hold:
  - an item creates a **new named Pomodoro entry without starting it**, whether via
    `<text> @r:id#NAME`, `@r^id+#NAME` project notes, `@r+id#NAME` toggles, or any other
    non-start creation path;
  - after-themes > cap;
  - after-themes > before-themes.
- Session starts (`=`, `#NAME=`, and link starts) are never refused.
- The refusal is an I/O-class error (exit 1).
  - JSON: `{"ok": false, "error": "…", "code": "plan_theme_cap_exceeded"}`. `code` is a
    new optional failure field, emitted only for this refusal.
  - The message names the current themes and gives the same `^` / `#now` / `p:<N>` hint.

**Implicit destination (C3).**

- Add an optional `role` to `PomodoroLinkEndpoint`:
  - `current`: the running timed entry;
  - `next_up`: chosen implicitly and not running;
  - `named`: an existing entry matched by `#NAME`;
  - `created`: a new entry.
- Fill `role` wherever a `pomodoro_link_destination` is produced.
- New `<text> @route:id[#NAME]` Pomodoro-task captures now also report
  `pomodoro_link_destination`, `pomodoro_name`, and `creates_pomodoro`. Today
  `capture/pomodoro_insert.rs` discards the selected entry and `plan.rs` sets them to
  `None`.
- Human output shows the destination as `→ under GOALS (next up)`,
  `→ into running GOALS (0945-1015)` or `→ new Pomodoro BOB`.

**Create rows in capture-complete.** In the `pomodoro_name` context, `creates_pomodoro`
candidates gain `plan_themes_after` and `plan_themes_cap`. Omit both when the daily note
or the config is unavailable.

**Docs and tests.**

- Update `docs/capture.md`: the JSON contract, a new "Plan budget and strict mode"
  section, and the destination role.
- Update `README.md` and `docs/plan.md` ("Surfaces").
- CLI tests cover:
  - warn versus quiet;
  - dry-run parity;
  - batch refusal in strict mode, and starts never refused;
  - each `role`;
  - create-row fields;
  - invalid config;
  - human output.

## Phase: close-drop — `~<K>` drop outcome for `=x` closes

**Grammar.** The close token becomes `=x[<N>][!<M>][~<K>]`.

- `!<M>` and `~<K>` may each appear at most once, in either order. Lists are
  comma-separated task numbers.
- Update `link_close_after_x` to accept `~` as the first character after the `x`.
- A dangling `~` is an Incomplete editing state, like `!`: it has
  `needs: ["pomodoro_close_task"]` and an `interactive_placeholder` span.
- A number listed in two lists, or an out-of-range number, is a precise-range error.
  Extend `close_selection.rs` and the messages in `markers.rs`.
- The same lexer serves `bob capture`, `capture-parse`, completion and rewrite,
  including the `@r:id=x…`, `^r:id=x…` and `<text> @r:id=x…` forms.

**Semantics.**

| Outcome                                  | 🍅 in the closed entry | Carried to the next placeholder | Task effect                                                                 |
| ---------------------------------------- | ---------------------- | ------------------------------- | --------------------------------------------------------------------------- |
| in progress (`N`, or the ledger default) | yes                    | yes                             | started `[/]`                                                               |
| deferred                                 | no (removed)           | yes                             | none                                                                        |
| complete (`!M`)                          | embedded               | no                              | closed                                                                      |
| **dropped (`~K`)**                       | **no (removed)**       | **no**                          | **none** (the next hooks run demotes Next → Ready; `#now` keeps it in view) |

- An omitted `<N>` keeps the ledger default for unlisted links. A typed `<N>` defers the
  unlisted ones, exactly as today.
- Dropped links count as "not carried" when `plan_ledger_close` decides whether to
  create a placeholder.
- Implement this in `capture_pomodoro_close/selection.rs` (a new outcome), `ledger.rs`
  (remove, don't carry) and `linked_tasks.rs` (no start, no Work Log).

**Spans and JSON.**

- New span kind: `pomodoro_close_drop`.
- `PomodoroCloseSpec` gains `drop: [u32]`, omitted when empty.
- `pomodoro_close` gains `drop` (omitted when empty).
- `task_links[].outcome` gains `dropped`, and `tasks[].role` gains `dropped`.
- Every `tasks[]` row gains `now: true` when the linked task's line has `#now` (use
  plan-core's `has_now_tag`), omitted when false.
- Human output lists dropped rows (e.g. `dropped 4 [[sase#^x]] · stays in NOW`) and a
  `Dropped 4, 5` summary.

**Docs.** Update:

- the grammar table and examples in `docs/capture.md` (`=x~4,5`, `=x1!2~3`, `=x0~2`);
- the outcome table, carry rules, JSON notes, and selection diagnostics;
- the `=x` help in `capture/cli.rs`;
- `README.md`.

**Tests.** Lexer tests (orders, dangling `~`, conflicts, range errors), planner tests,
editor-span tests, `capture-parse` JSON tests, and CLI close tests for all three close
forms, including human output.

## Phase: now-token — first-class `#now` in capture

**`#now` after the route marker.**

- `reject_legacy_bullet_markers` (`capture_language/markers.rs`) and its editor twin
  (`legacy_bullet_marker_diagnostic`) must treat a trailing `#now` as a tag, not a
  legacy marker. Every other trailing `#tag` keeps today's error.
- Resolve the route in front of a trailing `#now`, then move `#now` to the end of the
  body. The written line is `- [ ] #task <body> #now [created::…] ^id`: the tag goes
  before the fields.
- `#now` before the route keeps working.
- An item with no body text, such as a solo link, a toggle, or a whole-item operator,
  followed by `#now` is a usage error: "`#now` tags new task text; tag an existing task
  with Alt+N in Obsidian". Update the `placement.rs` and `editor_spans.rs` pins
  deliberately.

**Spans and completion.**

- Every exact `#now` token in the body gets a `now_tag` span.
- In the editor parse, a trailing `#n` or `#no` is an Incomplete state with
  `needs: ["now_tag"]` and a `now_tag` span. Execution is unchanged and still errors.
- `capture-complete` gets a new context, `now_tag`, with the single candidate
  `{replacement: "#now", label: "#now", text: "This week's bet", kind: "tag"}`.

**The `^` picker** (`capture_active_tasks.rs`).

- Also list open **Ready** tasks that have a block ID and `has_now_tag`, ordered after
  unqueued Next tasks.
- Add `now: true` to every active-task candidate whose text has `#now`, omitted when
  false. This applies to the `ActiveTask` struct and the capture-complete candidate
  JSON.

**Docs and tests.**

- Update `docs/capture.md`: the grammar, spans, the completion context, the `^` picker,
  and a "`#now`" section linking `docs/plan.md`.
- Tests for each rule, including `Fix it @sase^fix-it #now`, `Fix it @sase:fix-it #now`,
  `Fix it #now @sase`, and the solo-link error.

## Phase: ledger-plan-view — Bob Ledger Tools plan view, `bob-plan` block, and public API

The work is in bob-plugins `plugins/bob-ledger-tools/main.js`, plus a new `styles.css`.

**Pure helpers**, exported through `module.exports.helpers`:

- `computePlanBudget(content, caps)`: mirrors `docs/plan.md` exactly, with camelCase
  fields.
- `parsePlanCaps(yamlObject)`.
- `hasNowTag(text)`.
- `nowBudgetFromTasks(tasks, today, caps)`: the dash's predicate on Tasks-plugin task
  objects (`isBlocked`, tags, scheduled date, and path exclusions).

**Tests.** Add `scripts/test-ledger-tools-plan-budget.cjs` and register it in
`package.json`. Encode every bob-cli conformance example verbatim as a test vector,
citing `docs/plan.md` in a comment.

**Config.**

- Read `plan:` from `~/.config/bob/config.yml` the way bob-navigation-hotkeys does (it
  honors `XDG_CONFIG_HOME` and uses `parseYaml`).
- The plugin is not desktop-only, so guard `fs` with `Platform.isDesktopApp` and a
  try/require. Mobile and read errors fall back to the defaults.
- Re-read on each render; the reads are cheap.

**` ```bob-plan ` code block.**

- Target note: the note the block is in when that note is a canonical daily
  (`YYYY/YYYYMMDD.md`), otherwise today's daily, via the Daily Notes core-plugin format.
- It renders one row:
  - a PLAN chip (`PLAN 3/3 · 7/10`, whose tooltip lists each theme with its link count);
  - a NOW chip (`NOW 12/15`) that opens `dash#NOW Tasks`;
  - `★ GOALS · DECKS · BOB`;
  - one muted line per lint.
- The chips use the dash's chip language: small-caps label, tabular-nums value, a 2px
  accent start border, and a 14% accent background.
- It is red when over the cap and has an accessible `aria-label`.
- It re-renders live on `metadataCache` "changed" for the target note, and when the
  Tasks plugin's cache updates. If the Tasks plugin exposes no usable event, it falls
  back to a short interval. Debounce the re-renders and clean up on unload.
- It shows `–` values, never an error, when there is no daily note, no Pomodoros
  section, or no Tasks plugin.

**Public API.** Set
`this.api = { version: 1, caps(), planBudget({path?, content?}), nowBudget() }`. The
`content` option lets callers budget post-write text. Document it in the README as the
supported interface for the dash and the other plugins. Plugins still never import one
another's `main.js`.

**Finish.** Bump the version, update the README row, and deploy with `bob plugins sync`.

## Phase: link-picker — Ctrl+Shift+P edits the task behind a Task Link

This is bob-plugins `plugins/bob-navigation-hotkeys/main.js`.

**Detection.**

- A **dedicated Task Link bullet** is a bullet whose body, after optional 🍅 markers,
  optional `~~…~~`, and an optional trailing `#`, is exactly one block link `[[T#^id]]`
  or `![[T#^id]]`, with an optional alias.
- Resolve it with `resolveLinkTargetFile` plus `findTaskLineByBlockId`, or
  `getFirstLinkpathDest`. The target must be a unique `^id` on an open `#task` line.
- Read the target's live editor buffer when it is open
  (`getOpenMarkdownBufferContents`).
- A missing, duplicated, non-task or closed target shows a Notice and changes nothing.
- This applies anywhere, including dependency transclusions, not only in the ledger.

**Bare Ctrl+Shift+P.**

- Before the `isBulletLine` path in `openBulletPropertyPicker`, detect a dedicated Task
  Link and open the picker in **link mode** against the resolved task.
- The subtitle shows `↗ <note> · <task text>`.
- Hide the `dependsOn` row in link mode.

**Counted `N<Ctrl+Shift+P>`.**

- On a link line, target the current link plus the next N dedicated Task Link siblings:
  same depth, same parent (for ledger links, the same Pomodoro entry), in document
  order, skipping non-link siblings.
- Clamp at the parent's end, with the subtitle
  `N links of M requested · end of Pomodoro`.
- On a `#task` line, the existing behavior is unchanged.

**Writes.**

- Group targets by note and plan each note with the pure planners:
  `planCountedBulletPropertyBatch`, the priority roll per target, and
  `planScheduleLogEntry`.
- Write through the open editor (`getOpenMarkdownEditorForPath` +
  `applyEditorContentTransaction`), otherwise `vault.process` with a preimage guard, as
  `writeTaskMoveChange` does.
- **Future pruning.** For each task given a strictly future `scheduled` date, run the
  existing `shouldPrune` behavior: mark the target Blocked, and let
  `planDeferredPomodoroLinkCleanup` remove its links from today's open Pomodoros. The
  cleanup targets the resolved task's note path.
- Write order is targets, then the daily note. Refuse the whole operation if any
  preimage changed.
- **Notices** reuse the priority notice card. Examples:
  - `P2 → 3 tasks via Task Links · deferred 8–30 days · removed 3 links from today`;
  - `scheduled → 2026-10-02 · 1 task via Task Link`.

**Tests and finish.** Add harness tests to `scripts/test-navigation-hotkeys.cjs` for:

- bare and counted link mode;
- cross-note writes, both open-buffer and vault;
- prune;
- refusals;
- unchanged task-line behavior.

Then bump the version, update the README, and deploy.

## Phase: now-toggle — toggle #now from task lines and Task Links

This is also bob-navigation-hotkeys.

**Command.**

- Register `toggle-now-tag`, "Toggle #now (this week's bet)", with default hotkey
  `Alt+N`. Confirm the chord is free in `~/bob/.obsidian/hotkeys.json` and in Obsidian's
  defaults.
- In Vim normal mode, intercept the counted `N<Alt+N>` with the capture-phase keydown
  pattern used for `Ctrl+Shift+M`.
- Targets:
  - on a `#task` line, the current task plus the next N real tasks
    (`discoverCountedObsidianTaskTargets`);
  - on a dedicated Task Link, the resolved task plus the next N sibling links, reusing
    link-picker's discovery.

**Picker row.** Add a pinned `#now` row to the Ctrl+Shift+P property stage, in both task
mode and link mode.

- Its detail reads `this week · add` or `this week · remove`.
- Choosing it toggles immediately; there is no value stage.

**Toggle semantics.**

- If any target lacks `#now`, add it to every target. Otherwise remove it from all of
  them.
- **Insert:** ` #now` at the end of the description, immediately before the first
  trailing inline field `[k:: v]`, else before the trailing ` ^id`, else at the line's
  end.
- **Remove:** every whole-token `#now`, then collapse the doubled spaces.
- Never touch `#task`, fields, or the block ID. Writes follow link-picker's cross-note
  rules.

**Notice.** For example, `#now added · 3 tasks · NOW 13/15`, with 🔴 and "prune at the
weekly review" when over the cap.

- Compute the count from `app.plugins.plugins["bob-ledger-tools"]?.api?.nowBudget()`
  before the write, adjusted by the number of tasks that changed, because the Tasks
  cache lags behind the write.
- Omit the NOW suffix when the API is unavailable.

**Tests and finish.** Test insert and remove placement (fields, `^id`, `#hide`, already
tagged), the counted batch, link mode, the picker row, and the Notice with and without
the API. Bump the version, update the README, and deploy.

## Phase: link-notice-budget — plan budget in the Ctrl+Shift+Enter Notice

This is bob-plugins `plugins/block-id-prompt/main.js`.

- Append ` · plan T/Tc · L/Lc` to the Notices built by `reportPomodoroLinkOutcome`,
  `reportPomodoroUnlinkOutcome` and `reportTaskLinkOpenOutcome`.
- Add ` 🔴` when the plan is over the cap.
- Compute the budget with `api.planBudget({content: <post-write daily content>})` from
  bob-ledger-tools. Omit the suffix when the API is missing, or when the daily note
  wasn't part of the operation.
- Warn only, never refuse. Ctrl+Shift+Enter never creates a named theme, so strict mode
  does not apply.
- Test with the `createTaskLinkHarness` stubbing the API: present, absent, and over the
  cap. Bump the version, update the README, and deploy.

## Phase: mac-budget — plan budget meter, destination row, and create-row cap badge

This is bob-mac-capture.

**Models** (`Sources/CaptureCore/CaptureModels.swift`). Decode each of these with
`decodeIfPresent`, so an older Bob yields `nil`:

- `plan_budget` on `CaptureCommandSuccess`: a new `CapturePlanBudget`, whose meters have
  `count`, `cap`, `over` and an optional `before`, plus `added_themes` and `warnings`
  with `code` and `message`;
- the optional `code` on `CaptureCommandFailure`;
- `role` on the Pomodoro link endpoint;
- `plan_themes_after` and `plan_themes_cap` on completion candidates.

**Preview.**

- **Destination row.** Add a row above the preview items, e.g. `→ GOALS · next up`,
  `→ running GOALS 0945–1015` or `→ new Pomodoro BOB`, with a `timer` symbol.
- **Meter row.** Add two capsules, `Themes 3/3` and `Links 8/10`.
  - Green within the cap and red over it.
  - Show a `+1 BOB` delta chip whenever `after > before`.
  - Show the budget warnings as orange captions.
- Include both rows in the accessibility summary. Put the pure formatting in a new
  `CaptureCore` presentation type with its own tests.

**Completion.** A `pomodoro_name` create row with `plan_themes_after > plan_themes_cap`
shows a red `4/3` badge.

**Strict refusal.** The existing error callout shows the message. When
`code == "plan_theme_cap_exceeded"`, add a short hint line.

**Fixtures and docs.**

- Add real-bob fixtures (`plan-budget-over.json`, `plan-budget-strict-refusal.json`, and
  a destination-role fixture) plus the `fake-bob` branches.
- Add decoding tests with and without the fields.
- Update the README's Requirements and runtime-contract sections.
- Green macOS CI is required.

## Phase: mac-close-now — drop outcome, `#now` token, and NOW badges

This is bob-mac-capture.

**Spans.**

- `pomodoro_close_drop` maps to a new `.pomodoroCloseDrop` semantic category, in a muted
  gray that is distinct from neutral text.
- `now_tag` maps to a new `.nowTag` category in mint. Purple already means section and
  Pomodoro names in this palette, and teal means clipboard.
- Add both to the palette, which is exhaustive, and to `completionSpanKinds`.
- Add the `now_tag` need to the needs that request completion.

**Completion.** Add a `now_tag` context row: symbol `star.circle`, primary `#now`,
secondary "This week's bet", and a `NOW` badge.

**Close card** (`CapturePomodoroClosePresentation.swift`).

- Decode `PomodoroCloseSpec.drop` (default `[]`).
- Map the `dropped` outcome and role to a struck, dimmed row with its own glyph.
- The summary gains `Dropped 4, 5`, and the teaching hint mentions `~N`.
- Accessibility reads "drops from today".
- Rows with `now == true` show a `NOW` badge. A dropped `#now` row adds the caption
  "stays in NOW".

**Active-task picker.** Candidates with `now == true` show a `NOW` badge. Ready `#now`
tasks appear with their Ready status.

**Fixtures, docs and CI.** Add real-bob fixtures for the close, parse and complete
payloads, plus the `fake-bob` branches. Add presentation and model tests, and update the
README. Green macOS CI is required.

## Phase: rollout — PLAN chip, daily template block, config knobs, install, and end-to-end check

1. **Dash.**
   - Switch the NOW chip to `app.plugins.plugins["bob-ledger-tools"]?.api?.nowBudget()`,
     keeping the inline count and the cap 15 as the fallback.
   - Add a PLAN chip after NOW:
     - its value comes from `api.planBudget()` and reads `3/3 · 7/10`;
     - its accent is cyan, switching to `task-count-over` when over the cap;
     - its target is today's daily note;
     - it shows `–` without the API.
2. **Template.**
   - Insert this block between the `^gtd` task line and the `## Pomodoros` heading of
     `_templates/daily.md`:

     ````markdown
     ```bob-plan

     ```
     ````

   - Also insert it into today's daily note, if that note exists and lacks the block,
     touching nothing else.

3. **Config.**
   - Add the documented, commented `plan:` block (from the Design section, with the
     defaults) to `home/dot_config/bob/config.yml` in the chezmoi source, opened with
     `sase repo open chezmoi`.
   - Deploy it with chezmoi so `~/.config/bob/config.yml` matches.
   - If chezmoi can't be opened, skip this step and record a `PROPOSED FOLLOW-UP:`. The
     defaults already apply.
4. **Install and deploy.**
   - Run `cargo install --path . --locked --force` from an up-to-date bob-cli master
     checkout, so apollo's hooks service and tmux use the new `bob`.
   - Run `bob plugins sync -n -r "<opened bob-plugins path>"` for all plugins. Pull that
     checkout to origin master first.
5. **End-to-end check on apollo.** Run each of these and paste the key output into the
   phase notes:
   - `bob plan` and `bob plan -f json`;
   - `bob tmux-pomodoro`;
   - `bob task-status-hooks --dry-run -f json | jq .plan_budget`;
   - `bob capture --dry-run -f json -- 'Try it @bob:rollout-probe'`, checking
     `plan_budget` and `pomodoro_link_destination.role`;
   - `bob capture-parse -f json -- 'Try it @bob^probe #now'`;
   - `bob capture-parse -f json -- '=x1~2'`.

   Then:
   - Run `bob vault-sync` and confirm it with `bob vault-sync status --json`.
   - Update `docs/plan.md`'s "Surfaces" table so it is final.

6. **Bryan's checklist.** Put it in the final response. These are Bryan's steps and are
   not automated:
   - Add `#now` to at most 15 tasks you want this week. Type it before the fields, or
     use Alt+N or the Ctrl+Shift+P `#now` row.
   - From 2026-09-30, carry at most 3 open entries by hand and leave the rest.
   - Put GTD plus at most 3 themes in the daily note, highlight first.
   - On the MacBook and athena: reinstall `bob`, rebuild and install Bob Mac Capture,
     and run `bob plugins sync`.
   - Consider `plan.strict: true` if the plan is red on most days after a week.

## Deliberately not doing (from the report, plus these design calls)

- **Report exclusions:**
  - `[roadmap::]` or `[horizon::]` fields;
  - task-level Bases or a `roadmap.base`;
  - a hand-kept `roadmap.md`;
  - a `highlight::` field;
  - cron that rewrites plans or yesterday's note;
  - `#now` as a Next source;
  - breaking `@route:id`;
  - renaming `## Pomodoros`.
- **Deferred to Phase 2 by design:**
  - `stale_link`. It needs its own multi-day lookback semantics. Under the closed-day
    rule, today's links are re-chosen every morning, and the weekly `#now` review
    already covers the 7-day staleness question.
  - `bob pomodoro stats`, move-with-link-repair (K4), a `#next` tag, pull-a-theme (C5),
    and a "close day" command.
- **Rejected designs:**
  - A DataviewJS view file (`_meta/views/plan_budget.js`). The vault's `.gitignore`
    ignores `.js` outside `.obsidian/`, so the view would not sync to the other
    machines, and it would be a third copy of the rules. The `bob-plan` block and the
    ledger-tools API replace it.
  - Budget warnings duplicated into capture's `warnings: [String]`. They live only in
    `plan_budget.warnings`.
